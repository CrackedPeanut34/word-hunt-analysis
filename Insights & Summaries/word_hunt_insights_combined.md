# Word Hunt — All Insights (Simple to Complicated)

Every individual finding from this project, in one place, ordered roughly from the simplest/most basic to the most involved. For the narrative version of how these fit together, see `word_hunt_summary.md`. For how everything here was built, see `word_hunt_process_log.md`.

230 games total: 116 vs. Opponent 1 (dataset 1), 51 vs. Opponent 2 (dataset 2), 63 more vs. Opponent 1 after a strategy change (dataset 3).

## The basics: record and averages

| | vs. Opponent 1 (before) | vs. Opponent 2 | vs. Opponent 1 (after strategy change) |
|---|---|---|---|
| Games | 116 | 51 | 63 |
| Win % | 39.66% | 56.86% | 44.44% |
| My avg score | 10,506.9 | 10,078.4 | 11,461.9 |
| Opponent avg score | 11,482.8 | 9,439.2 | 11,758.7 |
| My avg words found | 34.41 | 32.53 | 34.30 |
| Opponent avg words found | 32.91 | 30.24 | 32.25 |
| My avg word value (pts/word) | 298.98 | 300.96 | 326.09 |
| Opponent avg word value | 343.11 | 302.38 | 358.30 |

Simplest takeaway available from raw counts alone: I consistently find slightly *more total words* than every opponent, in every dataset — this was never a volume problem.

## Word count alone is a mediocre predictor of score

Simple OLS, `score = slope × words + intercept`:

| | R² (mine) | R² (opponent) |
|---|---|---|
| Dataset 1 | 0.653 | 0.480 |
| Dataset 2 | 0.680 | 0.643 |

Word count explains my own score better than it explains either opponent's — consistent with opponents' scores depending more on word *quality*, which word count alone can't capture.

## The correlation leaderboard: word value and word-value rate dominate

Pearson correlation of each stat against winning (excluding `score_diff` itself, which trivially determines the winner by definition and isn't an insight):

| Rank | Dataset 1 | Dataset 2 | Dataset 3 |
|---|---|---|---|
| 1 | avg_word_value_diff (0.632) | words_over_800_diff (0.636) | words_over_800_diff (0.701) |
| 2 | words_over_800_diff (0.624) | avg_word_value_diff (0.614) | hv_rate_diff (0.563) |
| 3 | hv_rate_diff (0.562) | hv_rate_diff (0.614) | avg_word_value_diff (0.555) |
| 4 | words_diff (0.532) | words_diff (0.546) | words_diff (0.511) |

`hv_rate_diff` (the ≥800-rate differential — the *share* of words that clear 800, not just the count) only entered this leaderboard once it was explicitly engineered as a feature; before that it simply wasn't checked. It turned out to belong near the top in all three datasets — a real, previously invisible signal.

## Win rate splits by word value

Splitting games by whether my average word value beat the opponent's that game:

| | Dataset 1 | Dataset 2 | Dataset 3 |
|---|---|---|---|
| Win % when I'm ahead on word value | 87.5% (28/32) | 82.76% (24/29) | — |
| Win % when I'm behind | 21.43% (18/84) | 22.73% (5/22) | — |

Bucketed finer (50-point-wide bands of the differential), dataset 1 shows a near step-function: 0% win rate in the worst band (−300 to −250) climbing to 100% in the best (100 to 150).

## Win rate splits by high-value (≥800) rate — the sharpest split in the project

Splitting games by whether my *share* of words worth ≥800 beat the opponent's:

| | Dataset 1 | Dataset 2 | Dataset 3 |
|---|---|---|---|
| Win % when my ≥800 rate is higher | 68.3% (41 games) | 82.6% (23 games) | **90.5%** (21 games) |
| Win % when it isn't | 24.0% (75 games) | 35.7% (28 games) | 21.4% (42 games) |

A 44–69 point swing depending on which side of this line a game falls on, and it gets *more* decisive in dataset 3, not less, despite the strategy change happening on the "≥1400" tier specifically rather than this one.

## Opponent strength tiers

Splitting games into weak/mid/strong opponent-performance terciles (by the opponent's own word value that game):

| Tier | Dataset 1 win % | Dataset 2 win % |
|---|---|---|
| Weak | 53.85% | 64.71% |
| Mid | 44.74% | 58.82% |
| Strong | 20.51% | 47.06% |
| Drop, weak→strong | 33.3 pts | 17.6 pts |

Both opponents show the same direction (tougher performance that game → lower win chance for me), but the effect is roughly half as steep against Opponent 2 — consistent with her being the closer overall matchup.

## Streaks, session-order trend, and consistency

- **Streaks:** longest win streak 7 games (both datasets 1 and 2 identically); longest loss streak 9 (dataset 1) / 6 (dataset 2). Average streak length **2.04 games in both datasets** — results alternate about as choppily regardless of overall win rate.
- **No trend across the game sequence.** Correlation between game order and winning: dataset 1 r=−0.011 (p=0.91), dataset 2 r=0.059 (p=0.68). No detectable warm-up or fatigue effect in either dataset.
- **Volatility (coefficient of variation):** my score is about as volatile as opponents' in both datasets (CV ≈ 0.41–0.46 for everyone), but my average word value is consistently *more stable* than either opponent's (CV 0.25–0.26 mine vs. 0.28–0.31 theirs) — I'm the steadier player on word quality specifically, not on raw score.
- **No volume/quality tradeoff.** Correlation between words found and average word value is mildly *positive* for both me and both opponents in both datasets (0.30–0.47) — good games tend to be good on both axes, not a case of rushing for count at the cost of quality.

## Fair three-way comparison (adjusted for board difficulty)

"Adjusted points" = score minus that game's own average (`total_score / 2`) — cancels out board-to-board richness since it's always relative to what was actually scored that specific game.

| Entity | Games | Win % | Avg word value | Score share | Adjusted points |
|---|---|---|---|---|---|
| Opponent 1 (all games, incl. dataset 3) | 179 | 58.10% | 348.45 | 51.78% | +368.44 |
| Me (vs Opponent 1, all games) | 179 | 41.34% | 308.52 | 48.22% | −368.44 |
| Me (vs Opponent 2) | 51 | 56.86% | 300.96 | 51.51% | +319.61 |
| Opponent 2 | 51 | 43.14% | 302.38 | 48.49% | −319.61 |

Ranked: Opponent 1 is the strongest of the three, I'm a clear second, Opponent 2 is the weakest — I beat her more often than not with a real scoring edge, not a coin flip.

## The points-origin decomposition — the project's sharpest single finding

Breaking every game's score into three exact point bands (≤400, exactly 800, ≥1400 — computed exactly once the 1800/2200 splits were known) and summing each band's contribution to the total score gap:

| | ≤400 tier | =800 tier | ≥1400 tier | **Total gap** |
|---|---|---|---|---|
| All 167 pre-dataset-3 games | +87,600 | −5,600 | **−162,600** | −80,600 |
| vs. Opponent 1 (dataset 1) | +54,400 | −33,600 | **−134,000** | −113,200 |
| vs. Opponent 2 (dataset 2) | +33,200 | +28,000 | **−28,600** | +32,600 |

Read plainly: the ≥1400 tier alone costs −162,600 points across those 167 games — more than double the entire net deficit. In the ≤400 and =800 bands combined, I'm actually **+82,000 ahead**. Remove the rare big-word tier from the picture entirely and the overall record flips to a clear lead. Against Opponent 1 specifically, I'd be ahead overall if that tier didn't exist (+20,800 in the other two bands vs. a −134,000 deficit in this one, larger than the total loss). Against Opponent 2, I win the other two bands by enough margin to cover a same-direction (but much smaller) deficit here too.

**The ≥1800 tier itself:** across the 167 pre-dataset-3 games, I found a ≥1800-point word in only 0.9–2.0% of games; both opponents found one in 11.8–14.7% of their games. The very top of the value distribution belongs almost entirely to opponents, not marginally — a sharper, more concentrated version of the same pattern running through the whole project.

## The strategy change: dataset 1 vs. dataset 3, tested properly

Two-sample Welch t-tests (unequal variance), not just eyeballed averages:

**Relative to the opponent (differential view):**

| Metric | Before | After | p-value | Real? |
|---|---|---|---|---|
| Win % | 39.66% | 44.44% | 0.53 | No |
| Score differential | −975.9 | −296.8 | 0.21 | No |
| Word value differential | −44.1 | −32.2 | 0.36 | No |
| My words ≥1400/game | 0.40 | 0.81 | 0.017 | **Yes** |

**My own numbers, opponent set aside (own-stats view):**

| My stat | Before | After | p-value | Real? |
|---|---|---|---|---|
| Score | 10,506.9 | 11,461.9 | 0.186 | No |
| Words found | 34.41 | 34.30 | 0.938 | No (flat, as intended) |
| Words ≥800 | 3.85 | 4.76 | 0.111 | No (trend) |
| **Words ≥1400** | **0.40** | **0.81** | **0.017** | **Yes** |
| **Avg word value** | **298.98** | **326.09** | **0.026** | **Yes** |
| Words ≥1400 rate | 1.13% | 2.26% | 0.021 | **Yes** |
| High-value (≥800) rate | 10.6% | 13.0% | 0.076 | No (trend) |

Three things are independently, statistically real: the ≥1400 count roughly doubled, the ≥1400 *rate* roughly doubled, and average word value rose ~9% — all pointing the same direction, all on the user's own side regardless of what the opponent did. The downstream payoff (win rate, score gap) moved the same direction but isn't provable yet at 63 games. A trend in board richness (combined avg word value up 321→342, p=0.088) was flagged as a possible partial confound, though the opponent's own ≥1400 rate barely moved (+0.12 vs. my +0.41), so it doesn't fully explain the shift.

## Win-prediction modeling: the full history

**v1 — single-split models per dataset, own-side features only** (`my_score, my_words, my_words_over_800, my_words_over_1400, my_avg_word_value`): dataset 1 hit 74.51% test accuracy (baseline 62.75%) on a 65/51 split; dataset 2's 29/22 split badly overfit (86.2% train vs. 59.1% test) — the first clear demonstration in this project that sample size matters.

**The multicollinearity bug:** `score`, `words`, and `avg_word_value` aren't independent (`score = words × avg_word_value` by construction). Including all three as separate regression inputs let the fit swing unstably, occasionally producing backwards predictions (higher score sometimes predicting a *lower* win chance at fixed word count). Fixed by dropping `score` from the feature list; accuracy was unaffected (within a point either way), coefficients became correctly signed.

**Pooled 5-fold cross-validation** (167 games, stratified by win/opponent): 67.07% out-of-fold accuracy without an opponent-identity feature, **70.66% with one** — knowing who you're facing is worth +3.6 points, against a 55.09% baseline. This is more trustworthy than either dataset's single-split number, since every game gets evaluated exactly once as held-out data.

**The 1800-tier coefficient caveat:** when added as a model feature (after the missing-tier audit, see process log §3), its fitted coefficient came out *negative* — not a believable effect, just noise from only 2 of 167 games having one on my side. Kept in the front-end tool as-fitted with a visible caveat, rather than silently corrected, since the underlying issue (data sparsity) isn't something a coefficient adjustment can honestly fix.

**Refitting on all 230 games (post-strategy-change) made the multi-feature model *worse*:** cross-validated accuracy dropped to 63.6% (no opponent feature) / 64.4% (with it) from 68.26%/66.47% on the pre-dataset-3 data, and one coefficient (`my_words_over_1400`) flipped sign. The strategy-change data destabilized this particular model rather than improving it with more data.

**The simple score+opponent model — currently the best-performing option:** just `my_score` and a binary opponent-identity feature. A single 70/30 split gave 69.6% test accuracy (58.0% baseline). Properly 5-fold cross-validated on two independently-seeded fold assignments: 68.3% and 67.0% mean fold accuracy (~67.6% average) — checked on a second, differently-seeded split specifically to rule out one fold assignment being lucky. A score-only variant (for an opponent with no game history) cross-validates at ~65.2%.

**Fragility of a single train/test split, demonstrated directly:** testing the score-only model at 75/25, 80/20, and 85/15 splits gave test accuracies of 67.2%, 67.4%, and 64.7% — and at 85/15, the tiny 34-game test set happened to have a lopsided win rate, making the model's accuracy come out *exactly equal to* just guessing the majority class. The underlying fit hadn't gotten worse; the evaluation got noisier as the holdout shrank. This is the same lesson as the CV work, demonstrated a second, more concrete way.

**Adding ≥800 rate to the simple model made it worse, not better** — tested on two separate, differently-seeded 5-fold splits (with paired same-fold checks in both directions for rigor): ~65.2–65.3% either way, versus ~67–68% without it. Likely a milder version of the original score-redundancy problem, since `hv_rate` and `score` are derived from overlapping underlying data.

## The front-end predictor tool

A published, live calculator built around the above. Takes score, word count, and tier counts as input; originally ran the multi-feature opponent-conditioned model, later switched to the simpler score+opponent model once that was shown to outperform it on the full dataset. Also shows, for every input plus three engineered metrics (avg word value, ≥800 rate, ≥1400 "big-word" rate), what percentile that value falls at against the user's full 230-game history — with a deliberate fix so that a below-average entry (common for the rare tiers, where 0 is often the typical result) is shown as a plain comparison to the personal average rather than a misleadingly-framed "high percentile."
