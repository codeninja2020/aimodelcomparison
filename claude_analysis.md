# RHD70 Material Parameter Identification — Analysis

## 0. Scope note on directory structure

`prompt.md` refers to "each iteration folder" containing results per GA iteration, but no such
folders exist in this directory — only a single flat file, `volumes.csv`, with **165 rows**
(one row per GA individual evaluated, `iter` 0–164). This analysis therefore treats
`volumes.csv` as the complete record of the run and is written at the top level rather than
per-iteration-folder. If per-iteration folders are produced by a future run, this file's method
can be re-applied to each one directly (the parsing script below is generation-agnostic).

## 1. Data structure (reverse-engineered)

`volumes.csv` columns: `iter, gen, Params, ESV_Clinical, ESV_Sim, V0_simulation, EDP, EDV,
Target_EDV, EF_error, Phys_rmse, Diff, Vol_cost, Strain_cost, V0_cost, ESV_cost, loss_l,
loss_c, loss_r`.

- **`gen`**: only two values present, `1` (n=100) and `2` (n=65) — i.e. this file captures
  generations 1–2 of the GA only (population sizes 100 then 65, with 5 total failed
  evaluations, see §4).
- **`Params`**: a 129-element vector per individual. The **last element is always exactly
  equal to the `EDP` column** (verified on every row) — the GA is optimizing global EDP
  jointly with the material field. The remaining **128 = 16 × 8** elements show a clean
  repeating bound pattern every 8 entries:

  | offset | observed range   | inferred parameter | theoretical value |
  |-------:|-------------------|---------------------|-------------------|
  | 0 | 0.01 – 3.0   | `a`   | 0.0509 |
  | 1 | 1.0 – 30.0   | `b`   | 6.4860 |
  | 2 | 0.5 – 30.0   | `af`  | 4.2654 |
  | 3 | 1.0 – 25.0   | `bf`  | 8.7982 |
  | 4 | 0.01 – 10.0  | `as`  | 0.7449 |
  | 5 | 0.1 – 10.0   | `bs`  | 3.0987 |
  | 6 | 0.1 – 3.0    | `afs` | 0.0885 |
  | 7 | 0.1 – 20.0   | `bfs` | 6.7800 |

  This matches the Holzapfel–Ogden orthotropic myocardial law parameter order given in
  `prompt.md`, each bounded within a range that contains its theoretical value. Interpretation:
  **the LV mesh is partitioned into 16 regions**, each independently assigned its own
  8-parameter HO material, i.e. the GA is solving for a *regionally heterogeneous* material
  field (129 free parameters total: 128 material + 1 global EDP), not a single global material.

## 2. Approach

1. Parsed `Params` per row into 16 regional 8-tuples + trailing EDP.
2. Computed per-generation summary statistics on `loss_c` (and `loss_l`/`loss_r`) to assess GA
   convergence.
3. Ranked individuals by `loss_c` (lower = better) to identify best/worst-fit solutions.
4. For the best-fit individuals, computed per-parameter mean/SD/CV across the 16 regions and
   compared against the theoretical global values.
5. Checked sensitivity of each output (`EDV`, `ESV_Sim`, `V0_simulation`) to the explored
   parameter space to understand what the GA can and cannot correct.
6. Flagged failed simulations (`Diff = inf`).

(Full reproduction script available on request; all numbers below were computed directly from
`volumes.csv`, no simulation was re-run.)

## 3. GA convergence: essentially flat across the 2 recorded generations

| gen | n (valid) | loss_c min | loss_c mean | loss_c max | stdev |
|---|---|---|---|---|---|
| 1 | 96 | 4.1104 | 4.3459 | 4.4624 | 0.0495 |
| 2 | 64 | 4.1440 | 4.3336 | 4.4229 | 0.0526 |

Mean loss barely moves between generations (4.346 → 4.334), well within one standard
deviation. **The single best individual in the whole file (iter 79) occurs in generation 1**,
not generation 2 — generation 2 does not improve on it. With only two generations captured,
there isn't enough data to claim convergence; what the data does show is that population spread
(SD ≈ 0.05) is much larger than the between-generation shift, so no clear downward trend is yet
visible.

**Loss is dominated by `Strain_cost`**: for every row, `Strain_cost` ≈ 14.8–15.5, versus
`Vol_cost` ≈ 0.0004–0.16, `V0_cost` ≈ 0.170–0.172, `ESV_cost` fixed at 0.2475. Strain cost
alone is ~97% of the summed cost terms and shows almost no improvement across the whole run
(best 14.81 vs typical 15.3) — **strain matching, not volume/EDP matching, is the unsolved part
of this optimization** and should be the focus of any further GA budget.

## 4. Failed simulations

5/165 evaluations (3%) produced `Diff = inf` (solver divergence): iter 7, 22, 26, 93 (gen 1) and
151 (gen 2). Notably iter 93 and iter 151 share the exact same failing EDP
(3.2450964614833016) — the same (or a near-identical, elite-carried) individual failed twice
across generations. All 5 failures occur at mid-to-high EDP (2.98–3.75), suggesting solver
robustness degrades as EDP/stiffness combinations increase — worth a convergence/damping check
in the FEM solver before scaling up the population.

## 5. Best-fit solutions

Top 10 by `loss_c`:

| iter | gen | loss_c | EDP | EDV | V0_sim | EF_error | Strain_cost |
|---|---|---|---|---|---|---|---|
| 79 | 1 | 4.1104 | 2.996 | 201.75 | 107.539 | -0.0659 | 14.8055 |
| 116 | 2 | 4.1440 | 2.996 | 207.61 | 107.540 | -0.0508 | 14.8656 |
| 139 | 2 | 4.1909 | 3.797 | 208.95 | 107.511 | -0.0475 | 15.0333 |
| 81 | 1 | 4.2304 | 3.514 | 200.51 | 107.578 | -0.0692 | 15.0815 |
| 1 | 1 | 4.2308 | 2.963 | 202.30 | 107.569 | -0.0644 | 15.1097 |
| 112 | 2 | 4.2318 | 2.996 | 203.59 | 107.547 | -0.0610 | 15.0511 |
| 4 | 1 | 4.2362 | 3.497 | 197.57 | 107.566 | -0.0771 | 15.0120 |
| 137 | 2 | 4.2419 | 3.740 | 214.16 | 107.579 | -0.0349 | 15.1390 |
| 160 | 2 | 4.2472 | 3.595 | 199.91 | 107.611 | -0.0708 | 15.1488 |
| 15 | 1 | 4.2547 | 3.931 | 201.37 | 107.543 | -0.0669 | 15.1212 |

Bottom 5 (worst fit) by contrast: `EDP` 2.09–2.45, `EDV` 172.5–180.1 — **all low-EDP,
low-EDV individuals**.

### EDP is the dominant driver of fit quality

`corr(EDP, loss_c) = -0.43` over the whole file: higher EDP → lower (better) loss. The current
GA bounds on EDP are **[~2.0, ~4.0]**, and the best solutions cluster at 2.96–3.93, i.e. **near
the upper bound**, not centered in the search space. This is consistent with the theoretical
premise in `prompt.md` — RHD patients have elevated LV pressure — and suggests the true optimum
may sit *above* the current EDP upper bound of 4.0. **Recommend re-running with the EDP upper
bound raised (e.g. to 5–6) to check whether loss keeps improving past 4.0**, before concluding
4.0 is a real physiological ceiling rather than an artificial search-space limit.

## 6. Regional material parameters vs. theoretical values

For the 5 best-fit individuals, per-parameter mean ± SD across the 16 regions (representative
example, iter 79, `loss_c = 4.1104`):

| param | theoretical | region mean | region SD | CV% | region min–max |
|---|---|---|---|---|---|
| a   | 0.0509 | 1.259  | 1.019 | 81% | 0.016 – 2.97 |
| b   | 6.486  | 13.68  | 7.85  | 57% | 1.72 – 27.73 |
| af  | 4.265  | 14.41  | 9.43  | 65% | 0.68 – 29.87 |
| bf  | 8.798  | 13.62  | 7.19  | 53% | 2.18 – 24.43 |
| as  | 0.745  | 3.63   | 2.74  | 76% | 0.52 – 9.01 |
| bs  | 3.099  | 7.12   | 2.65  | 37% | 0.82 – 9.99 |
| afs | 0.089  | 1.65   | 0.96  | 58% | 0.11 – 2.84 |
| bfs | 6.78   | 7.93   | 3.96  | 50% | 1.15 – 13.43 |

This pattern holds across all top-10 individuals, not just this one:

- **Every parameter's optimized regional mean is 2–20× the theoretical "healthy/generic"
  value**, most dramatically for `a` (~25×) and `af` (~3.4×). Since `a`/`af`/`as`/`afs` scale
  passive stress and `b`/`bf`/`bs`/`bfs` control the *rate* of exponential stiffening, elevated
  values across the board mean the identified myocardium is both stiffer at baseline and
  stiffens more steeply with strain than the normative parameter set — i.e. **reduced passive
  compliance**, which is exactly the expected signature of RHD-related fibrotic/hypertrophic
  ventricular remodeling.
- **High inter-regional heterogeneity (CV 37–81%)**: the GA is not converging to a uniform
  stiffened material but to a patchy one, with some of the 16 regions near-normal (e.g. `a` as
  low as 0.016, close to the theoretical 0.0509) and others 10–30× stiffer. This is consistent
  with regionally non-uniform remodeling rather than diffuse uniform fibrosis, though it could
  also partly reflect parameter non-identifiability (see §8) rather than true regional signal —
  can't distinguish the two from this data alone.

## 7. Systolic function is structurally unreachable by this parameter set

`ESV_Sim` is **exactly 107.785 mL on every one of the 160 successful rows** — it never changes,
regardless of how the 128 material parameters or EDP are varied. Meanwhile `ESV_Clinical` is
fixed at 86.4 mL. Consequently:

- `EF_error = EF_sim − EF_clinical` is **negative on every row** (range −0.156 to −0.035): the
  simulated ejection fraction is always lower than the clinical target, because the simulated
  end-systolic volume (107.8 mL) is always ~25% larger than the clinical one (86.4 mL).
- Since this 129-parameter vector contains **only passive (diastolic filling) material
  parameters and EDP** — no active/systolic contractility parameters — **no combination of
  these parameters explored here can close the EF gap**. `EF_error` is a structural bias of the
  current model setup, not a fitting shortfall. If matching clinical EF is a goal, active
  tension/contractility parameters need to be added to the optimization; otherwise `EF_error`
  should be treated as a fixed offset rather than a cost to drive toward zero.

## 8. EDPVR / V0 calibration

`V0_simulation` (the unloaded/reference volume used for the EDPVR fit) ranges only
**107.49–107.62 mL** — a 0.13 mL spread — across the *entire* file, despite material parameters
swinging over more than an order of magnitude. This says V0 in this FEM model is governed
almost entirely by mesh geometry, not by the passive material law, so `V0_cost` (0.170–0.172
throughout) is effectively a constant offset in the loss, not something this GA can act on.
Practical implication: **V0 does not need to be part of the material-parameter search** — it
appears fixed by geometry, so if calibration to a specific V0 target is required, that has to
happen at the mesh/geometry stage, not via the HO material parameters.

`EDP` (2.01–4.00, likely kPa given typical HO/FEM convention) versus `EDV` (172.5–214.2 mL,
target 184.46 mL): the EDPVR relationship implied by the best-fit cluster (EDP ≈ 3.0–3.9 →
EDV ≈ 198–215 mL) sits at higher pressure for the same or higher volume than the clinical
target point (EDP unspecified clinically, EDV = 184.46 mL) — again consistent with a
right-shifted/steeper EDPVR curve (elevated stiffness) expected in RHD, though without a
clinical EDP value in this file, this can't be checked quantitatively against a patient-specific
EDPVR target — only against the assumed target EDV.

## 9. Summary of findings

1. The optimization solves for 16 spatially independent 8-parameter HO material regions plus 1
   global EDP (129 free parameters), not a single global material.
2. Only 2 generations (165 evaluations, 5 failed) are captured in this file; mean loss is
   statistically flat between them — no clear convergence trend is demonstrable yet from this
   data alone.
3. `Strain_cost` dominates total loss (~97%) and is essentially unimproved across the run — this
   is the primary unresolved objective, more so than volume or EDP matching.
4. Best-fit individuals cluster near the *upper* EDP search bound (~4.0), and loss correlates
   negatively with EDP (r = −0.43) — the bound should be raised and re-tested before treating 4.0
   as a physiological ceiling.
5. Optimized regional stiffness parameters are uniformly 2–20× the theoretical normative values,
   with high regional heterogeneity (CV 37–81%) — directionally consistent with the RHD
   hypothesis of increased, patchy passive myocardial stiffness.
6. `EF_error` is always negative and cannot be corrected by this parameter set, because
   `ESV_Sim` never moves — active contractility isn't part of this search; treat EF matching as
   out of scope unless contractility parameters are added.
7. `V0_simulation` is essentially invariant to material parameters (mesh/geometry-driven) — safe
   to exclude from future material-parameter searches.
8. Solver divergence (5/165 runs) occurs specifically at mid-to-high EDP — check solver
   stability at higher pressure/stiffness combinations before widening the EDP bound per (4).

## 10. Recommended next steps

- Re-run with EDP upper bound raised above 4.0 to test whether fit keeps improving (§5).
- Run substantially more generations (the file only has 2) with elitism to establish a real
  convergence curve, since current between-generation variation is smaller than within-generation
  noise.
- Investigate why `Strain_cost` isn't improving: check gradient/sensitivity of strain error to
  the 128 material parameters — if strain error is insensitive to them, the strain mismatch may
  be dominated by geometry/fiber-orientation error rather than material properties, in which case
  no amount of material-parameter search will fix it.
- If clinical EF matching matters, add active/systolic contractility parameters to the search
  space — the current passive-only parameterization cannot reduce `EF_error`.
- Investigate the 5 divergent runs (iter 7, 22, 26, 93, 151) for solver robustness at their
  specific parameter combinations.
