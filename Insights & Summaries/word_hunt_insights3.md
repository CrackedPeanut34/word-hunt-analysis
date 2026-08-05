# Word Hunt — Dataset 3 Insights (vs. Opponent 1, after the "find bigger words" strategy change)

63 games (IMG_5888–IMG_5950), against Opponent 1 again — the same opponent as dataset 1, but played after a deliberate change in approach: instead of just maximizing total words found, the goal shifted to prioritizing finding *longer* (higher-value) words. This dataset was extracted with the ≥1800 tier tracked natively from the start (dataset 1 and 2 needed a retroactive audit for it; see `word_hunt_process_log_combined.md` §0).

**Record:** 28 W – 35 L – 0 D. Win rate **44.44%**, up from dataset 1's 39.66% against this same opponent.

## Did the strategy change work? Be careful here.

The headline numbers move in the right direction, but only one of them clears the bar for "probably real" rather than "could be noise" at this sample size:

| Metric | Dataset 1 (before) | Dataset 3 (after) | Change | Statistically distinguishable from noise? |
|---|---|---|---|---|
| Win % | 39.66% | 44.44% | +4.8 pts | **No** (z-test on proportions, p=0.53) |
| Avg score diff (mine − opp) | −975.86 | −296.83 | +679 | **No** (Welch t-test, p=0.21) |
| Avg word value diff | −44.13 | −32.22 | +11.9 | **No** (p=0.36) |
| Avg my `words_over_1400` (big words/game) | 0.40 | 0.81 | +0.41 (~2×) | **Yes** (p=0.017) |

The one change that's actually statistically solid is the behavior itself: I am finding roughly twice as many 6+ letter words per game as before, and that increase is unlikely to be chance. Everything downstream of it — win rate, score differential, word-value gap — moved in a promising direction too, but none of those moves are large enough relative to their game-to-game noise to call "proven" yet with only 63 games. Read this as "the strategy change is real and measurable in behavior; whether it's paying off in wins is still an open question, not a confirmed yes."

## My own numbers, not just the differential (dataset 1 vs. dataset 3)

Everything above compares *me relative to the opponent*. That can hide real improvement if the opponent also got somewhat better in the same window, or hide a fake improvement if the opponent got worse. So it's worth testing my own raw stats against my own past self, opponent aside — same Welch t-test approach, dataset 1 (n=116) vs. dataset 3 (n=63):

| My stat | Dataset 1 | Dataset 3 | Change | Significant? |
|---|---|---|---|---|
| Score | 10,506.9 | 11,461.9 | +955.0 (+9.1%) | No (p=0.186) |
| Words found | 34.41 | 34.30 | −0.11 | No (p=0.938) — essentially flat |
| Words ≥800 | 3.85 | 4.76 | +0.91 (+23.6%) | No (p=0.111) — close, not quite |
| **Words ≥1400** | **0.40** | **0.81** | **+0.41 (+104%)** | **Yes (p=0.017)** |
| Words ≥1800 | 0.01 | 0.03 | +0.02 | No (p=0.336) — too rare to say |
| **Avg word value (score ÷ words)** | **298.98** | **326.09** | **+27.1 (+9.1%)** | **Yes (p=0.026)** |
| High-value word rate (share ≥800) | 10.6% | 13.0% | +2.4 pts | No (p=0.076) — a trend, not quite significant |

This is a better-grounded result than the differential view: **two of my own stats improved significantly on their own terms, independent of what the opponent did** — I'm finding roughly twice as many 6+ letter words per game, and my average points-per-word rose about 9%. Word count itself didn't move at all (34.41 → 34.30), which is exactly consistent with the stated goal: not finding *more* words, finding *better* ones. The score increase (+955, +9.1%) trends the same direction but isn't statistically solid yet on its own — score is noisier game-to-game than the tier counts or avg value that drive it.

Saved as `word_hunt_dataset1_vs_dataset3_own_stats_comparison.csv`.

## Word value differential, visualized

**[Word Value Differential — All Three Datasets](https://claude.ai/code/artifact/bf15f7bb-b153-4a52-87d5-0611d8e543ab)** — a per-game histogram (50-point bands, matching the bucket CSVs below) for all three datasets stacked win/loss. The visual story: dataset 1's mass sits well left of zero and is almost entirely coral (losses) until the gap closes; dataset 3 still leans left of zero on average (mean −32.2, versus dataset 1's −44.1) but has visibly more bars pushing right into positive territory, and — as in every dataset — once word value tips in my favor, the bars turn almost entirely gold.

## A confound worth flagging: were dataset 3's boards just richer?

`total_avg_word_value` (mine + opponent's combined points-per-word, a rough proxy for how generous the letter boards were) rose from 321.07 in dataset 1 to 341.69 in dataset 3 (p=0.088 — a trend, not quite significant). Opponent 1's own avg word value also rose in dataset 3 (343.11 → 358.30). That means at least part of the "more big words" story could be boards that simply had more long words available to find, not purely a change in my search behavior. It doesn't fully explain the jump in *my* `words_over_1400` rate specifically (I'd expect the opponent's rate to rise by a similar proportion if it were pure board effect, and it rose much less: opponent's over_1400 avg went from 1.17 → 1.29, only +0.12, versus my +0.41), but it means the improvement isn't as cleanly attributable to "I got better at finding big words" as it might look at first glance.

## Where the record still favors the opponent

Opponent 1 still out-paces me on every raw quality metric in dataset 3, same as dataset 1:
- Opponent avg word value: 358.30 vs my 326.09 (gap narrowed from −44.13 to −32.22, but didn't close)
- Opponent avg `words_over_1400`: 1.29 vs my 0.81
- Opponent avg score: 11,758.73 vs my 11,461.90 (still ahead, but the gap shrank from ~976 to ~297)

I still find slightly more total words on average (34.30 vs 32.25), same pattern as dataset 1 (34.41 vs 32.91) — that part of the matchup hasn't changed.

## Single-dataset win-prediction model: another small-sample warning

Fit the same style of model as before (56% train / 44% test split, hand-rolled logistic regression, features: score, words, words≥800, words≥1400, words≥1800, avg word value) on just these 63 games: **test accuracy landed at 50.0%, identical to the majority-class baseline (50.0%)** — the model added zero predictive value on held-out data, a more extreme version of dataset 2's overfitting warning (86.2% train vs 59.1% test). With 63 games split into a 35/28 train/test, there simply isn't enough held-out data for a single split to be trustworthy. See the pooled cross-validated model (`word_hunt_win_logistic_cv_*`) for the number actually worth trusting, and the chat discussion of what a better-behaved model/metric would look like going forward.

## Files
- `word_hunt_stats3.csv` — raw per-game data (mine/opponent's score, words, tier counts incl. ≥1800 natively, won flag, notes)
- `word_hunt_data_enriched3.csv` — enriched with differentials, avg word value, hv_rate, opponent tier, game_idx
- `word_hunt_summary_overall3.csv`, `_by_result3.csv`, `_by_total_score_bucket3.csv` (8k-wide bands, shared edges with datasets 1/2)
- `word_hunt_winrate_by_word_value_bucket3.csv` (50-wide bands, matches datasets 1/2), `_sign3.csv`
- `word_hunt_insight_opponent_tiers3.csv`
- `word_hunt_insight_correlation_leaderboard3.csv`, `word_hunt_regression_words_vs_score3.csv`
- `word_hunt_win_logistic_coefficients3.csv` / `_metrics3.csv`, `word_hunt_win_regression_coefficients3.csv` / `_metrics3.csv`
- `word_hunt_dataset1_vs_dataset3_own_stats_comparison.csv` — my own raw stats, before vs. after, with significance tests
- Folded into `word_hunt_data_combined.csv` as `dataset=3`, `opponent_label="Opponent 1"`, `vs_opponent_2=0`

**Follow-up pass (same day):** widened the total-score buckets for datasets 1/2/3 from 4k-wide to 8k-wide bands (same shared edges across all three, for comparability), and fixed dataset 3's word-value buckets from an inconsistent 75-wide bin to the 50-wide bins already used by datasets 1/2. Added the own-stats (non-differential) comparison table above, and the per-game word-value differential histogram chart.

Two rows have a flagged lower-bound note (`opp_words_over_800 is a lower bound`): IMG_5906 and IMG_5943, where the opponent's visible word list never dropped below the 800-point tier before hitting the "(N more)" cutoff. Higher tiers (≥1400/1800/2200) are exact in both cases since the visible list's lowest shown value was itself ≥800, guaranteeing nothing hidden exceeds it.
