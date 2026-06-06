# Stochastic Interest Rate Modelling and Prediction with CIR

This project implements a full short-rate modelling workflow using the **Cox-Ingersoll-Ross (CIR)** model on historical yield curve data. The main goal is to reconstruct the yield curve from a single observable input: the **3-month yield**, used as a proxy for the instantaneous short rate.

 **Stochastic Interest Rate Modelling and Prediction**. The Code handles noisy yield data, calibrates the CIR model, predicts missing maturities, tests multiple extensions, and compares out-of-sample performance.

---

## Project Idea

Interest rates do not move randomly in a completely unstructured way. Short-rate models try to describe their movement using stochastic processes. In this project, the short rate is modelled using the CIR process:

```text
dr_t = kappa * (theta - r_t) dt + sigma * sqrt(r_t) dW_t
```

where:

- `kappa` controls how quickly the rate mean-reverts,
- `theta` is the long-run average rate,
- `sigma` controls volatility,
- `r_t` is approximated using the 3-month yield.

The CIR model is useful because the square-root volatility term helps keep interest rates positive, provided the Feller condition is satisfied:

```text
2 * kappa * theta >= sigma^2
```

Once the model parameters are calibrated, I use the CIR zero-coupon bond pricing formula to reconstruct yields for longer maturities.

---

## Workflow:

The Code follows a complete modelling pipeline:

1. **Uploads and reads the data**
    - `train_data.csv`
    - `test_data.csv`
    - `test_data_3M.csv`

2. **Cleans the yield data**
    - Standardizes column names.
    - Converts yield values into decimal form.
    - Handles missing values using interpolation and forward/backward filling.
    - Removes weekend observations.
    - Clips outliers using training-data-based IQR bounds.
    - Ensures all yield values remain mathematically valid for CIR calibration.

3. **Implements CIR model functions**
    - Computes CIR bond pricing terms `A(t, T)` and `B(t, T)`.
    - Converts bond prices into continuously compounded yields.
    - Predicts the yield curve using only the test-day 3M yield.

4. **Calibrates the model**
    - Uses exact-transition Maximum Likelihood Estimation on the 3M short-rate series.
    - Uses curve calibration to minimize pricing error across training maturities.
    - Checks the Feller condition for calibrated parameters.

5. **Builds model extensions**
    - CIR++-style residual correction.
    - Validation-selected hybrid model.
    - Targeted CIR extension using CIR for short maturities and polynomial mappings for longer maturities.

6. **Evaluates predictions**
    - Calculates `R2`, `RMSE`, and `MAE` for each available test maturity.
    - Plots predicted vs actual yield paths.
    - Plots actual-vs-predicted scatter diagrams.
    - Plots sample reconstructed yield curves.

---

## Dataset Format

The notebook expects yield data across the following maturities:

| Raw Column | Tenor |
|---|---|
| `ZC025YR` | 3M |
| `ZC050YR` | 6M |
| `ZC075YR` | 9M |
| `ZC100YR` | 1Y |
| `ZC200YR` | 2Y |
| `ZC500YR` | 5Y |
| `ZC1000YR` | 10Y |
| `ZC2000YR` | 20Y |
| `ZC3000YR` | 30Y |

prediction rule: during testing, the model is allowed to use only the **3M yield** for that day. All other maturities must be reconstructed from the calibrated model or its approved extensions.

---

The notebook is designed to be run in Google Colab because it uses `google.colab.files.upload()` for data upload.

---

## Requirements

The code uses the following Python libraries:

```text
numpy
pandas
matplotlib
scipy
scikit-learn
google-colab
```

---

## Model Details

### Base CIR Model

The base model uses the 3M yield as the short rate and reconstructs the yield curve using the CIR closed-form bond pricing equation.

Two calibration approaches are included:

- **Short-rate MLE:** fits the dynamics of the 3M rate only.
- **Curve-calibrated CIR:** starts from MLE parameters and minimizes yield-curve error across all training maturities.

The curve-calibrated model is the stronger base model because it is trained directly on the yield curve reconstruction task.

### CIR++-Style Residual Extension

The CIR++-style extension learns a deterministic correction surface:

```text
actual_yield(tau) = CIR_yield(tau, r_3M) + correction(r_3M, tau)
```

The correction is estimated only from training data using polynomial features and Ridge regression.

### Hybrid and Targeted Extensions

The notebook also tests maturity-wise correction methods because different tenors behave differently. Short maturities are often captured well by CIR, while longer maturities can require extra flexibility due to slope and curvature effects in the yield curve.

The final targeted extension uses:

- CIR for short maturities: `6M`, `9M`, `1Y`
- Polynomial 3M-to-tenor mappings for longer maturities

---

## Results

In the uploaded notebook run, the available test actual maturities were:

```text
6M, 9M, 1Y, 2Y
```

The notebook still predicts all requested maturities from `6M` to `30Y`, but out-of-sample metrics can only be calculated for maturities present in `test_data.csv`.

### Model Comparison

| Model | Overall R2 | Overall RMSE | Overall MAE |
|---|---:|---:|---:|
| Base / Curve-Calibrated CIR | 0.8930 | 0.002196 | 0.001420 |
| CIR++ Residual Extension | 0.7254 | 0.003519 | 0.003143 |
| Validation-Selected Hybrid | 0.7628 | 0.003270 | 0.002707 |
| Final Targeted Extension | 0.8546 | 0.002560 | 0.001941 |

The strongest overall score in the notebook output is achieved by the **curve-calibrated CIR model**, with an out-of-sample `R2` of approximately **0.893**, which clears the required `0.85` benchmark.

The targeted extension also crosses the benchmark with an out-of-sample `R2` of approximately **0.855**, although it does not outperform the curve-calibrated CIR model overall.

---

## Key Observations

- The 3M yield carries meaningful information about nearby maturities.
- The CIR model performs very strongly on short maturities such as 6M, 9M, and 1Y.
- Performance weakens at longer maturities because long-term yields depend on more than the current short rate.
- A one-factor model cannot fully capture level, slope, and curvature changes in the yield curve.
- Extensions can improve some maturities but may also overfit if they are too flexible.
- The Feller condition can fail under high-volatility or low-rate regimes, so the notebook checks and penalizes violations during curve calibration.

---

## Limitations

This project is intentionally constrained by the rule that only the 3M yield can be used during prediction. Because of that, the model cannot fully observe the market’s slope or curvature on a given test day.

The main limitations are:

- The base CIR model is single-factor.
- It cannot perfectly fit arbitrary yield curve shapes.
- It struggles when the curve changes due to regime shifts or market shocks.
- Polynomial corrections may improve numerical scores but are less structurally interpretable than a true multi-factor stochastic model.
- In the uploaded run, only short-end test actuals were available for metric calculation, so long-maturity predictions could not be fully evaluated out of sample.

---

## Possible Improvements

Future improvements could include:

- A true two-factor CIR or Longstaff-Schwartz-style model.
- A jump-diffusion short-rate model for sudden rate shocks.
- CIR++ with a deterministic time-dependent shift to fit the initial curve more exactly.
- Kalman filtering for latent factor estimation.
- A stronger validation framework using rolling-window backtesting.
- Separate evaluation once full long-maturity test actuals are available.

---

## Conclusion

This project shows that a carefully calibrated CIR model can reconstruct a large part of the yield curve using only the 3M yield. The curve-calibrated CIR model performs especially well on the available out-of-sample test maturities and satisfies the required accuracy threshold. At the same time, the analysis highlights why one-factor short-rate models are limited: they are elegant, interpretable, and mathematically tractable, but they cannot fully represent all real-world yield curve movements.
