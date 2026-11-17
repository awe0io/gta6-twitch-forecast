[![make.fun · gta6-twitch-ccu-forecast-01](https://make.fun/badge/gta6-twitch-ccu-forecast-01.svg)](https://make.fun/bounty/gta6-twitch-ccu-forecast-01?utm_source=github&utm_medium=readme&utm_campaign=gta6-twitch-ccu-forecast-01)
[![rank](https://make.fun/badge/gta6-twitch-ccu-forecast-01/awe0io/gta6-twitch-forecast/rank.svg)](https://make.fun/bounty/gta6-twitch-ccu-forecast-01/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=gta6-twitch-ccu-forecast-01)
[![reward](https://make.fun/badge/gta6-twitch-ccu-forecast-01/awe0io/gta6-twitch-forecast/reward.svg)](https://make.fun/bounty/gta6-twitch-ccu-forecast-01?utm_source=github&utm_medium=readme&utm_campaign=gta6-twitch-ccu-forecast-01)

# GTA VI launch-day Twitch forecast

Entry for make.fun bounty `gta6-twitch-ccu-forecast-01`: forecast the
concurrent viewers in the "Grand Theft Auto VI" Twitch category for hours
2–24 of release day (2026-11-19 UTC), as five quantiles per hour, anchored
on the hour-1 print.

## Method (short)

Everything is computed at runtime from the case file, so it adapts to the
final case (42 comparables, including the four practice launches):

1. **Anchor** on the case's hour-1 print `p1`.
2. **Shape from similar launches only**: a comparable counts when it looks
   like GTA VI — console-only, rolling midnight-local unlock, no early
   access, no PC, no simultaneous global unlock (scored from its
   `platforms` / `unlock` text). The GTA VI category's own pre-release
   history entry is not a launch and is skipped.
3. **Steepness factor**: how front-loaded hour 1 is relative to the
   launch's true draw, as one standardised score
   `s = -z(ln p1) - z(ln viewers-per-channel at hour 1)
   + z(ln pre-release 24h mean viewers / p1)`. Big launches, concentrated
   hour-1 audiences and hour-1 prints already far above pre-release buzz
   all go with flatter curves, so each training curve's log-ratio is
   shifted by `-b_h · (s_curve - s_target)`, where `b_h` is the factor's
   slope at hour h: ~0 in the first hours, ~0.15 in the evening
   (hours 15–19), a shape stable across every LOO training set. The
   per-hour slopes are smoothed across neighbouring hours and shrunk
   toward the pooled slope (round 8, banded: evening hours 13–19 at
   95%, other hours at 85%), since each comes from only 6–11 curves. The
   target's `s` is clamped to the training pool's range (extended by half
   a range width), so the forecast never extrapolates wildly past the
   launches it learned from. When channels or pre-release history are
   missing, the factor uses only the features available; when it cannot
   be built, the forecast degrades to the plain p1-weighted median.
4. **Pacific-share early-hour adjustment** (round 5): the share of the
   fanbase in the first-unlock regions, as the mean viewers at UTC 11–14
   over the 7 pre-release days divided by the mean over all hours. A
   bigger first-wave share means a bigger hour-1 rush relative to the
   rest of the day, hence a deeper early dip, so a dedicated slope
   (pooled over hours 2–6, unshrunk, round 8, was 25% toward 0 in
   round 7, 50% before that) adjusts the log-ratios at hours 2–6 only. The factor's own per-hour slopes there are ~0, so
   folding this into the factor would dilute it to nothing. 7/7 LOO
   folds won, paired +0.0036, L2O +0.0020. Shipped on the user's call:
   the gain is below the +0.005 acceptance rule used in rounds 3–4, but
   the sign never flips and the mechanism is understood. GTA VI's own
   pacific share is 0.60 (z ~ −1.2), so in the final case the adjustment
   is small and in the shallower-dip direction.
5. **Night-share early-hour adjustment** (round 7): the share of the
   fanbase awake at night, as the mean viewers at UTC 02–05 over the 7
   pre-release days divided by the mean over all hours. The loss lives
   at hours 3–4 (the 02:00–04:00Z dead zone); a fanbase with a real
   night audience keeps watching through it, so its dip is shallower —
   the fitted slope is positive in every LOO fold. A dedicated slope
   (pooled over hours 3–4, unshrunk) adjusts the log-ratios at hours
   3–4 only. LOO +0.0030 alone (6/7 wins); stacked with the looser
   pacific (shrink 0.75), evening (weight 0.5) and slope (shrink 0.6)
   settings it is the round's passing candidate: LOO 0.1134, 6/7 folds
   won, L2O 0.1211 (35/42 wins), coverage back to 0.91. GTA VI's own
   night share is 0.66, the highest in the set (z ~ +1.7), so in the
   final case this is the largest single-hour adjustment in the model.
   Round 8: the night shrink is 1.25 (up from 1.0) in the joint stack.
6. **Per-hour sigma at hours 3–4 only** (round 8): the dead-zone hours
   have far wider residuals (sd 0.409/0.294 vs ~0.23 elsewhere), so
   their intervals use the per-hour IQR sigma instead of the pooled
   one; other hours keep the pooled sigma.
7. **Joint stack** (round 8): trim weight 0.75 (up from 0.5), sigma
   inflation 1.60 (up from 1.40), pacific shrink 1.0, night shrink
   1.25, banded slope map (evening 0.95 / rest 0.85). LOO 0.1083, 6/7
   folds won, L2O 0.1163 (28/42 wins), coverage 0.91 — the first round
   to clear the +0.005 / 5-of-7 bar, but only just (+0.0051) and with
   ALL 0.8039, over the 0.80 cap. The clean near-miss without the ALL
   breach is the same stack with pacific 0.75 and band 0.90/0.85:
   LOO 0.1089, ALL 0.7995, L2O 0.1171 (29/42 wins).
8. **Trim blend** (round 6): the log-median at each hour is
   TRIM_WEIGHT × (trimmed weighted mean) + (1 − TRIM_WEIGHT) ×
   (p1-weighted median) of the same adjusted log-ratios (the trimmed
   mean drops the min and max per hour). Trimming drops the steepest
   curves, so the centre moves flatter; it helps the flattest fold
   (Last of Us +0.0108) and costs the steepest (Zelda −0.0052). LOO
   +0.0015 (4/7 wins), L2O +0.0045 (31/42 wins). Shipped on the user's
   call: the gain is below the +0.005 acceptance rule, and coverage is
   0.89, under the 0.90 floor (restoring it with a wider sigma costs
   the whole gain). Round 8: TRIM_WEIGHT is 0.75, up from 0.5.
9. **Median**: weighted median of the (factor-adjusted) log-ratios
   `ln(viewers_h / viewers_1)` across the similar launches, with a
   Gaussian weight in log hour-1 size (σ = 1.0) — bigger launches have
   flatter curves.
10. **Spread**: one pooled lognormal σ from the interquartile range of the
   residuals of the adjusted log-ratios around the smoothed median,
   IQR / 1.349, × 1.60 (round 8, was × 1.40), floored at 0.15.
   Per-hour IQRs from 6–11 curves are too noisy to trust on their own,
   except at hours 3–4 (see item 6). Quantiles are
   `p1 * exp(median + z_τ * σ)`.
11. **Smoothing**: the log-median curve is smoothed across neighbouring
   hours ([0.25, 0.5, 0.25]), keeping the unlock waves (US East ~hour 6,
   US West ~hour 9, European evening ~hours 15–19) that sit at the same UTC
   hours for every rolling-midnight launch.

Robustness: `forecast_case` never raises — any unexpected input shape
falls back to the house persistence baseline, so the output is always a
valid forecast (23 rows, hours 2–24, five non-decreasing quantiles in
(0, 1e9]).

See `ANALYSIS.md` for the numbers (ratio curves, per-fold CV losses).

## Run

```sh
mkdir -p out
MAKEFUN_INPUT_DIR=input MAKEFUN_OUTPUT_DIR=out python3 main.py
```

Python 3.13, standard library only, no network. Reads every `*.in` in the
input dir, writes one `<id>.out` per case: exactly
`{"hours": [{"hour": h, "q": [q05, q25, q50, q75, q95]}]}` for h = 2..24.
Deterministic (same input → same bytes); ~0.07 s per case on this VM.

## Validation

Leave-one-out cross-validation on the 37 usable comparables in the kit
(`main.py` itself, called on synthetic cases holding out one launch):

| | mean pinball loss | 90% coverage | typical miss |
|---|---|---|---|
| This entry, similar launches (n=7) | **0.108** | 0.91 | 0.171 |
| House baseline, same folds | 0.344 | 0.78 | 0.548 |
| This entry, all launches (n=37) | 0.804 | — | — |
| House baseline, all launches | 0.930 | — | — |

Round 1 shipped 0.201 / 0.747; round 2 (steepness factor + pooled σ)
0.146 / 0.770; round 3 (per-hour factor slopes, shrunk 50% toward the
pooled slope, σ inflation 1.40) 0.132 / 0.786; round 4 (pre-release
evening share as a 4th factor component, weight 0.25, z capped at ±2.5)
0.126 / 0.787; round 5 (pacific-share early-hour adjustment, hours 2–6,
slope shrunk 50% toward 0) 0.122 / 0.792 — a further 3% on the
similar-launch loss, 7/7 folds won, paired +0.0036; round 6 (trim blend:
half weighted median, half trimmed weighted mean per hour) 0.121 /
0.796 — a further 1%, 4/7 folds won, paired +0.0015, L2O +0.0045
(31/42 wins), coverage 0.89; round 7 (night-share early-hour adjustment,
hours 3–4, unshrunk, + pacific shrink 0.75 + evening weight 0.5 + slope
shrink 0.6) 0.113 / 0.799 — a further 6%, 6/7 folds won, paired +0.0072,
L2O +0.0093 (35/42 wins), coverage back to 0.91; round 8 (joint stack:
trim 0.75 + σ ×1.60 + pacific 1.0 + night 1.25 + banded slope map
evening 0.95 / rest 0.85 + per-hour σ at hours 3–4) 0.108 / 0.804 — a
further 5%, 6/7 folds won, paired +0.0051, L2O +0.0049 (28/42 wins),
coverage 0.91. Round 8 clears the +0.005 / 5-of-7 bar only just
(+0.0051, by 0.00014) and its ALL is 0.8039, over the 0.80 cap — the
plateau is flat (15 near-identical stacks score 0.1083–0.1093) and
further rounds would be grid-chasing at the noise floor. The clean
near-miss without the ALL breach is the same stack with pacific 0.75
and band 0.90/0.85: 0.109 / 0.800, L2O 0.1171 (29/42 wins).
Leave-two-out on the similar set (21 pairs, 5 training curves) gives
0.116, still ahead of round 7's 0.121 on the same pairs, so the
round-8 gain survives with one fewer training curve. Round 9 (joint
fit of the early slopes on factor-adjusted log-ratios, Tsushima
diagnosis, asymmetric σ at hours 3–4) was a clean null: nothing moves
C4 by more than 0.0001, and the joint fit is worse (0.1096). See
`ANALYSIS.md` §15 for the full round-9 investigation. Round 10 (fine
grid around C4, pre-release windows, slope-map smoothing, median
definition, h3–4 trim, pool edge) was also a clean null: the fine
grid is flat to ±0.0006 and the only above-noise point (5-day band
window, 0.1062) is a single-fold artifact (Zelda +0.0253 alone, 4/7
folds lose). See `ANALYSIS.md` §16. Round 11 (soft pool weights,
h3–4 empirical quantiles, clamp pad, unsmoothed-median sigma, plus a
final-case sensitivity diagnostic) was also a clean null: the best
point is −0.0012 on one fold, and soft weights are catastrophic
(0.2422 / 0.4695). The diagnostic's ship-decision fact: C4 vs C2
differs by at most 0.019 log units on the real target, while the
s_target clamp pad is worth up to 0.83 — the clamp stays at pad 0.5.
See `ANALYSIS.md` §17.

The all-launch figure is dominated by simultaneous-global-unlock launches
whose hour-1 print is a pre-unlock trickle and whose viewership then
explodes 100–2000×; GTA VI's hour 1 is post-unlock, so that figure is not
representative of the final. The practice cases' own answers are not in
the kit, so CV is on the comparables, not on p00–p03.
