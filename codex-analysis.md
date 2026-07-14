# FEM/GA analysis for RHD_70

## Executive conclusion

This run does **not yet support a unique identification of the eight myocardial material parameters**. It does support a pressure/volume calibration result: the recorded candidate EDPs span 2.015–4.000 kPa (15.1–30.0 mmHg), and the clinical EDP of 2.178 kPa is 16.34 mmHg. Thus the clinical pressure is elevated in the usual physiological sense, provided the pressure in `config.json` and `volumes.csv` is in kPa. The EDPVR files appear to express their fitted pressures in mmHg, and this unit conversion must be made explicit in the workflow.

Several candidates reproduce EDV well, but the objective components do not converge together. Iteration 162 gives EDV = 184.380 mL (target 184.460 mL; error -0.080 mL), while iteration 117 gives the lowest recorded physics RMSE (0.237) and EDV = 184.759 mL (error +0.299 mL). Neither is the lowest strain-loss candidate. The lowest aggregate recorded cost occurs at iteration 79, but its EDV is 201.749 mL, 17.289 mL above target. Simulated ESV is exactly 107.785 mL for every one of the 176 records, versus 86.4 mL clinically, so the systolic constraint is not responding to the optimized parameters.

The reference-volume estimate is also not settled. `V0_simulation` is nearly constant at approximately 107.56 mL, 15.70 mL (17.1%) above the configured Klotz value of 91.861 mL. The zero-pressure intercept inferred from individual EDPVR files is much less stable (99.80–184.46 mL; mean 110.72 mL). These are evidence of a calibration/implementation issue or weak identifiability, not a defensible patient-specific V0 estimate.

## Data and interpretation

- `volumes.csv` contains 176 evaluated candidates (`iter` 0–175), not full populations for 176 GA generations. There is exactly one row per iteration. The `gen` field is 1 for rows 0–99 and 2 for rows 100–175. Consequently, population diversity, elite retention, and within-generation convergence cannot be reconstructed from this file.
- Each `Params` vector contains 129 values: 16 repeated blocks of the eight parameters `(a, b, af, bf, as, bs, afs, bfs)`, followed by EDP. This interpretation follows the 16 repetitions of the eight bounds in `config.json` and the final pressure-like value.
- Clinical targets are EDV = 184.46 mL, ESV = 86.4 mL, EDP = 2.17799 kPa, and Klotz V0 = 91.86108 mL.
- EDPVR files exist for 171 candidates. Files are missing for iterations 7, 22, 26, 93, and 151.
- `Strain_cost` is numerically equal (to rounding) to `loss_l + loss_c + loss_r`. The analysis therefore treats it as the total strain loss rather than double-counting its components.
- No explicit total-objective column is supplied. For comparison only, “aggregate recorded cost” below is `Vol_cost + Strain_cost + V0_cost + ESV_cost`. This assumption should be checked against the optimizer source.

## Analysis approach

1. Parsed all 176 parameter vectors and reshaped their first 128 values into 16 segments by 8 material parameters.
2. Checked completeness, ranges, constants, non-finite values, and agreement with clinical targets.
3. Compared the first, last, lowest-component-cost, and lowest-assumed-total-cost candidates. A GA should show improving best-so-far objective and contracting parameter spread; raw iteration-to-iteration oscillation alone is not convergence.
4. Summarized parameter distributions over all candidates, the final 20 candidates, and the ten lowest assumed-total-cost candidates, and compared them with the supplied theoretical global constants.
5. Read every available iteration EDPVR curve, extracted the zero-pressure optimized intercept and terminal pressure/volume, and calculated pointwise RMSE between `EDP_Simulation` and `EDP_Simulation_fitted`.
6. Audited whether volume, V0, ESV, strain, and physics objectives select the same candidate. They do not.

## Pressure and volume results

| Quantity | Result | Interpretation |
|---|---:|---|
| Clinical EDP | 2.178 kPa = 16.34 mmHg | Elevated filling pressure if kPa is the stored unit |
| Candidate EDP range | 2.015–4.000 kPa = 15.11–30.00 mmHg | Broad, with no contraction in the final records |
| Final candidate EDP | 2.963 kPa = 22.23 mmHg | Elevated, but not an identified optimum |
| EDV range | 172.516–214.165 mL | Wide response across candidates |
| Best EDV match | 184.380 mL at iteration 162 | Absolute error 0.080 mL (0.043%) |
| Lowest physics RMSE | 0.237 at iteration 117 | EDV 184.759 mL, absolute error 0.299 mL |
| Simulated ESV | 107.785 mL for every candidate | 21.385 mL (24.75%) above clinical ESV; objective is insensitive/frozen |
| `V0_simulation` range | 107.495–107.621 mL | Artificially narrow and inconsistent with Klotz V0 |
| Configured Klotz V0 | 91.861 mL | 15.70 mL below the approximately 107.56 mL simulation value |

The first candidate has EDV 201.426 mL and the final candidate 195.909 mL. This is an improvement relative to the target, but it is not monotonic and is inferior to candidates already evaluated at iterations 117 and 162. Over the final 20 records, EDV is 189.50 ± 6.65 mL and EDP is 2.929 ± 0.530 kPa. The retained search region has therefore not contracted around one pressure/volume solution.

The clinical EDV and ESV imply EF = `(184.46 - 86.4)/184.46` = 53.16%. Using the invariant simulated ESV and target EDV gives only 41.57%. This systolic mismatch should be fixed before passive parameters are interpreted biologically.

## Objective behavior and competing optima

| Selection rule | Iteration | EDP (kPa) | EDV (mL) | Physics RMSE | Strain cost | Volume cost |
|---|---:|---:|---:|---:|---:|---:|
| Lowest assumed aggregate cost | 79 | 2.996 | 201.749 | 2.362 | 14.806 | 0.0937 |
| Lowest physics RMSE | 117 | 2.630 | 184.759 | 0.237 | 15.294 | 0.00162 |
| Lowest volume cost | 162 | 3.514 | 184.380 | 5.304 | 15.221 | 0.000433 |
| Final recorded candidate | 175 | 2.963 | 195.909 | 1.877 | 15.180 | 0.0621 |

The assumed aggregate is dominated by strain cost: strain lies between 14.806 and 15.522, whereas volume cost is 0.00043–0.161, V0 cost is approximately 0.170–0.172, and ESV cost is the constant 0.2475. As a result, iteration 79 wins the aggregate despite a 9.37% EDV error. This scaling makes the optimization prefer small strain-loss changes over clinically large volume errors. The constant ESV and nearly constant V0 terms provide essentially no selection pressure.

`Phys_rmse` reaches its minimum at iteration 117 but later rises again; the final value is 1.877. `Diff` contains at least one infinite value. The optimization and any summary statistics should penalize/non-finitely reject such candidates explicitly.

## Material-parameter identifiability

The supplied theoretical global values are:

| Parameter | Theory | Mean, all candidates | Mean, final 20 | Final-20/theory |
|---|---:|---:|---:|---:|
| a | 0.0509 | 1.512 | 1.475 | 29.0× |
| b | 6.4860 | 15.286 | 15.998 | 2.47× |
| af | 4.2654 | 15.386 | 14.721 | 3.45× |
| bf | 8.7982 | 13.100 | 12.862 | 1.46× |
| as | 0.7449 | 5.208 | 5.146 | 6.91× |
| bs | 3.0987 | 5.255 | 5.044 | 1.63× |
| afs | 0.0885 | 1.556 | 1.520 | 17.2× |
| bfs | 6.7800 | 10.159 | 10.924 | 1.61× |

These means must **not** be reported as fitted constants. They are close to the midpoints of broad uniform bounds, and their spreads remain extremely large. For the final 20 candidates, pooled standard deviations across segments and candidates are 0.862 (`a`), 8.268 (`b`), 8.684 (`af`), 7.047 (`bf`), 2.903 (`as`), 2.898 (`bs`), 0.829 (`afs`), and 5.766 (`bfs`). Average within-candidate segment spreads are almost as large. The ten lowest assumed-cost candidates show the same broad dispersion rather than a tight posterior-like cluster.

Approximately 1.98% and 2.11% of all material coordinates lie within 2% of their lower and upper normalized bounds respectively. Boundary contact is not the dominant issue; the problem is that almost the whole prior range remains acceptable. With 128 material coordinates plus EDP but only a small collection of aggregate volume/strain targets, many compensating parameter combinations are expected. The data therefore identify response curves more readily than individual constitutive constants.

The theoretical comparison also raises a bounds issue: theoretical `afs = 0.0885` lies below the optimization lower bound of 0.1. The theoretical value is impossible for the GA to recover. This should be corrected or justified.

## EDPVR and V0 calibration

Across the 171 available EDPVR files:

- The optimized zero-pressure intercept (`EDV_Optimised` at zero pressure) ranges from 99.804 to 184.460 mL (mean 110.724, SD 15.251 mL).
- Terminal optimized/Klotz pressure ranges from 15.111 to 30.001 mmHg (mean 22.321 mmHg), consistent numerically with converting candidate EDP from kPa to mmHg.
- The simulation terminal volume ranges from 172.516 to 214.165 mL.
- Pointwise RMSE between `EDP_Simulation` and `EDP_Simulation_fitted` ranges from 3.305 to 17.781 in the stored pressure units (mean 8.067). Thus the fitted curve is not generally a close representation of the raw simulated points.

There are at least three different V0-like quantities: `config.json: V0_klotz = 91.861`, `volumes.csv: V0_simulation ≈ 107.56`, and the zero-pressure `EDV_Optimised` intercept in each EDPVR file. Their definitions and reference configurations must be reconciled. V0 should be estimated from the intercept of a clearly specified EDPVR model, with units and fitting domain fixed, not simultaneously treated as a nearly constant loss term and a highly variable curve intercept.

The physical reference file further deserves review: `physcurve/edpvr_curves_phys.csv` reaches `EDP_phys = 651279.8` at EDV 184.46 mL, whereas its optimized/Klotz endpoint is about 17 mmHg. This enormous scale discrepancy suggests a unit, exponent, or constitutive-evaluation problem in the raw physics curve.

## Recommended next iteration

1. **Fix units and labels.** Store pressure units in every column name or metadata. Verify that candidate EDP is kPa and EDPVR fitted/Klotz pressure is mmHg; apply 1 kPa = 7.50062 mmHg exactly once.
2. **Repair the systolic calculation.** Determine why `ESV_Sim` is invariant. Until ESV responds to parameters/loading, EF and the ESV objective cannot constrain the fit.
3. **Unify V0.** Define whether V0 is unloaded cavity volume, EDPVR zero-pressure intercept, or a Klotz-derived reference. Use one definition in geometry initialization, loss evaluation, and reporting.
4. **Correct EDPVR evaluation.** Investigate the 651,280 endpoint in the physical curve and the large raw-versus-fitted RMSE. Reject non-finite curves before GA selection.
5. **Rescale the objective.** Normalize residuals by clinical uncertainty or tolerances (for example, EDV/ESV in mL, strain in percentage points, EDP in mmHg) and report every weighted contribution plus total objective. The current strain term overwhelms the volume terms.
6. **Reduce dimensionality before re-running.** First fit eight global parameters plus EDP and V0. Introduce regional parameters only where segmental strain data demonstrate sensitivity. Regularize regional values toward global values and enforce physiologically motivated relationships.
7. **Make theory reachable.** Lower the `afs` bound below 0.0885 if the theoretical set is intended as a recoverable reference. Consider log-space optimization for positive exponential-law constants.
8. **Save complete GA history.** Record every individual, generation, total fitness, constraint status, parent/elite status, and random seed. Plot generation-wise best, median, and interquartile fitness and parameter diversity.
9. **Quantify identifiability.** After obtaining a stable global fit, use profile likelihood, bootstrap/noise perturbations, or Bayesian posterior sampling. Report intervals and parameter correlations, not only a point estimate.
10. **Validate out of sample.** Calibrate passive behavior to diastolic pressure-volume and ED strains, then test against held-out pressure levels or strain segments. Do not infer remodeling solely from elevated fitted stiffness when loading and unloaded geometry remain uncertain.

## Bottom line

The data are compatible with elevated RHD filling pressure, and individual runs can match the clinical EDV closely. However, the GA output presently shows structural non-identifiability, incompatible V0 definitions, a frozen ESV response, poorly balanced loss terms, and no demonstrated convergence. The eight theoretical parameters should remain priors/reference values; the broad GA parameter values should not yet be interpreted as patient-specific myocardial properties.
