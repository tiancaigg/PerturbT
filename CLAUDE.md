# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Research Goal

Develop a computational framework to **predict and engineer novel T cell states for immunotherapy** — moving from empirical screening to computationally guided design of T cells with specified functional properties (exhaustion resistance, sustained cytotoxicity, stemness).

The central challenge: current foundation models fine-tuned on single-gene perturbations default to approximately additive extrapolation (Effect_A + Effect_B) in combinatorial space — not because they are linear, but because additive extrapolation is the minimum-complexity solution under data sparsity. Explicit non-additive inductive bias makes interaction learning substantially more sample-efficient.

## Three-Stage Pipeline

| Stage | Task | Status |
|-------|------|--------|
| 1. Benchmark | Validate architecture on Norman 2019 (K562, 131 combinatorial perturbations). First stratified evaluation by genetic interaction type (additive / synergistic / suppressive). Compare vs baselines (GEARS, vanilla scGPT, CellOracle, scLAMBDA). | **Active** |
| 2. Predict | Train on T cell perturbation data (Zhou mouse CD8+, Schmidt/Belk human CD8+, Zhu human CD4+). Predict held-out Zhao o22R (stem-like) and oGCSFR (myeloid reprogramming) transcriptomes. Test cross-species transfer (mouse→human). | Planned |
| 3. Design | Gradient-based inverse optimization over TF activation + gene KO/OE space to identify combinatorial strategies producing desired T cell properties. | Planned |

## Dual-Track Architecture Approach

**Track A — scGPT + Multiplicative Attention (Method A)**
Modify scGPT's self-attention with: (1) a GRN-initialized bias matrix M from causally mapped T cell regulome (Zhou 2023), and (2) bilinear pairwise interaction terms capturing TF cooperativity/competition at gene-level resolution. Fine-grained non-additive modeling within a single-species 51M-parameter transformer.

**Track B — GeneCompass (frozen) + Custom Multiplicative Head (Method B)**
Use GeneCompass (~100M params, 120M+ human+mouse cells, 4 biological priors) as a frozen cross-species encoder. Lightweight custom head incorporates element-wise product terms (P_a ⊙ P_b) alongside individual perturbation embeddings. Frozen encoder preserves cross-species representations. Tests whether cross-species embeddings + coarser interaction terms suffice.

Comparison is a contribution: Track A tests gene-level interaction modeling; Track B tests embedding-level cross-species interaction. Both beating the additive baseline would confirm that the inductive bias — not the specific architecture — is the key ingredient.

## Key Datasets

| Dataset | Cell Type | Scale | Role |
|---------|-----------|-------|------|
| Norman 2019 (Science) | K562 | 91K cells, 131 combos | **Stage 1 benchmark** (annotated GI types) |
| Zhou 2023 (Nature) | Mouse CD8+ TILs in vivo | 42K cells, 180 TF KOs | Primary T cell training + GRN initialization for M matrix |
| McCutcheon 2023 (Nature Genetics) | Human CD8+ T | CRISPRi/a, 120 TFs | Human validation + bidirectional perturbation |
| Zhu 2025 (bioRxiv) | Human CD4+ T | 22M cells, ~10K genes | Large-scale T cell training |
| Zhao 2025 (Nature) | Engineered TILs | 29K scRNA, 13 receptors | **Held-out evaluation** (emergent states, e.g., o22R stem-like state) |

## Conda Environments

```bash
# scGPT training/evaluation (Stage 1 active work)
conda activate /conda_envs/scgpt

# CellOracle GRN inference (requires separate env for bedtools)
conda activate /conda_envs/celloracle_env
```

Jupyter notebooks are the primary interface. Launch from the relevant `notebooks/` subdirectory.

## Repository Structure

```
notebooks/
  scGPT/        # Stage 1 training and evaluation (Track A baseline)
  CellOracle/   # GRN inference → M matrix for attention bias
  GEARS/        # GEARS baseline (deprecated, see decrepted/)
data/
  splits/       # Norman 2019 h5ad with GEARS train/val/test splits
  ATAC_Seq/K562/  # K562 ATAC peaks for CellOracle motif scan
  genomes/hg38/ # Reference genome
model_checkpoints/
  scGPT_human/  # Pretrained scGPT checkpoint
```

## Key File Paths

| Path | Description |
|------|-------------|
| `data/splits/norman_2019_raw_sparse_with_splits_original_from_GEARS.h5ad` | Main Norman 2019 dataset; split labels in `obs['gears_split']` |
| `model_checkpoints/scGPT_human/` | Pretrained scGPT (`best_model.pt`, `vocab.json`, `args.json`) |
| `data/ATAC_Seq/K562/K562_ATAC_peaks.bed` | K562 bulk ATAC peaks (CellOracle input) |
| `notebooks/CellOracle/celloracle_grn_20260305_221104/K562_GRN_M_matrix.csv` | Current M matrix (1513 targets × 131 TFs) for GRN attention bias |

## Stage 1 Notebook Pipeline

Execution order for the active Stage 1 benchmark:

1. **`notebooks/CellOracle/celloracle_grn_norman2019_v4.ipynb`** — K562 GRN inference via CellOracle (bulk ATAC → motif scan → Lasso GRN → M matrix). Uses timestamped output dirs with checkpointing; set `CHOICE` variable to resume. Output: `K562_GRN_M_matrix.csv`.

2. **`notebooks/scGPT/S1_1b_stage1_scgpt_norman_leakfix.ipynb`** — Train vanilla scGPT baseline (S1). Saves `per_perturbation_results.csv` to `./save/dev_perturb_norman_leakfix-*/`.

3. **`notebooks/scGPT/S2_methodA_scgpt_grn_bias.ipynb`** — Train GRN-biased scGPT (Track A, S2). Reads M matrix from CellOracle output. Saves to `./save/dev_perturb_norman_grn_bias-*/`.

4. **`notebooks/scGPT/S1_2c_evaluate_scgpt_gi_stratified_v2.ipynb`** — Stratified evaluation by GI type and seen level. Auto-loads latest `save/` dir.

5. **`notebooks/scGPT/compare_models.ipynb`** — Side-by-side S1 vs S2 comparison.

## scGPT Architecture (Stage 1 / Track A)

**Model**: `scgpt.model.TransformerGenerator`. Only `encoder`, `value_encoder`, and `transformer_encoder` weights are transferred from pretraining. Architecture used for fine-tuning: `embsize=128, d_hid=512, nlayers=4, nhead=4` (smaller than pretrained 512/12/8).

**Task formulation**: Input = random control cell expression + binary perturbation flag vector → predict perturbed cell expression. Loss = `masked_mse_loss` over all genes (`include_zero_gene="all"`).

**Leakage fix (critical for Stage 1)**: HVGs and perturbed gene list computed on training split only; control pool restricted to training-split control cells only. Violated in the earlier `S1_1` notebooks (see `decrepted/`).

**GRN Attention Bias (Track A / S2)**:
- M matrix (1513×131 raw, aligned to 1251×1251 HVG space) injected as additive attention bias: `attn_scores += alpha * M_grn[gene_indices][:, gene_indices]`
- `alpha`: single learnable scalar shared across all layers and heads, initialized 0.1, converges ~0.1
- Implemented via `GRNAttentionBias` wrapper replacing `layer.self_attn` in each encoder layer
- Must call `set_gene_indices(input_gene_ids)` before each forward pass
- M is row-normalized before injection (`M_NORMALIZE=True`)
- `__getattr__` delegation required so `TransformerEncoderLayer` can still access `batch_first`, `num_heads`, etc. from the wrapped `nn.MultiheadAttention`

**Data format**:
- `obs['condition']`: perturbation label — `"SAMD1+ZBTB1"` (combo), `"ARID1A+ctrl"` (single), `"ctrl"` (control)
- `var_names`: Ensembl IDs in raw h5ad → must swap to gene symbols via `adata.var.set_index('gene_name')` before matching scGPT vocab

## Evaluation Conventions

**Primary metric**: Pearson correlation (PCC) on **top-20 DE genes** per perturbation (`pcc_top20`). All-gene PCC is dominated by unchanged genes and is inflated — do not use as the benchmark.

**Additive baseline**: For combo A+B, `delta_A + delta_B` from single-perturbation means. Both S1 and S2 currently fail to beat this baseline on combo perturbations (S2 narrows the gap: ΔPCC -0.378 → -0.296).

**GI type classification** (computed from expression, not Norman Table S3):
- `additive`: GI magnitude ≤ Q25; `synergistic`: positive mean GI on top-20; `suppressive`: negative mean GI
- GI magnitude = `‖delta_AB - delta_A - delta_B‖` on top-20 DE genes of the combo

**Seen level** (GEARS convention): `seen0` = neither component gene seen in training combos, `seen1` = one seen, `seen2` = both seen. S2 shows the largest gains on `seen2` (+0.233 PCC) at the cost of `seen0` and singles.

## Known Issues / Gotchas

- `np.float` removed in NumPy 2.0 — use `np.float64` explicitly
- `torchtext` is deprecated; suppress with `import torchtext; torchtext.disable_torchtext_deprecation_warning()`
- `flash_attn` not installed — always use `use_fast_transformer=False`
- CellOracle requires `celloracle_env` for bedtools; do not mix envs
- When reloading a GRN-biased model checkpoint, must rebuild `GRNAttentionBias` wrappers and `patched_attns` list before calling `set_gene_indices` — the wrapper is not serialized in `state_dict`
- `copy.deepcopy(model)` on the GRN-biased model can OOM; S2 notebook saves/reloads from disk instead
- GPU auto-selection in S2 notebook: `nvidia-smi` selects a free GPU and sets `CUDA_VISIBLE_DEVICES` **before** importing torch
