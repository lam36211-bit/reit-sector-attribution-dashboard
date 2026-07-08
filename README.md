# REIT Sector Performance & Rate-Sensitivity Dashboard

A production-quality Python project that replicates the monthly performance and attribution reporting workflow of an institutional real-estate portfolio. It holds ten best-in-class REITs across eight property sectors, benchmarks them against the REIT index (VNQ), decomposes returns by property sector via Brinson–Fachler attribution, and — the differentiator — quantifies REITs' rate sensitivity through the 2022–23 rate shock. Runs end-to-end in Google Colab; publishes cleanly to GitHub.

## Key Findings

**Risk-adjusted outperformance.** Over the 10-year window the portfolio compounded at
10.1% annually (+162% cumulative) versus 5.0% for the REIT index (VNQ) — roughly double
the return, with a Sharpe of 0.45 vs. 0.23. The edge came from return rather than lower
risk: portfolio volatility (21.7%) slightly exceeded VNQ's (20.7%), reflecting a
concentrated 10-name book.

**Property-type divergence drove results.** Returns dispersed enormously by property
sector — Welltower (healthcare, +16.3% CAGR), Prologis (industrial, +14.5%), and Equinix
(data centers, +12.3%) led, while BXP (office) was the only holding with a negative
10-year return (−2.5% CAGR). The overweight to structural winners (industrial, data
centers, towers) and underweight to office is the portfolio's central thesis.

**Attribution.** Active return of +1.0% vs. an equal-sector-weight policy benchmark came
almost entirely from sector allocation (+1.1%) rather than security selection (−0.05%).
The strongest single call was underweighting office (+7.7% relative, the sector returned
−22%); the largest drag was underweighting healthcare (−5.0%), which was in fact the
top-performing sector. (Selection is measured only in the two multi-name sectors,
Residential and Retail; the other six hold a single REIT each.)

**Drawdown.** Maximum drawdown was −40.4% during the Feb–Mar 2020 COVID crash, marginally
shallower than VNQ's −42.4% — the sector tilts offered limited downside protection in a
correlated sell-off.

**Rate sensitivity.** Daily correlation to long Treasuries (TLT) averaged just +0.06 over
the decade, but the rolling correlation rose to roughly [FILL FROM YOUR CHART] during the
2022–23 rate shock, confirming that listed REITs re-couple to duration when the cost of
capital moves — the core macro driver of the asset class.

---

## 1. Project Overview

The project builds a conviction-weighted REIT portfolio, pulls 10 years of daily prices, and produces the full institutional reporting stack:

- **Performance:** cumulative return vs. VNQ and SPY, CAGR, rolling returns
- **Risk:** annualized volatility, Sharpe, Sortino, Calmar, max drawdown, VaR/CVaR, underwater plot
- **Attribution:** per-REIT contribution-to-return waterfall plus a Brinson allocation/selection decomposition **by property sector** against an equal-sector-weight policy benchmark
- **Rate sensitivity:** rolling correlation of REITs (VNQ) with long Treasuries (TLT) — the chart that shows REITs re-coupling to rates in 2022
- **Delivery:** a standalone tabbed HTML dashboard (Performance / Risk / Attribution / Rate Sensitivity)

## 2. Real-World Finance Use Case

Every real-estate allocator — from a REIT mutual fund to the public-securities sleeve of a REPE shop — produces this exact report. The two questions an investment committee asks are *"How did we do, risk-adjusted?"* and *"Was it the right property types or the right names?"* Brinson attribution by sector answers the second and is the native language of real-estate portfolio reviews. The rate-sensitivity view speaks to the single biggest macro driver of listed real estate — the cost of capital — and shows you understand *why* office and mall REITs and long bonds sold off together in 2022.

## 3. System Architecture

```
Data Layer            Analytics Layer         Presentation
------------------    --------------------    -------------------
yfinance prices   ->  Returns engine      ->  Plotly figures
(10 REITs +           Metrics engine          Tabbed HTML dashboard
 VNQ/SPY/TLT)         Rolling metrics         CSV exports
FRED DGS3MO           Sector attribution      Summary report
(^IRX fallback)        - Contribution
Validation &           - Brinson
cleaning              Rate correlation
```

## 4. Required APIs and Data Sources

| Source | Data | Access |
|---|---|---|
| Yahoo Finance (`yfinance`) | 10y daily adjusted close: 10 REITs + VNQ, SPY, TLT | Free, no key |
| FRED (`fredapi`) | `DGS3MO` — 3-Month T-Bill (risk-free) | Free API key (optional) |
| Yahoo `^IRX` | 13-week T-bill yield — automatic fallback if no FRED key | Free, no key |

## 5. Required Python Libraries

`yfinance`, `pandas`, `numpy`, `plotly`, `fredapi` (optional), `matplotlib`. All installable in the first Colab cell.

## 6. Folder / File Structure

```
reit-sector-attribution-dashboard/
├── README.md
├── reit_sector_attribution_dashboard.ipynb   # the full Colab build
├── outputs/
│   ├── dashboard.html                        # tabbed interactive dashboard
│   ├── metrics_summary.csv
│   ├── contribution_by_asset.csv
│   └── brinson_attribution.csv
└── requirements.txt
```

## 7. Step-by-Step Build Guideline

1. Define config: 10 REITs with conviction weights, property-sector map, VNQ benchmark, SPY/TLT context.
2. Download 10 years of adjusted closes with a retry wrapper; validate coverage.
3. Pull the risk-free rate (FRED → `^IRX` → constant fallback chain).
4. Clean: align calendars, forward-fill gaps ≤ 5 days, drop the warm-up window.
5. Compute daily log returns (time-series math) and simple returns (attribution math).
6. Build the metrics engine, adding beta-vs-SPY and correlation-vs-TLT for REIT context.
7. Compute rolling 90-day volatility and rolling 252-day Sharpe.
8. Attribution: per-REIT contribution, then Brinson–Fachler **by property sector**.
9. Build the rolling VNQ–TLT correlation chart and the other five figures; assemble the dashboard.
10. Export CSVs + summary and push to GitHub.

## 8. Data Collection Pipeline

Batch download via `yf.download(..., auto_adjust=True)` with exponential-backoff retries and per-ticker coverage validation (any name with < 95% of expected trading days is flagged — relevant since some REITs have shorter listed histories). The risk-free series is re-indexed to the trading calendar and converted from annualized percent to a daily decimal rate.

## 9. Data Cleaning and Feature Engineering

Inner-join to a common trading calendar; forward-fill short gaps. Features: daily log and simple returns, cumulative wealth indices, excess returns, drawdown series, rolling windows, and pairwise correlations to TLT. Documented in-code: **log returns for time aggregation, simple returns for cross-sectional (portfolio/attribution) aggregation.**

## 10. Core Models / Algorithms

- **Annualization:** μ·252 for returns, σ·√252 for volatility.
- **Sharpe / Sortino / Calmar:** standard definitions; Sortino uses downside deviation.
- **Drawdown:** DD_t = W_t / max(W_0..t) − 1.
- **Contribution to return:** c_i = Σ_t w_i·r_{i,t}, proportionally scaled to close to the geometric total.
- **Brinson–Fachler by property sector:** Allocation = (w_p − w_b)(r_b,sector − r_b); Selection = w_b(r_p,sector − r_b,sector); Interaction = the cross-term.
- **Rate sensitivity:** rolling 90-day correlation of VNQ (and the portfolio) with TLT.

## 11. Visualizations & Dashboard Components

1. Growth of $10,000 — REIT portfolio vs. VNQ vs. SPY (log-scale toggle)
2. Underwater drawdown chart (portfolio vs. VNQ) with max-DD annotation
3. Rolling 90-day volatility + rolling 252-day Sharpe (dual panel)
4. Contribution-to-return waterfall by REIT
5. Brinson allocation/selection bar chart by property sector
6. **Rolling VNQ–TLT correlation** with the 2022–23 rate shock shaded — the standout chart
7. REIT correlation heatmap, ordered by property sector

Assembled into a standalone `dashboard.html` (Performance / Risk / Attribution / Rate Sensitivity tabs).

## 12. Performance Metrics

Total return, CAGR, annualized vol, Sharpe, Sortino, Calmar, max drawdown (with dates), 95% VaR/CVaR, hit rate, beta vs SPY, and correlation vs TLT — for the portfolio, VNQ, and every individual REIT.

## 13. Final Deliverables

- `reit_sector_attribution_dashboard.ipynb` — fully commented, sectioned Colab notebook
- `dashboard.html` — interactive tabbed dashboard
- Three CSV exports (metrics, contribution, Brinson)
- This README

## 14. Resume Description

> **REIT Sector Performance & Rate-Sensitivity Engine** — Python, pandas, Plotly
> Built an institutional-style performance and attribution engine for a 10-name REIT portfolio spanning 8 property sectors over 10 years of daily data; implemented a full risk-metrics suite (Sharpe, Sortino, Calmar, VaR/CVaR, drawdown analytics) and Brinson–Fachler allocation/selection attribution by property sector; quantified listed real estate's rate sensitivity via rolling REIT–Treasury correlation through the 2022–23 rate cycle; delivered results as a standalone interactive Plotly dashboard with automated FRED/Yahoo data pipelines.

## 15. Potential Upgrades

- Swap the equal-sector-weight benchmark for VNQ's **published sector weights** for a truer active-return read
- Carino/Menchero geometric linking for exact multi-period attribution
- **Public-vs-private valuation gap:** overlay a private-CRE index (e.g., NCREIF/Green Street) to show listed REITs reprice daily while private marks lag — a core REPE insight
- Factor attribution (Fama-French + a rates factor) alongside Brinson
- Marginal contribution to risk (MCTR) from the covariance matrix
- Streamlit deployment with editable tickers/weights

*Data: Yahoo Finance & FRED. Not investment advice.*
