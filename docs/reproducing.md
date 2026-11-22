# Reproducing the results

## 1. Environment

```bash
uv sync --frozen --all-extras     # exact versions from uv.lock
uv run pytest                     # unit tests, including temporal-leakage tests
```

On Windows without `uv` on the PATH, use `python -m uv` instead of `uv`.

## 2. Data

The data are not distributed with the code. Put them under `data/raw/`:

| Folder | Content | How to get it |
|---|---|---|
| `data/raw/aca_opendata/` | 5-min flows 1996–today, daily flows since 1951, daily reservoir status since 2000 | `llobregat aca-opendata stations ...` then `... data ...`; `scripts/download_aca_history.bat` runs the whole download (it can be resumed) |
| `data/raw/aca_rain/` | ACA rain gauge history, one Excel file per basin (`Precipitació_Llobregat.xlsx`) | Supplied by the ACA on request; not public |
| `data/raw/xema/` | XEMA half-hourly readings since 2009 | `llobregat xema probe / stations / data`; only needed for the `full` variant |

## 3. Pipeline

`scripts/run_all.sh` (or `scripts\run_all.bat`) runs every step in order. Outputs go to
`data/interim/` and `data/processed/sant_joan_despi/<variant>/`:

| Step | Command | Output |
|---|---|---|
| Merge the ACA history, fix units | `aca-merge`, `aca-rain` | `interim/aca_flow/`, `interim/aca_daily/`, `interim/aca_rain/` |
| Quality control | `qc-apply` (`qc-report`, `qc-event` to review) | `interim/aca_flow_qc/` |
| Inputs and folds per variant | `splits` | `interim/variants/` |
| 30-min data table | `build-dataset` | `dataset.parquet`, `meta.json` |
| Feature analysis | `analyse-features` | `feature_analysis/` |
| Point forecasts | `train`, `tune`, `combine`, `compare` | `models/`, `tuning/` |
| Intervals and warnings | `intervals` | `uncertainty/` |
| Hold-out, once | `final-evaluation` | `final/` (and the lock file `HOLDOUT_EVALUATED.json`) |
| Report | `report` | `report/` (tables, `report.md`, `figures/`) |
| Operational model | `operational-model` | `operational/` |

The README figures are copied from `report/figures/` to `docs/figures/`.

## 4. The hold-out

`final-evaluation` refuses to run a second time. The 2025–2026 hold-out has been used once,
for the results reported here; any later change to the system must be evaluated on new
hydrological years (2027 onwards), not on the hold-out again.

## 5. Outputs from older versions

Until November 2026 the code wrote Catalan folder, file and column names (`llarg/`,
`incertesa/`, `prediccions.parquet`, `plec`, `horitzo_h`...). To use those outputs with
the current code without re-running anything:

```bash
uv run python scripts/migrate_outputs_to_english.py          # dry run: lists the changes
uv run python scripts/migrate_outputs_to_english.py --apply
```

## 6. The operational model

`llobregat operational-model` retrains the frozen system on every complete hydrological year
except the last one, which is kept to calibrate the intervals, and writes:

- `manifest.json`: inputs, features per lead time, CQR adjustments, warning threshold,
  training years and library versions;
- `h<h>_lgbm.txt.gz`, `h<h>_q05.txt.gz` ... `h<h>_q95.txt.gz`: LightGBM models (gzipped text);
- `h<h>_ridge.joblib`: Ridge pipelines (scikit-learn; load them with the same version);
- `source_importance.csv`, `feature_importance.csv`.

`llobregat.operational.OperationalModel.load(path).predict(features)` turns a feature table
built with `llobregat.features` into forecasts, intervals and warnings. The live app
([llobregat-flood-forecast-live](https://github.com/joangordo/llobregat-flood-forecast-live)) uses exactly this.
