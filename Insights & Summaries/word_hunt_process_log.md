# Word Hunt Project — Process Log

Everything done in this project, start to finish, in the order it happened. This is the "how it was built" document — for what was actually *learned*, see `word_hunt_insights_combined.md`; for the narrative version of the story, see `word_hunt_summary.md`.

All extraction was done by reading screenshots directly (an LLM reading images, not OCR/computer vision code) — the "code" in this project is entirely pandas/numpy, used to turn hand-extracted numbers into CSVs and to fit the regression models by hand (no scikit-learn or statsmodels available in the working environment, so logistic regression was implemented from scratch via Newton-Raphson/IRLS).

## 1. Dataset 1 — extraction and base pipeline (116 games, vs. Opponent 1)

**Source:** 116 iPhone screenshots of Word Hunt's post-game results screen (IMG_5712–IMG_5827). Each shows a "YOU WON!"/"YOU LOST!" banner, then two columns ("You" and the opponent) each with a WORDS count, a SCORE, and a list of found words paired with point values, sorted descending and truncated after ~15 entries with "(N more)".

**Extraction rules** (established with the user before scaling up, validated on 5 sample images first):
- Only capture the *point values* next to words — never read or record the actual words.
- Tier counts are cumulative: ≥800 includes 1400s/1800s/2200s; ≥1400 includes 1800s/2200s.
- Capture both the user's and the opponent's stats per game.
- `my_won` = 1 only for an exact "YOU WON!" banner.

**Process:** split the 116 screenshots into batches, ran them through parallel background extraction passes, then manually re-verified every flagged edge case directly against the source image (truncated lists where the visible tier never dropped low enough to guarantee the hidden words were smaller — a "lower bound" flag; one draw game; unexpected 1800-point tier sightings). Assembled into `word_hunt_stats.csv`, sanity-checked with pandas (no duplicates, no nulls, tier counts internally consistent, `my_won` agrees with the sign of the score comparison).

**Enrichment** (`word_hunt_data_enriched.csv`) added, in the order features were requested over the project's life:

```python
e['total_score']   = e['my_score'] + e['opp_score']
e['score_diff']    = e['my_score'] - e['opp_score']
e['my_avg_word_value']  = e['my_score'] / e['my_words']          # points per word found
e['opp_avg_word_value'] = e['opp_score'] / e['opp_words']
e['avg_word_value_diff'] = e['my_avg_word_value'] - e['opp_avg_word_value']
e['my_hv_rate']  = e['my_words_over_800'] / e['my_words']         # share of words worth >=800
e['opp_hv_rate'] = e['opp_words_over_800'] / e['opp_words']
```

Plus a tercile-based `opp_tier` (weak/mid/strong, by the opponent's avg word value that game) and `game_idx` (1–116, chronological, for streak/trend checks).

**Summary tables built:** `word_hunt_summary_overall.csv`, `_by_result.csv`, `_by_total_score_bucket.csv` (4k-wide bands originally, later widened — see §12), `word_hunt_regression_words_vs_score.csv` (simple OLS via `scipy.stats.linregress`), `word_hunt_winrate_by_word_value_sign.csv` / `_bucket.csv` (50-point-wide bands), `word_hunt_insight_correlation_leaderboard.csv` (Pearson correlation of every stat against `my_won`, via `scipy.stats.pearsonr`), `word_hunt_insight_opponent_tiers.csv`.

**Win-prediction model (v1):** feature set `my_score, my_words, my_words_over_800, my_words_over_1400, my_avg_word_value` (own-side stats only, `my_words_over_2200` dropped for being constant), 65 train / 51 test split, seed 42. Fit both a linear probability model (OLS thresholded at 0.5) and a hand-rolled logistic regression:

```python
def sigmoid(z): return 1 / (1 + np.exp(-z))

def fit_logistic(X, y, lam=1e-4):
    mu, sigma = X.mean(0), X.std(0)
    Xs = (X - mu) / sigma
    Xd = np.column_stack([np.ones(len(Xs)), Xs])
    beta = np.zeros(Xd.shape[1])
    for _ in range(100):
        p = sigmoid(Xd @ beta)
        grad = Xd.T @ (y - p) - lam * np.r_[0, beta[1:]]
        H = -(Xd.T * (p * (1 - p))) @ Xd - lam * np.diag([0] + [1]*(len(beta)-1))
        beta = beta - np.linalg.solve(H, grad)     # Newton-Raphson step
    return beta, mu, sigma
```

First attempt had the step sign backwards (`beta + H^-1 grad` instead of `beta - H^-1 grad`), which diverged the coefficients to ~1e18 before it was caught and fixed. Corrected version converged in 6 iterations: 74.51% test accuracy (baseline 62.75%).

## 2. Dataset 2 — extraction and pipeline (51 games, vs. Opponent 2)

Same extraction methodology, same screen layout, 51 new screenshots (IMG_5828–IMG_5878), labeled `opp2_*` throughout instead of `opp_*`. One flagged lower-bound row (IMG_5851), two 1800-point-tier sightings that turned out fully countable. Same enrichment, same summary tables (suffix `2`), same bucket edges as dataset 1 for direct comparability.

**Win-prediction model:** same feature set, but only 51 games meant a 29/22 train/test split. This became an unplanned but useful demonstration of small-sample overfitting: 86.2% train accuracy vs. 59.1% test accuracy (dataset 1's model had nearly identical train/test accuracy — 73.85% vs. 74.51%). The events-per-predictor ratio at 29 training games was ~4.75, well under the standard 10–20 guideline.

## 3. Correction: the missing 1800-point tier (2026-08-04)

The original extraction tracked cumulative thresholds (≥800/≥1400/≥2200) with no dedicated ≥1800 column, so an 1800-point word was silently indistinguishable from a 1400. **Audit method:** only games where `words_over_1400 > 0` on either side could possibly contain an 1800 — that filtered 167 games down to 87 needing re-checking. Split into 6 parallel batches, each re-reading its assigned screenshots and reporting the exact 1400/1800/2200 split per side, cross-checked against the already-known ≥1400 and ≥2200 totals (every batch's reported sum matched exactly — zero discrepancies across all 87 games). Manually spot-checked a sample of newly-found 1800s directly against source images.

**Result:** 17 of 167 games had at least one ≥1800 word. Added `words_over_1800` (mine and opponent's) plus `words_over_1800_diff` to every base, enriched, and summary CSV in the project at the time. Recomputed the points-origin decomposition (§7) exactly instead of approximating every ≥1400 word as worth 1400 flat. Confirmed via re-running the cross-validated win-prediction model that this changed nothing about accuracy (same features, same values) and that adding the new tier as its own model feature didn't help (too rare — 2 occurrences in 167 games on the user's own side) — the front-end predictor tool needed no changes at that point.

## 4. Combined dataset and fair three-way comparison

Concatenated `word_hunt_data_enriched.csv` and `_enriched2.csv` into `word_hunt_data_combined.csv`, unifying `opp_*`/`opp2_*` into generic `opponent_*` columns and adding `dataset` (1/2), `opponent_label`, `combined_game_idx`, `dataset_game_idx`, a binary `vs_opponent_2` (for later model use), and normalized comparison metrics that cancel out board-to-board richness:

```python
e['my_score_share']       = e['my_score'] / e['total_score']
e['my_adjusted_points']   = e['my_score'] - e['total_score'] / 2   # score relative to that game's own midpoint
```

**Why "adjusted points":** an earlier comparison had mistakenly compared the user's *blended* 167-game average directly against Opponent 2's *isolated* 51-game average — an apples-to-oranges framing that made performance look closer than it was. Rebuilt `word_hunt_summary_by_player.csv` as matchup-specific rows (Me vs. Opponent 1 / Opponent 1 / Me vs. Opponent 2 / Opponent 2 / Me—Overall-blended-for-reference-only) with adjusted points and score share added to make a fair, board-difficulty-normalized comparison possible.

## 5. Points-origin decomposition

The project's sharpest single finding: broke each game's score into three point bands (≤400, exactly 800, ≥1400 — computed *exactly* once the 1800/2200 splits were known, not approximated) and summed each band's contribution to the score gap:

```python
for band, is_in_band in [('le400', lambda t: t <= 400), ('eq800', lambda t: t == 800), ('ge1400', lambda t: t >= 1400)]:
    # band_points = count_in_band * point_value, summed per side, differenced, summed across games
    ...
```

This is an exact algebraic identity — the three bands sum to `score_diff` with zero reconstruction error by construction. Result: the ≥1400 tier alone cost −162,600 points across all 167 games at the time, more than double the entire net deficit (−80,600) — in the ≤400/=800 bands combined, the user was actually +82,000 ahead. See `word_hunt_points_origin_decomposition.csv` / `_per_game.csv`.

## 6. Pooled 5-fold cross-validation

Replaced the two separate single-split models with one pooled model across all 167 games, evaluated with 5-fold cross-validation (every game tested exactly once) instead of one lucky/unlucky split, folds stratified jointly by (win/loss, opponent) for realistic composition:

```python
def make_stratified_folds(df, seed, k=5):
    strat_key = df['my_won'].astype(str) + '_' + df['vs_opponent_2'].astype(str)
    rng = np.random.RandomState(seed)
    fold_assign = np.full(len(df), -1)
    for key in strat_key.unique():
        idxs = df.index[strat_key == key].to_numpy()
        rng.shuffle(idxs)
        for i, ix in enumerate(idxs):
            fold_assign[df.index.get_loc(ix)] = i % k
    return fold_assign
```

Feature set: `my_score, my_words, my_words_over_800, my_words_over_1400, my_avg_word_value`, plus a version adding binary `vs_opponent_2`. Result: 67.07% out-of-fold accuracy without the opponent feature, **70.66% with it** — knowing who you're facing is worth +3.6 points on its own, against a 55.09% majority-class baseline.

## 7. Front-end predictor tool, and the score/words/avg-value multicollinearity fix

Built a published web calculator (Claude Artifact) around the pooled logistic model. Testing it surfaced a real bug: holding score fixed and sweeping word count produced **non-monotonic, occasionally backwards predictions** — because `my_score`, `my_words`, and `my_avg_word_value` aren't independent (`score = words × avg_word_value` by construction), and including all three as separate regression inputs let the fit distribute credit between them unstably. **Fix:** refit dropping `my_score` entirely as a model input (kept in the UI, since that's what's visible on screen — just not fed to the math): `my_words, my_words_over_800, my_words_over_1400, my_avg_word_value` (+ opponent flag). Accuracy was a wash (67.66%/69.46% vs. 67.07%/70.66% — within noise), but coefficients became correctly signed and predictions monotonic.

Also fixed a weaker spot in the "someone new" opponent case: originally interpolated `vsOpp2 = 0.5` as a placeholder with no data support; replaced with a properly, separately-fitted opponent-blind model instead of interpolating a single coefficient.

**Follow-up: added the 1800-point tier as a model input** at explicit request, once it existed as a column (§3). Fitted coefficient came out **negative** (−0.87 to −1.27 raw units) — not a believable causal effect, just noise from only 2 of 167 games having one on the user's side. Added it to the tool as-fitted with a visible caveat rather than silently overriding or flooring the coefficient, consistent with this project's general rule: fix structural bugs (like the multicollinearity above), but disclose — don't paper over — limitations that are just genuine data sparsity.

## 8. Dataset 3 — extraction and pipeline (63 games, vs. Opponent 1, after a strategy change)

New batch (IMG_5888–5950), against Opponent 1 again, played after the user deliberately shifted strategy toward finding *longer* words rather than maximizing total word count. Unlike datasets 1/2, **the ≥1800 tier was tracked natively from the first extraction pass** — no retroactive audit needed. Two rows flagged with the "saturated list" lower-bound edge case (IMG_5906, IMG_5943). Same enrichment/summary pipeline as before, `opp_tier` computed via terciles *within dataset 3 only* (labeled `(d3)` to keep separate from dataset 1's opponent-1 tiers, since board difficulty across the two time periods isn't assumed identical).

**Single-dataset win model:** included `my_words_over_1800` as a feature from the start (unlike datasets 1/2, which never got this retrofitted). Test accuracy landed at exactly the majority-class baseline (50.0%) — a stark demonstration of the same small-sample overfitting risk as dataset 2, at an even smaller test set (28 games).

**Before/after comparison (dataset 1 vs. dataset 3), done properly with significance tests** (`scipy.stats.ttest_ind`, Welch's, unequal variance) rather than just eyeballing the averages:

```python
t, p = stats.ttest_ind(dataset1[col], dataset3[col], equal_var=False)
```

Findings: `my_words_over_1400` roughly doubled (0.40→0.81/game, p=0.017) and `my_avg_word_value` rose ~9% (299.0→326.1, p=0.026) — both statistically real. Win rate (39.66%→44.44%) and score differential (−975.9→−296.8) moved the right direction but weren't statistically distinguishable from noise at this sample size (p=0.53, p=0.21). A trend in board richness (`total_avg_word_value` up 321→342, p=0.088) was flagged as a possible confound.

## 9. Bucket-width fixes and additional views (2026-08-05)

Widened `_by_total_score_bucket` tables from 4k-wide to 8k-wide bands (shared edges across all three datasets), and fixed dataset 3's word-value buckets from an inconsistent 75-wide bin back to the 50-wide bins datasets 1/2 already used. Added two new combined-perspective bucket tables: **Opponent 1, all games** (datasets 1+3 pooled, 179 games) and **all opponents combined** (all 230 games) — both bucketed by total score, plus a third bucketed by *just the user's own score* (since that showed a much cleaner, near-monotonic relationship with win rate than total-score bucketing, which gets diluted by board-richness variation). Built a per-game word-value-differential histogram (all three datasets, win/loss colored) as a published chart.

## 10. Engineered rate metrics, and closing a systemic propagation gap

Added `words_over_1400_rate` (≥1400 words ÷ total words) alongside the pre-existing `hv_rate` (≥800 rate) to every enriched/combined CSV:

```python
e['my_words_over_1400_rate'] = e['my_words_over_1400'] / e['my_words']
```

While wiring this in, audited where these rate metrics had and hadn't propagated downstream — the same category of gap as the missing-1800-tier problem in §3: a metric existed at the enriched-data layer but had silently never made it into `summary_overall*`, `_by_result*`, `_by_total_score_bucket*` (all variants), `summary_by_player`, `insight_opponent_tiers*`, the word-value bucket/sign tables, or the correlation leaderboards. Backfilled everywhere. The opponent-tier and word-value-bucket/sign tables additionally never had a full tier breakdown at all (not just the rate — the raw ≥800/1400/1800/2200 averages too), so that was added at the same time for consistency.

**New finding that fell out of this:** `hv_rate_diff` ranks **#3–4 by correlation with winning across all three datasets** — a real signal, invisible until now purely because it had never been in the leaderboard's candidate stat list. Also built `word_hunt_winrate_by_hv_rate_sign*.csv`: win rate when the user's ≥800 rate beats the opponent's is 68.3% / 82.6% / 90.5% across the three datasets respectively, versus 21.4% / 35.7% / 21.4% when it doesn't — one of the sharpest splits in the whole project.

## 11. Simple score+opponent model replaces the multi-feature model

Refitting the original multi-feature model (words, tier counts, avg word value, opponent) on the full 230-game combined dataset (it had only ever been fit through dataset 2) made cross-validated accuracy **worse**, not better: 63.6%/64.4% (down from 68.26%/66.47% on 167 games) — dataset 3's strategy-change data destabilized the fit, and one coefficient (`my_words_over_1400`, opponent-blind version) flipped sign.

In parallel, a much simpler two-feature model — just `my_score` and a binary `vs_opponent_2` — was tested: a single 70/30 split (69.6% test accuracy vs. 58.0% baseline), then properly 5-fold cross-validated on two independently-seeded fold assignments (68.3% and 67.0% mean fold accuracy, averaging ~67.6% — deliberately checked on a *second*, differently-seeded fold split specifically to make sure the first CV run's number wasn't a fluke of one fold partition). Also tested three single-split ratios (75/25, 80/20, 85/15) with score alone as the only feature — the 85/15 split's tiny 34-game test set happened to land exactly at its own majority-class baseline, a clean illustration of how a shrinking holdout set gets noisier even when the underlying model hasn't changed.

Tested whether adding `hv_rate` as a third feature improved the score+opponent model, using a **second, differently-seeded 5-fold split** specifically so the comparison wasn't an artifact of one fold assignment, plus paired same-fold checks in both directions for full rigor. Result: adding it consistently hurt (~65.2–65.3% vs. ~67–68%), on both fold assignments — likely the same flavor of score-redundancy problem as the original multicollinearity bug, just milder, since `hv_rate` and `score` share the same underlying words/tiers.

**Conclusion: the simple 2-feature model is the better one on the current data.** Switched the front-end predictor tool over to it.

## 12. Predictor tool v2 — new model, plus a percentile-against-history section

Updated the predictor: the win-probability gauge now runs on the score+opponent model (with a score-only variant for an unknown/new opponent, since there's no opponent-identity term to condition on). The word-count and tier-count inputs stay in the UI exactly as before, but no longer feed the probability — instead they now power a new section showing, for every input stat plus three engineered metrics (avg word value, ≥800 rate, ≥1400 "big-word" rate), what percentile that value falls at against all 230 of the user's own recorded games:

```javascript
function percentileOf(sortedHistoricalArray, v) {
  // binary search: fraction of historical games with value <= v
  var lo = 0, hi = sortedHistoricalArray.length;
  while (lo < hi) {
    var mid = (lo + hi) >> 1;
    if (sortedHistoricalArray[mid] <= v) lo = mid + 1; else hi = mid;
  }
  return Math.round((lo / sortedHistoricalArray.length) * 100);
}
```

The 230-value sorted arrays for each stat are embedded directly in the page (trivial size, no external data fetch needed). Percentiles are computed against the full personal history regardless of which opponent is selected in the tool, since word-finding performance is being compared to the player's own past, not adjusted for matchup.

## 13. Documentation consolidation (2026-08-05)

At this point the docs had sprawled to 8 files (`word_hunt_summary.md`, `insights.md`, `insights2.md`, `insights3.md`, `insights_combined.md`, `process_log.md`, `process_log2.md`, `process_log3.md`, `process_log_combined.md`). Consolidated down to exactly three: this process log (merging all four prior process logs into one chronological account), `word_hunt_summary.md` (a from-scratch narrative rewrite following a specific requested story arc — how the data came in, why Opponent 1 was better, what turned out to explain it, what changed with Opponent 2, what held up, the strategy change and its result, and an honest state of the modeling work), and `word_hunt_insights_combined.md` (every finding from the whole project in one place, ordered simple to complex, superseding the four separate insights files). The GitHub repo was pruned to just the updated `README.md` plus these three files for anything under `Insights & Summaries/` — all the CSVs stay as they were.
