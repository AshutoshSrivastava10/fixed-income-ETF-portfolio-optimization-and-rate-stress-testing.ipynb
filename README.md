# Fixed-Income ETF Portfolio Optimization & Rate Stress Testing

## Overview

This project builds and evaluates a constrained fixed-income ETF portfolio using mean-variance optimization in Gurobi. The investable universe spans Treasury, investment-grade corporate, high-yield corporate, and municipal bond exposures, while the iShares Core U.S. Aggregate Bond ETF (`AGG`) serves only as the benchmark.

The workflow uses a chronological train-test design to avoid look-ahead bias:

- **Training period:** January 5, 2010 to June 27, 2022
- **Testing period:** June 28, 2022 to August 31, 2026
- **Training observations:** 3,141 daily returns
- **Testing observations:** 1,047 daily returns

The final portfolio is evaluated against an equal-weight portfolio and AGG using out-of-sample return, volatility, Sharpe ratio, maximum drawdown, and tracking error. A duration-based stress test estimates the immediate effect of a parallel 100-basis-point increase in yields.

## Portfolio Universe

| ETF | Exposure | Role |
|---|---|---|
| `SHY` | 1–3 year U.S. Treasuries | Investable |
| `IEF` | 7–10 year U.S. Treasuries | Investable |
| `TLT` | 20+ year U.S. Treasuries | Investable |
| `LQD` | Investment-grade corporate bonds | Investable |
| `HYG` | High-yield corporate bonds | Investable |
| `MUB` | U.S. municipal bonds | Investable |
| `AGG` | Broad U.S. investment-grade bond market | Benchmark only |

## Methodology

### 1. Data preparation

- Download daily adjusted closing prices with `yfinance`.
- Use data from January 4, 2010 through August 31, 2026.
- Calculate simple daily returns from adjusted prices so that ETF distributions are reflected.
- Split observations chronologically, using the first 75% for estimation and the final 25% for out-of-sample testing.

### 2. Model inputs

Annualized expected returns and the covariance matrix are estimated using only the training period:

$$
\hat{\mu}=252\bar{r}_{\text{train}}
$$

$$
\hat{\Sigma}=252\operatorname{Cov}(r_{\text{train}})
$$

### 3. Gurobi quadratic model

The portfolio solves:

$$
\max_w \quad \mu^\top w-\lambda w^\top\Sigma w
$$

with risk-aversion parameter $\lambda=5$ and the following constraints:

$$
\sum_i w_i=1
$$

$$
0\leq w_i\leq0.35
$$

$$
w_{\text{HYG}}\leq0.20
$$

$$
w_{\text{SHY}}+w_{\text{IEF}}+w_{\text{TLT}}\geq0.30
$$

### 4. Out-of-sample evaluation

The optimized and equal-weight portfolios are initialized at the beginning of the test period and then held without rebalancing. They are compared with AGG using:

- Annualized return
- Annualized volatility
- Sharpe ratio, assuming a 0% risk-free rate
- Maximum drawdown
- Annualized tracking error versus AGG

### 5. Rate stress test

For a parallel 100-basis-point increase in yields, each ETF's immediate price impact is approximated using effective duration:

$$
\frac{\Delta P_i}{P_i}\approx-D_i\Delta y
$$

where $\Delta y=0.01$. Effective-duration inputs are dated August 31, 2026.

## Results

### Optimized allocation

| ETF | Weight |
|---|---:|
| `SHY` | 0.00% |
| `IEF` | 27.62% |
| `TLT` | 4.02% |
| `LQD` | 13.64% |
| `HYG` | 20.00% |
| `MUB` | 34.72% |

The portfolio allocated 31.64% to Treasury ETFs, satisfying the 30% minimum. HYG reached its 20% exposure limit, while MUB received the largest allocation at 34.72%.

### Out-of-sample performance

| Portfolio | Annualized Return | Annualized Volatility | Sharpe Ratio | Maximum Drawdown | Tracking Error vs. AGG |
|---|---:|---:|---:|---:|---:|
| **Optimized Portfolio** | **3.39%** | **5.83%** | **0.60** | **−9.18%** | 1.66% |
| Equal-Weight Portfolio | 2.75% | 6.39% | 0.46 | −10.32% | 1.40% |
| AGG Benchmark | 2.78% | 6.08% | 0.48 | −9.79% | 0.00% |

An initial investment of $1 grew to:

| Portfolio | Ending Value |
|---|---:|
| Optimized Portfolio | **$1.149** |
| Equal-Weight Portfolio | $1.119 |
| AGG Benchmark | $1.121 |

During the selected test period, the optimized portfolio produced the highest annualized return and Sharpe ratio, along with the lowest volatility and smallest maximum drawdown among the three portfolios.

### Interest-rate stress test

| Measure | Result |
|---|---:|
| Portfolio effective duration | 6.59 years |
| Parallel rate shock | +100 bps |
| Estimated portfolio return | −6.59% |
| Estimated portfolio loss | 6.59% |

MUB and IEF made the largest contributions to the estimated portfolio loss because of their relatively large weights. TLT had the largest standalone duration sensitivity, but its 4.02% allocation limited its contribution to total portfolio loss.

## Visualizations

The notebook generates and saves the following charts:

- `optimized_etf_allocations.png`
- `out_of_sample_cumulative_returns.png`
- `rate_shock_losses.png`

Add the generated PNG files to the repository if you want to display them directly in this README.

## Repository Structure

```text
.
├── Fixed_Income_ETF_Portfolio_Optimization_&_Rate_Stress_Testing.ipynb
├── README.md
├── requirements.txt
├── optimized_etf_allocations.png
├── out_of_sample_cumulative_returns.png
└── rate_shock_losses.png
```

The three PNG files are generated when the notebook's visualization cells are executed.

## Installation

Python 3.10 or newer is recommended.

```bash
python -m venv .venv
```

Activate the environment:

```bash
# macOS or Linux
source .venv/bin/activate

# Windows PowerShell
.venv\Scripts\Activate.ps1
```

Install the dependencies:

```bash
pip install -r requirements.txt
```

Then open the notebook with JupyterLab:

```bash
jupyter lab
```

The model contains only six continuous variables and can run under the size-limited Gurobi license included with `gurobipy`. A full commercial license is not required for this educational model.

## Reproducibility Notes

- The `yfinance` download end date is set to `2026-09-01`, which is exclusive and therefore includes prices through August 31, 2026.
- The train-test boundary is determined by observation count rather than a manually selected date.
- AGG is excluded from optimization and used only for out-of-sample comparison.
- The optimized allocation is estimated once and is not rebalanced during the test period.
- The duration stress test applies the original optimized target weights rather than the drifted ending weights of the buy-and-hold portfolio.
- Effective durations are recorded as of August 31, 2026.

## Limitations

- Historical sample means and covariances may not represent future market conditions.
- The model uses one fixed risk-aversion value rather than a full sensitivity analysis.
- The backtest excludes transaction costs, taxes, bid-ask spreads, and periodic rebalancing.
- The Sharpe ratio assumes a 0% risk-free rate.
- The stress test is a first-order duration approximation and excludes convexity.
- Credit spreads are assumed unchanged during the interest-rate shock.
- Municipal and corporate yields may not move one-for-one with Treasury yields.

## Potential Extensions

- Add a formal portfolio-duration constraint.
- Test multiple risk-aversion parameters.
- Use rolling or expanding-window estimation and periodic rebalancing.
- Apply Ledoit-Wolf covariance shrinkage.
- Incorporate turnover and transaction costs.
- Add convexity, key-rate-duration, yield-curve, and credit-spread stress scenarios.

## Data Sources

- [yfinance documentation](https://ranaroussi.github.io/yfinance/)
- [iShares fixed-income ETFs](https://www.ishares.com/us/products/bond-etfs)
- [SHY fund page](https://www.ishares.com/us/products/239452/ishares-13-year-treasury-bond-etf)
- [IEF fund page](https://www.ishares.com/us/products/239456/ishares-710-year-treasury-bond-etf)
- [TLT fund page](https://www.ishares.com/us/products/239454/ishares-20-year-treasury-bond-etf)
- [LQD fund page](https://www.ishares.com/us/products/239566/ishares-iboxx-investment-grade-corporate-bond-etf)
- [HYG fund page](https://www.ishares.com/us/products/239565/ishares-iboxx-high-yield-corporate-bond-etf)
- [MUB fund page](https://www.ishares.com/us/products/239766/ishares-national-amtfree-muni-bond-etf)
- [AGG fund page](https://www.ishares.com/us/products/239458/ishares-core-total-us-bond-market-etf)

## Disclaimer

This project is intended for educational and research purposes only. It is not investment advice, and historical or simulated results do not guarantee future performance.

## Author

Ashutosh Srivastava
