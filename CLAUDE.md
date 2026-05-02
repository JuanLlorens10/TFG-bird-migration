# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Environment

All work runs inside the project's virtual environment at `tfg_env/`. Always use its binaries:

```bash
# Launch JupyterLab
tfg_env/bin/jupyter lab

# Execute a notebook non-interactively (re-runs all cells in place)
tfg_env/bin/jupyter nbconvert --to notebook --execute --inplace notebooks/<name>.ipynb

# Run a Python script or one-off snippet
tfg_env/bin/python script.py
```

Key packages: pandas, numpy, scikit-learn, xgboost, lightgbm, hmmlearn, matplotlib, seaborn, plotly, scipy (Python 3.12).

## Data layout

```
data/
  raw/migration_original.csv      # Source GPS tracks (126 gulls, 2009-2015)
  processed/
    aves_procesado_markov.csv     # One location per bird per day (~22k rows), used by Markov notebook
    hmm.csv                       # Adds movement features + HMM state labels, used by ML notebooks
```

`hmm.csv` extends `aves_procesado_markov.csv` with: `step_length`, `bearing`, `turning_angle`, `estado_hmm` (0=migration, 1=stationary), `grid_x/grid_y/cell_id` (0.5°×0.5° spatial discretization), `target_cell/target_num` (next-day prediction target).

## Notebook pipeline

The notebooks must be run in order — each one's output feeds the next:

1. `dataExploration1.ipynb` → cleans raw GPS, produces `aves_procesado_markov.csv`
2. `HMM.ipynb` → fits Gaussian HMM, adds movement features and state labels, produces `hmm.csv`
3. `markov1.ipynb` → builds 12 monthly transition matrices from `aves_procesado_markov.csv`
4. `ML1.ipynb` / `ML2.ipynb` / `ML3.ipynb` → train classifiers on `hmm.csv` to predict next grid cell

ML1–3 differ only in the temporal feature set used (month+lunar → month → week-of-year). ML3 is the final version (best balance of accuracy and simplicity).

## Train/test split convention

Data is split **per animal**: first 80% of each bird's chronologically sorted records go to train, last 20% to test. This prevents data leakage across time. The LabelEncoder is fit only on training cells; test rows with unseen cells are dropped before evaluation.

## Git workflow

Commit and push to GitHub regularly so no work is ever lost. After each meaningful unit of work (new analysis, new cell, data fix, model result), stage and push:

```bash
git add <specific files>
git commit -m "short description of what changed and why"
git push
```

Commit messages should be concise and specific (e.g. `"add per-state accuracy breakdown to ML3"`, not `"update notebook"`). Never use `git add .` — always add files by name to avoid accidentally committing large data files or the `tfg_env/` directory. The `tfg_env/` virtualenv and `data/` files should be in `.gitignore` and never committed.

## Spatial grid

Coordinates are discretized into 0.5°×0.5° cells. `cell_id` is a string `"grid_x_grid_y"`. There are ~950 unique cells in the training set. The Markov model uses 1,253 unique cells across the full dataset.
