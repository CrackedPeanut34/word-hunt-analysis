# Word Hunt Project — Process Log

What was done, start to finish, across this whole project.

## 1. Data extraction (screenshots → CSV)

**Source:** 116 iPhone screenshots of Word Hunt (a GamePigeon game) post-game results screens, named IMG_5712.PNG through IMG_5827.PNG, in `/Users/levi/Desktop/Word Hunt/`.

**Screen layout:** each screenshot shows a "YOU WON!" / "YOU LOST!" (or occasionally "DRAW!") banner, then two side-by-side columns — "You" (left) and the opponent (right) — each with a WORDS count, a SCORE, and a list of found words paired with point values (100/400/800/1400/1800/2200), sorted descending, truncated after ~15 entries with "(N more)".

**Extraction rules established with the user up front:**
- Only capture the *numbers* next to words (point values), never read/count the actual words.
- 400 and below ("400" and "100" tiers) don't count toward the ≥800/≥1400/≥2200 tallies.
- Tier counts are cumulative (≥800 includes 1400s/2200s/1800s; ≥1400 includes 2200s/1800s).
- Capture both my stats and the opponent's stats per game (user chose "mine + opponent" over "mine only").
- Won flag = binary (1 if "YOU WON!", 0 otherwise).

**Process:**
1. Manually read and hand-extracted 5 sample screenshots (IMG_5712–5716) first and showed the user the results table to validate methodology before scaling up.
2. User confirmed the approach and clarified: cumulative tier thresholds, include opponent stats.
3. Split the remaining 111 screenshots into 8 batches of ~14 and ran them in parallel via background subagents, each independently reading its batch and returning CSV-formatted rows.
4. Batches self-flagged anomalies: truncated lists where the visible ≥800 words never dropped to the 400 tier before cutoff (meaning the ≥800 count could be a lower bound, not exact), an unexpected "DRAW!" banner, and non-standard point tiers (1800, in addition to the expected 100/400/800/1400/2200).
5. Manually re-verified all 7 flagged images directly (IMG_5717, IMG_5727, IMG_5758, IMG_5797, IMG_5807, IMG_5811, IMG_5827) by reading the actual screenshots again to confirm/correct the flagged values rather than trusting the batch output blindly. Confirmed the 1800-point tier is real (visually verified), confirmed the draw game, and confirmed which lower-bound flags were genuine (3 rows: IMG_5717, IMG_5727, IMG_5758 have opponent/my ≥800 counts that are undercounts since the full word list wasn't visible on screen).
6. Assembled everything into `word_hunt_stats.csv` (116 rows) and sanity-checked it with pandas: no duplicate files, no nulls, tier counts internally consistent (2200 ≤ 1400 ≤ 800 ≤ total words), 116 rows matching the 116 source images.

## 2. Base CSV schema (`word_hunt_stats.csv`)

`file, my_won, my_score, my_words, my_words_over_800, my_words_over_1400, my_words_over_2200, opp_score, opp_words, opp_words_over_800, opp_words_over_1400, opp_words_over_2200, notes`

`notes` is empty except for the 6 flagged rows described above (3 lower-bound truncation warnings, 1 draw-game note, 2 notes about the 1800-point tier, 1 note about both 1800 and 2200 tiers in the same game).

## 3. Enrichment (`word_hunt_data_enriched.csv`)

Built on top of the base CSV, adding (in the order they were requested across the conversation):
- `total_score`, `total_words` — mine + opponent's, combined per game
- `score_diff`, `words_diff`, `words_over_800_diff`, `words_over_1400_diff`, `words_over_2200_diff` — mine minus opponent's, per game
- `my_avg_word_value`, `opp_avg_word_value`, `total_avg_word_value` — score ÷ words (points per word found)
- `avg_word_value_diff` — mine minus opponent's average word value
- `my_hv_rate`, `opp_hv_rate`, `hv_rate_diff` — share of words worth ≥800, as a fraction of total words found (rate, not raw count)
- `opp_tier` — opponent bucketed into weak/mid/strong terciles by their avg word value
- `game_idx` — 1–116, chronological order by filename, used for streak/trend analysis

## 4. Summary tables produced

- `word_hunt_summary_overall.csv` — single-row: win rate, record, and averages for every stat + differential
- `word_hunt_summary_by_result.csv` — same averages, split by won vs. not-won
- `word_hunt_summary_by_total_score_bucket.csv` — same averages, split into 10 buckets of combined total_score (4k-wide bands, chosen via Freedman-Diaconis rule)
- `word_hunt_regression_words_vs_score.csv` — OLS linear regression of score on words, fit separately for mine / opponent's / total
- `word_hunt_winrate_by_word_value_sign.csv` — win rate split by whether avg_word_value_diff is positive or negative, plus word totals/diffs for each group
- `word_hunt_winrate_by_word_value_bucket.csv` — same, bucketed into 50-point-wide bands
- `word_hunt_insight_correlation_leaderboard.csv` — correlation of every numeric stat/diff column against winning, ranked
- `word_hunt_insight_opponent_tiers.csv` — win rate against weak/mid/strong opponent tiers

## 5. Win-prediction modeling

**Feature set (both models):** my_score, my_words, my_words_over_800, my_words_over_1400, my_avg_word_value — restricted to "stats I can see" (my side only, not opponent's). `my_words_over_2200` was dropped (constant/zero variance across all 116 games — no coefficient can be estimated from it).

**Split:** 65 train / 51 test games, randomly shuffled with a fixed seed (42) for reproducibility.

1. **Linear regression** (`word_hunt_win_regression_*.csv`) — fit OLS directly against the 0/1 win flag, thresholded predictions at 0.5. Flagged upfront that this isn't the theoretically correct tool for a binary target (predictions can fall outside [0,1]), but ran it since that's what was asked. Test accuracy 72.55%.
2. **Logistic regression** (`word_hunt_win_logistic_*.csv`) — implemented by hand via Newton-Raphson/IRLS in numpy (no sklearn/statsmodels available in this environment), with features standardized first for numerical stability and a small ridge penalty (λ=1e-4) for stability given multicollinearity between score/words/avg_word_value. First attempt had a sign error in the Newton step that caused coefficients to diverge to ~1e18 — caught it, fixed the update rule, refit cleanly (converged in 6 iterations). Test accuracy 74.51%, precision 0.667, recall 0.632, McFadden pseudo-R² 0.178 (train) / 0.037 (test).
3. Discussed sample-size adequacy: 116 games gives an events-per-predictor ratio of ~11.5 (borderline vs. the standard 10–20 guideline), and the 74.5% test accuracy has a 95% confidence interval of roughly ±12 points given only 51 test games. Estimated ~200 total games would comfortably clear standard guidelines, and predicted more data would likely yield a modest +2–5 point accuracy gain at best (train ≈ test accuracy already, meaning the simple model isn't obviously data-starved) — the bigger ceiling is that the feature set has no information about the opponent, which fundamentally caps how well "my stats alone" can predict a head-to-head outcome.

## 6. Charts (published as Claude artifacts, self-contained HTML/SVG, dataviz-skill-compliant)

1. **Total Score Distribution** — histogram of combined total_score across all 116 games, 10 bins, mean line overlaid. `https://claude.ai/code/artifact/e45d3a7c-796f-4278-b78e-a46e662621db`
2. **Words vs. Score scatter** — three series (mine/opponent's/total) with OLS regression lines overlaid, toggleable legend, hover tooltips per point. `https://claude.ai/code/artifact/237b49e0-d28b-4fd4-9dc8-c529747fed4e`
3. **Predicted Win Probability jitter plot** — logistic regression's predicted probability per game, jittered by actual outcome (won/not-won), colored by train/test split, misclassified points ringed. `https://claude.ai/code/artifact/9357e342-e5e0-4fea-b2f1-e8c87bfe49d6`
4. **Word Value Differential Distributions** (added after dataset 2 arrived) — two-panel histogram of `avg_word_value_diff`, dataset 1 and dataset 2 on identical bins/axis for direct comparison, with a zero reference line (the win/lose pivot for this stat) and each panel's mean marked. This turned out to be the clearest single visualization of the project's central finding — see `word_hunt_insights.md` and `word_hunt_insights2.md`. `https://claude.ai/code/artifact/6034b8ee-69da-403b-9e31-eeee4251e05a`

## 7. Exploratory insight pass

After the core analysis, ran a broader sweep for additional patterns: a correlation leaderboard against every stat, high-value-word *rate* (vs. raw count), win/loss streak lengths, a trend check over the game sequence (session order), opponent-strength segmentation, score/word-value volatility (coefficient of variation), and a check for a volume/quality tradeoff in word-finding. See `word_hunt_insights.md` for everything that came out of this.

## Full file list (all in `/Users/levi/Desktop/Word Hunt/`)

**Data:**
- `word_hunt_stats.csv` — raw extracted per-game data (source of truth for the screenshots)
- `word_hunt_data_enriched.csv` — full working dataset, all derived columns

**Summaries:**
- `word_hunt_summary_overall.csv`
- `word_hunt_summary_by_result.csv`
- `word_hunt_summary_by_total_score_bucket.csv`
- `word_hunt_regression_words_vs_score.csv`
- `word_hunt_winrate_by_word_value_sign.csv`
- `word_hunt_winrate_by_word_value_bucket.csv`
- `word_hunt_insight_correlation_leaderboard.csv`
- `word_hunt_insight_opponent_tiers.csv`

**Modeling:**
- `word_hunt_win_regression_coefficients.csv`, `_metrics.csv`, `_test_predictions.csv` (linear)
- `word_hunt_win_logistic_coefficients.csv`, `_metrics.csv`, `_test_predictions.csv`, `_all_predictions.csv` (logistic)

**Docs:**
- `word_hunt_summary.md` — narrative write-up of the core stats/splits/regression (written progressively as each was requested)
- `word_hunt_process_log.md` — this file
- `word_hunt_insights.md` — every insight/finding from the whole project, standalone

**Charts:** the 4 published artifact links above (not local files — hosted, private by default; the 4th was added after dataset 2 arrived and covers both datasets)
