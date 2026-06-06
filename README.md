# Stochastic Interest Rate Modelling: CIR and CIR++ Implementation

**Objective:** To implement, calibrate, and extend the Cox-Ingersoll-Ross (CIR) short-rate model, and evaluate its predictive power in reconstructing the yield curve from a single observable 3-Month rate.
**Institution:** Finance Club, IIT Roorkee
**Project Type:** Single-Person Project (2 Weeks)
**Final Out-of-Sample Predictive Accuracy (R-Squared):** 0.9285

---

## 1. The Intricacies of the CIR Model

### What is the CIR Model and Where is it Used?
Introduced by Cox, Ingersoll, and Ross in 1985, the CIR model describes the evolution of the instantaneous short rate. In the financial industry, quantitative analysts and institutional risk managers rely on models like CIR to price interest rate derivatives, value bonds, and manage portfolio risks. 

### The Mathematical Framework
The model is defined by the stochastic differential equation (SDE):
dr(t) = kappa(theta - r(t))dt + sigma * sqrt(r(t))dW(t)

It relies on two foundational properties:
1. **Mean Reversion:** Governed by the term kappa(theta - r(t)), the model assumes that if rates deviate too high or too low, they are pulled back toward a long-run mean (theta) at a specific speed of mean reversion (kappa).
2. **Strictly Positive Rates:** The diffusion term uses a square-root process. This ensures that interest rates cannot become negative, provided the Feller condition (2 * kappa * theta >= sigma^2) is satisfied. 

From this SDE, the model derives a closed-form zero-coupon bond pricing formula: P(t,T) = A(t,T)e^(-B(t,T)r(t)), where A and B are deterministic functions of the model parameters.

### Where and Why the Base CIR Model Fails
While mathematically tractable, the base CIR model has well-documented limitations. It fails heavily when data is non-stationary. If an economy experiences a multi-year downward trend in interest rates, the assumption of a static, long-term mean (theta) breaks down. Furthermore, the base model cannot perfectly fit an arbitrary initial term structure, struggles to capture sudden market shocks, and forces the yield curve into restrictive shapes because it relies entirely on a single factor.

---

## 2. Data Engineering & Preprocessing
The foundation of stochastic calibration relies on mathematically viable data. The raw dataset contained 1,976 daily observations across 9 specific bond maturities ranging from 3 Months to 30 Years.

* **Formatting Consistency:** The raw dataset contained hidden whitespace in column names (e.g., " ZC025YR"), which broke indexing and required algorithmic stripping.
* **Missing Value Analysis:** Financial data often hides missing days as 0.0 or negative numbers. Diagnostic checks confirmed the absolute minimum yield was 0.000486 (0.0486%), meaning the data was mathematically viable for square-root diffusion without requiring complex interpolation for lazy zero entries.
* **The Time-Step (dt) Definition:** The CIR model requires a precise change in time (dt). Setting dt = 1/365 would dilute volatility by assuming markets trade on weekends. We rigidly defined dt = 1/252 to reflect active trading days, ensuring all extracted parameters were correctly annualized.

---

## 3. The Base Calibration Pipeline: Failures and Lessons

We rigorously tested multiple calibration methodologies, each exposing a theoretical flaw in the base model.

### Unconstrained Ordinary Least Squares (OLS)
* **What we did:** Discretized the SDE into a linear regression formula to isolate kappa, theta, and sigma.
* **Why it failed:** The dataset contained a massive downward trend. Unconstrained OLS extrapolated this trend into infinity, resulting in a negative mean-reversion speed (kappa = -0.2548) and a negative long-term target (theta = -0.0053). This caused "mean aversion," actively pulling rates below zero and violating the fundamental premise of the CIR model.

### Constrained Least Squares Optimization
* **What we did:** Used optimization with strict boundaries (kappa > 0, theta > 0) to force the algorithm to respect the Feller condition.
* **Why it performed poorly:** To avoid breaking constraints during a downtrend, the optimizer found a mathematical loophole. It set the target (theta) to an absurd 343% (3.438) while slowing kappa to a microscopic crawl (0.000689). This acted as a mathematical magnet, violently pulling all long-term yield predictions upward. The model predicted a 30-Year yield of 7.07% when reality was 3.6%, leading to an out-of-sample R-squared of 0.6921.

### Maximum Likelihood Estimation (MLE)
* **What we did:** Attempted to optimize the Negative Log-Likelihood based on the exact probability distribution of the CIR process.
* **Why it failed:** We encountered the "Exploding Exponential" trap. The optimizer tested massive parameters causing e^(h * tau) to exceed 64-bit limits, resulting in NaN and Overflow crashes. Applying hard bounds (Regularized MLE) stabilized the math, yielding an R-squared of 0.8448. However, MLE became obsessed with fitting microscopic daily variance, cranking kappa to a violent 2.269. This perfected daily volatility but destroyed the macro shape of the curve.

---

## 4. The Model Evolution: Rejecting Complex Alternatives

To correct the systematic failures of the base model, we researched advanced extensions. We implemented them, backtested them, and empirically rejected them due to the Overfitting Trap.

* **Rejected: Jump-Diffusion (CIR-J)**
  * *Purpose:* Incorporates Poisson jump processes to handle sudden macroeconomic shocks.
  * *Why it failed:* Our dataset's primary challenge was a slow-moving macro-trend, not sudden overnight crashes. Exhaustive tuning proved that CIR-J overfit to a tiny 70-day window of micro-shocks, resulting in a degraded R-squared of 0.8722.
* **Rejected: Two-Factor Models (Longstaff-Schwartz) & Kalman Filters**
  * *Purpose:* Introduces a second stochastic factor to capture yield curve variations such as level versus slope.
  * *Why it failed:* Because the project constrained us to observing only the 3-Month rate, the two underlying drivers (x_t, y_t) were unobservable. Even when utilizing Dynamic State-Space Kalman Filtering to track these hidden components, the 7-parameter structure forced the algorithm to overfit, memorizing a 50-day micro-regime (R-squared = 0.9031).

---

## 5. The Champion Framework: Bayesian MCMC + CIR++

We achieved peak out-of-sample predictive power by combining a specific mathematical structure with an elite calibration mechanic.

1. **The Structure: Time-Dependent Parameters (CIR++)**
   We transitioned to the Brigo-Mercurio CIR++ model. By allowing parameters to act as deterministic functions of time, we calculated an Empirical Deterministic Shift exactly on t=0. This structure mathematically forces the model to fit the initial yield curve perfectly, acting as an error-correction layer that completely neutralizes the base model's systematic long-term overestimation.
2. **The Mechanic: Bayesian Markov Chain Monte Carlo (MCMC) + Regime Isolation**
   Calibrating over 8 years of chaotic macroeconomic data caused parameter collapse. We wrapped our calibration in an optimization loop to find the optimal "Lookback Window," utilizing Bayesian MCMC. MCMC integrates over parameter uncertainty via thousands of probability simulations rather than making a single strict guess. 
3. **The Result:** MCMC naturally smoothed out the noise that tricked the complex models, finding a highly stable optimal macro-regime of 280 Days (approx. 1.1 trading years). This setup flawlessly reconstructed the curve, achieving a peak Out-of-Sample R-squared of 0.9285.

---

## 6. Answers to Key Project Questions

### Part 1: Model Mechanics and Calibration
**Q: How sensitive is the calibrated yield curve to the choice of calibration methodology?**
**A:** The yield curve is hyper-sensitive to the methodology. Unconstrained MLE violently overfitted daily variance, resulting in extreme mean-reversion speeds (kappa > 2.0) that completely failed to predict long-term maturities. Ordinary Least Squares (OLS) and Generalised Method of Moments (GMM) proved vastly superior by prioritizing macro-level drift and variance moments over micro-daily noise.

**Q: Under what market conditions does the Feller condition break down in practice, and how do you handle it?**
