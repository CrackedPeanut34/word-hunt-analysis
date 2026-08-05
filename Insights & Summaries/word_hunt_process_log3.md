# Word Hunt Project — Process Log (Dataset 3, vs. Opponent 1, post-strategy-change)

Everything done to bring in the third batch of raw data and run the full pipeline against it.

## 1. Data extraction (screenshots → CSV)

**Source:** 63 new screenshots, IMG_5888.PNG through IMG_5950.PNG, same folder/game/screen layout as datasets 1 and 2 — against **Opponent 1 again**, but played after a deliberate strategy change (prioritizing finding longer/higher-value words over just maximizing total word count). Unlike datasets 1 and 2, the **≥1800 tier was tracked natively from the very first extraction pass** — no retroactive audit needed this time (see `word_hunt_process_log_combined.md` §0 for why that audit was needed for the earlier two datasets).

**Process:**
1. Split the 63 screenshots into 4 parallel background-agent batches (16/16/16/15) with the same extraction rules used throughout the project: only capture the numeric point value next to each word (never the word text itself), tiers are cumulative (≥800/≥1400/≥1800/≥2200), capture both my stats and the opponent's stats, `my_won` = 1 only for an exact "YOU WON!" banner.
2. Two images flagged as the "saturated list" edge case, where the opponent's visible ~15-word list never dropped below the 800-point tier before the "(N more)" cutoff, making `opp_words_over_800` a lower bound rather than an exact count (higher tiers stayed exact since the lowest *visible* value already sets a ceiling on anything hidden): **IMG_5906.PNG** (opp_words_over_800 recorded as 15, true count possibly up to 42) and **IMG_5943.PNG** (recorded as 15, true count possibly up to 34).
3. No draws in this batch.
4. Assembled into `word_hunt_stats3.csv` (63 rows). Automated checks: no duplicate files, no nulls outside the notes column, `over_800 ≥ over_1400 ≥ over_1800 ≥ over_2200` holds for both sides on every row, `my_won` agrees with the sign of `my_score − opp_score` on every row (0 mismatches).
5. Manual spot-check: re-read IMG_5888.PNG and IMG_5918.PNG directly against the CSV output, including IMG_5918's 2200-point word (`GRITTIER`, opponent side) — the first 2200-point word to appear in this dataset. Both matched exactly.

```python
import pandas as pd
batches = [pd.read_csv(f"dataset3_batch{i}.csv") for i in range(1, 5)]
df = pd.concat(batches, ignore_index=True)
df['_num'] = df['file'].str.extract(r'(\d+)').astype(int)
df = df.sort_values('_num').drop(columns='_num').reset_index(drop=True)
for side in ['my', 'opp']:
    a, b, c, d = (df[f'{side}_words_over_{t}'] for t in (800, 1400, 1800, 2200))
    assert (a >= b).all() and (b >= c).all() and (c >= d).all()
df.to_csv("word_hunt_stats3.csv", index=False)
```

## 2. Enrichment (`word_hunt_data_enriched3.csv`)

Same derived columns as datasets 1/2, computed against `opp_*` (this is Opponent 1 again, so the original `opp_` prefix is reused rather than a new one):
`score_diff, words_diff, words_over_800_diff, words_over_1400_diff, words_over_1800_diff, words_over_2200_diff, total_score, total_words, my_avg_word_value, opp_avg_word_value, total_avg_word_value, avg_word_value_diff, my_hv_rate, opp_hv_rate, hv_rate_diff, opp_tier, game_idx`.

`opp_tier` (weak/mid/strong) is computed via terciles of `opp_avg_word_value` **within dataset 3 only** (not pooled with dataset 1), labeled `... (d3)` to keep it distinct from dataset 1's own opponent-1 tiers, since board difficulty and the opponent's own game-to-game variance could differ between the two time periods.

```python
e['my_avg_word_value'] = e['my_score'] / e['my_words']
e['opp_avg_word_value'] = e['opp_score'] / e['opp_words']
e['avg_word_value_diff'] = e['my_avg_word_value'] - e['opp_avg_word_value']
e['my_hv_rate'] = e['my_words_over_800'] / e['my_words']
terciles = e['opp_avg_word_value'].quantile([1/3, 2/3]).values
e['opp_tier'] = e['opp_avg_word_value'].apply(
    lambda v: 'weak opponent (d3)' if v <= terciles[0]
    else 'mid opponent (d3)' if v <= terciles[1]
    else 'strong opponent (d3)')
```

## 3. Summary tables

Same structure as datasets 1/2, with `3` appended to filenames: `word_hunt_summary_overall3.csv`, `_by_result3.csv`, `_by_total_score_bucket3.csv` (same 4k-wide bucket edges as datasets 1/2 for direct comparability), `word_hunt_winrate_by_word_value_sign3.csv`, `_bucket3.csv`, `word_hunt_insight_opponent_tiers3.csv`.

## 4. Words → score regression and correlation leaderboard

`word_hunt_regression_words_vs_score3.csv` — same OLS fit (mine / opponent / total) via `scipy.stats.linregress`. `word_hunt_insight_correlation_leaderboard3.csv` — Pearson correlation of the same 18-stat list used in datasets 1/2 against `my_won`, via `scipy.stats.pearsonr`.

## 5. Single-dataset win-prediction model — included ≥1800 from the start, per request

Unlike datasets 1/2's individual (non-pooled) win models — which were fit before the ≥1800 tier existed and were never retrofitted — dataset 3's model includes `my_words_over_1800` as a feature from the outset, since this dataset never had the missing-tier problem. Feature set: `my_score, my_words, my_words_over_800, my_words_over_1400, my_words_over_1800, my_avg_word_value` (`my_words_over_2200` dropped — constant/zero across all 63 games). Same 56%-train ratio used throughout the project (63 games → 35 train / 28 test), same fixed seed (42), same hand-rolled Newton-Raphson/IRLS logistic regression with ridge penalty λ=1e-4, plus the linear-probability comparison model.

**Result:** test accuracy landed at exactly the majority-class baseline (50.0% vs 50.0%) — the single-split model added no predictive value at all on held-out data here, an even starker version of dataset 2's overfitting warning. Not a surprise at n=63 with a 35/28 split; treated as another concrete data point for "single train/test splits are unreliable at this sample size," reinforcing why the pooled cross-validated model remains the one worth trusting.

Files: `word_hunt_win_logistic_coefficients3.csv`, `_metrics3.csv` (logistic); `word_hunt_win_regression_coefficients3.csv`, `_metrics3.csv` (linear).

## 6. Before/after comparison (dataset 1 vs. dataset 3, same opponent)

Computed directly rather than just eyeballing the summary tables: two-sample Welch t-tests (`scipy.stats.ttest_ind`, unequal variance) on `score_diff`, `avg_word_value_diff`, and `my_words_over_1400` between dataset 1's 116 games and dataset 3's 63 games, plus a two-proportion z-test on win rate. Full results and interpretation in `word_hunt_insights3.md` and delivered directly in chat per the user's request — the short version: the *behavior change* (more ≥1400 words per game) is statistically real (p=0.017), but the *downstream* improvements (win rate, score differential, word-value gap) are all directionally positive yet not statistically distinguishable from noise at this sample size (p=0.21–0.53). Also flagged a confound: dataset 3's boards may simply have been somewhat richer on average (total avg word value trend, p=0.088), which could account for part of the shift.

## 7. Combined dataset update

Appended all 63 dataset-3 games to `word_hunt_data_combined.csv` (`Stats/`): `dataset=3`, `opponent_label="Opponent 1"`, `vs_opponent_2=0`, `combined_game_idx` continuing from 168–230, `dataset_game_idx` 1–63, plus all the same derived columns already present for datasets 1/2 (`my_adjusted_points`, `opponent_adjusted_points`, `my_score_share`, `opponent_score_share`). Combined file is now 230 rows across 3 dataset/opponent segments.

## 8. No new charts this round

The before/after comparison and model discussion were delivered directly in chat per the user's explicit request ("use the chat to..."), not as new published artifacts — no new Claude Artifacts were built for dataset 3.

## Full file list added this round (all in `/Users/levi/Desktop/Word Hunt/`)

**Data:** `Stats/word_hunt_stats3.csv`, `Stats/word_hunt_data_enriched3.csv`

**Summaries:** `Stats/word_hunt_summary_overall3.csv`, `_by_result3.csv`, `_by_total_score_bucket3.csv`, `_winrate_by_word_value_sign3.csv`, `_bucket3.csv`, `_insight_opponent_tiers3.csv`

**Regressions:** `Regressions/word_hunt_insight_correlation_leaderboard3.csv`, `Insights & Summaries/word_hunt_regression_words_vs_score3.csv`, `Regressions/word_hunt_win_logistic_coefficients3.csv` / `_metrics3.csv`, `Regressions/word_hunt_win_regression_coefficients3.csv` / `_metrics3.csv`

**Combined:** `Stats/word_hunt_data_combined.csv` (updated in place, now 230 rows)

**Docs:** `word_hunt_process_log3.md` (this file), `word_hunt_insights3.md`
