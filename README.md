# ETP Market Making

Quantitative analysis of leveraged ETP market making mechanics,
calibrated on real intraday events — built from a market maker perspective.

All modules use real 1-minute intraday data captured on 23 March 2026,
the day Brent fell -13.3% in under 2 minutes following a Trump Truth Social
post on Iran ceasefire talks.

---

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

**Key findings:**
- Restrike was not triggered today — the underlying held above -20% (66.3% max proximity)
- The 3BRL lost ~39.8% intraday purely from 3x leverage amplification, with no reset
- The MM suspension (theoretical) was entirely vol-driven: realized vol hit 300% ann. vs a 54% session baseline (6x)
- Spreads reached 800 bps (44x normal)
- The spread model `18 × (rvol/20)^2.2` captures the superlinear relationship between vol and spreads:
  spreads explode faster than vol because the MM faces both wider futures spreads AND larger delta exposure simultaneously
- Using a relative threshold (3x session baseline) rather than a fixed absolute threshold is critical
  in a prolonged stress regime — the Iran war context since 28 Feb had already elevated baseline vol to 54% ann.

---

## Output

Live dashboard → [spread_model.html](https://benjadeville.github.io/etp-mm/1_spread_model/spread_model.html)

# Module 2 — Delta Hedging Cost

Quantitative analysis of market maker delta hedging costs on a 3x leveraged
ETP during an extreme intraday event — calibrated on 23 March 2026.

---

## Context

At 11:08 London time, Brent 2nd month future fell -13.3% in under 2 minutes.
The MM was forced to rebalance from **278 to 320 futures contracts** (+42 contracts)
to maintain delta neutrality on a hypothetical $10M 3BRL exposure.

Total hedging cost: **201.5 bps** — of which **94% came from gamma loss** (190 bps),
not transaction costs (11 bps).

---

## Key concept — why short gamma dominates

A leveraged ETP market maker is structurally **short gamma**:
- MM sells 3BRL to clients and hedges with Brent futures
- When Brent moves sharply, the MM must rebalance at unfavorable prices
- The 3x daily reset forces continuous delta rebalancing
- Path variance — not bid-ask spreads — is the dominant cost

This is why MMs widen spreads dramatically in stress: they are pricing in
expected gamma loss, not just transaction costs.

---

## What this notebook models

### 1. Data
- Source: `yfinance` — ticker `BZ=F` (Brent front future, proxy for 2nd month)
- Granularity: 1-minute intraday
- Window: 23 March 2026

### 2. Computed series
- **Delta** — number of futures contracts to hedge `(AUM × leverage) / (contract_size × spot)`
- **Delta change** — contracts to buy/sell each bar to rebalance
- **Futures spread** — widens with vol: 1 tick normal → 5 ticks at stress peak
- **Transaction cost** — `delta_change × futures_spread` per bar, cumulated
- **Gamma loss** — `0.5 × leverage × return² × notional` per bar, cumulated

### 3. Parameters
| Parameter | Value |
|---|---|
| AUM exposure | $10,000,000 |
| Leverage | 3x |
| Gross exposure | $30,000,000 |
| Contract size | 1,000 barrels |
| Normal futures spread | 1 tick |
| Stress futures spread | 5 ticks (at 300% ann. vol) |

---

## Session results — 23 March 2026

| Metric | Value |
|---|---|
| Delta at open | 278 contracts |
| Delta at session low | 320 contracts |
| Delta rebalance | +42 contracts |
| Max futures spread | 5 ticks |
| Transaction costs | 11.4 bps |
| Gamma loss | 190.1 bps |
| Total MM cost | 201.5 bps |
| Gamma % of total | 94% |

**Key findings:**
- Gamma loss (190 bps) was 17x larger than transaction costs (11 bps) on a -13.3% Brent move
- 94% of total MM hedging cost came from short gamma exposure, not bid-ask spreads
- Delta increased mechanically from 278 to 320 contracts (+42) as Brent fell —
  the MM was forced to buy futures into a falling market to maintain delta neutrality
- This is the core paradox of leveraged ETP market making: the MM must buy when the market
  falls and sell when it rises, systematically trading against momentum at the worst prices
- Futures bid-ask spread widened from 1 to 5 ticks at the vol peak — but this was marginal
  compared to gamma loss, confirming that spread widening is primarily driven by gamma risk
  pricing, not by the MM's own transaction costs
- Practical implication: a MM quoting 800 bps spreads on 3BRL during this event
  was not being opportunistic — 201 bps of realized hedging cost justifies wide spreads
  even before accounting for inventory risk and gap risk

---

## Output

Live dashboard → [delta_hedging.html](https://benjadeville.github.io/etp-mm/2_delta_hedging/delta_hedging.html)

# Module 3 — Liquidity Gap

Analysis of order book disappearance and execution slippage during the
6-minute liquidity gap triggered by the Trump/Iran spike on 23 March 2026.

---

## Context

At 11:05 London time, Brent fell -13.3% in under 2 minutes triggering a
**6-minute liquidity gap** (liquidity score 97.4/100 — near-total book disappearance).

A retail investor trying to exit 3BRL during this window would have paid
**~215 bps in slippage** — 12x the normal 18 bps execution cost.
Combined with the 800 bps MM spread from Module 1, total execution cost
at worst: **~1,015 bps**.

---

## What this notebook models

### 1. Data
- Source: `yfinance` — ticker `BZ=F` (Brent front future, proxy for 2nd month)
- Granularity: 1-minute intraday
- Window: 23 March 2026

### 2. Computed series
- **Price gap** — bar-to-bar price jump (absolute %)
- **Volume ratio** — volume vs rolling 20-bar median
- **Amihud illiquidity ratio** — price impact per dollar traded
- **Liquidity score** — composite 0-100 combining vol, price gap, Amihud
- **Execution slippage** — estimated additional cost for a market order in gap

### 3. Liquidity score methodology
```
score = vol_score (40pts) + gap_score (30pts) + amihud_score (30pts)
```
| Component | Max | Condition |
|---|---|---|
| Realized vol | 40 pts | 300% ann. = max |
| Bar price gap | 30 pts | 2% gap = max |
| Amihud ratio | 30 pts | 95th percentile = max |

Score > 70 → liquidity gap flagged.

---

## Session results — 23 March 2026

| Metric | Value |
|---|---|
| Session low | $93.62 (-13.3%) |
| Max bar price gap | 5.08% |
| Normal volume | 52 contracts/min |
| Max liquidity score | 97.4/100 |
| Gap events (score > 70) | 6 minutes |
| Gap window | 11:05 → 12:30 |
| Max Amihud ratio | 284.47 |
| Normal slippage | 18 bps |
| Max slippage | 263 bps |
| Avg slippage in gap | 215 bps (12x normal) |
| Combined worst cost | ~1,015 bps (spread + slippage) |

**Key findings:**
- Liquidity score reached 97.4/100 — near-complete order book disappearance
- 6-minute gap window where the MM would absobe the risk
- Amihud ratio of 284.47 confirms extreme price impact per dollar traded
- Retail investors exiting 3BRL during the gap paid 12x normal execution costs
- Combined with Module 1 spreads (800 bps), total exit cost reached ~1,015 bps at worst
- The Amihud ratio is the most sensitive signal — it spikes before the liquidity score
  crosses 70, making it a useful early warning indicator

---

## Output

Live dashboard → [liquidity_gap.html](https://benjadeville.github.io/etp-mm/3_liquidity_gap/liquidity_gap.html)

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
jupyter notebook 2_delta_hedging/delta_hedging.ipynb
jupyter notebook 3_liquidity_gap/liquidity_gap.ipynb
```

> **Note on data availability:** yfinance provides 1-min intraday data
> for the last 7 days only. The notebook was run on 23 March 2026 to
> capture the Trump/Iran spike in real time. For reproducibility,
> the HTML output are committed to the repo.