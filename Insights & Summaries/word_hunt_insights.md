# Word Hunt — Everything Learned

116 games. 46 wins, 69 losses, 1 draw. Win rate 39.66% (40.0% if you don't count the draw).

## The headline finding

**Word value matters far more than word count.** When your average point-value-per-word beats your opponent's, you win 87.5% of the time (28 of 32 games). When it's worse, you win only 21.4% of the time (18 of 84 games). Bucketed into finer bands, it's almost a clean step function — win rate climbs from 0% to 100% as your word-value edge crosses zero:

- Word value diff −300 to −150: **0% win rate** (11 games, 0 wins)
- −150 to −50: ~10% win rate (39 games, 4 wins)
- −50 to 0: 42.4% win rate (33 games, 14 wins)
- 0 to 50: 80% win rate (20 games, 16 wins)
- 50+: **100% win rate** (12 games, 12 wins)

The striking part: in the worst buckets, you're still often finding *more total words* than your opponent (word-count differential stays positive, e.g. +6 to +10 extra words in the very worst bucket) — and you still lose every single one. Volume doesn't save you if your words are worth less on average.

**See it:** [Word Value Differential Distribution](https://claude.ai/code/artifact/6034b8ee-69da-403b-9e31-eeee4251e05a) plots this exact number across all 116 games. The shape says it plainly — mass piled up to the left of zero (mean −44.13), with 84 of 116 games sitting in negative territory and only 32 positive. This wasn't a handful of blowout losses dragging the average down; being the value-underdog was the *typical* game, not the exception.

## What actually predicts a win, ranked

Correlation of each stat against winning (excluding score_diff itself, since score differential *is* the win condition by definition, so it trivially correlates at 0.76 — not an insight, just the rule of the game):

1. **avg_word_value_diff — 0.632** (your average word value minus theirs)
2. words_over_800_diff — 0.624 (count of ≥800-point words, mine minus theirs)
3. words_diff — 0.532 (total words found, mine minus theirs)
4. words_over_1400_diff — 0.421
5. my_words (raw) — 0.383
6. my_score (raw) — 0.367
7. opp_words_over_1400 — −0.296
8. opp_avg_word_value — −0.282

Word value differential wins. Raw word count is a weaker signal than any of the "how good were the words" measures.

## Overall averages, mine vs. opponent's

- Score: you average 10,506.9 per game, opponent averages 11,482.8 — you're down about 976 points a game on average.
- Words found: you average 34.4, opponent averages 32.9 — you actually find *more* words on average.
- Average word value (score ÷ words): you average 298.98 points/word, opponent averages 343.11 — theirs are worth about 44 points more each, on average.
- Words worth ≥800: you average 3.85/game, opponent averages 4.99.
- Words worth ≥1400: you average 0.40/game, opponent averages 1.17.
- Words worth ≥2200: essentially never (you: 0.00, opponent: 0.03 — a genuinely rare tier).

Put together: you outwork opponents on quantity, they outwork you on quality, and quality is worth more.

## Averages split by win/loss

- When you win: your score averages 12,432.6, avg word value 321.3, words 38.4.
- When you don't win: your score averages 9,241.4, avg word value 284.3, words 31.8.
- The word-value gap **flips sign** depending on outcome: when you lose, your avg word value (284.3) is *below* the opponent's that game (365.3); when you win, yours (321.3) is *above* theirs (309.4). Winning isn't just "showing up and finding words" — it tracks a real jump in word quality specifically.

## Total combined score (yours + opponent's) — is a "high-scoring game" good for you?

Not reliably. Split into 10 bands from 4k to 44k combined score, win rate bounces between 32% and 67% with no clean trend — except the very top band (40k–44k, the 3 highest-scoring games on record), where you're the only band with a *positive* score differential on average (+400). That's a tiny sample though (n=3), so don't over-read it. Total combined score is a much weaker signal than word value differential.

## Opponent strength matters a lot (new finding — wasn't checked before)

Split opponents into three tiers by how valuable their words were that game (weak / mid / strong):

- **Weak opponents:** 53.85% win rate (39 games)
- **Mid opponents:** 44.74% win rate (38 games)
- **Strong opponents:** 20.51% win rate (39 games)

You do play better against tougher opponents — your own avg word value rises from 242 (vs weak) to 304 (vs mid) to 351 (vs strong) — but the opponents' value rises faster (247 → 330 → 452), so the gap actually *widens* against strong opponents rather than closing. You're not being outclassed by a fixed amount; the toughest opponents pull further ahead of your best response to them.

## No evidence of improving or declining over the 116-game sequence

Checked whether performance drifts across the order the games were played (win rate, score differential, word-value differential, all against game order 1–116). All essentially flat — correlation between game order and winning is ~0 (r = −0.011, not statistically significant, p = 0.91). No detectable "warming up" or "getting tired" pattern across this set of games. This is a real result even though it's a non-finding — it rules out session fatigue/practice-effect as an explanation for any given game's outcome.

## Streaks

- Longest win streak: 7 games in a row.
- Longest loss streak: 9 games in a row.
- 57 separate streaks across 116 games (average streak length ≈ 2 games) — results alternate fairly choppily rather than clustering into a few very long runs.

## Word quality as a rate, not just a count

Looked at what *share* of your found words are worth ≥800 (rather than the raw count): you average 10.6% of your words in that tier, opponents average 14.6%. So it's not just that opponents find more big words in absolute terms — proportionally more of their vocabulary skews valuable too. Interestingly, this rate-based version correlates with winning slightly *less* (0.562) than the raw-count differential (0.624), so for predicting outcomes, raw count of big words matters a bit more than the percentage.

## Consistency — who's more erratic, you or your opponents?

- Your score is slightly *more* variable game-to-game than opponents' (coefficient of variation 0.407 vs. 0.383).
- But your average word value is *more consistent* than opponents' (CV 0.25 vs. 0.283).
- Combined, this suggests your score swings come more from how many words you find each game (variable) than from the value of those words (comparatively stable) — while for opponents it's more mixed.

## No volume/quality tradeoff

Worth checking: do you sacrifice word value when you rush to find more words (a speed/quality tradeoff)? No — if anything, the opposite. Words found and average word value correlate *positively* for you (r = 0.351, and similarly for opponents at r = 0.303). Your best games tend to be good on both dimensions simultaneously; it's not that finding more words comes at the cost of finding worse ones. This likely reflects that some game boards are just better/easier for everyone than others, more than it reflects a personal strategic tradeoff.

## Words → score regression (how directly does word count predict score?)

Simple linear fit, score = slope × words + intercept, for each side:

- Mine: R² = 0.653 (word count explains 65% of the variance in my score)
- Opponent's: R² = 0.480 (explains only 48% of theirs)
- Total (combined): R² = 0.585

Word count predicts *your* score noticeably better than it predicts the *opponent's* score. That's consistent with everything else here: your score is driven more by how many words you find (a simpler, more linear relationship), while the opponent's score depends more on word quality, which word count alone doesn't capture — hence the weaker fit.

## Can "my stats alone" predict whether I win? (yes, modestly)

Built a model (logistic regression, the statistically correct tool for a win/loss target) using only stats about your own performance — not the opponent's, since in a live game you wouldn't know their score beforehand. Trained on 65 games, tested on 51 held-out games:

- Test accuracy: 74.5% (38 of 51 correct)
- Compare to baseline of just always guessing "not won" (the more common outcome): 62.75% — so the model beats blind guessing by about 12 points, not more.
- Precision (when it predicts a win, how often right): 66.7%
- Recall (of actual wins, how many it catches): 63.2%

Takeaway: your own stats carry real signal about whether you're likely to win, but there's a hard ceiling on how good this can get, because winning is inherently about beating a specific opponent whose performance that game isn't in the model. With only 116 games total, the accuracy estimate itself is also fairly uncertain (95% confidence range roughly 62.5%–86.5%) — more games (roughly 200+) would tighten that up and probably nudge true accuracy up a little, but the underlying "I don't know what the opponent will do" ceiling would remain no matter how much more data you collected with this same feature set.
