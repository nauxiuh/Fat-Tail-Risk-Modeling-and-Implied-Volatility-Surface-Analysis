# Fat-Tail Risk Modeling and Implied Volatility Surface Analysis

This project looks at what happens when you stop assuming financial returns are normally distributed. Using Bitcoin price history and S&P 500 options data, it covers distribution fitting, tail risk estimation (VaR/CVaR), implied volatility curve construction, VIX replication, and extracting risk-neutral moments from the options market.

The through-line is practical: standard models that assume normality tend to underestimate how bad things can get. This project shows by how much, and builds better alternatives.

## Repository Structure

```
├── FinancialDataAnalysis.ipynb             # Main analysis notebook
└── data/
    ├── bitcoin_data.csv                    # Bitcoin adjusted close prices
    └── spx-options-exp-2025-03-13-weekly.csv  # SPX options chain (exp. 2025-03-13)
```

## Analysis

### 1. Bitcoin Return Distribution: Why Normality Fails

Daily log returns are computed and tested for normality using Shapiro-Wilk and QQ-plots.

| Metric | Value |
|--------|-------|
| Mean | 0.0007 |
| Std Dev | 0.0357 |
| Skewness | -0.9791 |
| Kurtosis | 17.05 (normal = 3) |

Shapiro-Wilk p-value is essentially zero. Bitcoin returns are left-skewed and have kurtosis nearly six times that of a normal distribution, meaning extreme negative returns happen much more often than standard models would predict. Any risk framework built on normality is going to miss this.

### 2. Finding the Right Distribution (MLE + KS Test)

Three distributions are fitted via maximum likelihood and tested with Kolmogorov-Smirnov and QQ-plots:

| Distribution | KS p-value | Verdict |
|---|---|---|
| Normal | 4.04e-21 | Rejected |
| Student-t | 0.1081 | Acceptable |
| **Normal Inverse Gaussian (NIG)** | **0.5372** | **Best fit** |

NIG comes out on top because it has separate parameters for tail heaviness and asymmetry, which is exactly what you need for an asset that tends to crash hard but recover unevenly.

### 3. VaR and CVaR: Tail Risk Under Two Methods

Tail risk is estimated at 5% and 1% significance levels using historical simulation and a parametric NIG approach:

| Method | Level | VaR | CVaR |
|---|---|---|---|
| Historical Simulation | 5% | -5.59% | -8.63% |
| Historical Simulation | 1% | -10.48% | -14.18% |
| Parametric NIG | 5% | -5.34% | -8.73% |
| Parametric NIG | 1% | -10.69% | **-14.79%** |

At the 1% level, the NIG model gives a slightly more conservative CVaR than history alone. That is because it uses the full fitted distribution rather than the handful of extreme observations in the sample. For risk management purposes, underestimating CVaR means holding less capital than you actually need for tail events.

### 4. SPX Implied Volatility Curve

The implied volatility curve is built from SPX options expiring 2025-03-13 (data date: 2025-02-13, T = 30 days):
- Forward price recovered from ATM options via put-call parity (F = 6124.35, S0 = 6114.65)
- OTM calls converted to synthetic puts to fill in the right side of the curve
- Implied volatility solved numerically using bisection on the Black-Scholes formula

The curve shows a clear volatility smirk. Deep out-of-the-money puts are priced with much higher implied vol than calls at the same distance from the money. Investors are willing to pay up for downside protection, and the smirk reflects that demand directly.

### 5. VIX Replication from Scratch

The CBOE VIX methodology is replicated using the OTM options strip:

| | Value |
|---|---|
| Calculated VIX | **14.30%** |
| CBOE Reported VIX | 15.10% |
| Difference | 0.80 |

The 0.80 difference comes from a few sources: the dataset covers fewer strikes than CBOE uses, the calculation is anchored to a single expiry rather than interpolated across two, and the 1/K^2 weighting means any missing deep OTM puts have an outsized effect on the total. Even so, 14.30 vs 15.10 is a reasonable replication given those constraints. More importantly, going through this process makes clear that VIX is just a model-free variance swap rate, not a black box.

### 6. Real-World vs. Risk-Neutral Moments (BKM Framework)

Risk-neutral moments are extracted from options prices using the Bakshi-Kapadia-Madan framework and compared to historical S&P 500 monthly returns:

| Metric | Real-World (Monthly) | Risk-Neutral (30-day) |
|---|---|---|
| Mean | 0.0157 | 0.0027 |
| Std Dev | 0.1635 | 0.0427 |
| Skewness | -0.979 | **-6.927** |
| Kurtosis | 17.05 | **117.97** |

The skewness and kurtosis numbers are striking. The options market is pricing a left tail roughly seven times more extreme than what you see in realized returns. This is not just noise. The gap between real-world and risk-neutral distributions represents the volatility and skewness risk premiums, which are real and persistent phenomena that options traders and hedge funds actively try to harvest.

### 7. NIG Calibration to SPX Market Prices (Risk-Neutral)

A NIG distribution is calibrated to SPX put prices by minimizing mean squared error using differential evolution over 50,000 Monte Carlo paths, with a 70/30 train-test split:

| Parameter | Value | What it captures |
|---|---|---|
| alpha = 34.71 | Tail shape | Frequent but moderate jumps, not catastrophic ones |
| beta = -0.817 | Asymmetry | Heavy left tail, which is the source of the volatility skew |
| delta = 0.025 | Scale | Relatively low volatility over the 30-day window |
| mu = 0.024 | Location | Keeps the distribution centered under the risk-neutral measure |

Out-of-sample performance on the held-out test set shows the model prices closely match market prices across all moneyness levels, which is a good sign that the fitted distribution is capturing the actual implied risk-neutral density rather than just overfitting.

## Key Takeaways

- **Normal distribution is not appropriate for crypto or equity tail risk.** A kurtosis of 17 vs. 3 is a significant modeling error, not a minor correction.
- **NIG fits Bitcoin returns better than Student-t** because it handles skewness and tail weight independently.
- **CVaR tells you more than VaR.** At the 1% level, the gap between them is over 4 percentage points, which is material for capital planning.
- **The options market is structurally more pessimistic than history.** Risk-neutral skewness is roughly 7x more negative, and this difference is tradeable.
- **VIX is a variance swap rate built from OTM options.** Replicating it from scratch is the best way to understand why it moves the way it does.

## Dependencies

```bash
pip install numpy pandas scipy matplotlib
```

## Usage

```bash
jupyter notebook FinancialDataAnalysis.ipynb
```

Place `bitcoin_data.csv` and `spx-options-exp-2025-03-13-weekly.csv` in the same directory before running.
