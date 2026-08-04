# Word Hunt — Dataset 2 Insights (vs. Opponent 2)

51 games (IMG_5828–IMG_5878). 29 wins, 22 losses, 0 draws. Win rate **56.86%** — a completely different picture from dataset 1's 39.66%. Every finding below is stated fresh, with the dataset 1 number alongside it for comparison.

## The headline: opponent 2's word value is much closer to yours than opponent 1's was

The single biggest difference between the two datasets: your average word value (points per word found) is nearly **tied** with opponent 2's — 300.96 vs. 302.38, a gap of just −1.42 points. In dataset 1, your avg word value trailed the opponent's by −44.13 points, a real and consistent gap. That one number swing largely explains everything else that changed: your average score differential flipped from **−975.86 (behind)** in dataset 1 to **+639.22 (ahead)** in dataset 2, and your win rate jumped from 39.66% to 56.86%.

To be precise about what this does and doesn't mean: a near-tied *average word value* is not the same as a tied *overall skill level*. You clearly come out ahead of opponent 2 on the metrics that actually decide games — you win 56.86% of your head-to-head matches and outscore her by +639.22 points on average. The word-value gap narrowing (vs. dataset 1) is what changed between the two matchups; it isn't a claim that you and opponent 2 are equally good. See `word_hunt_summary_by_player.csv` for a fair, matchup-by-matchup breakdown (and an adjusted-for-board-difficulty rating) that makes this explicit.

**See it side by side:** [Word Value Differential Distributions](https://claude.ai/code/artifact/6034b8ee-69da-403b-9e31-eeee4251e05a) plots both datasets on the same bins and the same axis. Dataset 1's shape sits well left of zero (mean −44.13, mass concentrated in the −150-to-0 range, only 32 of 116 games positive). Dataset 2's shape is centered almost exactly on zero (mean −1.42) and visibly more balanced — negative and positive games are much closer to evenly split (22 vs. 29), with real mass pushed into the 0-to-100 range that dataset 1 barely reaches. Of everything in this project, this pair of histograms is the single clearest picture of what actually changed between the two opponent pools — not your game, theirs.

## Word value differential is still the strongest signal — and replicates almost exactly

Splitting games by whether your avg word value beat the opponent's that game:

| | Dataset 1 (vs. opp) | Dataset 2 (vs. opp2) |
|---|---|---|
| Win % when your word value is **higher** | 87.5% (28/32 games) | 82.76% (24/29 games) |
| Win % when your word value is **lower** | 21.43% (18/84 games) | 22.73% (5/22 games) |
| Gap between the two | 66.1 points | 60.0 points |

This is a strong replication — almost the same effect size shows up against a completely different opponent. Word value differential isn't a fluke of dataset 1's specific matchups; it looks like a real, general property of how these games are decided.

## Correlation leaderboard — same story, different order

Excluding score_diff (which trivially determines the winner):

| Dataset 1 (top 3) | r | Dataset 2 (top 3) | r |
|---|---|---|---|
| avg_word_value_diff | 0.632 | words_over_800_diff | 0.636 |
| words_over_800_diff | 0.624 | avg_word_value_diff | 0.614 |
| words_diff | 0.532 | words_diff | 0.546 |

Same three stats dominate in both datasets, just with avg_word_value_diff and words_over_800_diff swapping the #1/#2 spots by a hair. The overall pattern — quality-of-word measures beating raw word count — holds up.

## Words → score regression: opponent 2's score is much more predictable than opponent 1's was

| | Dataset 1 | Dataset 2 |
|---|---|---|
| Mine: R² | 0.653 | 0.680 |
| Opponent: R² | 0.480 | **0.643** |
| Total: R² | 0.585 | 0.694 |

In dataset 1, word count explained your score much better than the opponent's (a 17-point R² gap), which we read as "the opponent's score depends more on word *quality*, which word count alone doesn't capture." In dataset 2, that gap nearly closes (0.680 vs 0.643, only a 4-point difference) — consistent with the word-value finding above: opponent 2's performance is more similar in *shape* to your own than opponent 1's was.

## Win-prediction models — a real demonstration of the "you need more data" warning

Dataset 2 only has 51 games, so the train/test split had to shrink too — used the same ratio as before (56% train) instead of the same raw count, giving **29 train / 22 test**.

| | Dataset 1 (65 train / 51 test) | Dataset 2 (29 train / 22 test) |
|---|---|---|
| Linear regression test accuracy | 72.55% | 59.09% |
| Logistic regression test accuracy | 74.51% | 59.09% |
| Logistic regression train accuracy | 73.85% | **86.21%** |
| Baseline (always guess majority class) | 62.75% | 54.55% |
| Logistic precision (win) / recall (win) | 0.667 / 0.632 | 0.529 / 0.900 |
| McFadden pseudo-R² (train / test) | 0.178 / 0.037 | 0.280 / **−2.113** |

This is worth sitting with: dataset 2's model barely beats the naive baseline (59.09% vs 54.55%, only +4.5 points, versus dataset 1's +12 points), *and* shows a massive train/test accuracy gap (86.2% vs 59.1%) that dataset 1's model didn't have (73.85% vs 74.51% — nearly identical). That gap is the textbook signature of overfitting on too little data. With only 29 training games and ~19 wins spread across 4 meaningfully independent features, the events-per-predictor ratio is about 4.75 — less than half the already-borderline ratio dataset 1 had (11.5), and far under the 10–20 standard guideline. The logistic regression's coefficients reflect this too: my_score's odds ratio came out to roughly 712 per standard deviation, which is not a believable real-world effect size — it's the model fitting noise in a too-small training set. Recall also skewed heavily toward predicting "win" (0.9 recall, 0.529 precision — it cried "win" a lot and was right about half the time). This is exactly the failure mode predicted when we discussed sample size for dataset 1 — now we have a direct before/after showing what happens with even less data than before.

## Streaks — nearly identical average streak length

| | Dataset 1 | Dataset 2 |
|---|---|---|
| Longest win streak | 7 | 7 |
| Longest loss streak | 9 | 6 |
| Streak segments | 57 (of 116 games) | 25 (of 51 games) |
| Average streak length | 2.04 games | 2.04 games |

The average streak length is identical to two decimal places — results alternate about as choppily in both datasets, even though the overall win rate is very different.

## No trend over the game sequence — confirmed again

Dataset 1 showed no evidence of improving or declining across its 116-game sequence (r=−0.011, p=0.91). Dataset 2 shows the same null result: correlation between game order and winning is 0.059 (p=0.68), and neither score_diff nor avg_word_value_diff trend significantly across the 51 games (p=0.59 and p=0.39 respectively). Two for two — no detectable warm-up or fatigue pattern in either dataset.

## Opponent strength tiers — the effect is real but much gentler against opponent 2

| Tier | Dataset 1 win % | Dataset 2 win % |
|---|---|---|
| Weak opponent | 53.85% | 64.71% |
| Mid opponent | 44.74% | 58.82% |
| Strong opponent | 20.51% | 47.06% |
| **Drop, weak→strong** | **33.3 points** | **17.6 points** |

Both datasets show the same direction — you win less often against opponents whose words are worth more — but the effect is roughly half as steep against opponent 2's pool. Combined with the near-zero average word-value gap overall, this fits a consistent picture: opponent 2, even at their "strong" tier, isn't pulling as far ahead of you as opponent 1's strong tier did.

## High-value word rate — a notable flip

| | Dataset 1 | Dataset 2 |
|---|---|---|
| My rate (share of words worth ≥800) | 10.6% | 10.4% |
| Opponent's rate | 14.6% | **10.35%** |

Your own rate barely moved. The opponent's rate dropped a lot — from clearly ahead of you (14.6%) to essentially matching you (10.35%). This is the same "opponent 2's word quality is much closer to yours than opponent 1's was" story showing up in yet another metric — not evidence you're evenly matched overall (you still win 56.86% of these games).

## Volatility — same pattern, slightly more pronounced

| | Dataset 1 CV | Dataset 2 CV |
|---|---|---|
| My score | 0.407 | 0.456 |
| Opponent score | 0.383 | 0.459 |
| My avg word value | 0.250 | 0.258 |
| Opponent avg word value | 0.283 | 0.307 |

Same qualitative pattern as dataset 1: your word-value is more consistent than the opponent's in both datasets (you're the steadier performer on quality), while raw score volatility is roughly comparable between you and the opponent in both cases (dataset 2 has them essentially tied at 0.456 vs 0.459, versus a small gap in dataset 1).

## No volume/quality tradeoff — replicated, and stronger this time

| | Dataset 1 | Dataset 2 |
|---|---|---|
| corr(my_words, my_avg_word_value) | 0.351 | 0.395 |
| corr(opp_words, opp_avg_word_value) | 0.303 | 0.467 |

Same finding as before — no evidence that finding more words costs you word quality. If anything the positive relationship is a bit stronger in dataset 2, especially for the opponent (0.467). Good games are good on both axes; there's no rushing-for-volume tradeoff in either dataset.

## Bottom line

The two datasets tell a coherent, complementary story: the *mechanics* of what makes you win (word value beats word count, opponent quality matters, no session-order trend, no volume/quality tradeoff) replicate closely across two different opponent pools. What differs is simply *how tough opponent 2 is relative to you* — much closer to even than opponent 1 was, which is the entire reason the win rate, score differential, and word-value gap all moved together in the same direction. And the win-prediction model comparison turned into an unplanned but genuinely useful demonstration of why sample size matters: the same modeling approach that looked reasonably solid on 116 games fell apart on 51.
