# LNG Frontier Optimizer — Trader Edition v5.0

> **Black Swan Edition** — A quantitative Decision Support System for LNG cargo arbitrage, routing optimization, and risk-adjusted voyage analysis.

---

## Overview

LNG Frontier Optimizer is a real-time analytical dashboard engineered for commodity trading professionals operating in the global Liquefied Natural Gas market. The system integrates live FX data, stochastic price simulation, and a corrected economic model reflecting actual LNG trading contract structures — including SPA acquisition costs, ballast voyage charter, vessel heel, and sea state effects on cargo decay.

The application is purpose-built to answer the operative questions a trader faces at the start of a trading session:

- Does the current market justify this cargo against the acquisition cost already contracted?
- What is the break-even destination price per MMBtu for each route, accounting for all voyage costs?
- If the JKM rises above threshold during a voyage toward Europe, does cargo diversion become economically viable net of contractual penalties?
- How does a geopolitical shock scenario affect the margin, and which route survives stress testing?

---

## Tech Stack

| Component | Technology |
|---|---|
| Framework | React 18 (Vite) |
| Data Visualization | Recharts |
| FX Data Feed | Frankfurter API (ECB) — free, no API key, CORS-enabled |
| Price Simulation | Custom stochastic engine (OU + Merton Jump Diffusion + Hawkes) |
| Mathematical Core | Native JavaScript — no external math library |

All data sources are free and open. No API keys are required to run the application.

---

## Economic Model

### Acquisition Cost — SPA Contract Structure (v5.0 correction)

Prior to v5.0, the supply cost was calculated as `CAP_MMBTU × HH_spot`, which systematically underestimated the real acquisition cost by approximately 29%. The corrected formula reflects standard Sale and Purchase Agreement structures:

```
Acquisition Cost ($/MMBtu) = HH × SPA_slope + Tolling_Fee + Liquefaction_Fee
```

**Default parameters:**
- SPA Slope: `1.115×` (110–115% of HH, standard Sabine Pass / Freeport tolling agreement)
- Tolling Fee: `$2.50/MMBtu`
- Liquefaction overhead: `$0.30/MMBtu`
- **Effective cost at HH=$3.52: $4.55/MMBtu vs naive $3.52** — a 29% differential that can invert routing recommendations

### Charter Cost — Full Voyage Accounting (v5.0 correction)

Prior versions charged charter only for laden travel days. In reality, under a Time Charter contract the charterer bears the cost of the ballast return voyage. The corrected formula is:

```
Charter Cost = Charter_Rate × (laden_days + laden_days × ballast_factor)
```

At `ballast_factor = 0.85`, a 19-day EU laden voyage carries 35.2 total charter days — nearly doubling the charter cost relative to the naive calculation.

### Vessel Heel

The tank heel (default 1,500 m³) is a non-tradeable cooldown reserve maintained to prevent thermal shock to membrane tank panels. It is deducted from `loadMmbtu` before any P&L calculation:

```
loadMmbtu = (capacityM3 − heelM3) × LNG_density × MMBtu_per_ton
```

### Sea State — Boil-Off Amplification

The Natural Boil-Off Rate declared by the shipyard is a laboratory specification under calm sea conditions. In practice, mechanical stress on insulation panels increases BOG generation significantly:

| Sea State | Beaufort | BOR Multiplier |
|---|---|---|
| Calm | ≤3 | ×1.00 |
| Moderate | 4–5 | ×1.20 |
| Rough | 6–7 | ×1.50 |
| Storm | ≥8 | ×1.80 |

The effective BOR is: `borEff = borNominal × thermalFactor × seaStateMult`

On a 27-day Sabine Pass → Tokyo voyage in rough seas, the difference between nominal and effective BOG can represent $800k–$1.5M in cargo value.

### Thermal Gradient by Route

Ambient sea temperature increases BOG generation on equatorial routes:

| Route | Thermal Factor | Notes |
|---|---|---|
| EU (North Atlantic) | ×1.00 | Baseline |
| AP via Suez (equatorial) | ×1.10 | +10% BOG |
| AP via Cape (South Atlantic) | ×1.03 | +3% BOG |

### Dual-Fuel Engine — BOG Opportunity Cost

The system distinguishes between two fuel regimes:

- **BOG offsets HFO:** Boil-off gas generated during the voyage displaces Heavy Fuel Oil consumption (energy equivalent basis)
- **Opportunity cost regime:** When `HH × SPA_slope > HFO_threshold (~$11.85/MMBtu)`, burning BOG as fuel has a positive opportunity cost — the gas is worth more sold than burned. This is currently zero at HH≈$3.50 but becomes operative in high-price environments

### B/L Price Lock

When a cargo is sold at a fixed price at the Bill of Lading date (standard for long-term SPA deliveries), the revenue side of the trade is locked regardless of subsequent market movements. The lock is activatable per route in the Sidebar, making the tool functional for tracking existing positions against live market breaks.

---

## Primary KPI — Margin over Break-Even

The operative metric displayed throughout the dashboard is **Margin per MMBtu**, defined as:

```
Margin/MMBtu = Sell_Price − Break-Even_Destination_Price

Break-Even_Price = (Supply + Charter + Fuel + Regas + Ports) / delivered_MMBtu
```

This replaces the raw profit figure (which has no decision relevance without the acquisition cost context). A cargo generating $2M gross profit on a $10M cost base has a fundamentally different risk profile than $2M on a $2.5M cost base — the Margin/MMBtu figure captures this distinction.

---

## Stochastic Engine

### Ornstein-Uhlenbeck with Cholesky Correlation

Prices follow mean-reverting diffusion processes with correlated innovations:

```
dS_i = κ(μ_i − S_i)dt + σ_i · S_i · √dt · Z_i
```

where `Z` is drawn from a Cholesky-decomposed correlation matrix:
- ρ(HH, TTF) = 0.65
- ρ(HH, JKM) = 0.55
- ρ(TTF, JKM) = 0.70

### Jump Diffusion — Merton (1976)

Geopolitical supply shocks are modeled as a compound Poisson process superimposed on the OU dynamics:

```
dS = OU(κ, μ, σ)·dt + J·dN(λ)
J ~ LogNormal(0, σ_J²)    N ~ Poisson(λ·dt)
```

Jump asymmetry: destination prices spike while HH decouples (basis widening), reflecting real market behavior during supply disruptions.

### Hawkes Process — Cross-Market Jump Coupling

A price shock on TTF increases the conditional probability of a JKM shock by 60%, reflecting the competitive dynamics of the global LNG spot market:

```
λ_JKM(t) = λ_base + 0.60·λ_base · 𝟙{jump_TTF_occurred}
```

### Suez Congestion — Stochastic Delay

Canal waiting times are modeled as a log-normal random variable applied to AP voyage duration, affecting both cargo delivery and charter cost in each Monte Carlo path.

### Monte Carlo Output

700 simulations per route pair, producing:
- Expected P&L (mean)
- VaR at 95% and 99%
- Probability of profitable voyage
- Sharpe Ratio (risk-adjusted return)
- Volatility ratio EU/AP (triggers Risk Warning when > 1.8×)

---

## Cargo Diversion Engine

The system evaluates the economic viability of mid-voyage route diversion (EU → AP) at a configurable progress fraction (default: 50% of EU voyage completed):

```
Divert if: Revenue_AP − Revenue_EU > 0

JKM Threshold = TTF_USD + Diversion_Penalty/deliv_AP
              + (regasFee_AP − regasFee_EU)
              + ΔCharter(AP−EU) / deliv_AP
```

When the JKM spot price exceeds this threshold, the Diversion Signal activates with the exact delta revenue and the threshold price displayed.

---

## Routing: Suez Canal vs Cape of Good Hope

| Parameter | Via Suez | Via Cape |
|---|---|---|
| Distance | 16,100 nm | 20,200 nm |
| Extra transit | — | +4,100 nm / +7.1 days |
| Canal fee | ~$500k | None |
| War Risk Surcharge | Configurable | None |
| Thermal factor | ×1.10 | ×1.03 |

The system calculates the break-even automatically: Cape route becomes optimal when Suez fees + war risk surcharge exceed the additional charter and fuel cost of the longer voyage. Both routes account for ballast return days.

---

## Stress Test Scenarios

Three one-click deterministic stress scenarios apply price multipliers to the live feed non-destructively:

| Scenario | HH | TTF | JKM | Vol Mult |
|---|---|---|---|---|
| Black Sea Closure | ×1.10 | ×3.00 | ×1.50 | ×3.0 |
| US Gulf Hurricane | ×1.30 | ×1.10 | ×1.10 | ×1.8 |
| LNG Glut | ×0.75 | ×0.82 | ×0.80 | ×0.7 |

Stress multipliers are applied to all P&L calculations, Monte Carlo runs, diversion signals, and the Decision Engine simultaneously. The live OU price feed is unaffected and resumes on scenario deactivation.

---

## Parameters — Sidebar Reference

All parameters are runtime-configurable without code changes.

### SPA Contract
| Parameter | Default | Range | Description |
|---|---|---|---|
| SPA Slope | 1.115 | 1.00–1.30 | HH price indexation multiplier |
| Tolling Fee | $2.50 | $1.00–$5.00 | Liquefaction tolling per MMBtu |
| Liquefaction | $0.30 | $0.05–$1.00 | Loading overhead per MMBtu |

### Vessel
| Parameter | Default | Range | Description |
|---|---|---|---|
| Capacity | 174,000 m³ | 140k–210k | Q-Flex to Q-Max range |
| Heel | 1,500 m³ | 500–4,000 | Cooldown reserve (non-tradeable) |
| BOR Rate | 0.15%/day | 0.05–0.30 | Nominal boil-off (shipyard spec) |
| Speed | 19.5 kn | 14–22 | Cruise speed (affects travel days) |
| Sea State | Calm | 4 levels | BOR amplification multiplier |

### Charter
| Parameter | Default | Range | Description |
|---|---|---|---|
| Charter Rate | $85,000/day | $40k–$500k | Time charter daily rate |
| Ballast Factor | 0.85 | 0.50–1.20 | Return voyage as fraction of laden days |

### Financial
| Parameter | Default | Range | Description |
|---|---|---|---|
| Regas Fee EU | $0.28/MMBtu | $0.10–$0.80 | Gate / Rotterdam |
| Regas Fee AP | $0.35/MMBtu | $0.10–$0.80 | Sodegaura / Tokyo Gas |
| Diversion Penalty | $800,000 | $0–$3M | Contractual cancellation fee |

### Jump Diffusion
| Parameter | Default | Range | Description |
|---|---|---|---|
| λ (intensity) | 0.5/yr | 0–4 | Mean geopolitical events per year |
| σ_J (size) | 0.30 | 0.05–1.00 | Log-normal jump magnitude |

---

## Data Sources

| Data | Source | Refresh | Notes |
|---|---|---|---|
| EUR/USD, USD/JPY | Frankfurter API (ECB) | 60 seconds | Free, no key, CORS-enabled |
| HH / TTF / JKM | OU simulation seeded on EIA 2024–2025 averages | 2 seconds | Stochastic model, not live commodity prices |

**Important:** HH, TTF, and JKM prices are generated by the stochastic model (OU + Jump Diffusion), seeded from historical averages. They are not real-time commodity price feeds. The FX rates are live via the ECB. The system is designed for scenario analysis and pre-trade evaluation, not for live execution.

---

## Known Model Limitations

The following limitations are acknowledged and represent the boundary between this tool's intended use (pre-trade scenario analysis) and a full operational trading system:

1. **No real-time commodity price feed.** HH, TTF, and JKM use stochastic simulation. Integration with Platts, Argus, or Bloomberg B-PIPE would require a licensed data subscription.

2. **OU mean-reversion assumes a stable regime.** The long-run mean parameters (μ_HH=$3.52, μ_TTF=€29.50) are calibrated on 2024–2025 EIA/ICIS averages. In structural market breaks (e.g., EU energy crisis 2022), mean-reversion assumptions collapse. Stress test scenarios partially address this but do not substitute for regime-switching models.

3. **VaR is likely understated by 40–60%** relative to empirical historical VaR, due to the log-normal jump size assumption. Empirical gas price distributions have asymmetric fat tails. Calibrating σ_J and λ from historical Platts data would substantially improve risk quantification.

4. **No credit risk or terminal slot availability.** The model assumes the destination terminal has available regasification capacity and the counterparty is investment-grade. In practice, slot booking and credit assessment materially affect cargo routing decisions.

5. **Charter rate is treated as deterministic per voyage.** The Baltic Exchange LNG Index has historically ranged from $25k/day (2020 oversupply) to $450k/day (2022 crisis). The slider range has been extended to $500k/day to cover historical extremes, but the correlation between charter rates and commodity prices during crises is not captured stochastically.

6. **No hedge overlay.** The model assumes an unhedged position. A trading desk operating with TTF/JKM futures hedges would calculate P&L on the net delta position, not on the gross cargo value.

---

## Project Evolution

| Version | Key Change |
|---|---|
| v1.0 | Base dashboard — static parameters, OU pricing |
| v2.0 | Cholesky correlation matrix, live FX feed |
| v3.0 | Sidebar parametrica, Jump Diffusion (Merton 1976), Tornado Chart |
| v4.0 | Suez/Cape routing, Cargo Diversion Engine, Hawkes Process, Stress Tests, Sea State |
| v5.0 | **SPA acquisition cost, Heel, Ballast days, Margin/MMBtu KPI, B/L Price Lock** |

---

## Quick Start

```bash
# Install dependencies
npm install

# Start development server
npm run dev
```

Dependencies: `react`, `react-dom`, `recharts`. No additional packages required.

Google Fonts (JetBrains Mono, Barlow Condensed) are loaded via CDN import in the component stylesheet. The application functions without them, falling back to system monospace fonts.

---

## File Structure

```
src/
└── App.jsx          # Single-file application — all components, engine, and styles
```

The application is intentionally contained in a single file to simplify deployment and sharing. In a production context, the recommended refactoring would separate:

```
src/
├── engine/
│   ├── stochastic.js     # OU, Cholesky, Jump Diffusion, Hawkes
│   ├── calcProfit.js     # P&L engine with SPA, heel, ballast, dual-fuel
│   ├── monteCarlo.js     # MC runner
│   └── sensitivity.js   # Tornado / Spider analysis
├── components/
│   ├── Sidebar.jsx
│   ├── MarginPanel.jsx
│   ├── DecisionEngine.jsx
│   ├── StressPanel.jsx
│   └── ...
└── App.jsx
```

---

## Author

Developed by Luca — iterative engineering collaboration with Claude (Anthropic).

Architecture: React 18 + Recharts + custom stochastic engine.
Economic model: SPA contract structure, Merton Jump Diffusion, Hawkes Process, dual-fuel optimization, ballast-corrected charter accounting.

---

*This tool is intended for analytical and educational purposes. It does not constitute financial advice and should not be used as the sole basis for commercial trading decisions.*
