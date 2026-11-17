# Analysis — gta6-twitch-ccu-forecast-01

All numbers below come from the kit's practice cases p00–p03, which share
one set of 38 comparables (37 usable curves + the GTA VI category's own
pre-release history). The practice answers are NOT in the kit files, and
`grade-local` refuses this competition, so validation is leave-one-out
cross-validation (LOO CV) on the comparables themselves: hold out one
comparable launch, forecast its release-day hours 2–24 from its hour-1
print using the other comparables, score pinball loss on the log scale
(exactly as RULES.txt scores). The shipped `main.py` was used for the
final CV run, so the numbers below are the shipped code's, not a prototype's.
Sections 1–5 describe round 1 (shipped SIM 0.201); §6 describes round 2
(2026-11-17), which added the steepness factor and pooled sigma (SIM
0.146); §7 describes round 3 (2026-11-17), which replaced the single
pooled factor slope with per-hour slopes (shrunk 50% toward pooled) and
raised the sigma inflation to 1.40, and is what `main.py` ships now
(SIM 0.132).

## 1. What hour 1 means

Hour-1 prints vary 1000× across comparables, and the variation is unlock
timing, not launch size:

- Rolling-midnight console launches: hour 1 already reflects New Zealand /
  Australia / Japan / Korea (unlocked 11:00–15:00Z the day before) plus the
  UK at 00:00Z. p1 = 6k–365k.
- Simultaneous-global-unlock launches (e.g. Silksong 14:00Z, Black Myth
  02:00Z, Battlefield 6 ~14:00Z): hour 1 is a pre-unlock trickle
  (p1 = 164–4,941) and viewership then explodes 100–2000×.
- PC-simultaneous-at-midnight-GMT launches (Cyberpunk, Elden Ring): hour 1
  already holds the global PC surge (p1 = 852k, 875k) and the curve then
  decays/flattens.

GTA VI is console-only (PS5 / Xbox Series X|S), rolling midnight local, no
early access, no PC — so only the first group is the relevant analogue.
That is the whole design: condition on the launch type, not on the raw
pool.

## 2. Ratio curves of the similar launches

The 7 similar launches (weight 1.0): Ghost of Yōtei (p1 27k), Saros
(6.4k), Zelda: Tears of the Kingdom (130k), Final Fantasy VII Rebirth
(53k), The Last of Us Part II (365k), Ghost of Tsushima (132k), Stellar
Blade (27k). Median log-ratio curve (ratio = viewers_h / viewers_1) and
IQR-based σ:

| hour | median ratio | σ (IQR/1.349) | | hour | median ratio | σ |
|---|---|---|---|---|---|---|
| 2 | 0.82 | 0.10 | | 13 | 1.14 | 0.42 |
| 3 | 0.67 | 0.31 | | 14 | 1.39 | 0.45 |
| 4 | 0.52 | 0.31 | | 15 | 2.13 | 0.46 |
| 5 | 0.84 | 0.19 | | 16 | 2.26 | 0.46 |
| 6 | 1.00 | 0.24 | | 17 | 2.53 | 0.48 |
| 7 | 1.04 | 0.35 | | 18 | 2.60 | 0.59 |
| 8 | 0.94 | 0.41 | | 19 | 2.67 | 0.57 |
| 9 | 0.88 | 0.35 | | 20 | 2.64 | 0.37 |
| 10 | 0.85 | 0.28 | | 21 | 2.48 | 0.33 |
| 11 | 0.88 | 0.33 | | 22 | 2.30 | 0.34 |
| 12 | 1.03 | 0.36 | | 23 | 2.18 | 0.37 |
| | | | | 24 | 2.04 | 0.48 |

Shape reads as the unlock waves: dip to ~0.52 in hours 3–4 (post-midnight
lull after the first regions' rush), recovery at hour 6 (US East unlock
05:00Z), plateau ~0.9–1.0 through hour 11, then the climb to a ~2.6× peak
at hours 18–19 (European evening + US afternoon), easing to ~2.0× by
hour 24. The final case adds 4 more similar launches (Wolverine, RDR2,
God of War Ragnarök, Spider-Man 2 — all console-only rolling midnight),
so the similar pool grows from 7 to 11 curves at runtime.

Per-launch r24 vs p1 in the similar set: Yōtei 3.45 (27k), Saros 2.84
(6.4k), Zelda 1.90 (130k), FFVII Rebirth 1.31 (53k), Last of Us II 1.17
(365k), Ghost of Tsushima 2.04 (132k), Stellar Blade 2.47 (27k) — bigger
hour-1 prints go with flatter curves, which is why the median is weighted
by log hour-1 proximity (Gaussian σ = 1.5).

## 3. Weighting

Similarity weight from `platforms` / `unlock` text (multiplicative):
early access ×0.25 (campaign-only early access ×0.6; "no early access"
does not count), weak analogue ×0.3, free-to-play ×0.3, PC ×0.5,
simultaneous-global unlock ×0.35. Result: 7 launches at 1.0, the rest at
0.04–0.5 (sum of weights 15.8 over 37 curves). The GTA VI pre-release
history entry is excluded (not a launch; sparse ~12 h sampling, no hour-1
print). `history_hourly` / `channels` of the target case were inspected
and are not used: the hour-1 print is the anchor, and channels track
viewers closely enough to add no signal.

## 4. Method selection (LOO CV, mean pinball loss)

| method | similar (n=7) | all (n=37) |
|---|---|---|
| House baseline (persistence, σ=0.15√(h−1)) | 0.344, cov 0.78 | 0.930 |
| All-launch empirical quantiles | 0.314, cov 0.99 | 0.714 |
| Similar-only empirical quantiles | 0.233, cov 0.71 | 0.878 |
| Similar-only lognormal, smoothed | 0.226, cov 0.77 | 0.867 |
| Similar lognormal + p1-weighting (σ=1.5), smoothed | 0.208, cov 0.73 | 0.795 |
| **Shipped: as above, σ from unweighted IQR ×1.25, floor 0.15** | **0.201, cov 0.92** | **0.747** |

Variants around the shipped design (σ inflation 1.1–1.4, p1 σ 1.5–2.0,
floor 0.15/0.20) all land at 0.200–0.208 on the similar folds — the choice
is inside the noise of 7 folds, so the shipped variant was picked for its
honest coverage (0.92 vs the 0.90 target) rather than the last 0.002 of
loss. Smoothing the log-median and log-σ across hours ([0.25, 0.5, 0.25])
helped slightly (0.226 → 0.208 with p1-weighting) and never hurt.

Per-fold losses, shipped vs baseline (similar launches):

| launch | shipped | baseline |
|---|---|---|
| Ghost of Yōtei | 0.211 | 0.473 |
| Saros | 0.247 | 0.601 |
| Zelda: Tears of the Kingdom | 0.160 | 0.309 |
| Final Fantasy VII Rebirth | 0.193 | 0.173 |
| The Last of Us Part II | 0.302 | 0.247 |
| Ghost of Tsushima | 0.125 | 0.228 |
| Stellar Blade | 0.171 | 0.372 |

The two losing folds are the flattest curves in the set (FFVII r24 1.31,
Last of Us II r24 1.17); the median shape over-predicts their evening
rise. Typical miss on similar folds: 0.320 (shipped) vs 0.548 (baseline).

## 5. Caveats

- **Small n.** The shape rests on 7 similar curves in the practice kit
  (11 in the final). Per-hour IQRs from 7 points are noisy; the ×1.25
  inflation and 0.15 floor are the guard against over-narrow intervals,
  and the coverage (0.92) says they roughly work — but a single unusual
  final curve shape can still move the loss by ~0.05.
- **Unforecastable exploders.** The all-launch CV (0.747) is dominated by
  8 simultaneous-global launches with losses 1.3–3.1 (e.g. Silksong 3.15,
  Black Myth 2.90, Kingdom Come 2.76). No hour-1-anchored method can
  predict a 100–2000× post-unlock explosion without knowing the unlock
  hour; GTA VI's hour 1 is post-unlock, so these folds are not
  representative — but if GTA VI's hour-1 print came in bizarrely low
  (recorder failure, category misread), the forecast would under-predict
  the US unlock wave badly.
- **Practice targets are not scored here.** p00–p03's answers are not in
  the kit, so the CV above is on comparables, not on the practice targets
  themselves. The practice leaderboard score will differ.
- **Hour-1 magnitude.** The p1-proximity weighting assumes GTA VI's hour-1
  print is large (hundreds of thousands). If it is small, the weights
  shift toward the small-launch curves (Yōtei / Stellar / Saros) and the
  forecast gets steeper — the intended behaviour, but untested at that
  scale.
## 6. Round 2 — steepness factor and pooled sigma (2026-11-17)

Round 1's median was a p1-weighted median of raw log-ratios. Two new
signals were found in the kit data and tested in LOO CV (same folds,
same scoring; variant harness in /tmp/cv/variant.py, verified by re-running
the CV through the shipped `main.py` itself — identical numbers).

### 6.1 The signals

- **Viewers per channel at hour 1** (`vpc1 = p1 / channels_1`): corr with
  log(r24) is −0.674 on similar launches. Mechanism: a concentrated hour 1
  (few channels, many viewers each — the big streamers are already live)
  goes with a flatter curve; a diffuse hour 1 (many small channels) goes
  with a steeper one, as the streamer base keeps joining (median
  viewers-per-channel collapses 89 → 24 by hour 6).
- **Pre-release buzz** (`premean/p1` = mean viewers over the 24h before
  day_start, divided by p1): corr with log(r24) is +0.848 on similar
  launches (partial corr controlling log p1: 0.67), strongest in the
  evening hours (h15 +0.835, h18 +0.895) where the forecast matters most.
  A high ratio means hour 1 understates the launch's true draw.
- Both features are available at forecast time: `known[0].channels` and
  `history_hourly` in the target case, `hourly` rows in the comparables.
  `channels`/`history_hourly` add nothing as direct curve proxies (the
  channels ratio runs far ahead of the viewers ratio), only through these
  two hour-1 summaries.

The three features are highly correlated (log p1 vs vpc +0.897, log p1 vs
pre −0.736), so they were combined into **one standardised steepness
factor** `s = −z(ln p1) − z(vpc) + z(pre)` with a single pooled OLS slope
of log-ratio on `s` (per-hour intercepts removed). Pooled slopes fitted on
each LOO training set are stable: log p1 −0.196 (LOO range −0.223…−0.140),
vpc −0.353 (−0.394…−0.290), pre +0.284 (+0.216…+0.384); factor slope on
all 7 similar curves is +0.096, factor range [−4.20, +4.86]. The target's
factor is clamped to the training pool's range extended by half a range
width — a hard clamp to the range was tried first and cost 0.028 of SIM
loss in CV (it binds exactly on the extreme launches the factor helps
most), so the clamp only guards against wild extrapolation, it does not
bind on any CV fold.

### 6.2 Variant table (LOO CV, mean pinball loss)

| variant | similar (n=7) | cov | all (n=37) |
|---|---|---|---|
| Round 1 shipped | 0.2014 | 0.92 | 0.7472 |
| Median shrink 0.85 / 0.9 (blend to persistence) | 0.2041 / 0.2023 | 0.91 | 0.7624 / 0.7568 |
| p1 kernel σ 1.0 / 2.0 / ∞ | 0.2010 / 0.2057 / 0.2228 | 0.92 / 0.91 / 0.87 | 0.7210 / 0.7707 / 0.8397 |
| σ inflation 1.1 / 1.4 | 0.1993 / 0.2049 | 0.88 / 0.94 | 0.7598 / 0.7367 |
| σ floor 0.10 / 0.20 | 0.2016 / 0.2012 | 0.92 / 0.93 | 0.7473 / 0.7470 |
| No smoothing / 5-point smoothing | 0.2046 / 0.2003 | 0.93 | 0.7515 / 0.7457 |
| Soft similarity weights (all 37, weighted) | 0.3302 | 0.99 | 0.5350 |
| Pooled regression on log p1, shrink 0.5 / 1.0 | 0.1839 / 0.1768 | 0.89 | 0.7193 / 0.6951 |
| Pre feature, shrink 0.5 / 1.0 | 0.1811 / 0.1856 | 0.89 / 0.86 | 0.7915 / 0.8429 |
| vpc feature, shrink 0.5 / 1.0 | 0.1735 / 0.1546 | 0.93 / 0.88 | 0.7928 / 0.8970 |
| Marginal vpc=1.0 + pre=0.5 | 0.1420 | 0.91 | 0.9306 |
| Joint ridge [p1, vpc, pre], ridge 0.5 | 0.1711 | 0.85 | 0.7629 |
| Joint ridge [p1, vpc], ridge 1.0 | 0.1761 | 0.87 | 0.7355 |
| Empirical quantiles (no lognormal) | 0.2183 | 0.70 | 0.8216 |
| Pooled σ (one σ from pooled residuals) | 0.1948 | 0.94 | 0.7446 |
| **One-factor, shrink 1.0** | **0.1519** | **0.90** | **0.7955** |
| **One-factor + pooled σ** | **0.1471** | **0.93** | **0.7848** |
| **Shipped round 2: factor + pooled σ, p1 σ 1.0** | **0.1457** | **0.92** | **0.7697** |

### 6.3 What was rejected and why

- **Median shrink toward persistence** (the two losing round-1 folds were
  the flattest curves): no help at any level (0.95 = shipped, lower is
  worse). The flat-curve problem is a feature problem, not a shrinkage
  problem — the factor fixes it directly.
- **Marginal vpc + pre** (best SIM, 0.1420 / 0.1362 with pooled σ): rejected
  despite the best SIM number. The two marginal slopes double-count the
  same latent "front-loaded hour 1" component (features correlate up to
  +0.897), so the adjustment is ~2× the true partial effect; the all-launch
  figure (0.9306 — no better than the house baseline's 0.9296) exposes the
  over-adjustment. The one-factor model keeps 90% of the SIM gain
  (0.1457 vs 0.1362) with a far healthier ALL (0.7697 vs 0.9244).
- **Joint ridge regression**: principled, but worse than the one-factor on
  SIM (0.1711 best) — with 6 training curves, three correlated slopes are
  too many parameters; the single factor is the better bias/variance trade.
- **Soft similarity weights**: SIM collapses to 0.3302 (dissimilar launches
  pollute the similar median) even though ALL improves to 0.5350. The
  final is a similar-type launch; SIM is the metric that matters.
- **Empirical quantiles**: worse on both (0.2183, cov 0.70) — 6 points
  cannot support five empirical quantiles; the lognormal is the right
  smoothing.
- **σ inflation 1.1 / floor changes / 5-point smoothing**: all inside the
  noise of 7 folds (±0.002); kept the round-1 values (1.25 / 0.15 /
  [0.25, 0.5, 0.25]).

### 6.4 Per-fold losses, shipped round 2 vs round 1 vs baseline

| launch | round 2 | round 1 | baseline |
|---|---|---|---|
| Ghost of Yōtei | 0.148 | 0.211 | 0.473 |
| Saros | 0.142 | 0.247 | 0.601 |
| Zelda: Tears of the Kingdom | 0.184 | 0.160 | 0.309 |
| Final Fantasy VII Rebirth | 0.161 | 0.193 | 0.173 |
| The Last of Us Part II | 0.163 | 0.302 | 0.247 |
| Ghost of Tsushima | 0.086 | 0.125 | 0.228 |
| Stellar Blade | 0.136 | 0.171 | 0.372 |

Round 2 wins 6 of 7 folds against round 1 (paired mean −0.054, sd 0.049);
only Zelda worsens (+0.024). Typical miss 0.320 → 0.227.

### 6.5 Caveats (round 2)

- **Small n, again.** Slopes are fitted on 6 similar curves per CV fold
  (11 in the final, which should be more stable), and the factor's three
  components are correlated — the single-factor design is the guard
  against overfitting, but a single unusual final curve shape can still
  move the loss by ~0.05.
- **Factor extrapolation.** GTA VI's pre-release history spans months
  (trailer premiere 2026-08-27), so its premean/p1 could sit outside the
  similar range; the half-range clamp caps the adjustment there. If the
  final case's `history_hourly` is sparse in the last 24h, the pre feature
  falls back to the last 24 pre-release rows; if it is absent entirely,
  the factor drops the pre component (and likewise vpc when channels are
  missing), degrading gracefully to the p1-weighted median.
- **Practice targets are not scored here.** p00–p03's answers are not in
  the kit, so the CV above is on comparables, not on the practice targets
  themselves. The practice leaderboard score will differ.
- **Hour-1 magnitude.** The factor's p1 component assumes GTA VI's hour-1
  print is large (hundreds of thousands). If it is small, the weights
  shift toward the small-launch curves and the forecast gets steeper —
  the intended behaviour, but untested at that scale.
- **No early-access / no PC assumption.** If GTA VI had an unannounced
  early-access or streaming-embargo quirk, the similar set would be the
  wrong analogue; nothing in the case data suggests one.

## 7. Round 3 — per-hour factor slopes (2026-11-17)

Acceptance rule for this round (set before running): a change ships only
if it improves SIM LOO by > 0.005 AND wins or ties on ≥ 5 of 7 folds, or
improves the paired mean by > 0.01; AND does not push ALL above 0.80. A
leave-two-out check (21 pairs, 5 training curves) is required for the
final candidate. Harness: `/tmp/cv3/v3.py` (parameterised forecaster,
reproduces the shipped round-2 numbers exactly: SIM 0.1457, ALL 0.7697),
`/tmp/cv3/diag.py` (diagnostics), `/tmp/cv3/grid.py`, `/tmp/cv3/grid2.py`,
`/tmp/cv3/grid3.py`, `/tmp/cv3/l2o.py`, `/tmp/cv3/edge.py`.

### 7.1 Diagnostics that motivated the round

**Per-hour LOO residuals (SIM folds, round-2 config).** Bias is small
everywhere except the dip hours: h3 −0.119 (sd 0.46, cov 0.71, loss
0.241), h4 −0.151 (sd 0.31, cov 0.86, loss 0.179); h24 −0.117; all other
hours within ±0.07, sd 0.17–0.34, cov 0.86–1.00. Loss concentrates at h3
(0.241), h4 (0.179), h13–14 (0.169–0.178). The dip at hours 3–4 is both
the noisiest and the most under-covered part of the day.

**Per-hour factor slopes.** The factor's OLS slope on log-ratio, fit per
hour on each LOO training set (6 similar curves), is the same shape in
all 7 fits: ≈ 0.00 at h2–6 (SE 0.02–0.07), ≈ 0.05–0.08 at h7–8,
≈ 0.09–0.11 at h9–12, ≈ 0.13–0.17 at h13–19, ≈ 0.11–0.16 at h20–24.
Evening slopes are 4–6 standard errors from zero in every fold. The
pooled slope (~0.10) therefore under-adjusts the evening, where the
factor matters most, and over-adjusts the early hours, where it does
not. The shape is a property of the data, not of one fold — it survives
holding out any single launch.

**Zelda, the one losing fold (+0.024 in round 2).** Zelda's features say
flat (p1 130k, vpc 103.5, pre/p1 0.41 → factor s ≈ −1.64) but its curve
is steep in the evening (truth 2.24× at h15, 2.60× at h18 vs forecast
q50 1.33× / 1.72×). Its pre-release buzz is low for its size yet its
evening rise is the strongest in the set. No other feature in the kit
separates it: its absolute pre-mean (~53k) sits between the steep and
flat launches, and a pre-release trend feature (last / mean pre-release
viewers) was tested and rejected (§7.2). Zelda is the factor's residual
outlier — one fold in seven, and the factor still gets 6 of 7 right.

**In-sample vs LOO gap.** Fitting the target curve into its own training
pool gives mean loss 0.106 vs LOO 0.146 — a generalisation gap of ~0.04,
the price of learning the median and the slopes from 6 curves. The
in-sample 0.106 is close to the noise floor set by curve-shape
idiosyncrasy (see §7.6).

### 7.2 Candidate table (LOO CV, mean pinball loss)

| variant | similar (n=7) | cov | all (n=37) | wins | paired |
|---|---|---|---|---|---|
| Round 2 shipped | 0.1457 | 0.92 | 0.7697 | — | — |
| Slope per-hour (raw) | 0.1328 | 0.79 | 0.8355 | 5/7 | +0.0129 |
| Slope per-hour, smoothed | 0.1332 | 0.78 | 0.8367 | 5/7 | +0.0125 |
| Slope two-band (h≤8 / h≥9) | 0.1305 | 0.80 | 0.8164 | 5/7 | +0.0152 |
| Slope ramp (linear in h) | 0.1355 | 0.83 | 0.7979 | 5/7 | +0.0102 |
| Slope smooth, shrunk 50% to pooled | 0.1322 | 0.88 | 0.7984 | 6/7 | +0.0135 |
| Slope smooth, shrunk 25% | 0.1378 | 0.90 | 0.7838 | 5/7 | +0.0079 |
| Slope smooth, shrunk 50%, floor 0.5× pooled | 0.1307 | 0.88 | 0.7976 | 6/7 | +0.0150 |
| Slope two-band, shrunk 50% | 0.1320 | 0.88 | 0.7955 | 6/7 | +0.0137 |
| Slope ramp, shrunk 50% | 0.1369 | 0.88 | 0.7827 | 6/7 | +0.0088 |
| **Slope smooth, shrunk 50%, σ ×1.40 (shipped)** | **0.1322** | **0.91** | **0.7862** | **6/7 + 1 tie** | **+0.0135** |
| Slope smooth, shrunk 50%, σ ×1.60 | 0.1342 | 0.97 | 0.7718 | 7/7 | +0.0115 |
| Slope two-band, shrunk 50%, σ ×1.40 | 0.1325 | 0.94 | 0.7834 | 6/7 | +0.0132 |
| Slope ramp, shrunk 50%, σ ×1.40 | 0.1376 | 0.93 | 0.7699 | 6/7 | +0.0081 |
| σ per-hour (no slope change) | 0.1490 | 0.90 | 0.7764 | 2/7 | −0.0033 |
| σ per-hour, shrunk k=6 to pooled | 0.1465 | 0.92 | 0.7707 | 2/7 | −0.0008 |
| Median = weighted mean (no slope change) | 0.1456 | 0.96 | 0.7673 | 2/7 | +0.0001 |
| Median = trimmed mean | 0.1565 | 0.94 | 0.8141 | 1/7 | −0.0108 |
| Bias correction k=6 | 0.1478 | 0.93 | 0.8015 | 5/7 | −0.0021 |
| Drop pre from factor | 0.1597 | 0.94 | 0.7589 | 1/7 | −0.0140 |
| Drop vpc from factor | 0.1619 | 0.93 | 0.7368 | 2/7 | −0.0162 |
| Pre-release trend as 4th factor component | 0.1585 | 0.89 | 0.7549 | 3/7 | −0.0128 |

Notes: (1) All four unshrunk slope variants improve SIM by 0.010–0.015
but push ALL to 0.80–0.84 — the stronger evening slopes overshoot on
launch types GTA VI cannot resemble, whose curves diverge most in the
evening. Shrinking 50% toward the pooled slope keeps most of the SIM
gain while bringing ALL back under 0.80. (2) The better-centred median
shrinks the pooled residuals, so the old σ inflation (×1.25) left the
90% interval under-covering (cov 0.78–0.88); ×1.40 restores cov 0.91
and also improves ALL (0.7984 → 0.7862) at no SIM cost. ×1.60
over-covers (0.97) and costs SIM. (3) Per-hour σ, mean/trimmed median,
bias correction, dropping a factor component, and the pre-trend feature
all fail the acceptance rule — the numbers are in the table so the next
round does not re-test them blind.

### 7.3 Per-fold losses, shipped round 3 vs round 2 vs baseline

| launch | round 3 | round 2 | baseline |
|---|---|---|---|
| Ghost of Yōtei | 0.130 | 0.148 | 0.473 |
| Saros | 0.121 | 0.131 | 0.601 |
| Zelda: Tears of the Kingdom | 0.182 | 0.182 | 0.309 |
| Final Fantasy VII Rebirth | 0.139 | 0.169 | 0.173 |
| The Last of Us Part II | 0.150 | 0.173 | 0.247 |
| Ghost of Tsushima | 0.078 | 0.089 | 0.228 |
| Stellar Blade | 0.125 | 0.128 | 0.372 |

Round 3 wins 6 folds and ties Zelda; paired mean +0.0135, paired sd
0.011, paired t ≈ 3.3 (p ≈ 0.02) — the gain is significant, not a
7-fold coin flip. Typical miss 0.227 → 0.208.

### 7.4 Leave-two-out (21 pairs, 5 training curves)

Round 3: 0.1451. Round 2 on the same pairs: 0.1553. The round-3 gain
(+0.010) survives with one fewer training curve; the LOO→L2O
degradation (+0.013 for round 3, +0.010 for round 2) is the expected
cost of a smaller training pool, not a sharp collapse. Worst pairs all
involve the two hardest folds (Last of Us II, Zelda). With 11 similar
curves in the final case (7 + the 4 practice launches), the slope and
median estimates will be more stable than in this CV, so the final-case
loss is likely slightly better than the 0.132 LOO figure — but the
single-curve idiosyncratic component cannot be predicted away.

### 7.5 `streams` / `categories_hourly` — not used

Both are null in every practice case (RULES.txt: `streams` is the
category's top 20 live streams at the last valid read before the
checkpoint; `categories_hourly` is Twitch's top 20 categories once an
hour). In the final they could in principle carry signal — the top-20
sum vs the hour-1 mean would measure the within-hour trend, and the
category table would measure platform-wide traffic — but there is no
practice observation of either to calibrate any coefficient, and the
comparables carry neither field, so no mapping can be learned from the
kit at all. Any use would be an invented coefficient. The code ignores
both fields (graceful by construction) and the decision is recorded
here so it is not revisited without new data.

### 7.6 Fit ceiling — what the remaining error is made of

The round-3 LOO loss of 0.132 decomposes roughly as: ~0.106 in-sample
loss (the target curve fitted into its own training pool) + ~0.026
generalisation gap. The 0.106 floor is curve-shape idiosyncrasy: even
with the factor, individual launches deviate from the common median
shape by a typical |log residual| of ~0.21 (median absolute log miss),
and the per-hour residual sd is 0.17–0.46. No hour-1-anchored method can
predict that part — it is the launch's own shape, not a function of
anything observable at the checkpoint. The remaining generalisation gap
(~0.026) would shrink with more similar curves (11 in the final, vs 6
per CV fold here), which is the one structural improvement still
available — and it arrives for free at final time, since the program
reads the final case's own 42 comparables.

Further rounds would be chasing noise: every mechanism tested in rounds
1–3 that could move the loss by more than ~0.005 has been tested
(median form, sigma form, shrinkage, bias, factor components, trend,
per-hour slopes, per-hour sigma). The remaining candidates (a 4th factor
component, non-linear factor, day-of-week) each have ≤ 7 points to
support them and failed when tried. The honest statement is that 0.13
is close to this data's ceiling for the similar-launch class, and the
bigger risk to the final score is not the last 0.01 of CV but a
single-curve surprise (a Zelda-type evening, or an hour-1 print that
misstates the launch's draw) — which no further fitting on these 7
curves can insure against.

### 7.7 Caveats (round 3, unchanged from round 2 unless noted)

- **Small n, again.** Slopes are fitted on 6 similar curves per CV fold
  (11 in the final). The 50% shrinkage toward the pooled slope is the
  guard; the leave-two-out check (§7.4) is the evidence it holds.
- **Zelda stays the outlier.** Round 3 ties it (0.182) rather than
  fixing it; the stronger evening slopes would have made it worse
  unshrunk (+0.019–0.029), and the shrinkage is what keeps it level.
- **Factor extrapolation.** Unchanged: GTA VI's pre-release history
  spans months, so its premean/p1 could sit outside the similar range;
  the half-range clamp caps the adjustment.
- **Practice targets are not scored here.** Unchanged: p00–p03's
  answers are not in the kit; CV is on comparables, not on the practice
  targets themselves.

## 8. Round 4 — pre-release diurnal shape as a 4th factor component (2026-11-17)

Round 3's error was still 0.132, and this round was told to attack it with
genuinely new signals, not another turn of the same knobs. The first step
was to decompose the error, so the attack went at the centre and not at
the spread.

### 8.1 Error decomposition (round 3, LOO on the similar set)

For each fold, the shipped forecast was re-scored twice: once with the
median replaced by the best possible median per hour (oracle median,
σ kept), and once with σ replaced by the best per-fold σ (oracle σ,
median kept).

| launch | shipped | oracle median | oracle σ | oracle both |
|---|---|---|---|---|
| Ghost of Yōtei | 0.130 | 0.054 | 0.126 | 0.040 |
| Saros | 0.121 | 0.061 | 0.119 | 0.050 |
| Zelda: Tears of the Kingdom | 0.182 | 0.050 | 0.155 | 0.060 |
| FINAL FANTASY VII REBIRTH | 0.139 | 0.054 | 0.132 | 0.050 |
| The Last of Us Part II | 0.150 | 0.053 | 0.148 | 0.060 |
| Ghost of Tsushima | 0.078 | 0.061 | 0.029 | 0.010 |
| Stellar Blade | 0.125 | 0.052 | 0.123 | 0.050 |
| **Mean** | **0.132** | **0.058** | **0.125** | **0.050** |

Median error is 0.132 − 0.058 = **0.074 (56% of the loss)**; spread
error is 0.132 − 0.125 = **0.007 (5%)**. The oracle floor — even a perfect
median and a perfect σ — is 0.050, so the remaining room above the floor
is almost all median error. The spread is not worth chasing; the centre
is. Per-hour, the worst hours are h3 (loss 0.231, bias −0.100, sd 0.434,
coverage 0.71) and h4 (0.175, bias −0.136, sd 0.304) — variance, not
bias (both biases are inside their standard errors) — then h13–14
(0.156–0.158). Hours 20–23 are the best (0.087–0.097).

### 8.2 New-signal exploration

The target's own pre-release history was never used beyond its 24h mean,
and the comparables carry their own histories, so the mapping is
learnable in CV. Diurnal profiles were computed from each launch's
hourly rows over the 7 days before release (mean viewers by UTC hour,
per-hour share of the daily mean). Correlations with the held-out
launch's log-ratio (per hour, n = 7 similar curves):

- Diurnal profile vs log-ratio: h2 +0.86, h4 +0.82, h3 +0.59, h18 +0.62,
  h15 +0.50, h21 −0.74 — the shape of the pre-release day predicts the
  shape of release day at the same UTC hour, most strongly in the early
  hours.
- Evening share (h15–19 mean / daily mean) vs the shipped model's
  residual: h9 +0.81, h15 +0.72, h18 +0.69, h24 +0.63 — the model's
  biggest misses are exactly where the evening share bites, and Zelda
  (eve 2.40, the steepest evening in the set, and the model's residual
  outlier) is the tell.
- Night share (h0–4) vs h3/h4 residuals +0.66/+0.76; late share (h20–23)
  vs h9–h24 log-ratios −0.63 to −0.80 — consistent with a
  timezone-mix story: where the fanbase lives decides when the day's
  viewers arrive.
- Pre-release volatility (log sd of the hourly series) and day of week
  are weak (no |r| above 0.5 on the residual), so neither was pursued.

### 8.3 Candidate table (LOO CV, mean pinball loss)

| variant | SIM (n=7) | win/tie | paired mean | ALL (n=37) | verdict |
|---|---|---|---|---|---|
| Round 3 shipped | 0.1322 | — | — | 0.7862 | baseline |
| **eve band as 4th factor component, w 0.25** | **0.1256** | **6/7** | **+0.0066** | **0.7871** | **shipped** |
| eve band, w 0.50 | 0.1235 | 3/7 | +0.0086 | 0.7904 | rejected (win count) |
| eve band, w 0.75 | 0.1219 | 5/7 | +0.0103 | 0.7981 | rejected (ALL near the 0.80 line) |
| eve band, w 0.35 | 0.1251 | — | +0.0005 vs w 0.25 | — | rejected (not worth it) |
| diurnal profile, pooled γ | 0.1339 | — | −0.0017 | — | rejected |
| diurnal profile, per-hour γ | 0.1379 | — | −0.0057 | — | rejected |
| diurnal profile, sparse per-hour γ | 0.2015 | — | −0.0693 | — | rejected (Yōtei 0.370, Saros 0.283) |
| direct per-hour band adjustment (`extra`) | 0.16–0.19 | — | — | up to 1.37 | rejected (band shares have too little cross-curve variance; slopes explode) |
| kNN in factor space, k3 α 0.5 | 0.1299 | 4/7 | +0.0022 | — | rejected (below the +0.005 rule) |
| kNN, k3 α 0.3 | 0.1308 | — | +0.0014 | — | rejected |
| quantile regression, blend 0.5 | 0.1346 | — | −0.0024 | — | rejected (coverage 0.79) |
| quantile regression, blend 1.0 | 0.1493 | — | −0.0171 | — | rejected (coverage 0.66) |
| σ conditioned on \|s\| | 0.1318 | — | +0.0004 | — | rejected (spread error is 5%; §8.1) |
| eve w 0.25 + kNN | 0.1229 | — | +0.0027 vs eve alone | — | rejected (below the +0.005 stacking rule) |
| eve w 0.25 + sparse target history | 0.1256 | — | identical | — | rejected (no change; the fallback already covers it) |

The winning variant is the smallest one that passes the round-3
acceptance rule (SIM +0.005 with ≥5/7 folds won/tied, ALL ≤ 0.80). The
weight 0.25, not 1.0: at full weight the component helps Zelda even more
but hurts Yōtei/Saros and pushes ALL over the line. The 24-dimensional
profile failed outright — with 6 training curves, per-hour γ has no
degrees of freedom left. The direct per-hour band adjustment failed
harder: band shares vary little across curves (and hit 0), so the OLS
slopes explode. Quantile regression lost on both loss and coverage,
confirming the lognormal construction is the right shape. kNN and
σ-conditioning were both within noise, as the decomposition predicted.

### 8.4 The shipped change

`_eve_share` computes the target's pre-release evening share (mean
viewers UTC 15–19 over the 7 days before `day_start`, divided by the
mean over all hours; needs ≥12 distinct UTC hours, else None). The
factor becomes s = −z(ln p1) − z(vpc) + z(pre) + 0.25·z(ln eve), with
the evening z-score capped at ±2.5 (see §8.5). When the target or a
comparable lacks a usable profile, that curve falls back to the
3-component factor — the component is dropped, never the factor.

Two bugs were caught by validating through the shipped `main.py`
rather than trusting the exploration harness:

1. **Sparse-fallback bug (harness).** The first harness version returned
   `None` for the whole factor when the band was missing, overwriting the
   3-component factor — a fixed-interval sparse-history test then scored
   0.19, near round-1 level. Fixed: a missing band degrades to the
   3-component factor, and a component with fewer than 3 pool values is
   dropped on its own instead of failing the whole factor.
2. **Window bug (shipped code, first port).** The first `_eve_share`
   anchored its 7-day window on the last timestamp in the rows passed,
   and a comparable's `hourly` rows run 3 days past release — the window
   slid into release-day data and the shipped CV came out at SIM 0.1369,
   worse than round 3. Fixed: only rows strictly before `day_start`
   count, window anchored on `day_start`. Shipped CV then matched the
   harness (0.1257 vs 0.1255).

### 8.5 GTA VI's own evening share — the extrapolation risk

GTA VI's category history gives eve_share = **1.19** over the last 7
days (169 rows, 24 distinct UTC hours), far below the similar pool's
2.06–2.53: z ≈ **−9.45**. At weight 0.25 an uncapped component would
add −2.36 to the factor — more than the whole round-3 factor's range —
on the strength of a linear mapping learned where |z| never exceeded
1.99 in CV. The z-cap of ±2.5 (above every CV z, so it changes no CV
number: SIM 0.1256 → 0.1255) bounds the final-case bet to about 9% on
the evening peak. At p1 = 300k the factor goes from −8.17 (round 3) to
−8.79 (capped) vs −9.23 clamped (uncapped); the round-3 factor already
bets heavily on a flat curve for GTA VI (its premean/p1 is far below the
pool), so the evening component is a small addition to an already-large
flat bet, not a new one.

### 8.6 Per-fold losses, shipped round 4 vs round 3 vs baseline

| launch | round 4 | round 3 | baseline |
|---|---|---|---|
| Ghost of Yōtei | 0.128 | 0.130 | 0.345 |
| Saros | 0.121 | 0.121 | 0.329 |
| Zelda: Tears of the Kingdom | 0.159 | 0.182 | 0.366 |
| FINAL FANTASY VII REBIRTH | 0.130 | 0.139 | 0.350 |
| The Last of Us Part II | 0.148 | 0.150 | 0.360 |
| Ghost of Tsushima | 0.070 | 0.078 | 0.310 |
| Stellar Blade | 0.123 | 0.125 | 0.346 |
| **Mean** | **0.126** | **0.132** | **0.344** |

Round 4 wins 6 folds and ties Saros (0.121 vs 0.121); paired mean
+0.0066, paired sd 0.008. The biggest single gain is Zelda (0.182 →
0.159), the fold the component was built to fix; no fold got worse by
more than 0.002.

### 8.7 Leave-two-out (21 pairs, 5 training curves)

Round 4: **0.1370**. Round 3 on the same pairs: 0.1451. The gain
survives with one fewer training curve, and the gap (0.0081) is close
to the LOO gap (0.0066), so it is not a 7-fold artefact.

### 8.8 What the remaining error is made of now

Re-running the §8.1 decomposition on the shipped round-4 model: SIM
0.126, oracle median 0.055, oracle σ 0.119, oracle both 0.046. Median
error is still **0.071 (56%)**, spread error **0.007 (5%)**, and the
oracle floor is 0.046. The evening component took the median error
from 0.074 to 0.071 and fixed h13–14 (0.156–0.158 → 0.131–0.132), but
h3 (0.239, sd 0.446, coverage 0.71) and h4 (0.182, sd 0.314) are
unchanged: hour-3/4 viewership is where the launch's draw is still
settling (midnight unlocks rolling across timezones, streamers going
live at different times), and no pre-release signal in this data
predicts it — the diurnal profile correlates with h3/h4 at +0.59/+0.82
in-sample but the per-hour γ fit showed it does not generalise. The
honest statement is that 0.126 is close to this data's ceiling for the
similar-launch class: the oracle floor is 0.046 and the remaining
median error is dominated by early-hour variance that no tested signal
moves. A further round could plausibly only help if it finds a signal
that predicts h3/h4 specifically (e.g. the launch's own minute-level
unlock schedule, which is not in the kit), and with n = 7 any such
signal would need to survive the same LOO + L2O bar that killed the
profile, kNN and quantile-regression attempts here. The bigger risk to
the final score is not the last 0.01 of CV but a single-curve surprise
(a Zelda-type evening, or an hour-1 print that misstates the launch's
draw) — which no further fitting on these 7 curves can insure against.

### 8.9 Verification (round 4)

- LOO through the shipped `main.py`: SIM 0.1257, ALL 0.7869, coverage
  0.91, typical miss 0.197 (harness: 0.1255 / 0.7872).
- L2O through the shipped `main.py`: 0.1370.
- Practice inputs p00–p03: valid output (23 rows, hours 2–24, 5
  non-decreasing quantiles in (0, 1e9], ≤65536 bytes), two runs
  byte-identical, 0.34 s for all four (limit 8 s/case).
- Edge cases: the 19 round-3 cases still pass, plus 5 new
  evening-history cases (missing `history_hourly`, sparse history with
  6 distinct hours, history with all hours present, comparables with
  truncated `hourly`, garbage history rows).
- No secrets in the entry dir outside `.makefun/owner.json` (wallet
  address only).

## 9. Round 5 — hours 3–4, and the pacific share (2026-11-17)

Round 5 was told to attack h3/h4 specifically, the worst remaining
error, with genuinely new mechanisms. Verdict first: **null result —
nothing shipped** at the time of writing. The closest candidate, a
pacific-share early-hour adjustment, wins 7/7 LOO folds and survives
leave-two-out, but gains only +0.0036, below the +0.005 acceptance
rule. `main.py` is unchanged from round 4 (SIM 0.1257). This section
records the investigation so a later round does not redo it, and so
the near-miss is on the record. **Update: the user later decided to
ship the pacific adjustment anyway (7/7 wins was enough) — see §10
for the shipped version and its verification.**

### 9.1 h3/h4 diagnostics (shipped round-4 model, LOO, similar set)

| hour | loss | bias | sd | coverage |
|---|---|---|---|---|
| h2 | 0.115 | +0.081 | 0.215 | 0.86 |
| h3 | 0.239 | −0.104 | 0.446 | 0.71 |
| h4 | 0.182 | −0.138 | 0.314 | 0.86 |
| h5–h19 | 0.115–0.141 | −0.09…+0.06 | 0.22–0.27 | 0.86–1.00 |
| h20–23 | 0.080–0.088 | −0.04…−0.01 | 0.13–0.16 | 1.00 |

h3/h4 are the only hours with residual sd above 0.30; h3's sd (0.446)
is roughly double the day's typical 0.23. The biases (−0.10/−0.14)
are inside one standard error — it is variance, not bias. Per-fold
h3 log-ratios (viewers_h / viewers_1): Yōtei 0.30, Stellar 0.48,
Last of Us 0.54, Zelda 0.67, Tsushima 0.72, Saros 0.75, FFVII 0.91 —
Yōtei is the deep-dip outlier, FFVII the shallow one.

Pooled residuals (161 points): mean −0.030, sd 0.250, skewness
**−0.340** (downside mean −0.215 vs upside +0.176). Oracle per-hour
σ (sd_h × 1.40, in-sample) scores 0.1251 vs shipped 0.1257 — even a
perfect per-hour σ gains only +0.0006, so the spread is not the lever,
as round 4's decomposition already said (spread error 5% of loss).

### 9.2 Night share — degenerate feature

Night share (mean viewers UTC 02–05 over the 7 pre-release days /
daily mean) is near-zero for 5 of the 7 similar curves (Yōtei 0.0033,
Zelda 0.0166, Last of Us 0.0031, Tsushima 0.0043, Stellar 0.028) and
real only for Saros (0.517) and FFVII (0.332). In log space that is
−5.8…−0.7 (pool mean −3.77, sd 1.99): the feature is a near-binary
"has a night audience" flag, and its log is noise at the bottom (a
handful of stray night viewers moves it 10×). Out-of-fold it still
correlates with the shipped h3/h4 residuals (+0.70 / +0.79, n = 7),
driven by the two non-zero curves — but no linear use of it survives
LOO (see §9.3). GTA VI's own night share, anchored on release day
2026-11-19, is 0.66 (z ≈ +1.7 vs the pool): unlike most similar
launches, GTA VI has a real night audience.

### 9.3 Candidate table (LOO CV, mean pinball loss)

Acceptance rule (rounds 3–4): SIM improvement > 0.005 with ≥5/7
folds won/tied, or > 0.01 on paired mean; ALL ≤ 0.80.

| variant | SIM (n=7) | win/tie | paired | ALL | verdict |
|---|---|---|---|---|---|
| Round 4 shipped | 0.1257 | — | — | 0.7869 | baseline |
| night as 5th factor component, w 0.10 | 0.1277 | 1/0 | −0.0021 | 0.7905 | rejected |
| night as 5th factor component, w 0.25 | 0.1302 | 1/0 | −0.0045 | 0.7967 | rejected |
| night as 5th factor component, w 0.50 | 0.1349 | 1/0 | −0.0092 | 0.8016 | rejected |
| night early-separate (h2–6), shrink 0.5 | 0.1246 | 3/0 | +0.0010 | 0.7838 | rejected (below rule) |
| night early-separate (h2–6), shrink 1.0 | 0.1257 | 3/0 | −0.0000 | 0.7840 | rejected |
| night early-separate (h3–4), shrink 0.5 | 0.1238 | 5/0 | +0.0019 | 0.7841 | rejected (below rule) |
| night early-separate (h3–4), shrink 1.0 | 0.1226 | 5/0 | +0.0031 | 0.7825 | rejected (below rule) |
| night early, per-hour γ, shrink 0.5 | 0.1235 | 4/0 | +0.0022 | 0.7838 | rejected |
| night floored at 0.05, early h2–6 | 0.1254 | 4/0 | +0.0011 | 0.7848 | rejected |
| night floored at 0.05, early h3–4 | 0.1245 | 5/0 | +0.0012 | 0.7839 | rejected |
| wide night band (UTC 01–06), early h2–6 | 0.1239 | 5/0 | +0.0018 | 0.7837 | rejected |
| two-piece normal quantiles | 0.1269 | 2/0 | −0.0012 | 0.8124 | rejected (ALL over line) |
| smooth σ bump A 0.3–0.5, w 1.0–1.5 | 0.1253–0.1257 | 2/0 | +0.0000…+0.0004 | 0.7853–0.7861 | rejected (no effect) |
| **pacific early-separate (h2–6), shrink 0.5** | **0.1221** | **7/0** | **+0.0036** | **0.7921** | **nearest miss** |
| pacific early-separate (h2–6), shrink 1.0 | 0.1213 | 6/0 | +0.0044 | 0.7998 | rejected (below rule; ALL on the line) |
| pacific early-separate (h3–4), shrink 1.0 | 0.1256 | 5/0 | +0.0001 | 0.7894 | rejected |
| pacific per-hour γ, shrink 0.5 | 0.1218 | 6/0 | +0.0039 | 0.7972 | rejected (below rule) |
| pacific band 11–13 / 12–14 / 10–14 | 0.1218–0.1228 | 5–7/0 | +0.0029…+0.0039 | 0.7914–0.7925 | rejected (below rule) |

Why the night component fails despite a +0.75 out-of-fold
correlation: the correlation is carried by two points (Saros/FFVII),
and the fitted slopes are unstable on 6 training curves. The
early-separate form (a dedicated slope for the night feature, hours
2–6) is the right structure — the factor's per-hour slopes at h3 are
≈0, so a component folded into the factor is diluted to nothing —
but even so it only gains +0.001–0.003. The two-piece normal fails
despite the −0.34 skew: the fitted σ+/σ− mis-calibrates and ALL
breaches 0.80. The σ bump does nothing, exactly as the oracle
per-hour σ test in §9.1 predicted.

### 9.4 The pacific share — the near-miss, in full

Pacific share (mean viewers UTC 11–14 over the 7 pre-release days /
daily mean) is the share of the fanbase in the first-unlock regions
(New Zealand / Australia evening): Yōtei 1.32, Saros 1.17, Stellar
1.09, Tsushima 0.83, Last of Us 0.78, FFVII 0.65, Zelda 0.52.
Mechanism: a bigger first-wave share means a bigger hour-1 rush
relative to the rest of the day, hence a deeper early dip — the
fitted early slope γ is negative in **every** LOO fold (−0.057…
−0.119 for hours 2–6), i.e. the sign never flips. Per-fold (pacific
h2–6, shrink 0.5, vs shipped): Yōtei 0.1167 vs 0.1277 (+0.0109),
FFVII 0.1250 vs 0.1303 (+0.0054), Saros 0.1170 vs 0.1209 (+0.0040),
Last of Us 0.1458 vs 0.1484 (+0.0026), Zelda 0.1575 vs 0.1586
(+0.0011), Stellar 0.1225 vs 0.1235 (+0.0010), Tsushima 0.0701 vs
0.0704 (+0.0003) — 7/7 wins, paired mean +0.0036, sd 0.0037,
t ≈ +2.6. Leave-two-out (21 pairs): pacific 0.1350 vs shipped
0.1370 (+0.0020) — the gain survives with one fewer training curve.
The effect is robust to the exact band (11–13, 12–14, 10–14 all
0.1218–0.1228, 5–7 wins). It fails only on magnitude: +0.0036 <
+0.005, and the shrink-1.0 variant that reaches +0.0044 pushes ALL
to 0.7998, on the 0.80 line. GTA VI's own pacific share, anchored on
release day 2026-11-19, is 0.60 (pool log-mean −0.14, sd 0.31, so
z ≈ −1.2, low like Zelda/FFVII): in the final case the adjustment
would be small and in the shallower-dip direction.

### 9.5 Joint re-tune (round-4 settings confirmed)

Eve weight {0.15, 0.25, 0.35} × eve cap {2.0, 2.5, 3.0} × p1 σ
{0.8, 1.0, 1.2}, on top of the early adjustment: SIM 0.1236–0.1265,
all within noise of each other and of round-4 settings (eve 0.25,
cap 2.5, p1 σ 1.0); win counts 3–4/7. No re-tune is warranted.

### 9.6 What remains

h3/h4 moved only in the sense that the mechanism is now understood:
the dip depth tracks the first-unlock-region share (pacific) and,
independently, the night audience (night share) — both timezone-mix
proxies — but with 6 training curves per fold neither effect is
large enough to clear the +0.005 bar, and the two together were not
stacked in one harness (each alone is +0.001…+0.004; a combined fit
would need a two-feature early slope on 6 curves, which is the same
small-n wall). The remaining error is still median error at h3/h4
(sd 0.45/0.31 with no signal that moves it by >0.005), plus the
single-curve-surprise risk. A further round could plausibly only
help by (a) stacking pacific + night in one early slope and
accepting a sub-0.005 gain on the strength of 7/7 wins, or (b)
finding a signal that predicts h3/h4 from the launch's own
minute-level unlock schedule, which is not in the kit. Neither is a
modelling failure — it is the n = 7 wall.

### 9.7 Verification (round 5 — no code change)

- `main.py` unchanged from round 4 (commit 07a28f4); `git status`
  clean apart from this file; no `__pycache__` committed.
- LOO through the shipped `main.py` re-run: SIM 0.1257, ALL 0.7869,
  coverage 0.91, typical miss 0.197 — identical to round 4.
- Practice inputs p00–p03: valid output, two runs byte-identical,
  0.36 s for all four (limit 8 s/case).
- No secrets in the entry dir outside `.makefun/owner.json` (wallet
  address only).

## 10. Round 5 shipped — pacific share, on the user's call (2026-11-17)

The user decided to ship the pacific early-separate adjustment despite
the +0.0036 gain sitting below the +0.005 acceptance rule: 7/7 folds won
and the sign never flips was enough. Ported faithfully from the round-5
harness (`/tmp/cv5/v5b.py`, `forecast_b` with `early_feature: pacific`,
`early_hours: (2,3,4,5,6)`, `early_shrink: 0.5`): `_eve_share` generalised
to `_band_share(rows, day_start, lo, hi)`; the pacific share is a 5th
feature (log mean viewers UTC 11–14 / daily mean, 7 pre-release days);
`_pacific_fit` z-scores it on the pool and fits the pooled within-hour
OLS slope over hours 2–6, shrunk 50% toward 0; `adj_lr` subtracts
`gamma * (z_curve - z_target)` at hours 2–6, on top of the factor
adjustment. No z-cap (unlike the evening component): GTA VI's pacific
z is ≈ −1.2, well inside the pool.

### 10.1 Final CV (LOO through the shipped `main.py`)

| | mean pinball loss | 90% coverage | typical miss |
|---|---|---|---|
| Shipped, similar (n=7) | **0.1221** | 0.91 | 0.192 |
| Round 4, same folds | 0.1257 | 0.91 | 0.197 |
| House baseline, same folds | 0.3435 | 0.78 | 0.548 |
| Shipped, all launches (n=37) | 0.7921 | — | — |
| Round 4, all launches | 0.7869 | — | — |

Per-fold (shipped vs round 4): Yōtei 0.1167 vs 0.1277 (+0.0109),
FFVII 0.1250 vs 0.1303 (+0.0054), Saros 0.1170 vs 0.1209 (+0.0040),
Last of Us 0.1458 vs 0.1484 (+0.0026), Zelda 0.1575 vs 0.1586
(+0.0011), Stellar 0.1225 vs 0.1235 (+0.0010), Tsushima 0.0701 vs
0.0704 (+0.0003) — 7/7 wins, paired mean +0.0036, sd 0.0037,
t ≈ +2.6. Leave-two-out (21 pairs, shipped `main.py`): **0.1350** vs
round 4's 0.1370. ALL rose 0.7869 → 0.7921, still under the 0.80 cap
(the pacific adjustment is fitted on the similar pool but applies to
every curve when the similar pool is the fallback; the rise is the
price of the early-hour slope on launch types GTA VI cannot resemble).

### 10.2 Verification (round 5 shipped)

- LOO through the shipped `main.py` re-run: SIM 0.1221, ALL 0.7921,
  coverage 0.91, typical miss 0.192 — identical to the round-5 harness
  numbers (`forecast_b`), per-fold table identical too.
- L2O through the shipped `main.py`: 0.1350 (n=42).
- Practice inputs p00–p03: valid output (23 rows, 5 non-decreasing
  quantiles in (0, 1e9], ≤65536 bytes), two runs byte-identical,
  0.37 s for all four (limit 8 s/case).
- Edge-case suite (19 cases: no comparables, no similar, p1 = 0 /
  negative / string / None / 5 / 5e8, missing known / levels / history,
  garbage levels, broken comparable rows, all-zero viewers, empty case,
  missing channels, non-dict / None case, malformed JSON through
  `main()`): ALL PASS.
- No secrets in the entry dir outside `.makefun/owner.json` (wallet
  address only); no `__pycache__` committed.

## 11. Round 6 — ensembles and variance reduction, target 0.10 (2026-11-17)

The user asked for a sixth round after the dev leaderboard showed the
pay line at 0.1140 and the real top at 0.0952: the target is an
expected score of ~0.10–0.11 (dev top-10 level), down from the shipped
0.1221. The diagnosis says the gap is variance, not mean (oracle
median 0.055, oracle σ 0.119, oracle both 0.046 — median error is 56%
of the loss), so this round attacked variance with ensembles, in the
priority order the brief set. Verdict first: **null result — nothing
ships**. The best candidates cluster at 0.120–0.121, i.e. +0.001–0.002
over shipped, and none clears the acceptance rule in LOO. The one
near-miss is the trimmed-mean family (§11.3), whose case rests on
leave-two-out, not on LOO — it is left for the user to decide, as the
pacific adjustment was in round 5.

The CV harness had to be rebuilt (`/tmp/cv6/cv6.py`, `v6.py`,
`l2o6.py` — /tmp was cleared after round 5). The rebuild was verified
first: it reproduces the shipped numbers exactly (SIM 0.1221, cov
0.91, miss 0.192, ALL 0.7921; per-fold identical to §10.1; baseline
0.3435 / 0.9296).

### 11.1 Ensemble of median estimators — the weak members drag it down

Members tested alone (LOO SIM): raw p1-weighted median (no factor)
0.2002 (cov 0.71), factor-only 0.1261, weighted mean of full-adjusted
log-ratios 0.1294, trimmed weighted mean ("trim": drop min/max per
hour, weighted mean of the middle 4) **0.1214** (cov 0.90),
unweighted median 0.1266, wide p1 kernel (σ=2.0) 0.1232, narrow kernel
(σ=0.5) 0.1334, kNN in factor space (own implementation, k=3)
0.1781, all-pool median 0.5017 (SIM) / 0.5931 (ALL).

| variant | SIM (n=7) | cov | ALL | verdict |
|---|---|---|---|---|
| Round 5 shipped | 0.1221 | 0.91 | 0.7921 | baseline |
| full .5 + raw .5 | 0.1501 | 0.85 | 0.7722 | rejected |
| full .5 + raw .25 + knn .25 | 0.1451 | 0.86 | 0.7807 | rejected |
| full .5 + raw .25 + mean .25 | 0.1356 | 0.88 | 0.7835 | rejected |
| full .4 + raw .2 + knn .2 + mean .2 | 0.1408 | 0.89 | 0.7841 | rejected |
| full .5 + knn .5 | 0.1424 | 0.88 | 0.7891 | rejected |
| full .6 + mean .2 + unw .2 | 0.1233 | 0.89 | 0.7990 | rejected |
| full .5 + wide .5 | 0.1222 | 0.90 | 0.7981 | rejected |
| full .7 + allpool .3 | 0.1783 | 0.91 | 0.6041 | rejected |
| 9-model-ish (6 members) | 0.1377 | 0.89 | 0.7881 | rejected |
| full .5 + raw .3 + allpool .2 | 0.1417 | 0.91 | 0.6403 | rejected |
| full .34 + raw .33 + mean .33 | 0.1409 | 0.86 | 0.7805 | rejected |
| median across (full, trim, mean) | 0.1231 | 0.90 | 0.7947 | rejected |
| median across (full, trim, wide, mean) | 0.1229 | 0.89 | 0.7978 | rejected |
| median across (full, raw, trim) | 0.1233 | 0.91 | 0.7894 | rejected |
| midmean alone / .5 + full .5 | 0.1241 / 0.1222 | 0.92 / 0.90 | 0.8190 / 0.8045 | rejected (ALL over line) |

Lesson: averaging log-medians only helps when the members are close
to the full model. The raw member (0.2002, cov 0.71), kNN (0.1781)
and all-pool (0.5017) are too far away — blending them injects bias,
not variance reduction. The leaderboard's "9-model ensemble" works
because its members are all decent; ours are not diverse *and* good,
they are diverse *or* good. The only members within ~0.005 of shipped
are trim, wide and mean, and blending those gives at most +0.0015.

### 11.2 Bagging, winsorising, bias correction, shape/level — all flat

| variant | SIM | cov | ALL | verdict |
|---|---|---|---|---|
| bagged full median, 25 reps, alone | 0.1325 | 0.87 | 0.8054 | rejected (worse; ALL over line) |
| full .5 + bagged .5 | 0.1262 | 0.89 | 0.7987 | rejected |
| trim .5 + bagged .5 | 0.1245 | 0.90 | 0.7999 | rejected |
| full winsorised (0.10, 0.90) | 0.1218 | 0.91 | 0.7939 | rejected (+0.0003) |
| full winsorised (0.25, 0.75) | 0.1213 | 0.89 | 0.7992 | rejected (+0.0008) |
| full + nested bias correction, shrink 0.5 | 0.1220 | 0.91 | 0.7886 | rejected (+0.0001) |
| full + nested bias correction, shrink 0.25 | 0.1220 | 0.90 | 0.7911 | rejected |
| level from full + shape from raw | 0.1391 | 0.92 | 0.7728 | rejected |
| level from full + shape from trim | 0.1217 | 0.91 | 0.8015 | rejected (ALL over line) |
| shipped, no smoothing | 0.1244 | 0.86 | 0.7976 | rejected |
| shipped, wide (5-point) smoothing | 0.1232 | 0.93 | 0.7886 | rejected |

Bagging hurts: bootstrap resampling reweights the 6 curves at random
and destabilises the factor fit on each replicate (the factor fit on a
bootstrap sample with only 3–4 unique curves is noisy), and the
average of those medians does not recover the original median. The
nested bias correction confirms round 3's verdict — per-hour bias is
inside one standard error, so there is nothing to correct (this time
measured properly: each training curve forecast from the other five,
not in-sample). Smoothing: the standard [0.25, 0.5, 0.25] is right;
none and 5-point are both worse.

### 11.3 The near-miss: the trimmed-mean family

Trim = per hour, drop the min and max adjusted log-ratio, weighted
mean of the middle 4 (p1-kernel weights kept). It is the only
aggregation that beats the weighted median on its own (0.1214 vs
0.1221), and the blends cluster at 0.1206–0.1208:

| variant | SIM | cov | ALL | LOO wins | L2O |
|---|---|---|---|---|---|
| trim alone | 0.1214 | 0.90 | 0.7968 | 4/7 | — |
| full .5 + trim .5 (E13) | 0.1206 | 0.89 | 0.7962 | 4/7 | 0.1304, 31/42 wins |
| trim .75 + full .25 (T1) | 0.1208 | 0.90 | 0.7955 | 4/7 | 0.1296, 28/42 wins |
| trim .67 + full .33 | 0.1207 | 0.89 | 0.7955 | 4/7 | — |
| E13 + winsor (0.1, 0.9) | 0.1206 | 0.90 | 0.7972 | — | — |
| E13, σ ×1.50 | 0.1213 | 0.94 | 0.7894 | — | — |
| E13, σ ×1.60 | 0.1223 | 0.94 | 0.7829 | — | — |

Per-fold (E13 vs shipped): FFVII 0.1229 (+0.0021), Saros 0.1149
(+0.0021), Stellar 0.1205 (+0.0020), Last of Us 0.1350 (+0.0108),
Tsushima 0.0715 (−0.0014), Yōtei 0.1170 (−0.0003), Zelda 0.1627
(−0.0052). The gain is concentrated in one fold (Last of Us, the
flattest curve: trimming drops the steepest curves, so the centre
moves flatter) and Zelda, the steepest curve, pays for it. In L2O
(5-curve pools) the picture is broader — T1 wins 28/42 pairings
(+0.0054) and improves 6 of 7 curves (only Zelda loses, −0.0147 over
6 pairings; Last of Us +0.1144, Saros +0.0423, Tsushima +0.0322);
E13 wins 31/42 (+0.0045). The mechanism is real variance reduction,
and it strengthens as the pool shrinks — which is also the caveat:
the final pool has 11 similar curves, nearly twice the CV pools, so
the final-case gain is likely *smaller* than the LOO +0.0013–0.0015,
not larger. Expected final-case gain: +0.000 to +0.002. It does not
pass the acceptance rule (LOO +0.0015 < +0.005, 4/7 wins < 5, and
E13's coverage 0.89 is under the 0.90 floor — restoring coverage
with σ ×1.50 costs the whole gain, 0.1213).

### 11.4 Eve weight 0.50 — a one-fold story, again

Re-tested on the round-5 base (round 5's joint re-tune only went to
0.35): eve w 0.40 gives 0.1215 (cov 0.88), w 0.50 gives 0.1206 (cov
0.89, ALL 0.7915). Per-fold for w 0.50: Zelda 0.1404 (+0.0171),
FFVII 0.1207 (+0.0043), and losses on the other five folds
(Tsushima −0.0022, Yōtei −0.0024, Saros −0.0036, Stellar −0.0026,
Last of Us −0.0001) — 2/7 wins. The entire gain is Zelda alone, and
a better-centred median shrinks the residuals, so coverage falls to
0.89; σ ×1.50 restores it (cov 0.94) at 0.1209. Stacking eve 0.50
with trim (full .5 + trim .5, σ ×1.50) gives the round's best SIM,
0.1204 (cov 0.93, ALL 0.7897, L2O 0.1276, 32/42 wins), and eve 0.50
with pacific shrink 1.0 gives 0.1205 (cov 0.89, ALL 0.7985, 4/7
wins) — but stacking two sub-threshold effects that each fix one
fold is exactly the noise-chasing the acceptance rule exists to
stop, so neither ships.

### 11.5 Is 0.10 within reach? No — not with this data

The round's best honest number is 0.1204 (stacked sub-threshold
effects) or 0.1206 (single near-miss). The target needs −0.022 from
0.1221. The oracle floor is 0.046, so 0.10 is not theoretically
impossible — median error would have to fall from 0.067 to ~0.045,
a one-third cut — but no tested mechanism gets there, and the
mechanisms that do move the loss move it by +0.001–0.002, an order
of magnitude short. The remaining median error is idiosyncratic
curve shape: each launch deviates from the common median by a
typical |log residual| of ~0.21, and no hour-1-anchored signal in
this kit predicts which way a given launch deviates. The dev
leaderboard's real top (0.0952) is scored on the 4 practice
targets — four huge, flat, possibly easier launches — not on the
same 7-fold mean as our 0.1221, so the gap looks bigger than it is;
but even allowing for that, a 0.10 LOO mean is not on the table.

Two honest mitigations, neither a model change: (1) the final pool
has 11 similar curves vs 6 per CV fold, so the final-case loss
should already run a little better than the LOO 0.1221 — the
generalisation gap shrinks with pool size, and that improvement
arrives for free; (2) the single-case variance dominates everything
else — our folds span 0.070 (Tsushima) to 0.158 (Zelda), so the
final score is one draw from that range, and no further fitting on
these 7 curves insures against a Zelda-type day.

### 11.6 Verification (round 6 — no code change)

- `main.py` unchanged from round 5 (commit 72dd2e8); `git status`
  clean apart from this file; no `__pycache__` committed.
- LOO through the shipped `main.py` re-run: SIM 0.1221, ALL 0.7921,
  coverage 0.91, typical miss 0.192 — identical to round 5.
- Practice inputs p00–p03: valid output, two runs byte-identical,
  0.37 s for all four (limit 8 s/case).
- Edge-case suite re-run (19 cases, rebuilt in /tmp/cv6/edge6.py):
  ALL PASS.
- No secrets in the entry dir outside `.makefun/owner.json` (wallet
  address only).

**Update: the user later decided to ship the trim blend anyway
(E13, full .5 + trim .5) — see §12 for the shipped version and its
verification.**

## 12. Round 6 shipped — trim blend, on the user's call (2026-11-17)

Ported faithfully from the round-6 harness (`/tmp/cv6/v6.py`,
`forecast_v6` with `members=[("full", 0.5), ("trim", 0.5)]`,
`combine="mean"`, `smooth="std"`, `sigma_inflate=1.40`, no winsor,
no bias correction): a `_trim_mean` helper drops the min and max
adjusted log-ratio per hour and takes the p1-weighted mean of the
rest (with ≤3 samples it is the plain weighted mean); the median
loop now builds each hour's samples once and sets `med[h] = 0.5 *
wquantile(samples, 0.5) + 0.5 * trim_mean(samples)`. The spread is
unchanged: pooled IQR of the full-adjusted log-ratios around the
blended smoothed median, ×1.40, floored at 0.15 — exactly as the
harness computes it.

### 12.1 Final CV (LOO through the shipped `main.py`)

| | mean pinball loss | 90% coverage | typical miss |
|---|---|---|---|
| Shipped, similar (n=7) | **0.1206** | 0.89 | 0.190 |
| Round 5, same folds | 0.1221 | 0.91 | 0.192 |
| House baseline, same folds | 0.3435 | 0.78 | 0.548 |
| Shipped, all launches (n=37) | 0.7962 | — | — |
| Round 5, all launches | 0.7921 | — | — |

Per-fold (shipped vs round 5): Yōtei 0.1170 vs 0.1167 (−0.0003),
Saros 0.1149 vs 0.1170 (+0.0021), Zelda 0.1627 vs 0.1575 (−0.0052),
FFVII 0.1229 vs 0.1250 (+0.0021), Last of Us 0.1350 vs 0.1458
(+0.0108), Tsushima 0.0715 vs 0.0701 (−0.0014), Stellar 0.1205 vs
0.1225 (+0.0020) — 4/7 wins, paired mean +0.0015. Leave-two-out
(21 pairs, shipped `main.py`): **0.1304** vs round 5's 0.1350
(31/42 wins). ALL rose 0.7921 → 0.7962, still under the 0.80 cap.
Coverage fell 0.91 → 0.89, under the 0.90 floor: the better-centred
median shrinks the residuals, and restoring coverage with σ ×1.50
costs the whole gain (0.1213) — the user shipped it knowing that.

### 12.2 Verification (round 6 shipped)

- LOO through the shipped `main.py` re-run: SIM 0.1206, ALL 0.7962,
  coverage 0.89, typical miss 0.190 — identical to the round-6
  harness numbers (`forecast_v6`, E13), per-fold table identical too.
- L2O through the shipped `main.py`: 0.1304 (n=42).
- Practice inputs p00–p03: valid output (23 rows, 5 non-decreasing
  quantiles in (0, 1e9], ≤65536 bytes), two runs byte-identical,
  0.24 s for all four (limit 8 s/case).
- Edge-case suite (19 cases, /tmp/cv6/edge6.py): ALL PASS.
- No secrets in the entry dir outside `.makefun/owner.json` (wallet
  address only); no `__pycache__` committed.
