# Word Hunt Project — Process Log (Combined Cross-Dataset Work)

Everything done that spans both datasets at once, rather than treating them separately.

## 0. Correction (2026-08-04): the missing 1800-point tier

The original extraction tracked point values as cumulative thresholds only (≥800, ≥1400, ≥2200) — a word worth 1800 was correctly counted toward both the ≥800 and ≥1400 tallies, but there was no dedicated ≥1800 tier, so an 1800 was indistinguishable from a 1400 in the data. Six games had been informally flagged with a note during the original extraction when an 1800 happened to be noticed, but nothing had systematically checked for it everywhere it could occur.

**Audit method:** Only games where `my_words_over_1400 > 0` OR `opponent_words_over_1400 > 0` can possibly contain an 1800 (if that count is 0, there's nothing above 1400 to find). That filtered 167 games down to **87 that needed re-checking** — also confirmed the max count in that tier is 7, well under the 15-word display cap, so every relevant word is guaranteed visible in the original screenshot (no truncation risk for this check). Split the 87 into 6 parallel batches, each re-reading its assigned screenshots and reporting the exact count of 1400/1800/2200-point words per side, cross-checked against the already-known ≥1400 and ≥2200 totals.

**Verification:**
- Automated: every batch's reported (1400 + 1800 + 2200) count matched the previously-known ≥1400 total exactly, and every reported 2200 count matched the previously-known ≥2200 total exactly — zero discrepancies across all 87 games.
- Manual: directly re-read a sample of the newly-discovered (previously unflagged) 1800s — including both instances on my own side (IMG_5785, IMG_5848) — against the source screenshots. All matched.

**Result:** found 17 of 167 games with at least one ≥1800 word (2 on my side, 19 word-instances on the opponent side across both datasets — some games had more than one). Added `words_over_1800` (mine and opponent's) plus `words_over_1800_diff` to every base, enriched, and summary CSV in the project. Recomputed the points-origin decomposition (see `word_hunt_insights_combined.md`) **exactly** instead of approximating every ≥1400 word as worth 1400 — reconstruction error against the real score differential is now precisely 0, versus a small systematic undercount before. Confirmed via cross-validation that this changes nothing about the win-prediction models (same features, same values, byte-identical accuracy) and that adding the new tier as its own feature doesn't help (too rare — 2 occurrences in 167 games), so the front-end predictor tool needed no changes.

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

## 6. Front-end predictor tool, and a follow-up fix (dropping `my_score` as a model input)

Built a live win-probability calculator (a published artifact) around the pooled logistic model, taking score/words/tier counts as input. Testing it surfaced a real problem: holding score fixed and varying word count produced **non-monotonic, occasionally backwards predictions** (e.g. more score at a fixed word count sometimes *lowered* the predicted win chance). Root cause: `my_score`, `my_words`, and `my_avg_word_value` aren't independent (`score = words × avg_word_value` by definition), and including all three as separate regression inputs let the fit distribute credit between them in an unstable, occasionally sign-flipped way — a sharper, concrete symptom of the multicollinearity flagged as a caveat ever since the very first logistic regression.

**Fix:** refit both pooled models (with and without the opponent feature) dropping `my_score` entirely, keeping `my_words`, `my_words_over_800`, `my_words_over_1400`, and `my_avg_word_value` — same 5-fold CV, same fold assignment as the original run, for a clean before/after:

| | With score (original) | **Without score (fixed)** |
|---|---|---|
| No opponent feature — out-of-fold accuracy | 67.07% | **67.66%** |
| With opponent feature — out-of-fold accuracy | 70.66% | **69.46%** |

Accuracy is a wash (within a point either way — small-sample noise, not a real difference), but the coefficients are now all correctly signed and positive, and a spot-check (score fixed at 15,000, words swept 15→50) confirmed the predictions are monotonic and sane again instead of dipping and recovering. The front-end tool's input fields didn't change — it still asks for score, since that's what's visible on screen — only the math dropped the redundant term. New coefficient files: `word_hunt_win_logistic_cv_coefficients_noscore_no_opp.csv`, `_noscore_with_opp.csv`, `word_hunt_win_logistic_cv_metrics_noscore.csv`.

**Second follow-up: added the 1800-point tier as a model input, at explicit request, with a documented caveat.** Refit both pooled models (still no `my_score`) adding `my_words_over_1800` as a feature: 66.47% out-of-fold accuracy without the opponent feature (down from 67.66%), 68.26% with it (down from 69.46%). The fitted coefficient for this term is **negative** (−0.87 to −1.27 in raw units) — i.e. the model currently reads "having a ≥1800 word slightly hurts your win chance," which is not a believable causal effect. Root cause: only 2 of 167 games ever had one on my side, so there's essentially no data to estimate this coefficient from; the fit is dominated by noise. Rather than silently overriding or flooring the coefficient, added the field to the tool as requested with the fitted (negative) value intact, plus a visible caveat in the UI stating the sparsity issue directly, so the tool's behavior is honest about its own uncertainty rather than quietly "fixed" in a way that isn't actually justified by the data. New coefficient files: `word_hunt_win_logistic_cv_coefficients_with1800_no_opp.csv`, `_with1800_with_opp.csv`.

## 9. Correction (2026-08-05): a new rate metric, and closing the same kind of gap the 1800-tier fix should have closed

Added two engineered "high word value rate" metrics — the share of a player's own words that clear a tier, not just the raw count — to every enriched dataset: `hv_rate` (≥800 words ÷ total words, this already existed under that name) and the new **`words_over_1400_rate`** (≥1400 words ÷ total words, plus its `_diff` version). Rate is the more honest metric than a raw tier count for cross-comparison, since it isn't distorted by one player simply finding more total words than the other.

While wiring this in, audited where `hv_rate` (and by extension where `words_over_1400_rate` should also land) had and hadn't propagated — the same category of gap as the earlier missing-1800-tier problem: a metric existed at the base/enriched layer but silently never made it into the downstream summary tables that are supposed to mirror it. Found it **missing** from every one of: `word_hunt_summary_overall*.csv`, `_by_result*.csv`, `_by_total_score_bucket*.csv` (all datasets, plus the opponent-1-all and all-opponents combined versions), `word_hunt_summary_by_player.csv`, `word_hunt_insight_opponent_tiers*.csv`, `word_hunt_winrate_by_word_value_bucket*.csv` / `_sign*.csv`, and the correlation leaderboards. The opponent-tier and word-value-bucket/sign tables additionally never had a full ≥800/≥1400/≥1800/≥2200 tier breakdown at all (not just missing 1800 — missing the whole thing), which was backfilled at the same time for consistency with every other bucket-style table in the project.

**Result:** every overall/by-result/bucket/tier/leaderboard table across all three datasets now carries both rate metrics (and the opponent-tier / word-value tables now carry the full tier breakdown too). One genuinely new finding fell out of this: `hv_rate_diff` (≥800 rate differential) ranks **#3–#4 by correlation with winning across all three datasets** — a real, previously-invisible signal, since it was never in the leaderboard's candidate stat list before. Also extended `word_hunt_dataset1_vs_dataset3_own_stats_comparison.csv` with a `my_words_over_1400_rate` row: it's independently significant too (1.1% → 2.3%, p=0.021, roughly doubled) — consistent with the earlier finding that the ≥1400 *count* increase wasn't just a byproduct of finding more words overall (word count stayed flat), reinforcing that this is a real, targeted rate increase.

## 10. Simple score+opponent model, cross-validated, replaces the multi-feature model in the predictor tool

Refit the full multi-feature model (words, tier counts, avg word value, opponent) on all 230 games (it had only ever been fit through dataset 2): 5-fold CV accuracy actually **dropped** to 63.6%/64.4% (from 68.26%/66.47% on the old 167-game version) once dataset 3 was folded in — the strategy-change data made the richer model less stable, not more accurate, and one of its coefficients (`my_words_over_1400`) flipped sign in the opponent-blind version.

In parallel, a much simpler two-feature model (`my_score`, `vs_opponent_2` binary) was tested this session across a single 70/30 split, then properly 5-fold cross-validated on two independently-seeded fold assignments (68.3% and 67.0% mean fold accuracy — averaging ~67.6%), and a score-only variant for unknown opponents (~65.2%). Also tested whether adding `hv_rate` (≥800 rate) as a third feature improved it: it consistently made things worse (~65.2–65.3% vs ~67–68%) on two different fold assignments, likely the same flavor of redundancy-with-score problem as before, just milder.

**Conclusion: the simple 2-feature model now outperforms the richer one on the full 230-game dataset.** Switched the front-end predictor tool over to it — score and opponent identity drive the win estimate now, while words/tier-count inputs are kept in the UI purely to power a new percentile-comparison section (see below) rather than feeding the probability model. New coefficient files: `word_hunt_win_logistic_predictor_v2_score_opponent_coefficients.csv` (deployed), `_score_only_coefficients.csv` (deployed, unknown-opponent case), plus the stale 230-game refit of the old model kept for the record (`_230games_coefficients_with_opp.csv` / `_no_opp.csv`) even though it's no longer used.

## 11. Predictor tool: added a percentile-against-your-own-history section

At explicit request, the predictor now shows, after computing the win estimate, where the entered game's stats rank against all 230 of the user's own recorded games — for every raw input (score, words, ≥800/1400/1800 counts) plus three engineered metrics (avg word value, high-value rate ≥800, and the "big-word rate" ≥1400, i.e. `words_over_1400_rate`). Implemented as a binary search against sorted historical arrays embedded directly in the page (230 numbers per stat — trivial size, no external data fetch needed). Percentiles are computed against the full personal history regardless of which opponent is selected, since word-finding performance is being compared to the player's own past, not adjusted for matchup.

## Published to GitHub

The analysis (CSVs + markdown docs, not the raw screenshots — those stay local) is published at **https://github.com/CrackedPeanut34/word-hunt-analysis** (public repo).

## Full file list added this round

- `word_hunt_data_combined.csv`
- `word_hunt_summary_by_player.csv`
- `word_hunt_points_origin_decomposition.csv`, `word_hunt_points_origin_per_game.csv`
- `word_hunt_win_logistic_cv_metrics.csv`, `_coefficients_no_opp.csv`, `_coefficients_with_opp.csv`, `_oof_predictions.csv`
- `word_hunt_win_regression_cv_metrics.csv`
- `word_hunt_process_log_combined.md` (this file), `word_hunt_insights_combined.md`
