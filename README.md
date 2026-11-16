# Business Investment Analysis

An econometric study of what drives U.S. business investment, with forecasts and "what-if" scenarios. Everything lives in one Jupyter notebook, [Final_Script_Forecast_Simulation_v3.ipynb](Final_Script_Forecast_Simulation_v3.ipynb).

The notebook models quarterly growth in **real gross private domestic investment** (FRED `GPDIC1`) as a function of corporate profits, GDP, industrial production, corporate bond yields and producer prices, using U.S. data from 1962 to 2025Q2.

## Data

The notebook reads `merged_quarterly_data.csv`, which is **not included in this repository**. It is a quarterly file with these columns (all from FRED):

| Column | Series |
|---|---|
| `observation_date` | Quarter start date (1962-01-01 onward) |
| `GPDIC1` | Real gross private domestic investment, billions of chained 2017 $ (SAAR). **Dependent variable** |
| `CP` | Corporate profits after tax |
| `GDPC1` | Real GDP |
| `INDPRO` | Industrial production index |
| `WBAA` | Moody's BAA corporate bond yield (%) |
| `WPUFD41312` | Producer price index (final demand) |

The dataset has 254 quarterly observations (253 after differencing).

Section K5 also downloads newer BEA data (NIPA Table 1.1.6) from the [DBnomics API](https://db.nomics.world/), so it needs an internet connection.

## Setup

The notebook was written for **Google Colab**, but runs in any Jupyter environment with Python 3.

```bash
pip install pandas numpy matplotlib seaborn scipy statsmodels scikit-learn
```

Before running it:

1. Put `merged_quarterly_data.csv` somewhere reachable and update the path in cell 2 (`url = "/merged_quarterly_data.csv"`).
2. If you are not on Colab, delete or comment out `from google.colab import drive` in the first cell.

Then run all cells in order. A few cells save plots to the working directory (`eda_pairplot_updated.png`, `qqplot_updated.png`).

## What the notebook does

### 1. Data preparation and EDA
- Histograms and summary statistics of the raw series. Most are skewed, so they are transformed.
- **Transformations:** log-differences (quarterly growth rates) for `GPDIC1`, `CP`, `GDPC1` and `INDPRO`; simple first differences for `WBAA` and `WPUFD41312`.
- Histograms, box plots, a correlation heatmap and a scatterplot matrix of the transformed variables.
- **Augmented Dickey-Fuller tests:** every transformed series is stationary except the PPI change (p = 0.135).
- **Extra features:** squared GDP growth, a profits × industrial production interaction, and a `post_2008` dummy.
- **VIF check:** all VIFs are below 4, so multicollinearity is not a concern.

### 2. Regression models

| Model | Specification | Adj. R² |
|---|---|---|
| 1 | Basic OLS on the five drivers | 0.640 |
| 2 | + squared GDP growth | 0.657 |
| 3 | + profits × industrial production interaction | 0.640 |
| 4 | + `post_2008` dummy | 0.650 |
| 5 | Model 1 with standardized regressors | 0.640 |

**Diagnostics:**
- Breusch-Pagan and White tests find heteroskedasticity, so HC1 robust standard errors are reported.
- Breusch-Godfrey test (4 lags): no autocorrelation for Models 1, 3, 4 and 5. Model 2 shows some (p = 0.019).
- Shapiro-Wilk tests and Q-Q plots: residuals are not normal, except for Model 2 (p = 0.054).

**Model 4** is used for forecasting and scenarios.

### 3. Forecasting (K1–K5)

| Step | What it does |
|---|---|
| **K1** Accuracy test | Rolling one-quarter-ahead forecasts for 2010–2019 and 2021–2025 (2020 is excluded because of COVID). Compares Model 4 with actual drivers, Model 4 with VAR-forecast drivers, AR(1) and the historical mean. Includes a Diebold-Mariano test. |
| **K2** Point forecast | Forecasts the drivers 8 quarters ahead with a VAR (lag order chosen by AIC, max 4) and feeds them into Model 4, giving 95% prediction intervals. |
| **K3** Simulation | 10,000 simulated futures (VAR driver paths + bootstrapped Model 4 residuals), giving a fan chart and P(decline). |
| **K4** Backtest | Re-runs K2/K3 from each past quarter at 1–4 quarters ahead. Scores RMSE, interval coverage and Brier score. |
| **K5** Real-data check | Compares the forecasts with BEA data released after the dataset ends (2025Q3–2026Q2). |

**Main findings:**
- Model 4 + VAR is **not significantly better than the historical mean** (DM p = 0.98 for 2010–2019, 0.37 for 2021–2025).
- With the actual drivers known, Model 4 is much more accurate (RMSE 1.44 pp vs 2.03 pp in 2010–2019).
- K2's 95% intervals are well calibrated. K3's 90% intervals are too wide.
- On real post-2025Q2 data, the K2 forecast did well (RMSE 0.42 pp vs 0.65 pp for the mean), and all 4 quarters fell inside the K3 interval.

### 4. What-if analysis (L1–L3)

- **L1 Sensitivity:** the effect of a 1-standard-deviation move in each driver, with robust 95% CIs. GDP growth (+1.90 pp) and industrial production (+1.46 pp) are the only clearly significant drivers. Profits, bond yields and PPI have small or insignificant effects.
- **L2 Scenarios:** 10,000 simulations each (coefficient uncertainty + residuals) for recession, normal, boom and a custom scenario. You can edit the custom scenario's inputs in the cell to ask your own question.
- **L3 Dollars:** converts the scenarios into dollar changes from the latest investment level (about $4,383 billion, annual rate).

| Scenario | Median growth | Median change | P(decline) |
|---|---|---|---|
| Recession | −1.06% | about −$46 bn | 70% |
| Normal | +1.77% | about +$78 bn | 20% |
| Boom | +4.18% | about +$187 bn | 3% |
| Custom: GDP flat, yields +0.5 | +0.82% | about +$36 bn | 34% |

## Conclusion

Business investment is driven mainly by **demand** (GDP and industrial production growth), consistent with the accelerator effect. Profits and borrowing costs matter much less in this data. Model 4 is useful for explaining investment and for scenario analysis, but its forecasts are not significantly more accurate than a simple historical average.

**Limitation:** the data are national aggregates, so the results describe how businesses respond in total, not individual firms or industries.
