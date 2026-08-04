# Word Hunt Project — Process Log (Dataset 2, vs. Opponent 2)

Everything done to bring in the second batch of raw data and run the full pipeline from dataset 1 against it.

**2026-08-04 correction note:** a `words_over_1800` column was added retroactively to every CSV referenced below after auditing all games with any ≥1400 word. Full audit method and impact in `word_hunt_process_log_combined.md` § 0 and `word_hunt_insights_combined.md`. Nothing else here changed — the cumulative ≥800/≥1400/≥2200 counts were already correct.

## 1. Data extraction (screenshots → CSV)

**Source:** 51 new screenshots, IMG_5828.PNG through IMG_5878.PNG, same folder, same game (Word Hunt), same screen layout as dataset 1 — but this is a different opponent, labeled **"opponent 2" / `opp2`** throughout instead of `opp`.

**Process (identical methodology to dataset 1):**
1. Split the 51 screenshots into 4 parallel background-agent batches (~13 each) with the same extraction rules: only capture point values next to words (never read the words themselves), 400-and-below don't count toward tiers, tiers are cumulative, capture both my stats and opponent 2's stats, won flag = 1 only for an exact "YOU WON!" banner.
2. One image flagged as an edge case: **IMG_5851.PNG** — my ("You") column's visible 15-word list was entirely at the 800-point tier with 38 more words hidden off-screen, meaning `my_words_over_800=15` for that row is a lower bound, not a confirmed exact count (same class of issue as IMG_5717/5727/5758 in dataset 1). Verified this directly by re-reading the screenshot — confirmed the flag is accurate (own-column note added).
3. Two other images had the non-standard 1800-point tier (IMG_5838, IMG_5863) — both had their visible list drop to a lower tier before truncation, so their counts are complete despite the unusual point value.
4. No draws in this batch — every banner was a clean "YOU WON!" or "YOU LOST!"
5. Assembled into `word_hunt_stats2.csv` (51 rows) and sanity-checked: no duplicate files, no nulls, tier counts internally consistent, 51/51 rows present.

## 2. Enrichment (`word_hunt_data_enriched2.csv`)

Same derived columns as dataset 1, computed against `opp2_*` instead of `opp_*`:
`score_diff, words_diff, words_over_800_diff, words_over_1400_diff, words_over_2200_diff, total_score, total_words, my_avg_word_value, opp2_avg_word_value, total_avg_word_value, avg_word_value_diff, my_hv_rate, opp2_hv_rate, hv_rate_diff, opp2_tier, game_idx`

## 3. Summary tables

Same three tables as dataset 1, with `_2` appended to filenames and `opp2_` column naming:
- `word_hunt_summary_overall2.csv`
- `word_hunt_summary_by_result2.csv`
- `word_hunt_summary_by_total_score_bucket2.csv` — **used the same bucket edges as dataset 1** (4k-wide bands, 4k–44k) rather than re-deriving new ones, specifically so the two datasets' bucket rows line up for direct comparison. Dataset 2's total_score range (5,700–40,000) fits inside those same edges with no games in the 40k–44k band this time.

## 4. Words → score regression

`word_hunt_regression_words_vs_score2.csv` — same OLS fit (mine / opp2 / total), same method (scipy linregress).

## 5. Win-prediction modeling

**Important adjustment:** dataset 1 used a 65-train/51-test split out of 116 games. Dataset 2 only has 51 games total, so that exact split is impossible. Used the **same train/test ratio** as dataset 1 (65/116 ≈ 56.03% train) applied to 51 games: **29 train / 22 test**, same fixed seed (42) for the shuffle.

Same feature set (my_score, my_words, my_words_over_800, my_words_over_1400, my_avg_word_value; my_words_over_2200 dropped again — still constant/zero across all 51 games) and same two models:
1. **Linear regression** (thresholded at 0.5): `word_hunt_win_regression_coefficients2.csv`, `_metrics2.csv`, `_test_predictions2.csv`
2. **Logistic regression** (hand-rolled Newton-Raphson/IRLS, standardized features, small ridge penalty): `word_hunt_win_logistic_coefficients2.csv`, `_metrics2.csv`, `_test_predictions2.csv`, `_all_predictions2.csv`

This run surfaced exactly the problem flagged as a risk back in dataset 1's modeling discussion: with only 29 training games (≈19 wins) against 4 effectively-independent predictors, the events-per-predictor ratio (≈4.75) is well under even the loosest 10-per-predictor guideline. The result: **severe overfitting** — logistic regression hit 86.2% train accuracy but only 59.1% test accuracy, and its coefficients blew up to implausible sizes (e.g. an odds ratio of ~712 per std-dev of my_score). See `word_hunt_insights2.md` for the full breakdown — this is treated as a real finding, not just a caveat, since it's a direct, concrete demonstration of the earlier "you need more data" warning.

## 6. Charts (published as Claude artifacts)

Same three chart types, rebuilt with dataset 2's data (same bin edges/axes conventions as dataset 1 where possible, for visual comparability):
1. **Total Score Distribution 2** — `https://claude.ai/code/artifact/5e994013-e0bb-476b-a58b-9b47ae3c45df`
2. **Words vs. Score 2** (mine / opponent 2 / total, with regression lines) — `https://claude.ai/code/artifact/cefbe88d-79cf-484a-becf-18beabb3db75`
3. **Predicted Win Probability 2** (logistic jitter plot, 29 train / 22 test) — `https://claude.ai/code/artifact/0442437c-2984-4d91-9700-c8ecdad0b5dc`

A 4th chart was added after this dataset was built, covering both datasets at once rather than a `2`-suffixed dataset-2-only version: **Word Value Differential Distributions** — two-panel histogram of `avg_word_value_diff`, dataset 1 vs. dataset 2 on identical bins, the clearest single visual of why the two datasets differ (dataset 1 skewed negative, dataset 2 centered near zero). `https://claude.ai/code/artifact/6034b8ee-69da-403b-9e31-eeee4251e05a`

## 7. Word value differential splits

`word_hunt_winrate_by_word_value_sign2.csv` and `word_hunt_winrate_by_word_value_bucket2.csv` — same sign-split and bucketed-split analysis as dataset 1, now against opponent 2, including word totals and the word-count differential alongside win rate per group.

## 8. Exploratory insight pass

Same seven angles as the original insight sweep: correlation leaderboard (`word_hunt_insight_correlation_leaderboard2.csv`), high-value word rate, win/loss streaks, trend-over-session check, opponent-strength tiers (`word_hunt_insight_opponent_tiers2.csv`), score/word-value volatility, and the volume/quality tradeoff check.

## Full file list added this round (all in `/Users/levi/Desktop/Word Hunt/`)

**Data:** `word_hunt_stats2.csv`, `word_hunt_data_enriched2.csv`

**Summaries:** `word_hunt_summary_overall2.csv`, `word_hunt_summary_by_result2.csv`, `word_hunt_summary_by_total_score_bucket2.csv`, `word_hunt_regression_words_vs_score2.csv`, `word_hunt_winrate_by_word_value_sign2.csv`, `word_hunt_winrate_by_word_value_bucket2.csv`, `word_hunt_insight_correlation_leaderboard2.csv`, `word_hunt_insight_opponent_tiers2.csv`

**Modeling:** `word_hunt_win_regression_coefficients2.csv`, `_metrics2.csv`, `_test_predictions2.csv` (linear); `word_hunt_win_logistic_coefficients2.csv`, `_metrics2.csv`, `_test_predictions2.csv`, `_all_predictions2.csv` (logistic)

**Docs:** `word_hunt_process_log2.md` (this file), `word_hunt_insights2.md`

**Charts:** the 3 published artifact links above, plus the shared cross-dataset histogram noted in section 6
