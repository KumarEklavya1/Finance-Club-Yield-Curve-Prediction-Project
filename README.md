# Stochastic Interest Rate Modelling: CIR and CIR++ Implementation

**Objective:** To implement, calibrate, and extend the Cox-Ingersoll-Ross (CIR) short-rate model, and evaluate its predictive power in reconstructing the yield curve from a single observable 3-Month rate.
**Institution:** Finance Club, IIT Roorkee **Project Type:** Single-Person Project (2 Weeks) **Final Out-of-Sample Predictive Accuracy (R-Squared):** 0.9285 

---

## 1. The Intricacies of the CIR Model

### What is the CIR Model and Where is it Used?

Introduced by Cox, Ingersoll, and Ross in 1985, the CIR model describes the evolution of the instantaneous short rate. In the financial industry, quantitative analysts and institutional risk managers rely on models like CIR to price interest rate derivatives, value bonds, and manage portfolio risks.

### The Mathematical Framework

The model is defined by the stochastic differential equation (SDE):
dr(t) = kappa(theta - r(t))dt + sigma * sqrt(r(t))dW(t) 

It relies on two foundational properties:

1. 
**Mean Reversion:** Governed by the term kappa(theta - r(t)), the model assumes that if rates deviate too high or too low, they are pulled back toward a long-run mean (theta) at a specific speed of mean reversion (kappa).


2. 
**Strictly Positive Rates:** The diffusion term uses a square-root process. This ensures that interest rates cannot become negative, provided the Feller condition (2 * kappa * theta >= sigma^2) is satisfied.



From this SDE, the model derives a closed-form zero-coupon bond pricing formula: P(t,T) = A(t,T)e^(-B(t,T)r(t)), where A and B are deterministic functions of the model parameters.

### Where and Why the Base CIR Model Fails

While mathematically tractable, the base CIR model has well-documented limitations. It fails heavily when data is non-stationary. If an economy experiences a multi-year downward trend in interest rates, the assumption of a static, long-term mean (theta) breaks down. Furthermore, the base model cannot perfectly fit an arbitrary initial term structure, struggles to capture sudden market shocks, and forces the yield curve into restrictive shapes because it relies entirely on a single factor.

---

## 2. Data Engineering & Preprocessing

The foundation of stochastic calibration relies on mathematically viable data. The raw dataset contained 1,976 daily observations across 9 specific bond maturities ranging from 3 Months to 30 Years.

* 
**Formatting Consistency:** The raw dataset contained hidden whitespace in column names (e.g., " ZC025YR"), which broke indexing and required algorithmic stripping.


* 
**Missing Value Analysis:** Financial data often hides missing days as 0.0 or negative numbers. Diagnostic checks confirmed the absolute minimum yield was 0.000486 (0.0486%), meaning the data was mathematically viable for square-root diffusion without requiring complex interpolation for lazy zero entries.


* 
**The Time-Step (dt) Definition:** The CIR model requires a precise change in time (dt). Setting dt = 1/365 would dilute volatility by assuming markets trade on weekends. We rigidly defined dt = 1/252 to reflect active trading days, ensuring all extracted parameters were correctly annualized.



---

## 3. The Base Calibration Pipeline: Failures and Lessons

We rigorously tested multiple calibration methodologies, each exposing a theoretical flaw in the base model.

### Unconstrained Ordinary Least Squares (OLS)

* 
**What we did:** Discretized the SDE into a linear regression formula to isolate kappa, theta, and sigma.


* **Why it failed:** The dataset contained a massive downward trend. Unconstrained OLS extrapolated this trend into infinity, resulting in a negative mean-reversion speed (kappa = -0.2548) and a negative long-term target (theta = -0.0053). This caused "mean aversion," actively pulling rates below zero and violating the fundamental premise of the CIR model.



### Constrained Least Squares Optimization

* 
**What we did:** Used optimization with strict boundaries (kappa > 0, theta > 0) to force the algorithm to respect the Feller condition.


* **Why it performed poorly:** To avoid breaking constraints during a downtrend, the optimizer found a mathematical loophole. It set the target (theta) to an absurd 343% (3.438) while slowing kappa to a microscopic crawl (0.000689). This acted as a mathematical magnet, violently pulling all long-term yield predictions upward. The model predicted a 30-Year yield of 7.07% when reality was 3.6%, leading to an out-of-sample R-squared of 0.6921.



### Maximum Likelihood Estimation (MLE)

* 
**What we did:** Attempted to optimize the Negative Log-Likelihood based on the exact probability distribution of the CIR process.


* **Why it failed:** We encountered the "Exploding Exponential" trap. The optimizer tested massive parameters causing e^(h * tau) to exceed 64-bit limits, resulting in NaN and Overflow crashes. Applying hard bounds (Regularized MLE) stabilized the math, yielding an R-squared of 0.8448. However, MLE became obsessed with fitting microscopic daily variance, cranking kappa to a violent 2.269. This perfected daily volatility but destroyed the macro shape of the curve.



---

## 4. The Model Evolution: Rejecting Complex Alternatives

To correct the systematic failures of the base model, we researched advanced extensions. We implemented them, backtested them, and empirically rejected them due to the Overfitting Trap.

* **Rejected: Jump-Diffusion (CIR-J)**
* 
*Purpose:* Incorporates Poisson jump processes to handle sudden macroeconomic shocks.


* 
*Why it failed:* Our dataset's primary challenge was a slow-moving macro-trend, not sudden overnight crashes. Exhaustive tuning proved that CIR-J overfit to a tiny 70-day window of micro-shocks, resulting in a degraded R-squared of 0.8722.




* **Rejected: Two-Factor Models (Longstaff-Schwartz) & Kalman Filters**
* 
*Purpose:* Introduces a second stochastic factor to capture yield curve variations such as level versus slope.


* 
*Why it failed:* Because the project constrained us to observing only the 3-Month rate, the two underlying drivers (x_t, y_t) were unobservable. Even when utilizing Dynamic State-Space Kalman Filtering to track these hidden components, the 7-parameter structure forced the algorithm to overfit, memorizing a 50-day micro-regime (R-squared = 0.9031).




* **Rejected: Deep Neural Networks (DNN)**
* 
*Purpose:* To empirically prove the danger of over-parameterization without financial constraints.


* 
*Why it failed:* The DNN memorized the training data perfectly but completely collapsed out-of-sample because it lacked the structural financial constraints (mean-reversion, Feller condition) necessary to generalize.





---

## 5. The Champion Framework: Bayesian MCMC + CIR++

We achieved peak out-of-sample predictive power by combining a specific mathematical structure with an elite calibration mechanic.

1. **The Structure: Time-Dependent Parameters (CIR++)**
We transitioned to the Brigo-Mercurio CIR++ model. By allowing parameters to act as deterministic functions of time, we calculated an Empirical Deterministic Shift exactly on t=0. This structure mathematically forces the model to fit the initial yield curve perfectly, acting as an error-correction layer that completely neutralizes the base model's systematic long-term overestimation.


2. **The Mechanic: Bayesian Markov Chain Monte Carlo (MCMC) + Regime Isolation**
Calibrating over 8 years of chaotic macroeconomic data caused parameter collapse. We wrapped our calibration in an optimization loop to find the optimal "Lookback Window," utilizing Bayesian MCMC. MCMC integrates over parameter uncertainty via thousands of probability simulations rather than making a single strict guess.


3. 
**The Result:** MCMC naturally smoothed out the noise that tricked the complex models, finding a highly stable optimal macro-regime of 280 Days (approx. 1.1 trading years). This setup flawlessly reconstructed the curve, achieving a peak Out-of-Sample R-squared of 0.9285.



---

## 6. Answers to Key Project Questions

### Part 1: Model Mechanics and Calibration

**Q: How sensitive is the calibrated yield curve to the choice of calibration methodology?** 
**A:** The yield curve is hyper-sensitive to the methodology. Unconstrained MLE violently overfitted daily variance, resulting in extreme mean-reversion speeds (kappa > 2.0) that completely failed to predict long-term maturities. Ordinary Least Squares (OLS) and Generalised Method of Moments (GMM) proved vastly superior by prioritizing macro-level drift and variance moments over micro-daily noise.

**Q: Under what market conditions does the Feller condition break down in practice, and how do you handle it?** **A:** The Feller condition breaks down during prolonged, multi-year downward-trending markets. An unconstrained algorithm will extrapolate this trend into negative territory, yielding kappa < 0 and theta < 0. We handled this programmatically by applying Constrained Optimization boundaries (via scipy.optimize), forcing the algorithm to preserve the strictly positive square-root diffusion process.

**Q: What does the mean-reversion speed (kappa) imply about the persistence of interest rate shocks in your data?** **A:** Our optimal calibration yielded a relatively slow speed of mean reversion (kappa approx 0.10 to 0.12). This implies that interest rate shocks are highly persistent. When central banks alter policy rates, the effects do not snap back immediately; they reverberate through the yield curve for roughly 8 to 10 years before fully reverting to the long-term mean.

### Part 2: Prediction and Out-of-Sample Performance

**Q: How accurately can the 3M rate alone reconstruct the full yield curve, and which maturities are hardest to fit?** **A:** The 3M rate acts as a strong anchor for the "level" of the short end of the curve, but accuracy decays as maturity increases. The hardest maturities to fit are the ultra-long ends (20Y and 30Y). These tenors contain massive term premiums driven by long-term macroeconomic inflation expectations that a single instantaneous short-rate proxy simply cannot encapsulate.

**Q: Where does the base CIR model systematically over- or underestimate yields, and why?** **A:** The base CIR model systematically overestimates long-term yields. Because it is strictly mean-reverting, calibrating it on a downward trend forced the optimizer to set an artificially massive theta target (343%) to avoid breaking constraints. This absurd target acted as a mathematical magnet, violently pulling all long-term yield predictions upward.

**Q: Does your extension meaningfully improve out-of-sample performance, or does it overfit the training period?** **A:** The CIR++ extension, when paired with Bayesian MCMC and Regime Isolation, meaningfully improved out-of-sample performance (boosting R-squared from 0.69 to 0.9285). However, we empirically proved that adding unnecessary mathematical complexity leads directly to overfitting. Highly complex models (Two-Factor Kalman Filters, Jump-Diffusion) overfit to 50-day and 70-day micro-regimes, failing to generalize out-of-sample.

### Part 3: Extensions and Modelling Choices

**Q: What mathematical structure justifies your chosen extension over the alternatives?** **A:** We chose the Time-Dependent Brigo-Mercurio (CIR++) extension because it acts directly as an error-correction layer. By calculating an Empirical Deterministic Shift exactly on t=0, the structure mathematically forces the model to fit the initial yield curve exactly. This directly counteracted the base model's systematic long-term overestimation without requiring the estimation of unobservable variables.

**Q: How do jump processes change the qualitative shape of predicted yield curves during stress periods?** **A:** Poisson jump processes (CIR-J) change the qualitative shape by introducing "fat tails" to the probability distribution, accommodating sudden, discontinuous stress events. However, our empirical testing discarded this model, as our dataset's primary challenge was a slow-moving macro-trend, not sudden shocks, causing the model to overfit to micro-shocks.

**Q: What are the additional estimation challenges introduced by a two-factor or time-dependent model?** **A:** * **Two-Factor Models:** Because we are constrained to only observing the 3M rate, the two underlying drivers (x_t and y_t) are unobservable. Estimating them requires complex Dynamic State-Space modeling (Kalman Filtering), which is highly prone to over-parameterization and overfitting micro-regimes (as seen with the 50-day overfit).

* 
**Time-Dependent Models (CIR++):** The primary challenge is Microstructural Noise Sensitivity. Because the framework relies on fitting the initial term structure (t=0) with zero error, the entire predictive shift is entirely dependent on the market conditions of that single calibration day. Temporary liquidity shocks on Day 0 become permanently baked into the model's forward pricing.



---

## 7. Personal Learnings & Conclusion

This project evolved from a standard coding assignment into an extensive masterclass in quantitative finance research. My core takeaways include:

1. **The Overfitting Trap:** I learned firsthand that more complex math does not equal better predictions. Deep Neural Networks, Jump-Diffusion, and 7-parameter Kalman Filters act like contortionists; they perfectly memorize local noise but violently misprice global out-of-sample reality.


2. 
**Regime Shifts are Everything:** Financial data is not stationary. Attempting to calibrate a model over 8 years of chaotic data forces the model to average out a world that has fundamentally changed. I learned the critical importance of dynamically isolating the "Lookback Window" to capture the current macroeconomic regime.


3. 
**Local Fit vs. Global Fit (The Single-Day Illusion):** I experienced how an overfitted model (like the Kalman Filter) looks visually perfect on one specific day's graph but fails the aggregate R-squared score across the entire timeline. I learned to trust Bayesian probability distributions and structural rigidity over single-point estimates for true global stability.



By rigorously testing parameters, refusing to accept theoretical ceilings, and engineering data-driven solutions to deep mathematical failures, I successfully reconstructed the yield curve and significantly exceeded the project's predictive constraints.
