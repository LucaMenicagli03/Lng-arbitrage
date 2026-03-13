# LNG Trading Dashboard
 
> **Real-time P&L calculator and stochastic risk engine for LNG cargo trading.**  
> Routes: Henry Hub → TTF (Europa / Rotterdam) · Henry Hub → JKM (Asia-Pacific / Tokyo).  
 
---
 
## Table of Contents
 
1. [Overview](#overview)
2. [Feature Matrix](#feature-matrix)
3. [Mathematical Models](#mathematical-models)
4. [Architecture](#architecture)
5. [Parameter Reference](#parameter-reference)
6. [Getting Started](#getting-started)
7. [Known Limitations & Calibration Notes](#known-limitations--calibration-notes)
8. [Changelog](#changelog)
9. [Disclaimer](#disclaimer)
 
---
 
## Overview
 
The LNG Trading Dashboard is a self-contained single-page application built with React and Recharts. It is designed as a **pre-trade decision support tool** for LNG cargo traders and risk managers, covering the full economics of a single spot voyage from US Gulf Coast liquefaction terminals (Henry Hub-indexed SPA) to either European (TTF) or Asian Pacific (JKM) regasification terminals.
 
The application requires **no backend infrastructure**. The only live network call is a lightweight FX rate fetch from the public [Frankfurter / BCE API](https://www.frankfurter.app) (EUR/USD, USD/JPY), with a graceful fallback if the feed is unavailable.
 
All price simulation, Monte Carlo sampling, and risk metric computation run entirely in the browser on the main thread, re-evaluated every two seconds via a tick engine.
 
---
 
## Feature Matrix
 
### Live Market Feeds
 
| Feed | Source | Fallback |
|---|---|---|
| EUR/USD | Frankfurter/BCE | Last known rate |
| USD/JPY | Frankfurter/BCE | Last known rate |
| HH, TTF, JKM | Stochastic simulation (OU + Jump) | — |
 
> **Note:** HH, TTF, and JKM prices are **simulated** via calibrated stochastic processes (see [Mathematical Models](#mathematical-models)). The application does not consume live commodity price feeds. Traders must manually calibrate the long-run mean parameters to current market levels.
 
---
 
### P&L Engine (`calcProfit`)
 
The core function computes the full economics of a single laden voyage for a given route (`EU` or `AP`):
 
| Component | Formula / Notes |
|---|---|
| **Cargo loaded** | `(capacityM3 − heelM3) × LNG_density × MMBtu/ton` |
| **Delivery (MMBtu)** | `loaded × (1 − BOR_effective)^travel_days` — continuous compounding over voyage days |
| **BOG (Boil-Off Gas)** | `loaded − delivery` |
| **Acquisition cost** | `HH × SPA_slope + tolling_fee + liquefaction` (per MMBtu) |
| **Charter cost** | `charterRate × (laden_days + ballast_days)` |
| **Fuel cost** | `net_HFO_consumed × HFO_price + BOG_opportunity_cost` |
| **BOG opportunity cost** | `BOG_MMBtu × max(0, sell_price − HFO_equiv_price)` *(Fix v6.1a)* |
| **Regas fee** | `delivery × regasFee_per_MMBtu` |
| **Port & canal fees** | Sum of origin port, destination port, canal, war risk |
| **Capital cost** | `supply_value × (SOFR/100) × (travel_days / 365)` *(Fix v6.1b)* |
| **Revenue** | `delivery × sell_price` (market or B/L locked) |
| **Break-even price** | `total_cost / delivery` — minimum destination price for profitability |
| **Margin/MMBtu** | `sell_price − break_even_price` |
| **Total margin** | `margin_per_MMBtu × delivery` |
 
**Effective BOR** is modulated by route thermal factor and sea state multiplier:  
`BOR_eff = BOR_nominal × thermal_factor[route] × sea_state_multiplier`
 
---
 
### Routing Engine
 
#### Trade Routes
 
| Route ID | Destination | Nautical Miles | Notes |
|---|---|---|---|
| `EU` | Rotterdam / TTF | 9,200 nm | Direct Atlantic |
| `AP_SUEZ` | Tokyo / JKM | 16,100 nm | Via Suez Canal (default) |
| `AP_CAPE` | Tokyo / JKM | 20,200 nm | Via Cape of Good Hope |
 
#### Suez vs. Cape Break-Even
 
`calcRouteBreakeven` computes the marginal cost of each routing option and identifies the optimal path. The Cape route eliminates canal fees but adds ~4,100 nm (≈ 9 extra laden days at 19.5 kn), whose extra charter cost must be weighed against Suez fees plus any active war risk surcharge.
 
#### Cargo Diversion Engine (`calcCargoDiversion`)
 
Models the economics of diverting a cargo already en route to EU toward the AP market at a given voyage progress fraction (default 50%). Accounts for:
- Remaining cargo at the diversion point (continuous BOR decay applied)
- Backtrack nautical mile penalty (~15% of distance already covered)
- Diversion penalty cost (configurable, default $800k)
- War risk and canal fees on the redirected AP leg
- JKM threshold: the minimum JKM price at which diversion becomes value-accretive over EU delivery
 
---
 
### Monte Carlo Risk Engine (`runMC`)
 
Generates `N = 2,000` correlated scenarios per tick using a fully correlated trivariate stochastic process (HH, TTF, JKM). Each scenario independently produces P&L outcomes for both EU and AP routes. Risk metrics are computed from the resulting empirical distributions.
 
#### Correlation Structure
 
The three price processes are correlated via a Cholesky decomposition of the correlation matrix `CORR`:
 
```
         HH    TTF   JKM
HH    [  1.00  0.65  0.55 ]
TTF   [  0.65  1.00  0.70 ]
JKM   [  0.55  0.70  1.00 ]
```
 
#### Risk Metrics Produced
 
| Metric | Description |
|---|---|
| `mean` | Expected margin (E[P]) |
| `std` | Standard deviation of margin |
| `var95` | Value at Risk at 95% confidence (5th percentile of P&L distribution) |
| `var99` | Value at Risk at 99% confidence |
| `cvar95` | Conditional VaR / Expected Shortfall at 95% — coherent risk measure |
| `cvar99` | CVaR at 99% |
| `probP` | Probability of positive P&L (%) |
| `sharpe` | `E[P] / std(P)` — risk-adjusted return ratio |
| `p95` | 95th percentile of P&L (upside) |
| `volRatio` | `std(AP) / std(EU)` — relative volatility of routes |
 
---
 
### Kelly Criterion Position Sizing
 
A **Half-Kelly** position size hint is computed from the EU Monte Carlo distribution:
 
```
Kelly_full  = E[r] / σ²
Half_Kelly  = Kelly_full / 2   (standard risk management convention)
```
 
The result is displayed as a percentage of capital and is **indicative only**. It is suppressed when E[Margin] ≤ 0.
 
---
 
### Break-Even Sensitivity Matrix
 
A 7×7 heatmap grid spanning:
- **HH rows:** ±20% around live HH price  
- **Destination columns (TTF or JKM):** ±25% around live destination price
 
Each cell displays the net margin per MMBtu at that (HH, Destination) price combination, color-coded from deep red (loss > $2/MMBtu) through amber (near break-even) to deep green (profit > $2/MMBtu). The live market cell is highlighted with a framed border.
 
---
 
### Sensitivity / Tornado Analysis (`calcSpider`)
 
Computes the ±1% sensitivity of EU total margin to seven key drivers:
 
1. HH Gas Price  
2. TTF Destination Price  
3. Charter Rate  
4. SPA Slope  
5. Boil-Off Rate  
6. Ballast Factor  
7. JKM (AP alternative)
 
Results are sorted by absolute impact and rendered as a horizontal tornado chart.
 
---
 
### Stress Scenarios
 
Three geopolitical / market stress scenarios are available as one-click overlays. Each applies multipliers to the base stochastic parameters:
 
| Scenario | Key Impact | Multipliers |
|---|---|---|
| **Black Sea Closure** | TTF ×3.0, Russian supply disruption | `mHH:1.10, mTTF:3.00, mJKM:1.50, volMult:3.0` |
| **US Gulf Hurricane** | Sabine Pass offline, HH +30% | `mHH:1.30, mTTF:1.10, mJKM:1.10, volMult:1.8` |
| **LNG Glut** | Structural oversupply, prices -18–25% | `mHH:0.75, mTTF:0.82, mJKM:0.80, volMult:0.7` |
 
Stress mode alters both the deterministic P&L calculation and the Monte Carlo simulation simultaneously.
 
---
 
### 2-State Markov Regime Model
 
The simulation engine operates under a 2-state hidden Markov model (Normal ↔ Crisis) with per-tick transition probabilities:
 
| Transition | Probability / tick | Interpretation |
|---|---|---|
| Normal → Crisis | 0.008 | ~1 crisis per 208 ticks (≈ 7 min at 2s tick) |
| Crisis → Normal | 0.12 | ~8-tick crisis duration (≈ 16 seconds) |
 
The Crisis regime amplifies volatility, dampens mean-reversion, and increases jump intensity:
 
| Parameter | Normal | Crisis |
|---|---|---|
| Volatility multiplier | ×1.0 | ×2.8 |
| Mean-reversion speed (κ) | ×1.0 | ×0.30 |
| Jump intensity multiplier | ×1.0 | ×2.5 |
 
The current regime is always displayed in the dashboard footer.
 
---
 
## Mathematical Models
 
### 1. Ornstein-Uhlenbeck (OU) Mean-Reversion
 
Each price process follows a discrete-time OU process with mean-reversion speed κ and long-run mean μ:
 
```
dX = κ(μ − X) dt + σ X √dt · ε
```
 
Long-run calibration:
- HH: μ = $3.52/MMBtu
- TTF: μ = $29.50/MMBtu (EUR-denominated, converted at live FX)  
- JKM: configurable via `muJkm` parameter (default $13.50/MMBtu; calibrate to current market)
 
### 2. Merton Jump-Diffusion
 
Jump events are superimposed on the OU diffusion. Jump arrival follows a Poisson process with intensity λ (configurable). Jump sizes are drawn from a **Student-t distribution with ν = 4 degrees of freedom**, producing fat tails approximately 35% wider than Gaussian (correcting the log-normal assumption used prior to v6.0).
 
Jumps are directionally asymmetric:
- TTF and JKM: predominantly upward jumps (energy crisis realism)
- HH: lower jump intensity (×0.25 of TTF) and smaller magnitude (σ_jump × 0.5)
 
### 3. Hawkes Self-Exciting Process
 
A Hawkes process overlay increases the jump arrival rate by +60% conditional on a jump having occurred in the same tick across correlated commodities. This models the empirically observed clustering of jump events in energy markets (e.g., one supply disruption triggering rapid cascading repricing across interconnected hubs).
 
### 4. Charter Rate Stochastic Correlation
 
The charter rate in each Monte Carlo path is shocked by a crisis multiplier:
 
```
charterCrisisShock = 1 + 0.65 × |Student-t(ν=6)|    if nTTF > 0
charterCrisisShock = 1                                 otherwise
```
 
This models the empirical positive correlation (ρ ≈ 0.65) between LNG spot shipping rates and gas price volatility spikes, a significant source of P&L risk that is absent from models assuming deterministic charter rates.
 
### 5. CVaR (Expected Shortfall)
 
CVaR is computed as the arithmetic mean of all simulated P&L outcomes falling below the VaR threshold:
 
```
CVaR_α = E[P&L | P&L < VaR_α]
```
 
CVaR is a coherent risk measure (satisfies sub-additivity) and is more appropriate than VaR alone for fat-tailed distributions.
 
---
 
## Architecture
 
```
App.jsx  (single-file SPA)
│
├── DESIGN TOKENS (T)            — Semantic color system + typography
├── PHYSICAL CONSTANTS           — LNG density, HFO price, fuel consumption
├── ROUTE DEFINITIONS            — NM distances, thermal factors
├── SEA STATE CONFIG             — BOR multipliers per Beaufort scale
├── STRESS SCENARIOS (SS)        — Black Sea, Hurricane, Glut
├── REGIME MODEL                 — 2-state Markov parameters
├── DEFAULT PARAMETERS (DP)      — All configurable defaults
│
├── resolveRoute()               — Route metadata resolver (EU / AP_SUEZ / AP_CAPE)
├── calcProfit()                 — Core P&L engine (v6.1)
├── calcRouteBreakeven()         — Suez vs. Cape optimizer
├── calcCargoDiversion()         — In-voyage diversion economics
├── getMU() / choleskyL()        — OU calibration + Cholesky decomposition
├── randn() / studentT()         — Normal / Student-t samplers (Box-Müller)
├── poissonSample()              — Poisson jump count sampler
├── calcCVaR()                   — Expected Shortfall computation
├── runMC()                      — Full Monte Carlo engine (N=2,000 paths)
├── calcSpider()                 — Tornado sensitivity analysis
├── calcBreakEvenMatrix()        — 7×7 sensitivity heatmap engine
├── fetchFX()                    — Frankfurter/BCE EUR/USD, USD/JPY feed
│
├── UI ATOMS
│   ├── Tip                      — Recharts custom tooltip
│   ├── SH                       — Section header
│   ├── Card                     — Panel container with accent border
│   ├── Badge                    — Status label chip
│   └── Ticker                   — Animated live price display
│
├── PANELS (React components)
│   ├── BreakEvenMatrixPanel     — 7×7 margin heatmap with hover info
│   ├── PnlDetailPanel           — Full cost waterfall breakdown
│   ├── StressPanel              — One-click stress scenario selector
│   └── Sidebar                  — Collapsible scenario parameter editor
│
└── App()                        — Root component
    ├── State: params, prices, MC results, regime, stress mode
    ├── Effect: FX feed polling (60s interval)
    ├── Effect: Tick engine (2s interval) — OU + Jump simulation
    └── Layout: Header → Row A (tickers) → Row B (P&L + charts)
                       → Row C (optimal route) → Row D (matrices) → Footer
```
 
---
 
## Parameter Reference
 
All parameters are configurable via the **Scenario Editor** sidebar (▶ toggle). Default values reflect a typical US Gulf Coast → global spot cargo.
 
### SPA Contract
 
| Parameter | Default | Range | Description |
|---|---|---|---|
| `spaSlope` | 1.115 | 1.00 – 1.30 | HH multiplier in SPA formula. Typical range: 110–115% of HH |
| `sTolling` | $2.50 | $1.00 – $5.00 | Tolling fee per MMBtu (Sabine Pass / Freeport) |
| `sLiquefaction` | $0.30 | $0.05 – $1.00 | Loading & liquefaction overhead per MMBtu |
 
### Vessel
 
| Parameter | Default | Range | Description |
|---|---|---|---|
| `capacityM3` | 174,000 m³ | 140k – 210k | Cargo tank capacity (Q-Flex: 174k, Q-Max: 210k) |
| `heelM3` | 1,500 m³ | 500 – 4,000 | Non-tradeable cooldown volume |
| `borRate` | 0.15%/day | 0.05% – 0.30% | Boil-off rate at calm sea (shipyard nominal spec) |
| `speedKnots` | 19.5 kn | 14 – 22 | Laden voyage speed |
| `seaState` | CALM | CALM / MODERATE / ROUGH / STORM | BOR multiplier: ×1.00 / ×1.20 / ×1.50 / ×1.80 |
 
### Charter
 
| Parameter | Default | Range | Description |
|---|---|---|---|
| `charterRate` | $85,000/day | $40k – $500k | Time charter equivalent rate |
| `ballastFactor` | 0.85 | 0.50 – 1.20 | Ballast voyage duration as fraction of laden voyage |
 
### Routing & Risk
 
| Parameter | Default | Description |
|---|---|---|
| `useCape` | false | Route AP cargo via Cape of Good Hope instead of Suez |
| `suezWarActive` | false | Apply war risk surcharge to Suez transits |
| `canalAP` | $500,000 | Suez Canal transit fee |
| `warRiskSuez` | $300,000 | War risk surcharge (active when `suezWarActive = true`) |
| `diversionPenalty` | $800,000 | Fixed cost applied to any cargo diversion event |
 
### Terminal Fees
 
| Parameter | Default | Description |
|---|---|---|
| `portOrigin` | $280,000 | US Gulf origin port fees |
| `portDest` | $380,000 | Destination port fees |
| `regasFeeEU` | $0.28/MMBtu | EU regasification terminal fee |
| `regasFeeAP` | $0.35/MMBtu | AP regasification terminal fee |
 
### Stochastic / Risk
 
| Parameter | Default | Description |
|---|---|---|
| `lambdaJump` | 0.5 | Poisson jump arrival intensity (per year) |
| `sigmaJump` | 0.30 | Jump size scale parameter (Student-t standard deviation) |
| `muJkm` | $13.50 | JKM long-run mean for OU simulation — **calibrate to current market** |
| `sofr` | 5.30% | SOFR proxy for capital cost computation — update manually |
| `alertThresholdEU` | $0.30 | Amber alert margin threshold for EU route |
| `alertThresholdAP` | $0.30 | Amber alert margin threshold for AP route |
 
### B/L Price Lock
 
When enabled, fixes the sell price at the Bill of Lading value, overriding the simulated market price for both deterministic P&L and Monte Carlo paths. Useful for modeling fixed-price long-term supply agreements.
 
---
 
## Getting Started
 
### Prerequisites
 
- Node.js ≥ 18  
- A React project scaffolded with [Vite](https://vitejs.dev/) (recommended) or Create React App
 
### Dependencies
 
```json
"dependencies": {
  "react": "^18",
  "react-dom": "^18",
  "recharts": "^2"
}
```
 
### Installation
 
```bash
# 1. Scaffold a new Vite + React project (skip if adding to existing project)
npm create vite@latest lng-dashboard -- --template react
cd lng-dashboard
 
# 2. Install dependencies
npm install recharts
 
# 3. Replace src/App.jsx with the provided App.jsx
 
# 4. Start the development server
npm run dev
```
 
The application will be available at `http://localhost:5173` by default.
 
### Build for Production
 
```bash
npm run build
# Output: dist/ — a fully static bundle with no server requirements
```
 
The production build is a single HTML + JS bundle deployable to any static hosting provider (GitHub Pages, Netlify, Vercel, S3, etc.).
 
### Fonts (Optional, Recommended)
 
The dashboard uses JetBrains Mono for monospaced data values and Barlow Condensed for labels. To load them from Google Fonts, add the following to `index.html`:
 
```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Barlow+Condensed:wght@400;700;800&family=JetBrains+Mono:wght@400;700&display=swap" rel="stylesheet">
```
 
Both fonts have system fallbacks (`Arial Narrow` and `Courier New`) and the application renders correctly without them.
 
---
 
## Known Limitations & Calibration Notes
 
### Price Feeds
 
The HH, TTF, and JKM prices displayed are **generated by the stochastic model**, not sourced from live market data. The simulation is calibrated to reasonable long-run mean values but will drift from realized market prices. Traders should:
 
1. Treat the price tickers as scenario generators, not live quotes.
2. Calibrate `muJkm` to the current JKM forward curve center (default $13.50 reflects 2024–25 oversupply; adjust for prevailing conditions).
3. Calibrate `sofr` to the current Fed Funds / SOFR rate (default 5.30%).
 
### Monte Carlo Sample Size
 
With `N = 2,000` paths, CVaR99 estimates (bottom 20 paths) carry non-trivial sampling error. For production risk management, increase `N` to 10,000–50,000. Note that computation occurs synchronously on the main thread; at high `N`, consider moving the MC engine to a Web Worker to avoid UI freezing.
 
### Regime Model Calibration
 
The 2-state Markov transition probabilities (`pNormal→Crisis = 0.008`, `pCrisis→Normal = 0.12`) are indicative. These imply a crisis frequency of roughly one event per 7 minutes at the 2-second tick rate. For backtesting or longer time horizons, these parameters should be estimated from historical volatility regime data.
 
### Single-Cargo Scope
 
The model prices a **single spot cargo**. It does not model:
- Portfolio effects across multiple simultaneous cargoes
- Long-term SPA contract optionality
- Storage optionality at regasification terminals
- Freight forward curve (FFA) hedging instruments
 
### FX Risk
 
TTF prices are converted from EUR to USD at the live BCE rate. JKM is assumed USD-denominated. The model does not hedge FX risk; traders with EUR-denominated cost bases should apply appropriate FX adjustments.
 
---
 
## Changelog
 
### v6.1 — Current Release
 
- **FIX v6.1a — BOG Opportunity Cost:** Corrected the dual-fuel opportunity cost formula. Previous versions compared acquisition cost (HH × slope, always < $11.85/MMBtu HFO threshold) instead of the sell price, causing perpetually zero opportunity cost. The correct formula applies `max(0, sell_price − HFO_equiv_price)` per MMBtu of BOG consumed as fuel.
- **FIX v6.1b — Capital Cost:** Introduced explicit cost of capital: `supply_value × (SOFR/100) × (travel_days / 365)`. At SOFR = 5.3% on a $120M supply value over 19 days, this amounts to ~$330k — previously invisible, causing systematically low break-even prices.
- **Configurable JKM Long-Run Mean (`muJkm`):** The JKM OU mean-reversion target was previously hardcoded as `HH × 3.8 + 1.5`, which became stale during the 2024–25 JKM compression cycle. Now a direct trader-configurable parameter.
- **Break-Even Matrix expanded to ±25% (Dest) / ±20% (HH):** Broadened from ±15% / ±15% (v6.0) following trader feedback on the need for wider scenario coverage.
 
### v6.0
 
- 2-State Markov Regime Model (Normal ↔ Crisis)
- Student-t(ν=4) jump size distribution (fat tails, replaces log-normal)
- CVaR / Expected Shortfall as coherent risk measure
- Hawkes self-exciting process overlay
- Charter rate — volatility crisis correlation (ρ = 0.65)
- Half-Kelly position sizing hint
- Break-Even Sensitivity Matrix (7×7 heatmap)
- Alert threshold system (amber ring on low-but-positive margin)
- Semantic color system v6.0
- Cargo Diversion Engine fixes (actual remaining nautical miles replacing heuristic)
- Break-Even Matrix Panel v6.1 rendering fixes (removed `position:absolute` inside `<td>`, eliminated `rgb()` interpolation NaN vectors, added matrix row guard)
 
---
 
## Disclaimer
 
This application is provided for **informational and educational purposes only**. It does not constitute financial advice, trading recommendations, or an offer to buy or sell any commodity or financial instrument. All price simulations are stochastic models that do not predict future market prices.
 
LNG trading involves substantial financial risk. Users are solely responsible for all trading decisions. The authors of this software accept no liability for any financial loss incurred through its use.
 
---
 
*LNG Trading Dashboard — v6.1 Trader Edition*
 
