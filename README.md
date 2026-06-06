# Stochastic Interest Rate Modelling: CIR and CIR++ Implementation

[cite_start]**Objective:** To implement, calibrate, and extend the Cox-Ingersoll-Ross (CIR) short-rate model, and evaluate its predictive power in reconstructing the yield curve from a single observable 3-Month rate[cite: 1452, 2771].
[cite_start]**Institution:** Finance Club, IIT Roorkee [cite: 1437, 2772]
[cite_start]**Project Type:** Single-Person Project (2 Weeks) [cite: 1439, 1441]
[cite_start]**Final Out-of-Sample Predictive Accuracy (R-Squared):** 0.9285 [cite: 2772]

---

## 1. The Intricacies of the CIR Model

### What is the CIR Model and Where is it Used?
[cite_start]Introduced by Cox, Ingersoll, and Ross in 1985, the CIR model describes the evolution of the instantaneous short rate[cite: 1456]. [cite_start]In the financial industry, quantitative analysts and institutional risk managers rely on models like CIR to price interest rate derivatives, value bonds, and manage portfolio risks[cite: 1448, 2775]. 

### The Mathematical Framework
The model is defined by the stochastic differential equation (SDE):
[cite_start]dr(t) = kappa(theta - r(t))dt + sigma * sqrt(r(t))dW(t) [cite: 1457]

It relies on two foundational properties:
1. [cite_start]**Mean Reversion:** Governed by the term kappa(theta - r(t)), the model assumes that if rates deviate too high or too low, they are pulled back toward a long-run mean (theta) at a specific speed of mean reversion (kappa)[cite: 2776].
2. [cite_start]**Strictly Positive Rates:** The diffusion term uses a square-root process[cite: 1459, 2777]. [cite_start]This ensures that interest rates cannot become negative, provided the Feller condition (2 * kappa * theta >= sigma^2) is satisfied[cite: 1459, 2778]. 

[cite_start]From this SDE, the model derives a closed-form zero-coupon bond pricing formula: P(t,T) = A(t,T)e^(-B(t,T)r(t)), where A and B are deterministic functions of the model parameters[cite: 1462, 1463].

### Where and Why the Base CIR Model Fails
[cite_start]While mathematically tractable, the base CIR model has well-documented limitations[cite: 1467, 2780]. [cite_start]It fails heavily when data is non-stationary[cite: 2781]. [cite_start]If an economy experiences a multi-year downward trend in interest rates, the assumption of a static, long-term mean (theta) breaks down[cite: 2781]. [cite_start]Furthermore, the base model cannot perfectly fit an arbitrary initial term structure, struggles to capture sudden market shocks, and forces the yield curve into restrictive shapes because it relies entirely on a single factor[cite: 1467, 2782].

---

## 2. Data Engineering & Preprocessing
[cite_start]The foundation of stochastic calibration relies on mathematically viable data[cite: 2783]. [cite_start]The raw dataset contained 1,976 daily observations across 9 specific bond maturities ranging from 3 Months to 30 Years[cite: 1473, 2784].

* [cite_start]**Formatting Consistency:** The raw dataset contained hidden whitespace in column names (e.g., " ZC025YR"), which broke indexing and required algorithmic stripping[cite: 2785].
* [cite_start]**Missing Value Analysis:** Financial data often hides missing days as 0.0 or negative numbers[cite: 2786]. [cite_start]Diagnostic checks confirmed the absolute minimum yield was 0.000486 (0.0486%), meaning the data was mathematically viable for square-root diffusion without requiring complex interpolation for lazy zero entries[cite: 2787].
* [cite_start]**The Time-Step (dt) Definition:** The CIR model requires a precise change in time (dt)[cite: 2788]. [cite_start]Setting dt = 1/365 would dilute volatility by assuming markets trade on weekends[cite: 2789]. [cite_start]We rigidly defined dt = 1/252 to reflect active trading days, ensuring all extracted parameters were correctly annualized[cite: 2790].

---

## 3. The Base Calibration Pipeline: Failures and Lessons

[cite_start]We rigorously tested multiple calibration methodologies, each exposing a theoretical flaw in the base model[cite: 2791].

### Unconstrained Ordinary Least Squares (OLS)
* [cite_start]**What we did:** Discretized the SDE into a linear regression formula to isolate kappa, theta, and sigma[cite: 2792].
* [cite_start]**Why it failed:** The dataset contained a massive downward trend[cite: 2793]. [cite_start]Unconstrained OLS extrapolated this trend into infinity, resulting in a negative mean-reversion speed (kappa = -0.2548) and a negative long-term target (theta = -0.0053)[cite: 2794]. [cite_start]This caused "mean aversion," actively pulling rates below zero and violating the fundamental premise of the CIR model[cite: 2795].

### Constrained Least Squares Optimization
* [cite_start]**What we did:** Used optimization with strict boundaries (kappa > 0, theta > 0) to force the algorithm to respect the Feller condition[cite: 2796].
* [cite_start]**Why it performed poorly:** To avoid breaking constraints during a downtrend, the optimizer found a mathematical loophole[cite: 2797]. [cite_start]It set the target (theta) to an absurd 343% (3.438) while slowing kappa to a microscopic crawl (0.000689)[cite: 2798]. [cite_start]This acted as a mathematical magnet, violently pulling all long-term yield predictions upward[cite: 2799]. [cite_start]The model predicted a 30-Year yield of 7.07% when reality was 3.6%, leading to an out-of-sample R-squared of 0.6921[cite: 2800].

### Maximum Likelihood Estimation (MLE)
* [cite_start]**What we did:** Attempted to optimize the Negative Log-Likelihood based on the exact probability distribution of the CIR process[cite: 2801].
* [cite_start]**Why it failed:** We encountered the "Exploding Exponential" trap[cite: 2802]. [cite_start]The optimizer tested massive parameters causing e^(h * tau) to exceed 64-bit limits, resulting in NaN and Overflow crashes[cite: 2803]. Applying hard bounds (Regularized MLE) stabilized the math, yielding an R-squared of 0.8448[cite: 2804]. However, MLE became obsessed with fitting microscopic daily variance, cranking kappa to a violent 2.269[cite: 2805]. This perfected daily volatility but destroyed the macro shape of the curve[cite: 2806].

---

## 4. The Model Evolution: Rejecting Complex Alternatives

To correct the systematic failures of the base model, we researched advanced extensions[cite: 2807]. We implemented them, backtested them, and empirically rejected them due to the Overfitting Trap[cite: 2808].

* **Rejected: Jump-Diffusion (CIR-J)**
  * [cite_start]*Purpose:* Incorporates Poisson jump processes to handle sudden macroeconomic shocks[cite: 1497, 2809].
  * [cite_start]*Why it failed:* Our dataset's primary challenge was a slow-moving macro-trend, not sudden overnight crashes[cite: 2810]. [cite_start]Exhaustive tuning proved that CIR-J overfit to a tiny 70-day window of micro-shocks, resulting in a degraded R-squared of 0.8722[cite: 2811].
* **Rejected: Two-Factor Models (Longstaff-Schwartz) & Kalman Filters**
  * [cite_start]*Purpose:* Introduces a second stochastic factor to capture yield curve variations such as level versus slope[cite: 1495, 2812].
  * [cite_start]*Why it failed:* Because the project constrained us to observing only the 3-Month rate, the two underlying drivers (x_t, y_t) were unobservable[cite: 2813]. [cite_start]Even when utilizing Dynamic State-Space Kalman Filtering to track these hidden components, the 7-parameter structure forced the algorithm to overfit, memorizing a 50-day micro-regime (R-squared = 0.9031)[cite: 2814].

---

## 5. The Champion Framework: Bayesian MCMC + CIR++

[cite_start]We achieved peak out-of-sample predictive power by combining a specific mathematical structure with an elite calibration mechanic[cite: 2817].

1. **The Structure: Time-Dependent Parameters (CIR++)**
   [cite_start]We transitioned to the Brigo-Mercurio CIR++ model[cite: 2818]. [cite_start]By allowing parameters to act as deterministic functions of time, we calculated an Empirical Deterministic Shift exactly on t=0[cite: 1498, 2819]. [cite_start]This structure mathematically forces the model to fit the initial yield curve perfectly, acting as an error-correction layer that completely neutralizes the base model's systematic long-term overestimation[cite: 2820].
2. **The Mechanic: Bayesian Markov Chain Monte Carlo (MCMC) + Regime Isolation**
   [cite_start]Calibrating over 8 years of chaotic macroeconomic data caused parameter collapse[cite: 2821]. [cite_start]We wrapped our calibration in an optimization loop to find the optimal "Lookback Window," utilizing Bayesian MCMC[cite: 2822]. [cite_start]MCMC integrates over parameter uncertainty via thousands of probability simulations rather than making a single strict guess[cite: 2823]. 
3. [cite_start]**The Result:** MCMC naturally smoothed out the noise that tricked the complex models, finding a highly stable optimal macro-regime of 280 Days (approx. 1.1 trading years)[cite: 2824]. [cite_start]This setup flawlessly reconstructed the curve, achieving a peak Out-of-Sample R-squared of 0.9285[cite: 2825].

---

## 6. Answers to Key Project Questions

### Part 1: Model Mechanics and Calibration
[cite_start]**Q: How sensitive is the calibrated yield curve to the choice of calibration methodology?** [cite: 1504, 2826]
[cite_start]**A:** The yield curve is hyper-sensitive to the methodology[cite: 2827]. [cite_start]Unconstrained MLE violently overfitted daily variance, resulting in extreme mean-reversion speeds (kappa > 2.0) that completely failed to predict long-term maturities[cite: 2828]. [cite_start]Ordinary Least Squares (OLS) and Generalised Method of Moments (GMM) proved vastly superior by prioritizing macro-level drift and variance moments over micro-daily noise[cite: 2829].

[cite_start]**Q: Under what market conditions does the Feller condition break down in practice, and how do you handle it?** [cite: 1505, 2830]
[cite_start]**A:** The Feller condition breaks down during prolonged, multi-year downward-trending markets[cite: 2831]. [cite_start]An unconstrained algorithm will extrapolate this trend into negative territory, yielding kappa < 0 and theta < 0[cite: 2832]. [cite_start]We handled this programmatically by applying Constrained Optimization boundaries (via scipy.optimize), forcing the algorithm to preserve the strictly positive square-root diffusion process[cite: 2833].

[cite_start]**Q: What does the mean-reversion speed (kappa) imply about the persistence of interest rate shocks in your data?** [cite: 1506, 2834]
[cite_start]**A:** Our optimal calibration yielded a relatively slow speed of mean reversion (kappa approx 0.10 to 0.12)[cite: 2835]. [cite_start]This implies that interest rate shocks are highly persistent[cite: 2836]. [cite_start]When central banks alter policy rates, the effects do not snap back immediately; they reverberate through the yield curve for roughly 8 to 10 years before fully reverting to the long-term mean[cite: 2837].

### Part 2: Prediction and Out-of-Sample Performance
[cite_start]**Q: How accurately can the 3M rate alone reconstruct the full yield curve, and which maturities are hardest to fit?** [cite: 1508, 2838]
[cite_start]**A:** The 3M rate acts as a strong anchor for the "level" of the short end of the curve, but accuracy decays as maturity increases[cite: 2839]. [cite_start]The hardest maturities to fit are the ultra-long ends (20Y and 30Y)[cite: 2840]. [cite_start]These tenors contain massive term premiums driven by long-term macroeconomic inflation expectations that a single instantaneous short-rate proxy simply cannot encapsulate[cite: 2841].

[cite_start]**Q: Where does the base CIR model systematically over- or underestimate yields, and why?** [cite: 1509, 2842]
[cite_start]**A:** The base CIR model systematically overestimates long-term yields[cite: 2843]. [cite_start]Because it is strictly mean-reverting, calibrating it on a downward trend forced the optimizer to set an artificially massive theta target (343%) to avoid breaking constraints[cite: 2844]. [cite_start]This absurd target acted as a mathematical magnet, violently pulling all long-term yield predictions upward[cite: 2845].

[cite_start]**Q: Does your extension meaningfully improve out-of-sample performance, or does it overfit the training period?** [cite: 1510, 2846]
[cite_start]**A:** The CIR++ extension, when paired with Bayesian MCMC and Regime Isolation, meaningfully improved out-of-sample performance (boosting R-squared from 0.69 to 0.9285)[cite: 2847]. [cite_start]However, we empirically proved that adding unnecessary mathematical complexity leads directly to overfitting[cite: 2848]. [cite_start]Highly complex models (Two-Factor Kalman Filters, Jump-Diffusion) overfit to 50-day and 70-day micro-regimes, failing to generalize out-of-sample[cite: 2849].

### Part 3: Extensions and Modelling Choices
[cite_start]**Q: What mathematical structure justifies your chosen extension over the alternatives?** [cite: 1512, 2850]
[cite_start]**A:** We chose the Time-Dependent Brigo-Mercurio (CIR++) extension because it acts directly as an error-correction layer[cite: 2851]. [cite_start]By calculating an Empirical Deterministic Shift exactly on t=0, the structure mathematically forces the model to fit the initial yield curve exactly[cite: 2852]. [cite_start]This directly counteracted the base model's systematic long-term overestimation without requiring the estimation of unobservable variables[cite: 2853].

[cite_start]**Q: How do jump processes change the qualitative shape of predicted yield curves during stress periods?** [cite: 1513, 2854]
[cite_start]**A:** Poisson jump processes (CIR-J) change the qualitative shape by introducing "fat tails" to the probability distribution, accommodating sudden, discontinuous stress events[cite: 2855]. [cite_start]However, our empirical testing discarded this model, as our dataset's primary challenge was a slow-moving macro-trend, not sudden shocks, causing the model to overfit to micro-shocks[cite: 2856].

[cite_start]**Q: What are the additional estimation challenges introduced by a two-factor or time-dependent model?** [cite: 1514, 2857]
[cite_start]**A:** * **Two-Factor Models:** Because we are constrained to only observing the 3M rate, the two underlying drivers (x_t and y_t) are unobservable[cite: 2858]. Estimating them requires complex Dynamic State-Space modeling (Kalman Filtering), which is highly prone to over-parameterization and overfitting micro-regimes (as seen with the 50-day overfit)[cite: 2859].
* [cite_start]**Time-Dependent Models (CIR++):** The primary challenge is Microstructural Noise Sensitivity[cite: 2860]. [cite_start]Because the framework relies on fitting the initial term structure (t=0) with zero error, the entire predictive shift is entirely dependent on the market conditions of that single calibration day[cite: 2861]. [cite_start]Temporary liquidity shocks on Day 0 become permanently baked into the model's forward pricing[cite: 2862].

---

## 7. Personal Learnings & Conclusion

[cite_start]This project evolved from a standard coding assignment into an extensive masterclass in quantitative finance research[cite: 2863]. My core takeaways include:

1. [cite_start]**The Overfitting Trap:** I learned firsthand that more complex math does not equal better predictions[cite: 2864]. [cite_start]Jump-Diffusion and 7-parameter Kalman Filters act like contortionists; they perfectly memorize local noise but violently misprice global out-of-sample reality[cite: 2865, 2866]. 
2. [cite_start]**Regime Shifts are Everything:** Financial data is not stationary[cite: 2867]. [cite_start]Attempting to calibrate a model over 8 years of chaotic data forces the model to average out a world that has fundamentally changed[cite: 2868]. [cite_start]I learned the critical importance of dynamically isolating the "Lookback Window" to capture the current macroeconomic regime[cite: 2869].
3. [cite_start]**Local Fit vs. Global Fit (The Single-Day Illusion):** I experienced how an overfitted model (like the Kalman Filter) looks visually perfect on one specific day's graph but fails the aggregate R-squared score across the entire timeline[cite: 2870]. [cite_start]I learned to trust Bayesian probability distributions and structural rigidity over single-point estimates for true global stability[cite: 2871].

[cite_start]By rigorously testing parameters, refusing to accept theoretical ceilings, and engineering data-driven solutions to deep mathematical failures, I successfully reconstructed the yield curve and significantly exceeded the project's predictive constraints[cite: 2872].
