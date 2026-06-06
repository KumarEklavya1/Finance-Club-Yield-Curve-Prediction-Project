# Finance-Club-Yield-Curve-Prediction-Project
# Stochastic Interest Rate Modelling: CIR and CIR++ Implementation

**Objective:** To implement, calibrate, and extend the Cox-Ingersoll-Ross (CIR) short-rate model, and evaluate its predictive power in reconstructing the yield curve from a single observable 3-Month rate. 
**Institution:** Finance Club, IIT Roorkee

---

## 📌 Executive Summary
This project develops a robust quantitative framework to predict the full term structure of interest rates (6 Months to 30 Years) using only the instantaneous 3-Month short rate ($r_t$). By identifying the structural flaws of single-factor mean-reverting models in non-stationary macroeconomic environments, this project successfully extends the base framework to a Time-Dependent CIR++ model calibrated via Bayesian Markov Chain Monte Carlo (MCMC). 

**Final Out-of-Sample Predictive Accuracy ($R^2$): 0.9285**

---

## 🛠️ Milestone A: Data Engineering & Preprocessing
The foundation of stochastic calibration relies on continuous, mathematically viable data. The raw dataset containing daily bond yields across 9 maturities was processed as follows:
* **Chronological Alignment:** Formatted the `Date` column into continuous `datetime` objects to serve as the time-series index.
* **Time Step Definition:** The stochastic differential equation requires a precise time step ($dt$). We defined $dt = 1/252$ to reflect the standard annualization of 252 active trading days. This ensures our parameters (like volatility $\sigma$) scale correctly to annualized figures rather than daily variances.
* **Data Integrity:** Minimum/Maximum yield checks confirmed the absence of negative rates or absolute zero entries (which would break the Feller condition) and highlighted no extreme typo-induced outliers.

---

## 🧮 Milestone B: Base CIR Model Calibration & Mechanics
The continuous CIR model is defined by the stochastic differential equation: 
$$dr_{t}=\kappa(\theta-r_{t})dt+\sigma\sqrt{r_{t}}dW_{t}$$

**Calibration Methodology & Sensitivity:** The yield curve proved hyper-sensitive to the choice of calibration methodology. Unconstrained Maximum Likelihood Estimation (MLE) violently overfitted the daily variance, resulting in extreme mean-reversion speeds ($\kappa > 2.0$) that completely failed to predict long-term maturities. Conversely, Generalised Method of Moments (GMM) and Bayesian MCMC proved vastly superior by prioritizing macro-level drift and variance moments over micro-daily noise.

**The Feller Condition & Market Conditions:** Unconstrained Ordinary Least Squares (OLS) on a multi-year downward-trending market forces the speed of mean reversion ($\kappa$) and the long-term mean ($\theta$) into negative territory. This breaks the Feller condition ($2\kappa\theta\ge\sigma^{2}$) in practice because the algorithm extrapolates the downward trend to below zero. We handled this by applying a Constrained Optimization (via `scipy.optimize`), strictly bounding $\kappa, \theta > 0$.

---

## 📉 Milestone C: The Prediction Challenge & Base Shortcomings
Using the calibrated parameters and the closed-form CIR pricing formulas, we theoretically reconstructed the yield curve using only the 3M short rate.

**Prediction Analysis & Failures:**
1. **Systematic Overestimation:** The base CIR model systematically overestimates long-term yields. Because it is strictly mean-reverting, our constrained calibration on 8 years of trending data resulted in an artificially massive $\theta$ target (exceeding 300%). This acts as a mathematical magnet, pulling long-term yield predictions violently upward, resulting in a failing base out-of-sample $R^{2}$ of 0.6921.
2. **Hardest Maturities to Fit:** The ultra-long ends (20Y and 30Y) are the most difficult to fit. The 3M rate effectively dictates the "Level" of the curve, but long-term maturities contain macro-term premiums (like inflation expectations) that a single short-rate proxy simply cannot encapsulate.

---

## 🚀 Milestone D: Model Extensions and Hyperparameter Tuning
To correct the structural overestimation, we pursued advanced mathematical extensions and conducted exhaustive hyperparameter tuning across all 1,976 historical windows to isolate the true macroeconomic regime without succumbing to data snooping.

**Evaluating the Extensions:**
1. **Time-Dependent Parameters (CIR++):** We utilized the Brigo-Mercurio CIR++ model, calculating an Empirical Deterministic Shift ($\Psi(\tau)$) exactly on $t=0$. This structure mathematically forces the model to fit the initial yield curve exactly, acting as an error-correction layer that directly counteracts the base model's long-term overestimation.
2. **Jump-Diffusion (CIR-J):** We implemented and empirically discarded Jump-Diffusion. While jump processes change the qualitative shape of the curve by adding fat tails for sudden stress events, our dataset's primary challenge was a slow-moving macro-trend. Exhaustive tuning proved that CIR-J overfit to a tiny 70-day window of micro-shocks, resulting in a degraded $R^2$ (0.8722).
3. **Two-Factor Models (Longstaff-Schwartz):** Two-factor models introduce massive estimation challenges because the second macro-factor ($y_t$) is unobservable. Even when utilizing a Dynamic State-Space Kalman Filter, the 7-parameter structure forced the algorithm to overfit to a 50-day micro-regime.

**🏆 The Champion Model: Bayesian MCMC + CIR++**
By combining the CIR++ deterministic shift with Bayesian Markov Chain Monte Carlo (MCMC) estimation, we achieved our peak performance. MCMC integrated over parameter uncertainty, finding a highly stable optimal macro-regime of **280 Days** (approx. 1.1 trading years). This successfully reconstructed the curve with an elite **Out-of-Sample $R^{2}$ of 0.9285**.

---

## 🔍 Milestone E: Critical Analysis & Real-World Implications

**1. The Implication of Mean Reversion ($\kappa$)**
Our optimal Bayesian calibration yielded a relatively slow speed of mean reversion ($\kappa \approx 0.10$). In a real-world scenario, this implies that interest rate shocks are highly persistent. When central banks alter policy rates, the effects do not snap back immediately; they reverberate through the yield curve for roughly 8 to 10 years before fully reverting to the long-term mean.

**2. Overfitting in Complex Quantitative Frameworks**
Our empirical testing proved that increasing mathematical complexity actually *degrades* out-of-sample generalization if the model is allowed to overfit high-frequency micro-regimes. The highly complex Two-Factor Kalman Filter and Jump-Diffusion models overfit to 50-day and 70-day windows, respectively. The Bayesian CIR++ model meaningfully improved performance specifically because its probabilistic nature and 280-day regime prevented it from memorizing market noise. We verified this by deploying a Deep Neural Network (DNN) which entirely collapsed out-of-sample due to parameter memorization.

**3. Limitations of the CIR++ Extension (Microstructural Noise)**
While the time-dependent CIR++ extension mathematically cured our predictive variance, it introduces a dangerous theoretical vulnerability in real-world trading: **Microstructural Noise Sensitivity**. Because the framework relies on fitting the initial term structure ($t=0$) with zero error, the entire predictive shift ($\Psi(\tau)$) is entirely dependent on the market conditions of that single calibration day. If Day 0 contains a sudden, transient liquidity shock, that temporary anomaly is permanently baked into the model's forward pricing, potentially mispricing real-world derivatives over the long term.

---
*Project successfully completed for the IIT Roorkee Finance Club Quantitative Challenge.*
