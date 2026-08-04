# Word Hunt — Combined Cross-Dataset Insights

167 games total: 116 vs. Opponent 1, 51 vs. Opponent 2, unified into `Stats/word_hunt_data_combined.csv`.

**2026-08-04 correction:** a missing 1800-point tier was found and backfilled across the whole project — see `word_hunt_process_log_combined.md` for the full audit. All 87 games with any word worth ≥1400 were individually re-checked to split that count into exact 1400 / 1800 / 2200 tiers (previously the points-origin decomposition below approximated every ≥1400 word as worth exactly 1400). The numbers in this section are updated to the exact values; every other analysis in the project (correlations, win-rate splits, regression models) was confirmed unaffected, since they only ever used the cumulative ≥800/≥1400/≥2200 thresholds, which were already correct.

## Headline: it isn't "word value" broadly — it's specifically the ≥1400-point tier

Earlier analysis treated "avg word value" as one blended number. Breaking every game's score down into three point bands (≤400, exactly 800, and ≥1400 — now computed **exactly**, using each word's real value instead of assuming every ≥1400 word is worth 1400) and summing each band's contribution to the score gap across all 167 games gives a much sharper answer to "where do the points actually come from":

| | ≤400 tier | =800 tier | ≥1400 tier | **Total gap** |
|---|---|---|---|---|
| **All 167 games** | +87,600 | −5,600 | **−162,600** | −80,600 |
| **vs. Opponent 1** (116 games) | +54,400 | −33,600 | **−134,000** | −113,200 |
| **vs. Opponent 2** (51 games) | +33,200 | +28,000 | **−28,600** | +32,600 |

Read the "All 167 games" row plainly: the ≥1400 tier alone costs you **162,600 points** — double your *entire* net deficit (80,600). In the ≤400 and =800 tiers combined, you're actually **ahead by +82,000 points** across your whole history. The entire reason you're behind overall is the rare, high-value word tier — remove it from the picture and you're winning by a wide margin.

This holds up against both opponents individually, just to different degrees:
- **Vs. Opponent 1:** you're actually ahead in the low+mid tiers combined (+54,400 − 33,600 = +20,800), and the ≥1400 tier deficit (−134,000) is larger than your total loss (−113,200) — i.e., you'd be *ahead* against Opponent 1 if the ≥1400 tier didn't exist.
- **Vs. Opponent 2:** you dominate the ≤400 and =800 tiers (+33,200 and +28,000), enough to overcome a ≥1400 tier deficit that exists *even here* (−28,600) — you still lose the high-value-word battle against Opponent 2, you just win by a wide enough margin everywhere else that it doesn't matter.

**The real story:** you consistently lose the ≥1400-point-word battle against everyone, and now that this is measured exactly rather than approximated, the gap is even more decisive than first estimated. Whether you win the *game* depends entirely on whether your edge in the more common ≤400/800 tiers is big enough to cover that gap — which it is against Opponent 2, and isn't against Opponent 1.

## The 1800-point tier itself, now that it's tracked

Across all 167 games, only **2 of your own words** were ever worth 1800+ (both single occurrences, one per opponent), versus **19 word-instances from opponents** (17 from Opponent 1's games, 6 from Opponent 2's — some games had more than one). Per-game rate, from `word_hunt_summary_by_player.csv`:

| Entity | Avg ≥1800 words per game |
|---|---|
| Opponent 1 | 0.147 |
| Opponent 2 | 0.118 |
| Me (vs. Opponent 1) | 0.009 |
| Me (vs. Opponent 2) | 0.020 |

Both opponents find a ≥1800-point word roughly 12-15% of the time; you find one roughly 1-2% of the time. This is the sharpest, most concentrated version yet of the same pattern running through this whole project — it's not that opponents are marginally better at finding good words, it's that the very top of the value distribution belongs almost entirely to them. Checked whether adding this as its own model feature helps predict wins: it doesn't — cross-validated accuracy stayed flat or dropped slightly (67.66%→66.47% without the opponent-identity feature, 69.46%→68.26% with it) when added, since 2 occurrences out of 167 games is too sparse to learn from. The existing win-prediction models and the front-end predictor tool are correctly left as-is.

(Method note: the ≥1400 tier's point value is approximated as exactly 1,400 per word, since the rare 1800/2200-point outliers can't be separated from ordinary 1,400s in the extracted data. Only 5 of 167 games contain a flagged 1800/2200 word, so this slightly understates the true ≥1400 tier deficit in a handful of games — a small, conservative bias that doesn't change the conclusion. Percentage-of-gap figures are omitted above because when tiers partially offset each other with opposite signs, percentages can exceed 100% or flip sign in a way that's more confusing than informative — the raw point totals tell the story more clearly. Full per-game detail is in `word_hunt_points_origin_per_game.csv`.)

## Fair three-way comparison: Me vs. Opponent 1 vs. Opponent 2

(See `word_hunt_summary_by_player.csv`.) The correct way to compare all three is matchup-by-matchup, not blending your combined average against an opponent who never played those extra games:

| Entity | Games | Win % | Avg Score | Avg Word Value | Score Share | **Adjusted Points** |
|---|---|---|---|---|---|---|
| Opponent 1 | 116 | 59.48% | 11,482.8 | 343.11 | 52.29% | **+487.93** |
| Me (vs Opponent 2) | 51 | 56.86% | 10,078.4 | 300.96 | 51.51% | **+319.61** |
| Me (vs Opponent 1) | 116 | 39.66% | 10,506.9 | 298.98 | 47.71% | **−487.93** |
| Opponent 2 | 51 | 43.14% | 9,439.2 | 302.38 | 48.49% | **−319.61** |

"Adjusted points" = score minus that game's own average (`total_score / 2`), which cancels out board-to-board richness/difficulty since it's always measured relative to what was actually scored in that specific game. Ranked: **Opponent 1 is the clear strongest of the three, you're a clear second, Opponent 2 is the weakest** — you beat her 56.86% of the time with a real scoring edge, not by a coin flip.

## Does knowing who you're playing improve win prediction? Yes — pooled 5-fold cross-validation

Pooled all 167 games into one model (instead of two separate, much smaller ones) and added a binary `vs_opponent_2` feature — "do you know your opponent going in." Evaluated with 5-fold cross-validation (every game tested exactly once as held-out data) instead of one lucky/unlucky train-test split:

| | Without opponent feature | **With opponent feature** |
|---|---|---|
| Mean fold accuracy | 67.14% (±4.4%) | **70.78% (±6.8%)** |
| Overall out-of-fold accuracy | 67.07% | **70.66%** |
| Precision / Recall | 0.652 / 0.573 | 0.691 / 0.627 |
| McFadden R² (out-of-fold) | 0.094 | 0.112 |

Baseline (always guess the majority class): 55.09%. Both models clear it comfortably; knowing which opponent you're facing is worth **+3.6 accuracy points** on its own. The feature's coefficient translates to roughly a **2.85x swing in your predicted odds of winning** just from facing Opponent 2 instead of Opponent 1, holding your own stats constant — a big, intuitive effect that matches everything else in this analysis.

This number (70.66%) is also more trustworthy than either of the two original single-split results (74.51% on dataset 1 alone, 59.09% on dataset 2 alone) — those were each at the mercy of one specific random split (the dataset 2 one was badly overfit, as flagged earlier). Cross-validating the pooled data gives one honest estimate instead of two conflicting, individually-shaky ones.

## Files
- `word_hunt_data_combined.csv` — all 167 games, unified `opponent_*` columns (regardless of dataset), `dataset`/`opponent_label` identifying who, plus `vs_opponent_2`, adjusted-points, and score-share columns
- `word_hunt_summary_by_player.csv` — the fair per-matchup 3-way comparison table above
- `word_hunt_points_origin_decomposition.csv` — the tier-by-tier point-gap breakdown (combined + per dataset)
- `word_hunt_points_origin_per_game.csv` — same breakdown at per-game granularity
- `word_hunt_win_logistic_cv_metrics.csv`, `_coefficients_no_opp.csv`, `_coefficients_with_opp.csv`, `_oof_predictions.csv` — pooled logistic regression cross-validation results
- `word_hunt_win_regression_cv_metrics.csv` — pooled linear regression cross-validation results, for comparison
- `word_hunt_process_log_combined.md` — how all of the above was built
- `word_hunt_insights_combined.md` — this file
