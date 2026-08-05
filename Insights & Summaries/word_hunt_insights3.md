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
- `word_hunt_summary_overall3.csv`, `_by_result3.csv`, `_by_total_score_bucket3.csv`
- `word_hunt_winrate_by_word_value_bucket3.csv`, `_sign3.csv`
- `word_hunt_insight_opponent_tiers3.csv`
- `word_hunt_insight_correlation_leaderboard3.csv`, `word_hunt_regression_words_vs_score3.csv`
- `word_hunt_win_logistic_coefficients3.csv` / `_metrics3.csv`, `word_hunt_win_regression_coefficients3.csv` / `_metrics3.csv`
- Folded into `word_hunt_data_combined.csv` as `dataset=3`, `opponent_label="Opponent 1"`, `vs_opponent_2=0`

Two rows have a flagged lower-bound note (`opp_words_over_800 is a lower bound`): IMG_5906 and IMG_5943, where the opponent's visible word list never dropped below the 800-point tier before hitting the "(N more)" cutoff. Higher tiers (≥1400/1800/2200) are exact in both cases since the visible list's lowest shown value was itself ≥800, guaranteeing nothing hidden exceeds it.
