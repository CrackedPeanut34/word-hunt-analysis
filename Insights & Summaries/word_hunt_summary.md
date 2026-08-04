# Word Hunt Stats Summary

Source: `word_hunt_data_enriched.csv` (116 games, IMG_5712–IMG_5827)

**The throughline of this whole project turned out to be one derived stat: `avg_word_value` (score ÷ words) — points per word found, not raw score and not raw word count.** Every later analysis (correlation ranking, win-rate splits, the win-prediction models, and the second dataset comparison) kept landing back on this same number as the strongest single signal for winning. See `word_hunt_insights.md` for the full case, and the [Word Value Differential Distribution](https://claude.ai/code/artifact/6034b8ee-69da-403b-9e31-eeee4251e05a) chart for the picture of it.

**2026-08-04 correction note:** a `words_over_1800` tier (previously silently folded into ≥1400) was audited in and backfilled across every CSV. It doesn't change anything below. Full audit in `word_hunt_process_log_combined.md` § 0.

## Record
- **Win %:** 39.66% (40.00% excluding the one draw)
- **Record:** 46 W – 69 L – 1 D

## Overall averages (per game)

| Metric | Mine | Opponent | Total |
|---|---|---|---|
| Score | 10,506.9 | 11,482.8 | 21,989.7 |
| Words | 34.41 | 32.91 | 67.33 |
| Avg word value (score ÷ words) | 298.98 | 343.11 | 321.07 |
| Words ≥800 | 3.85 | 4.99 | — |
| Words ≥1400 | 0.40 | 1.17 | — |
| Words ≥2200 | 0.00 | 0.03 | — |

## Average differentials (mine − opponent, per game)

| Stat | Avg Diff |
|---|---|
| Score | −975.86 |
| Words | +1.50 |
| Avg word value | −44.13 |
| Words ≥800 | −1.14 |
| Words ≥1400 | −0.78 |
| Words ≥2200 | −0.03 |

Takeaway: you typically find *more total words* than opponents but *fewer high-value ones*, and each of your words is worth ~44 points less on average — that gap in word value (driven by the ≥800/≥1400 tiers) is basically the whole story behind the average score deficit. The [distribution of this gap](https://claude.ai/code/artifact/6034b8ee-69da-403b-9e31-eeee4251e05a) across all 116 games shows it's not a few outlier blowouts — most individual games sit on the negative side.

## Averages grouped by result

| Stat | Not Won (n=70) | Won (n=46) |
|---|---|---|
| My score | 9,241.4 | 12,432.6 |
| My words | 31.8 | 38.4 |
| My avg word value | 284.3 | 321.3 |
| My words ≥800 | 3.11 | 4.98 |
| My words ≥1400 | 0.26 | 0.61 |
| My words ≥2200 | 0.00 | 0.00 |
| Opp score | 12,324.3 | 10,202.2 |
| Opp words | 33.2 | 32.4 |
| Opp avg word value | 365.3 | 309.4 |
| Opp words ≥800 | 5.77 | 3.80 |
| Opp words ≥1400 | 1.56 | 0.59 |
| Opp words ≥2200 | 0.06 | 0.00 |
| Total score | 21,565.7 | 22,634.8 |
| Total words | 65.06 | 70.78 |
| **Score diff** | **−3,082.9** | **+2,230.4** |
| Words diff | −1.40 | +5.91 |
| Avg word value diff | −80.97 | +11.93 |
| Words ≥800 diff | −2.66 | +1.17 |
| Words ≥1400 diff | −1.30 | +0.02 |
| Words ≥2200 diff | −0.06 | 0.00 |

Wins aren't just "more words" — your avg word value flips from *below* the opponent's when you lose (284 vs 365) to *above* it when you win (321 vs 309). Winning tracks word quality at least as much as word count.

## Averages grouped by total-score bucket

Buckets match the histogram (combined total_score = mine + opponent's, per game).

| Bucket | Games | Win % | My Score | Opp Score | Total Score | My Words | Opp Words | Total Words | My Avg Value | Opp Avg Value | Score Diff | Words Diff | Avg Value Diff |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 4k–8k | 1 | 0.00% | 2,400.0 | 3,100.0 | 5,500.0 | 15.0 | 16.0 | 31.0 | 160.00 | 193.75 | −700.0 | −1.00 | −33.75 |
| 8k–12k | 9 | 44.44% | 5,111.1 | 5,355.6 | 10,466.7 | 24.4 | 24.3 | 48.8 | 208.67 | 222.58 | −244.4 | +0.11 | −13.91 |
| 12k–16k | 16 | 37.50% | 7,006.3 | 7,193.8 | 14,200.0 | 29.0 | 26.2 | 55.2 | 242.17 | 274.59 | −187.5 | +2.81 | −32.42 |
| 16k–20k | 28 | 32.14% | 8,553.6 | 9,750.0 | 18,303.6 | 33.5 | 32.5 | 66.0 | 260.40 | 307.32 | −1,196.4 | +0.96 | −46.92 |
| 20k–24k | 24 | 41.67% | 10,658.3 | 11,529.2 | 22,187.5 | 34.7 | 33.2 | 67.9 | 309.78 | 349.37 | −870.8 | +1.50 | −39.59 |
| 24k–28k | 19 | 42.11% | 12,110.5 | 14,021.1 | 26,131.6 | 35.5 | 37.6 | 73.1 | 343.81 | 376.43 | −1,910.5 | −2.11 | −32.62 |
| 28k–32k | 5 | 40.00% | 14,440.0 | 16,240.0 | 30,680.0 | 41.0 | 37.8 | 78.8 | 356.20 | 430.50 | −1,800.0 | +3.20 | −74.30 |
| 32k–36k | 6 | 66.67% | 16,900.0 | 17,466.7 | 34,366.7 | 43.0 | 38.2 | 81.2 | 398.80 | 472.85 | −566.7 | +4.83 | −74.05 |
| 36k–40k | 5 | 40.00% | 19,060.0 | 20,140.0 | 39,200.0 | 46.0 | 41.8 | 87.8 | 416.00 | 487.42 | −1,080.0 | +4.20 | −71.43 |
| 40k–44k | 3 | 33.33% | 21,333.3 | 20,933.3 | 42,266.7 | 51.7 | 38.3 | 90.0 | 418.78 | 547.21 | +400.0 | +13.33 | −128.43 |

Words≥800/≥1400/≥2200 and their diffs per bucket are in the CSV (omitted above for table width).

Notes:
- Win % doesn't trend cleanly with total score — 32k–36k is your best bucket (66.7%) but it's only 6 games.
- Avg word value diff is negative in **every single bucket** — even the 40k–44k bucket where you win the score race overall (+400 diff) does so on raw word *count* (you find 51.7 vs their 38.3), not word quality — opponents' avg word value stays higher than yours across the entire score range, and the gap actually widens at the top (−128 diff in 40k–44k, the worst of any bucket).

## Linear regression: words → score

Simple OLS, `score = slope * words + intercept`, n=116 games per fit.

| Series | Equation | R² | r | p-value |
|---|---|---|---|---|
| Mine | score = 412.44 × words − 3,686.71 | 0.653 | 0.808 | 5.7e-28 |
| Opponent | score = 468.59 × words − 3,940.46 | 0.480 | 0.692 | 7.3e-18 |
| Total | score = 455.22 × words − 8,659.11 | 0.585 | 0.765 | 1.8e-23 |

Takeaways:
- Word count explains score well for you (R²=0.65) — each additional word is worth ~412 points on average.
- It explains opponent score less well (R²=0.48) — a wider spread, consistent with them finding more high-value (800+/1400+) words per game so their score depends more on word *quality* than *quantity* relative to you (matches the avg-word-value gap above: opponents average ~343 pts/word vs your ~299).
- All three fits are highly significant (p ≪ 0.001), so the relationships aren't noise, just varying in how tightly score tracks word count.

## Files
- `word_hunt_stats.csv` — original per-game raw data (my/opp score, words, tier counts, won flag, notes)
- `word_hunt_data_enriched.csv` — full working dataset: adds `total_score`, `total_words`, `my_avg_word_value`, `opp_avg_word_value`, `total_avg_word_value`, and 6 differential columns (`score_diff`, `words_diff`, `words_over_800_diff`, `words_over_1400_diff`, `words_over_2200_diff`, `avg_word_value_diff`)
- `word_hunt_summary_overall.csv` — single-row overall averages/win rate, incl. total_words and avg_word_value (pandas-ready)
- `word_hunt_summary_by_result.csv` — averages grouped by won/not-won, incl. total_words, avg_word_value, and all diffs (pandas-ready)
- `word_hunt_summary_by_total_score_bucket.csv` — averages grouped by total-score histogram bucket, incl. total_words, avg_word_value, and all diffs (pandas-ready)
- `word_hunt_regression_words_vs_score.csv` — linear regression stats (slope, intercept, r, R², p-value) for mine/opponent/total
- `word_hunt_summary.md` — this file
