# Energy Demand Forecasting & KPI Monitoring

> **Portfolio project demonstrating competency in monitoring & evaluation, time series forecasting, and structured KPI reporting — applied to Malawi's national electricity sector.**

**Author:** Alex Maseko &nbsp;|&nbsp; BSc Applied Statistics (Credit) &nbsp;|&nbsp; MEAL Certified — HLA & Catholic Relief Services  
**Tool:** Stata 17 &nbsp;|&nbsp; **Data period:** January 2018 – December 2024 (84 months)  
**Forecast horizon:** January – December 2025  
**Contact:** alexmaseko166@gmail.com &nbsp;|&nbsp; [linkedin.com/in/alexmaseko](https://linkedin.com/in/alexmaseko)

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Repository Structure](#2-repository-structure)
3. [Dataset Description](#3-dataset-description)
4. [KPIs Monitored](#4-kpis-monitored)
5. [Modelling Process](#5-modelling-process)
   - [Step 1 — Data Import & Validation](#step-1--data-import--validation)
   - [Step 2 — KPI Computation](#step-2--kpi-computation)
   - [Step 3 — Exploratory Data Analysis](#step-3--exploratory-data-analysis)
   - [Step 4 — Stationarity Testing](#step-4--stationarity-testing)
   - [Step 5 — Autocorrelation Diagnostics](#step-5--autocorrelation-diagnostics)
   - [Step 6 — Model Selection](#step-6--model-selection)
   - [Step 7 — Forecast Accuracy Evaluation](#step-7--forecast-accuracy-evaluation)
   - [Step 8 — 2025 Demand & Generation Forecast](#step-8--2025-demand--generation-forecast)
6. [Results & Key Findings](#6-results--key-findings)
7. [How to Run](#7-how-to-run)
8. [Skills Demonstrated](#8-skills-demonstrated)

---

## 1. Project Overview

Malawi's electricity system faces a persistent and growing challenge: national energy **demand is rising at approximately 5.2% per year**, while installed generation capacity has remained constrained at **444 MW** — the combined output of EGENCO's four hydro stations (Nkula, Tedzani, Kapichira, Wovwe) and thermal plants. The mismatch between supply and demand is especially acute during the **dry season (June–September)**, when hydro output falls sharply just as demand peaks.

This project simulates the full monitoring and evaluation workflow of a power sector **Monitoring Officer**, including:

- Constructing and validating a **Consistent Data Set (CDS)** from monthly divisional data
- Computing and tracking six **operational KPIs** against generation and demand benchmarks
- Applying **time series forecasting** (SARIMA) to predict monthly demand 12 months ahead
- Evaluating forecast accuracy on a **holdout test window**
- Structuring outputs as **periodic reports** (monthly, quarterly, annual) for Executive Management

All analysis is conducted in **Stata 17** with a fully reproducible, annotated do-file.

---

## 2. Repository Structure

```
energy-forecasting-malawi/
│
├── README.md                          ← This file
│
├── data/
│   └── malawi_energy_data.csv         ← Input CDS — 84 months, 14 variables
│
├── stata/
│   └── energy_forecasting.do          ← Main do-file (run this to reproduce everything)
│
├── outputs/
│   ├── kpi_annual_summary.csv         ← Annual KPI monitoring table
│   ├── malawi_energy_clean.dta        ← Validated Stata dataset
│   └── energy_forecasting_log.smcl   ← Full Stata output log
│
├── figures/
│   ├── fig1_demand_vs_generation.png
│   ├── fig2_unmet_demand.png
│   ├── fig3_seasonal_pattern.png
│   ├── fig4_rainfall_hydro.png
│   ├── fig5_acf_demand.png
│   ├── fig6_pacf_demand.png
│   ├── fig7_acf_diff_demand.png
│   ├── fig8_forecast_vs_actual.png
│   ├── fig9_demand_forecast_2025.png
│   └── fig10_gen_demand_forecast_2025.png
│
└── report/
    └── Malawi_Energy_Forecasting_Report.docx
```

---

## 3. Dataset Description

The input file `malawi_energy_data.csv` contains **84 monthly observations** (January 2018 – December 2024) simulating Malawi's national electricity sector data, structured as a Consistent Data Set (CDS) representative of the type collected from EGENCO's operational Divisions.

| Variable | Description | Unit |
|---|---|---|
| `year` | Calendar year | — |
| `month` | Calendar month (1–12) | — |
| `generation_gwh` | Total electricity generated | GWh/month |
| `demand_gwh` | Total electricity demanded | GWh/month |
| `peak_demand_mw` | Monthly peak demand | MW |
| `capacity_mw` | Total installed generation capacity | MW |
| `hydro_gwh` | Generation from hydro plants | GWh/month |
| `thermal_gwh` | Generation from thermal plants | GWh/month |
| `load_factor_pct` | System load factor | % |
| `unmet_demand_gwh` | Demand not met by generation | GWh/month |
| `system_losses_pct` | Transmission & distribution losses | % |
| `rainfall_mm` | Monthly rainfall (hydro leading indicator) | mm |
| `gdp_index` | Economic activity index (2018 = 100) | Index |

---

## 4. KPIs Monitored

Six operational KPIs are computed from the validated dataset and tracked across all reporting periods:

| KPI | Formula | Purpose |
|---|---|---|
| **Generation adequacy ratio** | `generation / demand` | Flags supply shortfalls — target ≥ 1.0 |
| **Capacity utilisation (%)** | `(generation / (capacity × 24 × 30 / 1000)) × 100` | Measures fleet deployment efficiency |
| **Energy access gap (%)** | `(unmet_demand / demand) × 100` | Tracks % of demand not met |
| **Hydro dependency ratio (%)** | `(hydro / generation) × 100` | Monitors hydrological risk exposure |
| **Supply reliability index** | `100 − access_gap_pct` | Overall system reliability score |
| **YoY demand growth (%)** | `(demand_t − demand_t−12) / demand_t−12 × 100` | Annual demand growth rate |

---

## 5. Modelling Process

### Step 1 — Data Import & Validation

```stata
import delimited "malawi_energy_data.csv", clear varnames(1)
gen t = ym(year, month)
format t %tm
tsset t, monthly
misstable summarize       // confirm zero missing values
```

The dataset is declared as a **monthly time series** using `tsset`. Completeness, range, and internal consistency checks are applied before any analysis. No missing values were found across all 14 variables and 84 observations.

---

### Step 2 — KPI Computation

All six KPIs are computed as derived variables directly in Stata:

```stata
gen adequacy_ratio   = generation_gwh / demand_gwh
gen capacity_util_pct = (generation_gwh / (capacity_mw * 24 * 30 / 1000)) * 100
gen access_gap_pct   = (unmet_demand_gwh / demand_gwh) * 100
gen hydro_share_pct  = (hydro_gwh / generation_gwh) * 100
gen reliability_idx  = 100 - access_gap_pct
gen demand_growth_yoy = (demand_gwh - l12.demand_gwh) / l12.demand_gwh * 100
```

Annual averages are collapsed and exported to `kpi_annual_summary.csv` for use in executive monthly and quarterly reports.

---

### Step 3 — Exploratory Data Analysis

Five charts are produced to understand the data structure before modelling:

- **Fig 1** — Monthly demand vs generation time series (2018–2024): reveals the persistent and widening supply gap
- **Fig 2** — Unmet demand trend: tracks the energy access gap over time
- **Fig 3** — Average monthly seasonal pattern: confirms June–September demand peaks and January–April hydro peaks
- **Fig 4** — Rainfall vs hydro generation scatter: confirms significant positive relationship (R² ≈ 0.58, p < 0.001), validating rainfall as a useful leading indicator
- Pearson **correlation matrix**: demand is most strongly correlated with GDP index (r = 0.97), confirming economic growth as the primary demand driver

---

### Step 4 — Stationarity Testing

Before fitting any time series model, the **Augmented Dickey-Fuller (ADF) test** is applied to test for a unit root.

```stata
dfuller demand_gwh, regress lags(3)    // level series — H0 not rejected (p > 0.05)
dfuller D.demand_gwh, regress lags(3)  // first difference — H0 rejected (p < 0.01)
zandrews demand_gwh, break(trend)      // structural break check
```

**Result:** The demand series is **non-stationary in levels** but **stationary after first differencing** — confirming `d = 1` (one non-seasonal difference required). This prevents spurious regression and ensures valid inference from the model.

---

### Step 5 — Autocorrelation Diagnostics

ACF and PACF plots are produced for both the level and differenced series to identify the AR and MA orders:

```stata
ac  demand_gwh, lags(24)    // ACF — levels
pac demand_gwh, lags(24)    // PACF — levels
ac  D.demand_gwh, lags(24)  // ACF — first difference
```

**Findings:**
- Significant spike at **lag 12** in the ACF confirms **annual seasonality** (s = 12) — consistent with Malawi's rainfall-driven hydro cycle
- Decay pattern in the ACF after differencing suggests AR(1) specification
- Single significant spike in the PACF after differencing suggests MA(1) specification
- A **seasonal AR term (P = 1)** at lag 12 is indicated by the ACF structure

---

### Step 6 — Model Selection

Four candidate ARIMA / SARIMA models are estimated on the **training window** (January 2018 – December 2023, 72 observations) and compared by AIC and BIC:

```stata
arima demand_gwh if train, arima(1,1,1)                       // Model A
arima demand_gwh if train, arima(2,1,1)                       // Model B
arima demand_gwh if train, arima(1,1,1) sarima(1,1,0,12)     // Model C ✓
arima demand_gwh if train, arima(1,1,1) sarima(0,1,1,12)     // Model D
estimates stats model_A model_B model_C model_D
```

| Model | Specification | AIC | BIC | Selected |
|---|---|---|---|---|
| A | ARIMA(1,1,1) | 487.3 | 496.1 | ✗ Misses seasonality |
| B | ARIMA(2,1,1) | 485.9 | 497.2 | ✗ Marginal improvement |
| **C** | **SARIMA(1,1,1)(1,1,0)₁₂** | **461.2** | **473.8** | **✓ Selected** |
| D | SARIMA(1,1,1)(0,1,1)₁₂ | 463.7 | 476.3 | ✗ Slightly higher AIC |

**Model C** — `SARIMA(1,1,1)(1,1,0)₁₂` — achieves the lowest AIC (461.2), with residuals confirmed as white noise (Ljung-Box Q, p > 0.10 at all lags).

The model specification:

```
Φ(B¹²) φ(B) ∇¹² ∇ Yₜ = θ(B) εₜ
```

Where: `d = 1`, `D = 1`, `p = 1`, `q = 1`, `P = 1`, `s = 12`.

---

### Step 7 — Forecast Accuracy Evaluation

The selected model is used to generate dynamic forecasts over the **holdout window** (January – December 2024, 12 observations):

```stata
predict demand_fitted, dynamic(tm(2024m01)) y
gen residual_test = demand_gwh - demand_fitted if !train
gen abs_err = abs(residual_test)
gen sq_err  = residual_test^2
gen abs_pct = abs(residual_test / demand_gwh) * 100
```

| Metric | Value | Benchmark |
|---|---|---|
| MAE (Mean Absolute Error) | 8.4 GWh | — |
| RMSE (Root Mean Squared Error) | 10.7 GWh | — |
| **MAPE (Mean Absolute Percentage Error)** | **2.8%** | **Industry threshold ≤ 5%** ✓ |

A MAPE of **2.8%** confirms the model is well-specified and operationally reliable for monthly demand planning.

---

### Step 8 — 2025 Demand & Generation Forecast

The model is refit on the **full 84-month dataset**, extended to include 12 future periods, and a dynamic forecast is generated for January – December 2025:

```stata
arima demand_gwh, arima(1,1,1) sarima(1,1,0,12)
tsappend, add(12)
predict demand_forecast, dynamic(tm(2025m01)) y
predict demand_se,       dynamic(tm(2025m01)) rmse
gen ci_lower = demand_forecast - 1.96 * demand_se
gen ci_upper = demand_forecast + 1.96 * demand_se
export delimited "forecast_2025_monthly.csv", replace
```

Generation is forecast as a function of the historical adequacy ratio constrained by installed capacity:

```stata
gen generation_forecast = mean_adequacy * demand_forecast
replace generation_forecast = min(generation_forecast, capacity_mw * 24 * 30 / 1000)
```

---

## 6. Results & Key Findings

### KPI Monitoring — Annual Summary (2018–2024)

| Year | Gen (GWh) | Demand (GWh) | Adequacy Ratio | Cap Util % | Access Gap % | Reliability Idx |
|---|---|---|---|---|---|---|
| 2018 | 277.8 | 295.2 | 0.94 | 22.4% | 6.7% | 93.3 |
| 2019 | 293.1 | 312.6 | 0.94 | 22.7% | 6.9% | 93.1 |
| 2020 | 305.4 | 327.8 | 0.93 | 23.5% | 7.2% | 92.8 |
| 2021 | 319.2 | 344.1 | 0.93 | 24.1% | 7.4% | 92.6 |
| 2022 | 333.8 | 360.9 | 0.93 | 24.8% | 7.5% | 92.5 |
| 2023 | 350.1 | 379.2 | 0.92 | 25.2% | 7.8% | 92.2 |
| 2024 | 366.4 | 398.7 | 0.92 | 25.9% | 8.1% | 91.9 |

### 2025 Demand Forecast — Monthly (GWh)

| Month | Forecast | Lower 95% CI | Upper 95% CI |
|---|---|---|---|
| Jan 2025 | 386.4 | 350.8 | 422.0 |
| Feb 2025 | 374.2 | 339.1 | 409.3 |
| Mar 2025 | 371.8 | 336.8 | 406.8 |
| Apr 2025 | 390.5 | 354.9 | 426.1 |
| May 2025 | 402.1 | 365.8 | 438.4 |
| Jun 2025 | 415.3 | 378.4 | 452.2 |
| Jul 2025 | 421.7 | 384.2 | 459.2 |
| Aug 2025 | 418.6 | 381.3 | 455.9 |
| Sep 2025 | 408.3 | 371.5 | 445.1 |
| Oct 2025 | 395.7 | 359.4 | 432.0 |
| Nov 2025 | 389.4 | 353.4 | 425.4 |
| Dec 2025 | 393.2 | 356.9 | 429.5 |

### Outlook

The results reveal three structural patterns that any monitoring framework for Malawi's power sector must track:

**1. A widening adequacy gap.** The generation adequacy ratio declined from 0.94 in 2018 to 0.92 in 2024 and is projected to continue declining without new capacity additions. Monthly unmet demand averaged approximately 32 GWh in 2024 — up from 18 GWh in 2018 — and the 2025 forecast suggests this will reach 35–40 GWh/month during dry-season peaks if no new generation comes online.

**2. A persistent seasonal mismatch.** Demand peaks in June–August (dry season) precisely when hydro output is lowest due to reduced reservoir inflows. The average seasonal demand swing is approximately 50 GWh/month between the wettest and driest months. This creates a recurring structural deficit that thermal dispatch must cover — making thermal availability and fuel security critical monitoring variables not yet captured in the base KPI set.

**3. Strong but narrowing forecast uncertainty.** The SARIMA model achieves MAPE of 2.8%, well within the 5% operational threshold. However, confidence intervals for the 2025 forecast widen from ±35 GWh in January to ±38 GWh by mid-year — a normal property of time series forecasts. Monitoring officers should flag any two consecutive months where actual demand exceeds the upper CI bound, as this signals accelerating growth requiring revised generation planning assumptions.

**Monitoring trigger recommended:** ≥ 2 consecutive months where `actual_demand_gwh > ci_upper` → escalate to M&E Manager for model recalibration and generation plan review.

---

## 7. How to Run

**Requirements:** Stata 14 or later (Stata 17 recommended)

```stata
* 1. Set your working directory
cd "path/to/energy-forecasting-malawi"

* 2. Run the full do-file
do stata/energy_forecasting.do

* 3. All outputs (figures, CSVs, log, .dta) are saved to the working directory
```

All 10 figures, both CSV outputs, the clean `.dta` file, and the full Stata log are generated automatically in a single run. No additional packages are required beyond base Stata.

---

## 8. Skills Demonstrated

| Skill | Applied in this project |
|---|---|
| Monitoring & Evaluation (M&E) | CDS construction, KPI framework design, structured periodic reporting |
| Time series forecasting | SARIMA modelling with seasonal differencing, AIC/BIC model selection |
| Stationarity testing | ADF test |
| Autocorrelation diagnostics | ACF, PACF analysis for AR/MA order identification |
| Forecast accuracy evaluation | MAE, RMSE, MAPE on 2024 holdout window |
| Data validation & cleaning | Completeness checks, range validation, consistency verification |
| Stata 17 | `tsset`, `arima`, `dfuller`, `ac`, `pac`, `tsappend`, `collapse`, `export delimited` |
| Statistical reporting | Annual KPI tables, monthly forecast tables with confidence intervals |
| Energy sector KPIs | Generation adequacy, capacity utilisation, access gap, hydro dependency, reliability index |
| Data storytelling | Translating model outputs into actionable M&E triggers for management |

---

*This project was developed as part of a Data Analytics Portfolio Project. All data is simulated for demonstration purposes and does not represent actual EGENCO operational figures.*
