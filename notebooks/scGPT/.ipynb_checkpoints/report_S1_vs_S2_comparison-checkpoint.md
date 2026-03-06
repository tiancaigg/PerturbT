# scGPT Model Comparison: S1 vs S2

**Date**: 2026-03-06
**S1 notebook**: `S1_2c_evaluate_scgpt_gi_stratified_v2.ipynb`
**S2 notebook**: `S2_2c_evaluate_scgpt_gi_stratified_v2-Copy1.ipynb`

---

## Models

| | S1 | S2 |
|--|----|----|
| Model name | `scGPT_vanilla_leakfix` | `scGPT_grn_bias` |
| Save dir | `dev_perturb_norman_leakfix-Mar05-20-28` | `dev_perturb_norman_grn_bias-Mar06-12-39` |
| Training date | Mar 05 | Mar 06 |
| n perturbations evaluated | 114 | 114 |

---

## Stratified Results: Combo Perturbations vs Additive Baseline

### PCC (top-20 DEGs, higher is better)

| GI Type | n | S1 PCC | S2 PCC | Delta |
|---------|---|--------|--------|-------|
| additive | 19 | 0.531 ± 0.357 | 0.586 ± 0.413 | +0.055 |
| synergistic | 28 | 0.695 ± 0.162 | 0.719 ± 0.276 | +0.024 |
| suppressive | 30 | 0.520 ± 0.284 | 0.675 ± 0.284 | **+0.155** |
| **ALL COMBOS** | 77 | 0.587 ± 0.278 | **0.669 ± 0.317** | **+0.082** |
| SINGLES | 37 | **0.446 ± 0.405** | 0.266 ± 0.309 | **-0.180** |

### MSE (top-20 DEGs, lower is better)

| GI Type | n | S1 MSE | S2 MSE | Delta |
|---------|---|--------|--------|-------|
| additive | 19 | 0.311 ± 0.216 | 0.277 ± 0.256 | -0.034 |
| synergistic | 28 | 0.817 ± 0.599 | 0.603 ± 0.565 | **-0.214** |
| suppressive | 30 | 0.890 ± 0.637 | 0.546 ± 0.383 | **-0.344** |
| **ALL COMBOS** | 77 | 0.720 ± 0.592 | **0.500 ± 0.450** | **-0.220** |
| SINGLES | 37 | **0.378 ± 0.413** | 0.445 ± 0.471 | +0.067 |

---

## Gap to Additive Baseline: Delta(scGPT - Additive)

Negative ΔPCC = scGPT worse than additive; positive ΔMSE = scGPT worse than additive.

| GI Type | S1 ΔPCC | S2 ΔPCC | S1 ΔMSE | S2 ΔMSE |
|---------|---------|---------|---------|---------|
| additive | -0.445 ± 0.361 | -0.390 ± 0.416 | +0.283 ± 0.213 | +0.249 ± 0.256 |
| synergistic | -0.267 ± 0.162 | -0.244 ± 0.265 | +0.681 ± 0.581 | +0.467 ± 0.559 |
| suppressive | -0.439 ± 0.291 | -0.284 ± 0.275 | +0.677 ± 0.516 | +0.333 ± 0.469 |
| **ALL** | -0.378 ± 0.282 | **-0.296 ± 0.313** | +0.581 ± 0.512 | **+0.361 ± 0.467** |

Both models still lose to the simple additive baseline (delta_a + delta_b) on all combo types. S2 narrows the gap.

**Statistical tests (Mann-Whitney, ΔPCC additive vs other GI types):**

| Comparison | S1 p-value | S2 p-value |
|-----------|------------|------------|
| additive vs synergistic | p = 0.0703 | p = 0.1620 |
| additive vs suppressive | p = 0.6591 | p = 0.4059 |

No significant difference in performance across GI types in either model.

---

## Seen-Level Stratification

| Seen Level | n | S1 PCC | S2 PCC | Delta |
|------------|---|--------|--------|-------|
| seen0 | 10 | 0.461 ± 0.313 | 0.354 ± 0.305 | -0.107 |
| seen1 | 52 | 0.595 ± 0.293 | 0.669 ± 0.314 | +0.074 |
| **seen2** | 15 | 0.643 ± 0.166 | **0.876 ± 0.106** | **+0.233** |
| single | 37 | **0.446 ± 0.405** | 0.266 ± 0.309 | -0.180 |

S2 improves strongly for seen2 pairs (both component genes seen during training). S2 is worse for seen0 (unseen gene pairs) and singles.

---

## GI Magnitude → Prediction Error

Does greater non-additivity predict worse scGPT performance?

| Metric | S1 Spearman rho | S1 p | S2 Spearman rho | S2 p |
|--------|----------------|------|----------------|------|
| GI mag vs MSE | +0.510 | 2.20e-06 | +0.453 | 3.46e-05 |
| GI mag vs PCC | +0.010 | 9.31e-01 | +0.099 | 3.94e-01 |
| GI mag vs ΔMSE | **+0.342** | **2.33e-03** | +0.041 | 7.22e-01 |

Key finding: In S1, higher GI magnitude predicts a larger gap between scGPT and the additive baseline (rho=+0.342, p=0.002). In S2, this relationship disappears (p=0.72), indicating the GRN bias reduces the systematic tendency to fail more on strongly non-additive interactions.

---

## Summary

### Is S2 (scGPT_grn_bias) improved over S1 (scGPT_vanilla_leakfix)?

**Yes, for combo perturbations overall (+0.082 PCC, -0.220 MSE).**

**Improvements in S2:**
- All combo GI types show better PCC and MSE
- Suppressive GI type shows largest PCC gain (+0.155)
- seen2 pairs show dramatic improvement (+0.233 PCC, reaching 0.876)
- The systematic correlation between GI magnitude and prediction gap is eliminated

**Regressions in S2:**
- Singles performance dropped substantially (-0.180 PCC, 0.446 -> 0.266)
- seen0 (fully unseen gene pairs) also regressed (-0.107 PCC)

### Interpretation

The GRN bias appears to leverage prior gene regulatory network information that is most useful when both component genes of a combo have been seen during training (seen2). This comes at the cost of generalization to single-gene and fully unseen (seen0) perturbations. Both models still fail to beat the simple additive baseline, but S2 closes the gap somewhat (ΔPCC: -0.378 -> -0.296).
