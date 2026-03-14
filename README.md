# LNG Frontier Optimizer

**v7.2 — Full Fundamentals Edition**

A browser-based pre-trade analysis dashboard for LNG cargo economics. Calculates real-time P&L, break-even prices, route optimization (Suez vs Cape), cargo diversion signals, and Monte Carlo risk metrics for Henry Hub → TTF (Europe) and Henry Hub → JKM (Asia-Pacific) voyages.

Built entirely with React and free-tier public APIs. No backend required.

---

## Overview

The tool addresses a specific operational problem: given a US Gulf Coast LNG cargo loaded under an SPA contract, which destination maximizes margin after accounting for all voyage costs — and what is the risk profile of that decision?

It is designed as a **pre-trade scenario planning tool**, not an execution system. Every data point carries an explicit quality label (LIVE / ~PROXY / SIM) so the user always knows the epistemic basis of each number.

---

## Data Architecture

| Variable | Source | Quality | Latency | Notes |
|---|---|---|---|---|
| Henry Hub spot | FRED API `DHHNGSP` | ✅ LIVE | ~1 business day | Federal Reserve St. Louis |
| Brent crude | FRED API `DCOILBRENTEU` | ✅ LIVE | ~1 business day | Anchor for HFO and JKM proxy |
| EUR/USD | BCE via Frankfurter | ✅ LIVE | ~60 seconds | Daily ECB fixing |
| NG Futures M1–M4 | EIA API v2 `RNGC1–4` | ✅ LIVE | Daily | Real NYMEX forward curve |
| US NG Storage | EIA API v2 | ✅ LIVE | Weekly | Working gas Bcf, HH leading indicator |
| US LNG Exports | EIA API v2 | ✅ LIVE | Weekly | Weekly volumes, JKM supply signal |
| EU Gas Storage | GIE AGSI+ | ✅ LIVE | Daily (19:30 CET) | Fill %, injection/withdrawal, TTF leading indicator |
| EU LNG Terminals | GIE ALSI+ | ✅ LIVE | Daily | Sendout GWh/d, inventory, TTF bearish signal |
| TTF front-month | Yahoo / ICE TTF=F via Cloudflare Worker | ⚡ PROXY | ~15 min | CORS proxy required |
| HFO Rotterdam | Brent × 6.80 + 115 | ~ PROXY | Derived | VLSFO formula, R²≈0.89 vs Platts 2019–2025 |
| JKM | HH + Brent 3-regime regression | ~ PROXY | Derived | R²≈0.71 normal regime, ±10–15% error |

**Data not available for free:** JKM real-time (Platts), charter rates live (Clarksons/Baltic), intraday TTF granularity.

---

## Features

### P&L Engine
- Full voyage cost model: SPA acquisition, charter (laden + ballast), HFO fuel (cubic speed model), BOG opportunity cost, regas fees, port costs, Suez/war risk surcharges, SOFR capital cost
- Break-even price calculation per route
- Dual-fuel BOG model: BOG burns as fuel when sell price < HFO equivalent
- B/L price lock for hedged positions
- Dynamic HFO price from Brent (was hardcoded $480/MT in older versions)

### Route Optimization
- EU (Rotterdam TTF) vs AP (Tokyo JKM) margin comparison
- Suez Canal vs Cape of Good Hope break-even analysis
- Cargo diversion signal at configurable voyage progress (default 50%)
- Sea state multiplier on boil-off rate (Beaufort scale)

### Break-Even Matrix
- 7×7 sensitivity matrix: HH supply cost (rows) × destination price (cols)
- **Dynamic range:** automatically adjusts to always show loss zones, regardless of current market spread
- Hover tooltip with full BEP and acquisition cost breakdown

### Monte Carlo Risk — N=3,000
- Ornstein-Uhlenbeck price simulation with Markov regime switching (Normal/Crisis)
- Correlated shocks: HH / TTF / JKM via Cholesky decomposition (ρ matrix calibrated)
- Merton jump diffusion with Student-t(ν=4) jump sizes (fat tails)
- Charter rate crisis correlation: ρ≈+0.65 with TTF jump events
- Seedable PRNG (Mulberry32) for reproducible results
- VaR95, VaR99, CVaR95, CVaR99, Sharpe ratio, P(profit)

### Fundamental Signals Panel
- EU gas storage fill % gauge with TTF bullish/bearish signal
- ALSI+ LNG terminal sendout (high sendout → TTF bearish)
- US storage surplus vs year-ago (HH bullish/bearish)
- US LNG export volumes vs 4-week average (JKM supply signal)

### Forward Curve
- M1–M4: real EIA NYMEX futures (filled dots)
- M5–M12: OU simulation anchored to real prices (open dots)
- Contango / Backwardation / Flat classification

### Scenario Tools
- Stress tests: Black Sea Closure (TTF ×3), US Gulf Hurricane (HH +30%), LNG Glut (HH −25%)
- Tornado chart: ±1% sensitivity on all major cost drivers
- Charter rate sensitivity curve
- Half-Kelly position sizing hint (EU route)

---

## Accuracy Assessment

| Variable | Estimated Error | Impact per Cargo |
|---|---|---|
| HH spot | ±0% (FRED real) | $0 |
| TTF | ±2–4% (15min lag) | ~$150k |
| JKM | ±10–15% (formula) | ±$700k–$1.5M |
| HFO | ±5–8% (Brent formula) | ~$150k |
| Charter | ±50–100% (manual input) | **±$1M–$3M** |
| **Total P&L EU** | **~±5% if charter calibrated** | **~±$500k** |
| **Total P&L AP** | **~±15%** | **~±$2M** |

**Practical conclusion:** The EU route P&L is operationally reliable if the charter rate is updated to the current Baltic LNG Index level. The AP route should be treated as an indicative range, not a point estimate, due to JKM proxy limitations.

---

## Tech Stack

- **React 18** — UI framework, single-file component
- **Recharts** — all charts (LineChart, AreaChart, BarChart, ComposedChart)
- **Vite** — build tooling (recommended)
- No backend, no database, no authentication layer

---

## Setup

### Prerequisites
- Node.js ≥ 18
- A Vite + React project scaffold

### Installation

```bash
git clone https://github.com/YOUR_USERNAME/lng-frontier-optimizer
cd lng-frontier-optimizer
npm install
```

Replace the generated `src/App.jsx` with the project `App.jsx`, then:

```bash
npm run dev
```

### API Keys

The application requires three free API keys. Replace the values in the `API` constant at the top of `App.jsx`:

```javascript
const API = {
  FRED: "YOUR_FRED_KEY",   // register at fred.stlouisfed.org/docs/api/api_key.html
  EIA:  "YOUR_EIA_KEY",    // register at eia.gov/opendata/
  GIE:  "YOUR_GIE_KEY",    // register at agsi.gie.eu/account (select AGSI + ALSI)
};
```

All three keys are free and require only email registration. No credit card required.

### TTF Proxy (Cloudflare Worker)

Yahoo Finance does not allow direct browser fetch due to CORS. A Cloudflare Worker proxy is required for live TTF data. Without it, TTF falls back to OU simulation.

1. Create a free account at [cloudflare.com](https://cloudflare.com) (no credit card required, 100k requests/day free)
2. Go to **Workers & Pages → Create application → Create Worker**
3. Replace the default code with:

```javascript
export default {
  async fetch(request) {
    const url =
      "https://query1.finance.yahoo.com/v8/finance/chart/TTF=F" +
      "?interval=1d&range=5d";
    try {
      const resp = await fetch(url, {
        headers: {
          "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)",
          "Accept": "application/json",
        },
      });
      const data = await resp.json();
      return new Response(JSON.stringify(data), {
        headers: {
          "Content-Type": "application/json",
          "Access-Control-Allow-Origin": "*",
          "Cache-Control": "max-age=900",
        },
      });
    } catch (e) {
      return new Response(JSON.stringify({ error: e.message }), {
        status: 502,
        headers: {
          "Content-Type": "application/json",
          "Access-Control-Allow-Origin": "*",
        },
      });
    }
  },
};
```

4. Click **Save and Deploy**
5. Copy your Worker URL (e.g. `https://ttf-proxy.yourname.workers.dev/`) and replace the endpoint in `fetchTTFProxy()` inside `App.jsx`

---

## Configuration

All physical constants and model parameters are centralized in the `CFG` object at the top of `App.jsx`. No rebuild is required if you edit these directly in the UI sidebar (Scenario Editor panel, toggle with the ▶ button).

Key parameters to calibrate before use:

| Parameter | Default | Notes |
|---|---|---|
| Charter Rate | $85,000/day | **Update daily from Baltic LNG Index** |
| SOFR Rate | 5.30% | US fed funds rate proxy, update quarterly |
| HFO Crack Spread | $115/MT | VLSFO Rotterdam premium over Brent |
| Suez Canal Fee | $500,000 | SCZONE published rates, update monthly |
| SPA Slope | 1.115 | Your actual contract HH multiplier |
| Tolling Fee | $2.50/MMBtu | Your actual tolling agreement |

---

## Physical Model

### Boil-Off Rate
```
BOR_effective = BOR_nominal × thermal_factor × sea_state_multiplier
BOG = load_MMBtu × (1 - (1 - BOR_eff)^travel_days)
Delivered = Load - BOG
```
Sea state multipliers: Calm ×1.00 / Moderate ×1.20 / Rough ×1.50 / Storm ×1.80

### Fuel Consumption (cubic speed model)
```
fuel_MT/day = FUEL_COEFF × speed_knots³
```
Calibrated so that 19.5 knots yields ~97 MT/day laden (Q-Flex 174k m³ spec).
BOG displaces HFO: net HFO = max(0, fuel_required − BOG_HFO_equivalent)

### BOG Opportunity Cost
```
BOG_opp_cost = BOG_MMBtu × max(0, sell_price − HFO_per_MMBtu)
```
When sell price < HFO equivalent ($15.10/MMBtu at current HFO $611/MT), BOG opportunity cost is zero — burning BOG is economically free relative to HFO.

### JKM Proxy (3-regime regression)
```
Glut   (HH<2.8, Brent<72):  JKM = 1.30×HH + 0.060×Brent + 1.5
Normal (default):            JKM = 1.55×HH + 0.070×Brent + 2.0
Crisis (Brent>92 or HH>5):  JKM = 2.00×HH + 0.140×Brent + 3.5
```
Recalibrated on 2023–2025 data to reflect LNG structural oversupply. R²≈0.71 in normal regime. Error: ±10–15%.

### HFO Price Proxy
```
HFO_VLSFO ($/MT) = Brent ($/bbl) × 6.80 + 115
```
Historical calibration: Rotterdam VLSFO 2019–2025, R²≈0.89.

---

## Monte Carlo Model

The simulation uses an Ornstein-Uhlenbeck process with Merton jump diffusion:

```
dS = κ(μ−S)dt + σS·dW + J·dN(λ)
```

Where:
- **κ = 0.18** — mean-reversion speed (anchored to real prices, not hardcoded)
- **σ** — regime-adjusted volatility (×2.8 in Crisis regime)
- **J** — jump size drawn from Student-t(ν=4) — fat tails vs Gaussian
- **λ** — jump intensity (default 0.5/year, configurable)
- **dN** — Poisson process; Hawkes-like: P(JKM jump | TTF jump) += 60%

Price correlations: HH/TTF ρ=0.65, HH/JKM ρ=0.55, TTF/JKM ρ=0.70 (Cholesky decomposed).

Charter crisis correlation: ρ≈+0.65 with TTF jump events (Baltic LNG Index historically spikes with supply disruptions).

PRNG: Mulberry32 (seedable, reproducible per session).

---

## Data Quality Transparency

Every market variable in the UI carries a badge indicating its epistemic status:

| Badge | Meaning |
|---|---|
| `● LIVE` | Real data from authoritative API, fetched this session |
| `~ PROXY` | Derived from real data via formula or non-official source |
| `⟳ SIM` | Ornstein-Uhlenbeck simulation (API unavailable or not yet fetched) |

The Data Sources banner at the top of the dashboard lists the current status of all 11 data feeds simultaneously.

---

## Limitations

The following limitations are inherent to the free-data constraint and should be understood before operational use:

1. **JKM is a proxy, not a price.** Platts JKM is proprietary. The regression formula carries ±10–15% error. AP route margins should be interpreted as a range, not a point estimate.

2. **Charter rate is manual input.** The default $85,000/day may differ significantly from the current Baltic LNG Index. Update this parameter daily for operationally meaningful results.

3. **TTF has ~15 minute delay.** The Cloudflare Worker caches Yahoo Finance data for 15 minutes (max-age=900). Not suitable for real-time hedging decisions.

4. **FRED data is T+1.** Henry Hub and Brent spot prices reflect the previous business day's settlement. The OU simulation evolves intraday from that anchor.

5. **No position management.** The tool evaluates a single hypothetical cargo. It does not track portfolios, hedges, or aggregate exposure.

6. **Monte Carlo is synchronous (N=3,000).** Results are statistically valid for VaR95 (150 tail observations) but VaR99 has ~30 tail observations — adequate for directional guidance, not regulatory reporting.

---

## Roadmap

The following improvements are identified but require either paid data access or significant architectural changes:

- **Web Worker for Monte Carlo** — moves N=5,000 simulation off the UI thread, eliminates ~120ms blocking per recalculation
- **Cloudflare KV cache** — persist FRED/EIA data across sessions, eliminate re-fetch on page reload
- **Baltic LNG Index integration** — charter rate from live source (currently no free API exists)
- **Platts JKM** — eliminates the largest remaining accuracy gap (paid subscription required)
- **AGSI+ historical chart** — 90-day storage fill trend for seasonal context

---

## License

MIT License. Free for personal and commercial use.

If you use this tool in a professional context, please note the data limitations described above and verify all figures against authoritative market sources before making trading decisions.

---

## Acknowledgements

- [FRED / Federal Reserve Bank of St. Louis](https://fred.stlouisfed.org) — Henry Hub and Brent data
- [U.S. Energy Information Administration](https://www.eia.gov/opendata/) — NG storage, futures, LNG exports
- [Gas Infrastructure Europe](https://agsi.gie.eu) — AGSI+ and ALSI+ European storage and LNG terminal data
- [Frankfurter / European Central Bank](https://www.frankfurter.app) — EUR/USD exchange rate
- [Cloudflare Workers](https://workers.cloudflare.com) — TTF CORS proxy (free tier)
