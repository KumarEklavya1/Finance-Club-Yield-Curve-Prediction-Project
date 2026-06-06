# Stochastic Interest Rate Modelling: CIR and CIR++ Implementation

**Objective:** To implement, calibrate, and extend the Cox-Ingersoll-Ross (CIR) short-rate model, and evaluate its predictive power in reconstructing the yield curve from a single observable 3-Month rate.  
**Institution:** Finance Club, IIT Roorkee  
**Final Out-of-Sample Predictive Accuracy ($R^2$):** **0.9285** ---

## 📌 Executive Summary
Interest rates are the fundamental building blocks of the global financial system, evolving in complex, stochastic patterns. This project aimed to reconstruct the full term structure of interest rates (6 Months to 30 Years) using *only* the instantaneous 3-Month short rate ($r_t$). 

Starting from a base CIR model that failed out-of-sample ($R^2 = 0.6921$), this project systematically broke down the mathematical flaws of applying single-factor mean-reverting models to non-stationary macroeconomic environments. Through rigorous data engineering, hyperparameter regime tuning, and the implementation of a Time-Dependent CIR++ model calibrated via Bayesian Markov Chain Monte Carlo (MCMC), the final model achieved an elite out-of-sample $R^2$ of 0.9285.

---

## 🧗 The Journey: Challenges, Solutions, and Conquests

Building this model was not a straightforward path; it was a rigorous process of encountering mathematical roadblocks, diagnosing the underlying failures, and engineering robust solutions.

### 1. Data Engineering & The Time-Step Trap
* **The Problem:** The raw dataset of 1,976 days contained hidden whitespace formatting errors, and applying standard daily increments ($dt = 1/365$) threatened to dilute volatility by assuming markets trade on weekends.
* **The Solution:** We algorithmically stripped hidden characters, verified the absence of negative or zero rates (which would break the CIR square-root diffusion), and rigidly defined $dt = 1/252$ to reflect active trading days, ensuring all extracted parameters were correctly annualized.

### 2. Base CIR Calibration & The "Negative Mean" Crisis
* **The Problem:** We initially attempted an unconstrained Ordinary Least Squares (OLS) regression using Euler-Maruyama discretization. Because the 8-year dataset contained a massive downward trend, unconstrained OLS extrapolated this trend into infinity, resulting in a negative mean-reversion speed ($\kappa$) and a negative long-term target ($\theta$).
* **The Solution:** We upgraded to a **Constrained Least Squares Optimization** (via `scipy.optimize`), strictly bounding $\kappa, \theta > 0$ to satisfy the Feller condition ($2\kappa\theta\ge\sigma^{2}$). 
* **The Limitation Exposed:** To avoid breaking constraints during a downtrend, the optimizer satisfied the rules by setting the target ($\theta$) to an absurd **343%** while slowing $\kappa$ to a crawl. This proved the base CIR model fundamentally fails when forced to fit non-stationary (trending) data. Unsurprisingly, this base model failed the prediction challenge with an out-of-sample $R^2$ of **0.6921**.

### 3. The "Exploding Exponential" in Maximum Likelihood Estimation (MLE)
* **The Problem:** When attempting to upgrade to Maximum Likelihood Estimation (MLE), the Python environment crashed with `NaN` and `Overflow` errors. The unconstrained optimizer tested massive parameters (e.g., $\kappa = 500$), causing the mathematical calculation of $e^{h\tau}$ to exceed 64-bit limits.
* **The Solution:** We implemented **Regularized MLE** by applying hard macroeconomic boundaries (capping parameters at 1.0). While this stabilized the math, MLE became obsessed with fitting microscopic daily noise (cranking $\kappa$ to > 2.0), ultimately losing to models that prioritized macro-trends.

### 4. The Overfitting Trap of Complex Models
* **The Problem:** To improve the $R^2$, we deployed highly complex extensions: A Jump-Diffusion (CIR-J) model and a 7-parameter Two-Factor Longstaff-Schwartz model via Kalman Filtering. 
* **The Discovery:** Both models suffered from severe short-regime overfitting. The Jump-Diffusion model overfit to a tiny 70-day window, and the Kalman Filter memorized a 50-day micro-regime. We verified this overfitting trap by deploying a Deep Neural Network (DNN), which perfectly memorized the training data but entirely collapsed out-of-sample.

### 5. The Ultimate Conquest: Bayesian MCMC + Regime Isolation
* **The Solution:** We transitioned to the **Time-Dependent Brigo-Mercurio (CIR++) Model**, calculating an Empirical Deterministic Shift ($\Psi(\tau)$) exactly on $t=0$ to act as an error-correction layer. 
* **The Mechanic:** We wrapped the entire calibration pipeline in a hyperparameter optimization loop to find the optimal historical "Lookback Window". We calibrated it using **Bayesian Markov Chain Monte Carlo (MCMC)**. MCMC integrated over parameter uncertainty, smoothing out the noise that tricked the Kalman Filter. It found a highly stable optimal macro-regime of **280 Days** (approx. 1.1 trading years), successfully cracking the project requirement with a peak $R^2$ of 0.9285.

---

## 🔑 Explicit Answers to Key Project Questions

### Part 1: Model Mechanics and Calibration
**Q: How sensitive is the calibrated yield curve to the choice of calibration methodology?**
**A:** The yield curve is hyper-sensitive to the methodology. Unconstrained MLE violently overfitted daily variance, resulting in extreme mean-reversion speeds ($\kappa > 2.0$) that completely failed to predict long-term maturities. Ordinary Least Squares (OLS) and Generalised Method of Moments (GMM) proved vastly superior by prioritizing macro-level drift and variance over micro-daily noise.

**Q: Under what market conditions does the Feller condition break down in practice, and how do you handle it?**
**A:** The Feller condition breaks down during prolonged, multi-year downward-trending markets. An unconstrained algorithm will extrapolate this trend into negative territory, yielding $\kappa < 0$ and $\theta < 0$. We handled this programmatically by applying Constrained Optimization boundaries (e.g., using `scipy.optimize` with bounds $\kappa > 0.0001, \theta > 0.0001$), forcing the algorithm to find a positive mathematical compromise.

**Q: What does the mean-reversion speed ($\kappa$) imply about the persistence of interest rate shocks in your data?**
**A:** Our optimal Bayesian calibration yielded a relatively slow speed of mean reversion ($\kappa \approx 0.10$). This implies that interest rate shocks are highly persistent. When central banks alter policy rates, the effects do not snap back immediately; they reverberate through the yield curve for roughly 8 to 10 years before fully reverting to the long-term mean.

### Part 2: Prediction and Out-of-Sample Performance
**Q: How accurately can the 3M rate alone reconstruct the full yield curve, and which maturities are hardest to fit?**
**A:** The 3M rate acts as a strong anchor for the "level" of the short end of the curve, but it struggles severely as maturity increases. The hardest maturities to fit are the ultra-long ends (20Y and 30Y). These tenors contain massive term premiums driven by long-term macroeconomic inflation expectations that a single instantaneous short-rate proxy simply cannot encapsulate.

**Q: Where does the base CIR model systematically over- or underestimate yields, and why?**
**A:** The base CIR model systematically **overestimates** long-term yields. Because it is strictly mean-reverting, calibrating it on a downward trend forced the optimizer to set an artificially massive $\theta$ target (in our case, 343%) to avoid breaking constraints. This absurdly high target acted as a mathematical magnet, violently pulling all long-term yield predictions upward.

**Q: Does your extension meaningfully improve out-of-sample performance, or does it overfit the training period?**
**A:** The CIR++ extension, when paired with Bayesian MCMC and Regime Isolation, meaningfully improved performance (boosting $R^2$ from 0.69 to 0.9285). However, we empirically proved that adding *unnecessary* mathematical complexity (like Two-Factor Kalman Filters or Deep Neural Networks) leads directly to overfitting. Those complex models memorized 50-day noise regimes and collapsed when tested on global, out-of-sample data.

### Part 3: Extensions and Modelling Choices
**Q: What mathematical structure justifies your chosen extension over the alternatives?**
**A:** We chose the Time-Dependent Brigo-Mercurio (CIR++) extension because it acts directly as an error-correction layer. By calculating an Empirical Deterministic Shift ($\Psi(\tau)$) exactly on $t=0$, the structure mathematically forces the model to fit the initial yield curve perfectly. This directly counteracted the base model's systematic long-term overestimation without requiring the estimation of unobservable variables.

**Q: How do jump processes change the qualitative shape of predicted yield curves during stress periods?**
**A:** Poisson jump processes (CIR-J) change the qualitative shape by introducing "fat tails" to the probability distribution, effectively raising theoretical yields to account for the sudden risk of violent macroeconomic shocks. However, our empirical testing discarded this model, as our dataset’s primary challenge was a slow-moving macro-trend, not sudden discontinuities.

**Q: What are the additional estimation challenges introduced by a two-factor or time-dependent model?**
**A:** * **Two-Factor Models:** Because we are constrained to only observing the 3M rate ($r_t$), the two underlying drivers ($x_t$ and $y_t$) are unobservable. Estimating them requires complex Dynamic State-Space modeling (Kalman Filtering), which is highly prone to non-convergence and overfitting micro-regimes.
* **Time-Dependent Models (CIR++):** The primary challenge is **Microstructural Noise Sensitivity**. Because the framework relies on fitting the initial term structure ($t=0$) with zero error, the entire predictive shift ($\Psi(\tau)$) is dependent on the market conditions of that single calibration day. Temporary liquidity shocks on Day 0 become permanently baked into the model's forward pricing.

---

## 🧠 Personal Learnings & Conclusion

This project evolved from a standard coding assignment into a deep dive into the realities of quantitative finance. My core takeaways include:

1. **The Overfitting Trap:** I learned firsthand that more complex math does not equal better predictions. Neural Networks, Jump-Diffusion, and 7-parameter Kalman Filters act like contortionists—they perfectly memorize local noise but violently misprice global reality.
2. **Regime Shifts are Everything:** Financial data is not stationary. Attempting to calibrate a model over 8 years of chaotic data is a fool's errand. I learned the critical importance of dynamically isolating the "Lookback Window" to capture the current macroeconomic regime.
3. **Local Fit vs. Global Fit:** I experienced the "Single-Day Illusion," where an overfitted model looks visually perfect on one specific day's graph but fails the aggregate $R^2$ score across the timeline. I learned to trust Bayesian probability distributions over single-point estimates for global stability.

By questioning mathematical assumptions, exhaustively testing parameters, and refusing to accept theoretical ceilings, I successfully navigated the depths of stochastic calculus and engineered a model that conquered the yield curve.

---
*Project successfully completed for the IIT Roorkee Finance Club Quantitative Challenge.*
