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
   per-hour slopes are smoothed across neighbouring hours and shrunk 50%
   toward the pooled slope, since each comes from only 6–11 curves. The
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
   (pooled over hours 2–6, shrunk 50% toward 0) adjusts the log-ratios
   at hours 2–6 only. The factor's own per-hour slopes there are ~0, so
   folding this into the factor would dilute it to nothing. 7/7 LOO
   folds won, paired +0.0036, L2O +0.0020. Shipped on the user's call:
   the gain is below the +0.005 acceptance rule used in rounds 3–4, but
   the sign never flips and the mechanism is understood. GTA VI's own
   pacific share is 0.60 (z ~ −1.2), so in the final case the adjustment
   is small and in the shallower-dip direction.
5. **Trim blend** (round 6): the log-median at each hour is half the
   p1-weighted median and half the trimmed weighted mean of the same
   adjusted log-ratios (the trimmed mean drops the min and max per
   hour). Trimming drops the steepest curves, so the centre moves
   flatter; it helps the flattest fold (Last of Us +0.0108) and costs
   the steepest (Zelda −0.0052). LOO +0.0015 (4/7 wins), L2O +0.0045
   (31/42 wins). Shipped on the user's call: the gain is below the
   +0.005 acceptance rule, and coverage is 0.89, under the 0.90 floor
   (restoring it with a wider sigma costs the whole gain).
6. **Median**: weighted median of the (factor-adjusted) log-ratios
   `ln(viewers_h / viewers_1)` across the similar launches, with a
   Gaussian weight in log hour-1 size (σ = 1.0) — bigger launches have
   flatter curves.
7. **Spread**: one pooled lognormal σ from the interquartile range of the
   residuals of the adjusted log-ratios around the smoothed median,
   IQR / 1.349, × 1.40, floored at 0.15. Per-hour IQRs from 6–11 curves
   are too noisy to trust on their own. Quantiles are
   `p1 * exp(median + z_τ * σ)`.
8. **Smoothing**: the log-median curve is smoothed across neighbouring
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
| This entry, similar launches (n=7) | **0.121** | 0.89 | 0.190 |
| House baseline, same folds | 0.344 | 0.78 | 0.548 |
| This entry, all launches (n=37) | 0.796 | — | — |
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
(31/42 wins), coverage 0.89. The round-6 target (~0.10, ensembles /
bagging / winsorising / nested bias correction) was not reached: no
candidate cleared the +0.005 / 5-of-7 rule in LOO, and the trim blend
was shipped on the user's call despite that and despite coverage
0.89. Leave-two-out on the similar set (21 pairs, 5 training curves)
gives 0.130, still ahead of round 5's 0.135 on the same pairs, so the
round-6 gain survives with one fewer training curve. See `ANALYSIS.md`
§11–§12 for the full round-6 investigation, the per-fold numbers, and
what was rejected.

The all-launch figure is dominated by simultaneous-global-unlock launches
whose hour-1 print is a pre-unlock trickle and whose viewership then
explodes 100–2000×; GTA VI's hour 1 is post-unlock, so that figure is not
representative of the final. The practice cases' own answers are not in
the kit, so CV is on the comparables, not on p00–p03.
