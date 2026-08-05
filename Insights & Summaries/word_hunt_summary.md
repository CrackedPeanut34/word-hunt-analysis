# Word Hunt — Summary

**Preface:** I wanted to actually analyze, learn from, and predict my Word Hunt games against my friends — not just eyeball the results screen after each match and move on. This is the story of what that turned up, told in the order it actually unfolded: first why I was losing, then what explained it, then what changed when a second opponent entered the picture, then what happened after I deliberately changed how I play, and finally an honest account of how well any of this can actually be predicted.

**How the game works, briefly** (full detail in the repo's `README.md`): Word Hunt is a head-to-head iMessage/GamePigeon game — both players get the same 4×4 letter grid and 80 seconds to find as many words as possible; higher score wins. Points depend only on word length: 3 letters = 100, 4 = 400, 5 = 800, 6 = 1,400, 7 = 1,800, 8 = 2,200 (+400 per letter beyond that). A 6-plus-letter word — referred to below as a "big word" or the "≥1400 tier" — is rare to find under time pressure but worth disproportionately more than a short one.

## 1. Getting the data in

167 of my own screenshots (later 230) of Word Hunt's post-game results screen, hand-read one at a time — score, word count, and how many words cleared each point tier, for both me and the opponent, never the actual words themselves. Three batches came in over time: 116 games against one friend ("Opponent 1"), then 51 against a second friend ("Opponent 2"), then a further 63 against Opponent 1 again, played after I'd deliberately changed how I approach the board (more on that in §6). Everything landed in a set of CSVs — raw extracted numbers, then an enriched layer adding derived stats like average point-value-per-word, then summary tables, then a handful of small regression models — all of it detailed in `word_hunt_process_log.md`.

## 2. Opponent 1 was better than me — and it wasn't about who found more words

Against Opponent 1, my record was 46–69–1: a 39.66% win rate. The surprising part is *why* he was ahead. I actually found slightly **more** total words per game than he did (34.41 vs. 32.91) — so this was never a story about him outworking me. What he had was quality: he found more words worth ≥800 points (4.99 vs. my 3.85/game) and dramatically more worth ≥1400 (1.17 vs. my 0.40/game, nearly 3× the rate). He wasn't playing a bigger volume game — he was finding *bigger* words.

## 3. The "how": word value turned out to be nearly the whole story

Once "average word value" (points per word found, not raw score) got isolated as its own number, it explained almost everything else. Split every game by whether my average word value beat the opponent's that game: I won **87.5%** of the time when it did (28 of 32 games), and only **21.4%** of the time when it didn't (18 of 84 games). Bucketed more finely, it's close to a step function — win rate climbs smoothly from 0% at the worst word-value deficits to 100% at the biggest edges. It's also the single strongest correlate of winning across every dataset in this project (r ≈ 0.6, replicated against a completely different opponent later).

A couple of honest caveats belong right here, not buried later:
- **Word value is partly derived from score itself** (it's score ÷ words), so this isn't a fully independent explanation of winning — it's close to restating the outcome in different units. It's still a genuinely useful lens (it separates "found more words" from "found better words," which raw score can't do), just not an outside cause.
- **This assumes the two players' words are roughly comparable in difficulty within a game** (same board, so in principle yes) — but that assumption has never actually been tested directly, e.g. by bucketing games by how word-value-rich the board was overall and checking whether the relationship holds the same way in generous boards as in stingy ones. That test is still on the to-do list.
- **The flip side is worth naming too:** my style — finding more total words even when they're not individually as valuable — does get me out of jams. There are real games in the lower word-value buckets that I still won on volume alone, wins I "shouldn't" have had on word quality terms. It's not a large effect, but it's there, and it's part of why word value is the dominant story rather than the *only* one.

## 4. Then Opponent 2 arrived, and the story replicated — with an easier matchup

Against Opponent 2, my record flipped to 56.86% (29–22). Every mechanic from §2–3 held up: word value differential was still the top correlate (r ≈ 0.61, nearly identical effect size to Opponent 1), and the win-rate-by-word-value-edge split looked almost the same shape (82.8% win rate when ahead, 22.7% when behind). What was different wasn't the *rules of the game* — it was how close the matchup was. My average word value against Opponent 2 was 300.96, hers was 302.38 — a gap of just 1.4 points, compared to a real, persistent ~44-point gap against Opponent 1. I live in a world of much closer word-value games against her, and that's the entire reason winning came easier: not a different game, an easier one.

## 5. What didn't change — and got backed up, not just repeated

A handful of findings from the Opponent 1 data held up essentially unchanged once Opponent 2's games came in, which is worth treating as confirmation rather than coincidence:
- **No session-order trend.** Neither dataset shows any drift in performance across the order games were played (r ≈ 0 both times) — no warm-up effect, no fatigue effect.
- **No volume/quality tradeoff.** Finding more words doesn't cost word quality — the correlation between word count and average word value is mildly *positive* in both datasets, not negative.
- **Streak behavior is nearly identical** — average streak length 2.04 games in both datasets, despite very different overall win rates.
- **I'm the more consistent player on word quality, in both datasets** — my average-word-value volatility is lower than either opponent's, even though my raw score is comparably volatile to theirs.
- **The opponent-strength-tier effect points the same direction against both opponents** (win rate drops as the opponent's word value that game rises) — just steeper against Opponent 1 (53.85%→20.51% weak-to-strong) than Opponent 2 (64.71%→47.06%).

## 6. Then I changed strategy — and it's a real, if partial, story

At some point I deliberately shifted from "find as many words as possible" toward "prioritize finding longer words." The next 63 games against Opponent 1 (dataset 3) let me test this properly, dataset 1 vs. dataset 3, with actual significance tests rather than eyeballing averages:

| | Before | After | Statistically real? |
|---|---|---|---|
| My words ≥1400/game | 0.40 | 0.81 (≈2×) | **Yes** (p=0.017) |
| My avg word value | 298.98 | 326.09 (+9%) | **Yes** (p=0.026) |
| My total words found | 34.41 | 34.30 (flat) | — (unchanged, as intended) |
| Win rate vs. Opponent 1 | 39.66% | 44.44% | Not yet (p=0.53) |
| Score differential | −975.9 | −296.8 | Not yet (p=0.21) |

The behavior change is real and it's exactly what was intended — nearly double the big-word rate, a genuine jump in average word value, with total word count staying flat (I didn't just find more of everything). The downstream payoff — win rate, score gap — moved the right direction too, but at only 63 games neither is statistically distinguishable from noise yet. One honest confound: the boards in dataset 3 may simply have run a bit richer on average (total combined word value trended up too, p=0.088), so part of the shift isn't cleanly attributable to me alone. Verdict: **the strategy change measurably worked in the way it was supposed to (finding better words without sacrificing volume); whether it's paying off in wins is still an open question, not a closed one.**

## 7. Models: partly figured out, partly still not

Predicting wins turned out to be a genuinely harder problem than finding what correlates with them, and the honest state of it is a mix of real progress and real limits.

**What went wrong first:** the first multi-feature models (score, word count, tier counts, average word value, all together) suffered from a structural problem — score is just words × avg-word-value by definition, so putting all three in one regression let the fit swing unstably between them, occasionally producing backwards predictions (more score sometimes predicting a *lower* win chance). Dropping score from that feature list fixed the instability without costing meaningful accuracy.

**What happened when the strategy-change data got folded in:** re-fitting that same multi-feature model on the full 230-game history actually made it *worse* under honest cross-validation (63.6–64.4% accuracy, down from 68.26/66.47% on the smaller, pre-strategy-change dataset) — more data didn't help this model, because the strategy shift changed the relationship between the features and winning.

**What actually works best right now, on the full data:** a far simpler model — just my score and which opponent I'm facing — cross-validates at **~67–68% accuracy**, reliably beating a ~58% baseline by about 10 points, confirmed stable across two independently-shuffled 5-fold splits. Adding the ≥800 word-value rate as a third feature was tested and made it *worse*, not better, on two separate fold assignments — likely the same score-redundancy problem in a milder form. So: not a solved problem, but not "no idea" either. The honest state is "a simple, real, modest edge — roughly 10 points better than a coin flip weighted by base rates — with several more complex attempts tried and found not to help." The front-end predictor tool runs on this simpler model today, and shows a percentile breakdown against personal history for everything else that was collected but didn't make the final model.

## Where the rest of the detail lives

- `word_hunt_process_log.md` — every extraction pass, every CSV, every model, with the actual Python behind it
- `word_hunt_insights_combined.md` — every individual finding from the whole project, in one place, simple to complex
- The full data (`Stats/`, `Regressions/`) — nothing above is asserted without a CSV backing it up
