# Raw_Data_Analysis

Data analysis code and experimental datasets accompanying:

> **An automated materials acceleration platform for active-learning-driven atomic layer deposition**  
> M.B. Alghalayini\*, T. Kodalle\*, A. Gashi, A. Razumtcev, M. Surendran, S. Aloni, A.M. Schwartzberg, E.S. Barnard  
> *Digital Discovery*, 2026 (DOI: TBD)

This repository contains the Gaussian process (GP) modeling pipeline used to process PE-ALD experimental campaign data. It loads raw ellipsometry-derived growth rate measurements from [ALDBot](https://github.com/MF-ALDBot/ALDBot) experiments, fits GP models, computes predictive accuracy (RMSE) and information gain metrics, and generates the synthesis-property relationship analysis (including SHAP feature importance) reported in the paper.

## Repository structure

```
Raw_Data_Analysis/
├── analyze_full_campaign.ipynb      # Main analysis notebook (start here)
├── models/
│   ├── gpmodel_base.py              # Abstract GP base class
│   └── gpmodel_constant_PriorMean.py  # Constant prior mean GP (used in paper)
├── campaigns/
│   ├── BO_4D/                       # 4D active learning campaign (paper Fig. 2)
│   │   ├── BO_4D_campaign_overview.yaml          # Full per-run metadata
│   │   ├── combined_runs_df.csv                  # All runs with parameters and growth rates
│   │   ├── combined_optimizer_df.csv             # GP optimizer recommendations per step
│   │   ├── BO_4D_campaign_growth_and_rmse_summary_fixedCauchy.csv  # Per-run growth rate summaries
│   │   ├── RMSE.json                             # RMSE vs. experiment count (100 trials)
│   │   └── exp_info_gain.csv                     # Cumulative information gain per step
│   ├── Random_4D/                   # 4D random sampling campaign (paper Fig. 2 baseline)
│   │   └── ...                      # Same structure as BO_4D
│   ├── BO_12D/                      # 12D active learning campaign (paper Fig. 3)
│   │   └── ...
│   └── Random_12D/                  # 12D random sampling campaign (paper Fig. 3 baseline)
│       └── ...
├── Sweeps/
│   ├── precursor_dose_time_sweep.json   # Validation sweep: p₄ (Fig. 3b)
│   ├── process_pressure_sweep.json      # Validation sweep: p₂ (Fig. 3c)
│   └── plasma_purge_time_sweep.json     # Validation sweep: p₁₂ (Fig. 3d)
├── RMSE_heldout_sets_4d.json        # Pooled held-out test sets for 4D RMSE comparison
├── RMSE_heldout_sets_12d.json       # Pooled held-out test sets for 12D RMSE comparison
└── pyproject.toml                   # Python dependencies
```

## Experimental campaigns

Four PE-ALD TiO₂ campaigns are included, corresponding to the two experimental designs (active learning and random sampling) applied to two parameter spaces:

| Folder | Dimensionality | Experiments | Role in paper |
|---|---|---|---|
| `BO_4D` | 4D subspace | 100 (5 random init + 95 GP) | Fig. 2 — active learning campaign |
| `Random_4D` | 4D subspace | 100 (5 shared init + 95 random) | Fig. 2 — random sampling baseline |
| `BO_12D` | 12D full space | 200 (15 random init + 185 GP) | Fig. 3 — active learning campaign |
| `Random_12D` | 12D full space | 200 (15 shared init + 185 random) | Fig. 3 — random sampling baseline |

The 4D and 12D active learning / random sampling pairs share the same initialization experiments to ensure a fair comparison.

## Installation

This project uses [uv](https://github.com/astral-sh/uv) for dependency management. Python 3.11 is required.

```bash
# Install uv if needed
pip install uv

# Install all dependencies
uv sync

# Alternatively, install with pip
pip install -e .
```

Core dependencies: `gpcam`, `scikit-learn`, `numpy`, `pandas`, `matplotlib`, `plotly`, `shap`, `umap-learn`, `pyyaml`.

## Usage

Open `analyze_full_campaign.ipynb` in Jupyter and run the cells sequentially. The notebook:

1. Loads the campaign overview YAML from the relevant `campaigns/` subfolder
2. Fits `GPModelConstMean` to the accumulated experimental data, sequentially over all run indices
3. Computes RMSE on 100 randomly drawn held-out test sets (50 experiments each), drawn from the pooled data of both the active learning and random sampling campaigns so that the same test sets are used for both
4. Computes per-step information gain (KL divergence between consecutive GP posteriors)
5. Generates GP posterior predictions, uncertainty maps, and SHAP feature importance analysis
6. Writes processed results to the campaign's subfolder under `campaigns/`

To switch between campaigns (e.g., 4D vs. 12D), set `campaign_num` at the top of the notebook.

## Data format

### Campaign overview YAML (`campaigns/<name>/<name>_campaign_overview.yaml`)

Each campaign folder contains a YAML file with the full campaign metadata and one entry per experiment. Key fields per run:

| Field | Description |
|---|---|
| `run_id` | Unique experiment identifier |
| `precursor_dose_time` | TDMAT dose duration (ms) |
| `plasma_prep_time` | O₂ plasma preparation time (ms) |
| `plasma_duration` | Plasma exposure time (ms) |
| `plasma_purge_time` | Post-plasma purge time (ms) |
| `H2_plasma_flow_rate` | Gas flow through plasma source (sccm) - In this study, O₂ was used instead of H₂ |
| `ALD_purge_time` | Post-precursor purge time (ms) |
| `process_pressure` | Chamber pressure during precursor half-cycle (mTorr) |
| `plasma_pressure` | Chamber pressure during plasma half-cycle (mTorr) |
| `RF_power_setpoint` | RF source power (W) |
| `Ar_process_flow_rate` | Ar flow through plasma source (sccm) |
| `ALD_purge_flow_rate` | Ar flow through precursor line (sccm) |
| `precursor_purge_time` | Precursor line purge duration (ms) |
| `dep_rate_mean` | Growth rate — mean over cycles 10–25 (Å/cycle) |
| `dep_rate_deviation` | Growth rate standard deviation (Å/cycle) |
| `dep_rate_fit` | Growth rate from linear thickness fit (Å/cycle) |
| `Initial_thickness` | Cumulative film thickness at experiment start (Å) |
| `Number_of_ALD_cycles` | ALD cycles per experiment (fixed at 25) |
| `campaign_id` | ID of the ALDBot run batch this experiment belongs to |

The first 9 of 25 cycles are discarded to remove inter-experiment effects; growth rate is averaged over the remaining 16 cycles.

### Validation sweeps (`Sweeps/`)

The three JSON files in `Sweeps/` contain the independent parameter sweep experiments used to validate the GP model predictions (Fig. 3b–d). Each file corresponds to a single-parameter sweep where all other parameters are held fixed:

| File | Parameter swept |
|---|---|
| `precursor_dose_time_sweep.json` | p₄ — precursor dose time |
| `process_pressure_sweep.json` | p₂ — process pressure |
| `plasma_purge_time_sweep.json` | p₁₂ — plasma purge time |

These sweep experiments were conducted independently and were not included in the GP model's training data.

## GP model

Two model classes are provided, both inheriting from `GPModelBase` (`models/gpmodel_base.py`):

- **`GPModelConstMean`** (`gpmodel_constant_PriorMean.py`) — Uses a constant prior mean $\mu(\mathbf{x}) = c_1$ with $c_1$ estimated from data. This is the model used throughout the paper.

Both models use an ARD Matérn-3/2 kernel with independent length scales per parameter dimension, fit by maximizing the log marginal likelihood via `gpCAM`.

## Citation

If you use this code or data, please cite:

```
@article{alghalayini2026aldbot,
  title   = {An automated materials acceleration platform for active-learning-driven atomic layer deposition},
  author  = {Alghalayini, Maher B. and Kodalle, Tim and Gashi, Arian and Razumtcev, Aleksandr and Surendran, Mythili and Aloni, Shaul and Schwartzberg, Adam M. and Barnard, Edward S.},
  journal = {Digital Discovery},
  year    = {2026},
  doi     = {TBD}
}
```
