# Word Hunt Analysis

I wanted to actually analyze, learn from, and predict my Word Hunt games against my friends — not just eyeball the results screen after each match and move on. This repo is that project: 116 games against one opponent, 51 against a second, then a third batch of 63 more games against the first opponent again after I deliberately changed strategy — 230 games total, going from raw screenshot extraction all the way to regression modeling and a head-to-head skill comparison.

**Start here:** [`Insights & Summaries/word_hunt_insights_combined.md`](Insights%20%26%20Summaries/word_hunt_insights_combined.md) — the sharpest finding in the whole project: it isn't "word value" broadly that decides these games, it's specifically the rare ≥1400-point words. Remove that tier from the picture and I'm net ahead across my entire history.

## How Word Hunt works

Word Hunt is a head-to-head word game played over iMessage via the GamePigeon extension. Both players get the same randomly-generated 4×4 grid of letters and a fixed **80-second timer**. You form words by dragging across adjacent letters (including diagonals) — no letter can be reused within the same word, and the shortest valid word is 3 letters. Both players search the identical board at the same time, entirely independently; whoever ends with the higher score wins.

Scoring is **based purely on word length**, not letter rarity (unlike Scrabble — a Q is worth the same as an E here):

| Word length | Points |
|---|---|
| 3 letters | 100 |
| 4 letters | 400 |
| 5 letters | 800 |
| 6 letters | 1,400 |
| 7 letters | 1,800 |
| 8 letters | 2,200 |
| 9+ letters | +400 per additional letter |

That scoring curve is the reason this whole project's headline finding exists: 6+ letter words (the ≥1400 tier) are worth disproportionately more than short ones, but they're also much rarer to spot on a 4×4 board under time pressure — so a player who's simply better at finding *long* words gets rewarded heavily, independent of how many total words either side finds. Every "word value," "tier," and "≥1400" reference throughout this repo's docs and CSVs is describing exactly this scoring table.

*(Sources: [thewordfinder.com](https://www.thewordfinder.com/word-hunt-solver/), [Word Hunt: Cracking the Code (Medium)](https://medium.com/@abhay.khanna_37314/word-hunt-cracking-the-code-9344188b1edb), [dcode.fr Word Hunt solver](https://www.dcode.fr/word-hunt-game-pigeon-solver).)*

## What's here

Raw screenshots aren't included in this repo (they're personal photos, not analysis output) — only the extracted data and everything built on top of it.

### `Insights & Summaries/`
Narrative write-ups, in the order the project unfolded:
- `word_hunt_summary.md` — core stats, splits, and regression for dataset 1
- `word_hunt_insights.md` — every finding from dataset 1
- `word_hunt_insights2.md` — dataset 2, opponent 2, with dataset-1 comparisons throughout
- `word_hunt_insights_combined.md` — cross-dataset work: the points-origin decomposition (the headline finding), a fair three-way skill comparison, and pooled cross-validated win prediction
- `word_hunt_insights3.md` — dataset 3: 63 more games against opponent 1, played after deliberately shifting strategy toward finding longer words, with a statistically-tested before/after comparison against dataset 1
- `word_hunt_process_log.md`, `word_hunt_process_log2.md`, `word_hunt_process_log3.md`, `word_hunt_process_log_combined.md` — methodology: how each stage was built, what was double-checked, and mistakes caught along the way (a bug in a hand-rolled logistic regression, a flawed apples-to-oranges comparison that got corrected, etc.)

### `Stats/`
The working datasets and summary tables (all pandas-ready CSVs):
- `word_hunt_stats.csv` / `word_hunt_stats2.csv` — raw extracted per-game data
- `word_hunt_data_enriched.csv` / `_enriched2.csv` / `_combined.csv` — enriched with derived metrics (score/word differentials, average word value, high-value word rate, adjusted points, etc.)
- `word_hunt_summary_overall*.csv`, `_by_result*.csv`, `_by_total_score_bucket*.csv` — grouped averages
- `word_hunt_summary_by_player.csv` — the fair three-way comparison (me vs. each opponent, matchup by matchup)
- `word_hunt_winrate_by_word_value_*.csv` — win rate split by word-value differential (the single best predictor found)
- `word_hunt_points_origin_decomposition.csv` / `_per_game.csv` — the tier-by-tier point breakdown behind the headline finding
- `word_hunt_insight_correlation_leaderboard*.csv`, `_opponent_tiers*.csv` — correlation ranking and opponent-strength segmentation

### `Regressions/`
Model fits and their evaluation:
- `word_hunt_regression_words_vs_score*.csv` — simple words→score OLS fits
- `word_hunt_win_regression_*` — linear regression predicting win/loss (flagged as the wrong tool for a binary target, kept for comparison)
- `word_hunt_win_logistic_*` — logistic regression predicting win/loss, including a hand-rolled Newton-Raphson implementation (no sklearn available in the original environment)
- `word_hunt_win_logistic_cv_*`, `word_hunt_win_regression_cv_*` — the pooled, 5-fold cross-validated versions of the above, evaluated on all 167 games at once instead of a single train/test split

## Method notes worth knowing before trusting a number here

- Word point values were extracted by reading each screenshot's results screen (only the numeric point value next to each word, never the word text itself). A handful of games have `_over_800` counts flagged as **lower bounds** rather than exact — the game's results screen only shows the top ~15 words per column, and a few games had 15+ words all worth ≥800 with more hidden off-screen.
- Two win-prediction models exist per dataset stage: **linear regression thresholded at 0.5** (not the statistically correct tool for a binary target, but requested and kept for comparison) and **logistic regression** (the correct tool). Prefer the logistic numbers.
- Sample size is genuinely small for classification modeling — 51 games for the second opponent gave a models that badly overfit under a single train/test split (86% train vs. 59% test accuracy). Cross-validating the pooled 167-game dataset fixed this and is the most trustworthy win-prediction result in the repo.
