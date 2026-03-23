# Module 1 — ETP Spread Model

Real-data analysis of market maker spread dynamics on a 3x leveraged ETP
during an extreme intraday event — calibrated on 23 March 2026.

---

## Context

At 11:08 London time, Trump posted on Truth Social that the US and Iran had
held "very good and productive conversations" on ending hostilities.

Brent 2nd month future: **$107.92 → $93.62 (-13.3%) in under 2 minutes.**
3BRL (WisdomTree 3x Daily Brent) implied intraday move: **-39.8%**.

The market maker desk suspended quotes for 56 minutes —
not because of a restrike, but because of realized vol explosion
and delta hedging difficulty on the underlying futures.

---

## Product specs — 3BRL

| Parameter | Value |
|---|---|
| Issuer | WisdomTree |
| Underlying | ICE Brent 2nd month future |
| Leverage | 3x daily |
| Restrike trigger | -20% intraday on the **underlying** (not on 3BRL) |
| Restrike effect | Intraday reset — new base price, leverage recalculated |
| Exchange | LSE (3BRL.L) |

**Why the 2nd month?** Tracking the 2nd month future (vs front contract) reduces
roll frequency and avoids expiry-related vol spikes that would trigger restrikes
more often. Tradeoff: slightly less liquid than the front in stress regimes.

---

## What this notebook models

### 1. Data
- Source: `yfinance` — ticker `BZ=F` (Brent front future, proxy for 2nd month)
- Granularity: 1-minute intraday
- Window: 23 March 2026 (captured same day — yfinance 1min window = 7 days)

### 2. Computed series
- **Log returns** — 1-min
- **Realized vol** — rolling 20-bar (20min), annualized × √(252 × 390)
- **Intraday drawdown** — from open price $107.92
- **Restrike proximity** — % of -20% barrier reached on underlying
- **MM spread** — power law model: `spread = 18 × (rvol/20)^2.2`, clipped 800 bps

### 3. Spread model
```
spread_bps = base_spread × (realized_vol / vol_normal) ^ β
```
- `base_spread` = 18 bps (normal market)
- `vol_normal` = 20% annualized
- `β` = 2.2 (superlinear — spreads explode faster than vol)
- Cap at 800 bps (consistent with observed MM behavior)

### 4. Book suspension methodology
Absolute vol thresholds are not meaningful in a prolonged stress regime
(Iran war context since 28 Feb). Instead, suspension is flagged when
realized vol exceeds **3x the session baseline** (session median):

- Session vol baseline: **54% ann.** (median)
- Suspension threshold: **162% ann.** (3x baseline)
- More robust than a fixed threshold in war-driven vol environments

---

## Session results — 23 March 2026

| Metric | Value |
|---|---|
| Brent 2nd future open | $107.92 |
| Session low | $93.62 (-13.3%) |
| Session close | $101.59 (-5.9%) |
| Restrike barrier | $86.34 — **not breached** |
| Max restrike proximity | 66.3% |
| 3BRL intraday low (est.) | -39.8% |
| 3BRL end of day (est.) | -17.6% |
| Normal MM spread | 18 bps |
| Vol baseline (session median) | 54% ann. |
| Suspension threshold | 162% ann. (3x baseline) |
| Spread at spike | 800 bps (44x normal) |
| Realized vol at spike | ~300% ann. |
| Book suspended | 56 min (11:05 → 12:49) |

**Key finding:** restrike was not triggered today — the underlying held above
-20%. The 3BRL lost ~39.8% intraday purely from 3x leverage amplification.
The MM suspension was entirely vol-driven, not restrike-driven.

---

## Output

Live dashboard → [spread_model.html](https://benjadeville.github.io/etp-mm/1_spread_model/spread_model.html)

---

## Stack
```
Python · yfinance · pandas · numpy · matplotlib · scipy
```

---

## Run it
```bash
pip install yfinance pandas numpy matplotlib scipy
jupyter notebook 1_spread_model/spread_model.ipynb
```

> **Note on data availability:** yfinance provides 1-min intraday data
> for the last 7 days only. The notebook was run on 23 March 2026 to
> capture the Trump/Iran spike in real time. For reproducibility,
> the chart and HTML output are committed to the repo.