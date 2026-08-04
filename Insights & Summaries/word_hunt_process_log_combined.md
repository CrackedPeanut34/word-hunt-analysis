# Word Hunt Project — Process Log (Combined Cross-Dataset Work)

Everything done that spans both datasets at once, rather than treating them separately.

## 1. Combined dataset (`word_hunt_data_combined.csv`)

Concatenated `word_hunt_data_enriched.csv` (116 games, Opponent 1) and `word_hunt_data_enriched2.csv` (51 games, Opponent 2) into one 167-row table. Unified the opponent-specific column names (`opp_*` / `opp2_*` → generic `opponent_*`) and added:
- `dataset` (1 or 2) and `opponent_label` ("Opponent 1" / "Opponent 2") — identifies who was played, per the request to have "something signifying who the opponent is"
- `combined_game_idx` (1–167, chronological by filename across both datasets) and `dataset_game_idx` (the original per-dataset index, preserved)
- `vs_opponent_2` — binary (1/0) version of the same information, added specifically as a model feature
- `my_score_share`, `opponent_score_share` — each player's share of that game's total_score (score ÷ total_score)
- `my_adjusted_points`, `opponent_adjusted_points` — score minus that game's own midpoint (`total_score / 2`); a board-difficulty-normalized rating, since a rich/high-scoring board lifts both players' baselines equally and this cancels that out

## 2. Fair three-way player comparison (`word_hunt_summary_by_player.csv`)

**Correction made here:** an earlier version of this comparison mistakenly compared a blended "Me" average (167 games, weighted across two very different opponents) directly against each opponent's isolated stats — a misleading apples-to-oranges comparison that made it look like performance against Opponent 2 was closer than it actually is. Rebuilt as matchup-specific rows: **Me (vs Opponent 1)** using only dataset-1 games, **Opponent 1**, **Me (vs Opponent 2)** using only dataset-2 games, **Opponent 2**, plus a clearly-labeled **Me — Overall (blended)** row kept for reference only, with a note not to use it for head-to-head comparisons. Added `avg_score_share_pct` and `avg_adjusted_points` columns to every row for a normalized comparison.

## 3. Points-origin decomposition (`word_hunt_points_origin_decomposition.csv`, `_per_game.csv`)

Investigated "where do the points actually come from" — broke each game's score into three bands per player: **≤400** (residual/remainder), **exactly 800** (the only value observed in [800, 1400)), and **≥1400** (approximated as exactly 1400 per word, since the rare 1800/2200-point outliers can't be separated from ordinary 1400s using the extracted cumulative tier counts). This decomposition is an exact algebraic identity — the three bands' differentials sum to `score_diff` with zero reconstruction error by construction — with one disclosed approximation: 5 of 167 games contain a flagged 1800/2200-point word, so the ≥1400 tier's true point total is a small underestimate in those specific games (the shortfall leaks into the ≤400 residual instead). Summed each band's contribution to the total score gap, both across all 167 games and per dataset separately. See `word_hunt_insights_combined.md` for what this revealed — it ended up being the sharpest single finding in the whole project.

## 4. Timestamp investigation (not pursued further)

Checked the status-bar clock across both datasets to see if real play-time data was recoverable. Dataset 1's 116 screenshots span only 7:43–7:56 (13 minutes) and dataset 2's 51 screenshots span only 11:41–11:47 (6 minutes) — both physically impossible for live gameplay at that volume, meaning these timestamps reflect a rapid screenshot/export session (scrolling through game history), not actual play time. Concluded there's no valid time-of-day or fatigue signal to extract here and stopped, rather than sinking further extraction effort into a dead end.

## 5. Pooled win-prediction model with 5-fold cross-validation

**Why:** the two datasets' win-prediction models (from earlier work) each used a single random train/test split — dataset 1's (65/51) held up reasonably, but dataset 2's (29/22) was badly overfit (86.2% train vs. 59.1% test accuracy) simply from having too little data. Pooling both datasets and cross-validating fixes both problems: more data to fit on, and every game gets evaluated exactly once as held-out data instead of the result depending on one lucky/unlucky split.

**Method:**
- Pooled all 167 games, feature set = `my_score, my_words, my_words_over_800, my_words_over_1400, my_avg_word_value` (`my_words_over_2200` dropped again — still constant/zero across all 167 games), plus the new binary `vs_opponent_2` feature in a second version, to directly test whether knowing the opponent's identity improves prediction.
- 5-fold cross-validation, folds stratified jointly by (win/loss, dataset) so every fold has a realistic mix of both outcomes and both opponents (fold sizes 32–35, fold win rates 43.8–45.7%, fold opponent-2 share 28–32%).
- Same hand-rolled Newton-Raphson/IRLS logistic regression as before (standardized features per training fold, small ridge penalty for stability), plus the linear-regression-thresholded version for comparison, both run with and without the opponent feature.
- Overall out-of-fold accuracy: 67.07% (logistic, no opponent feature) → **70.66%** (logistic, with opponent feature) → beats the pooled baseline of 55.09% by 12–16 points depending on version. Linear regression version came in slightly weaker across the board and produced more invalid (outside [0,1]) predictions when the opponent feature was added (7 of 167) — one more mark in favor of logistic regression as the right tool.

## Published to GitHub

The analysis (CSVs + markdown docs, not the raw screenshots — those stay local) is published at **https://github.com/CrackedPeanut34/word-hunt-analysis** (public repo).

## Full file list added this round

- `word_hunt_data_combined.csv`
- `word_hunt_summary_by_player.csv`
- `word_hunt_points_origin_decomposition.csv`, `word_hunt_points_origin_per_game.csv`
- `word_hunt_win_logistic_cv_metrics.csv`, `_coefficients_no_opp.csv`, `_coefficients_with_opp.csv`, `_oof_predictions.csv`
- `word_hunt_win_regression_cv_metrics.csv`
- `word_hunt_process_log_combined.md` (this file), `word_hunt_insights_combined.md`
