# Factor Runner

Rendered output from a configurable factor-model runner and its automatic-grouping head. Each PDF
in `outputs/` is self-contained: it documents either how candidate factors were discovered or how
their factor-mimicking portfolios performed and loaded on MKT, SMB, VAL, MOM, BAB and QMJ. Every
report opens with an abstract written to be read *instead of* the report.

The runner lives in a private research repo; this repo holds only its output.

## Latest catalogues

| Report | Scope | PDF |
|---|---|---|
| Original automatic factor grouping, top 90% of names | No pre-cluster Sharpe screen; exposure clustering followed by PCA/varimax into five frozen factors | [PDF](outputs/2026-09-06_automatic-factor-grouping-top-90.pdf) |
| Original grouped factor model, in sample | Original five-factor construction; 2012--2022, 131 monthly returns; combined Sharpe 0.910 | [PDF](outputs/2026-09-06_grouped-factor-model-top-90-is.pdf) |
| Original grouped factor model, out of sample | Frozen construction; 2023--2026-07, 43 monthly returns; combined Sharpe 0.239 | [PDF](outputs/2026-09-06_grouped-factor-model-top-90-oos.pdf) |
| Bootstrap Sharpe-screened grouping, top 90% of names | 2,000 stationary-block resamples; 100/140 signals retained, 62 primitive groups and five frozen factors | [PDF](outputs/2026-09-06_automatic-factor-grouping-top-90-bootstrap.pdf) |
| Bootstrap-screened grouped factor model, in sample | Bootstrap-screened construction; 2012--2022, 131 monthly returns; combined Sharpe 1.096 | [PDF](outputs/2026-09-06_grouped-factor-model-top-90-bootstrap-is.pdf) |
| Bootstrap-screened grouped factor model, out of sample | Frozen construction; 2023--2026-07, 43 monthly returns; combined Sharpe 0.577 | [PDF](outputs/2026-09-06_grouped-factor-model-top-90-bootstrap-oos.pdf) |
| No clustering, bootstrap Sharpe > 0, in sample | 100 admitted signals used as 100 single-signal factors; 2012--2022; combined Sharpe 1.366 | [PDF](outputs/2026-09-07_no-clustering-screen-sharpe-0-is.pdf) |
| No clustering, bootstrap Sharpe > 0, out of sample | Frozen 100-factor construction; 2023--2026-07; combined Sharpe 1.255 | [PDF](outputs/2026-09-07_no-clustering-screen-sharpe-0-oos.pdf) |
| No clustering, bootstrap Sharpe > 0.5, in sample | 21 admitted signals used as 21 single-signal factors; 2012--2022; combined Sharpe 2.330 | [PDF](outputs/2026-09-07_no-clustering-screen-sharpe-0-5-is.pdf) |
| No clustering, bootstrap Sharpe > 0.5, out of sample | Frozen 21-factor construction; 2023--2026-07; combined Sharpe 0.867 | [PDF](outputs/2026-09-07_no-clustering-screen-sharpe-0-5-oos.pdf) |
| No clustering, bootstrap Sharpe > 1, in sample | Five admitted signals used as five single-signal factors; 2012--2022; combined Sharpe 1.246 | [PDF](outputs/2026-09-07_no-clustering-screen-sharpe-1-is.pdf) |
| No clustering, bootstrap Sharpe > 1, out of sample | Frozen five-factor construction; 2023--2026-07; combined Sharpe 0.374 | [PDF](outputs/2026-09-07_no-clustering-screen-sharpe-1-oos.pdf) |
| Direct HDBSCAN, no PCA/varimax, in sample | Sharpe > 0 survivors clustered directly into seven signed factors; 2012--2022; combined Sharpe 1.380 | [PDF](outputs/2026-09-07_direct-hdbscan-no-pca-varimax-is.pdf) |
| Direct HDBSCAN, no PCA/varimax, out of sample | Frozen seven-factor construction; 2023--2026-07; combined Sharpe 1.191 | [PDF](outputs/2026-09-07_direct-hdbscan-no-pca-varimax-oos.pdf) |

## Running it

This public repository contains reports only. The commands below must be run from the root of the
private research repository, with its Python 3.13 environment, gold `daily_features` data and any
external control-return CSVs referenced by the selected config.

Start with a bounded smoke run:

```powershell
.venv\Scripts\python.exe -m experiments.factor_model_runner.run `
  --config src\experiments\factor_model_runner\example_model.py `
  --n 300 --seed 0
```

After the PDF and diagnostics pass inspection, run the complete screened universe:

```powershell
.venv\Scripts\python.exe -m experiments.factor_model_runner.run `
  --config src\experiments\factor_model_runner\example_model.py `
  --n 0 --seed 0
```

`--n` samples symbols, not rows; every selected symbol keeps its full history. `--n 0` means all
eligible symbols and is also the default. Runtime depends on the number of factors and their
history, so it is deliberately not estimated here.

### Reproduce the published catalogues

```powershell
# Standard VAL/MOM/BAB/QMJ reconstruction using each published construction
.venv\Scripts\python.exe -m experiments.factor_model_runner.run `
  --config src\experiments\factor_model_runner\standard_rebuild_faithful.py `
  --n 0 --seed 0

# Standalone GPOA quality and EBIT/EV value factors
.venv\Scripts\python.exe -m experiments.factor_model_runner.run `
  --config src\experiments\factor_model_runner\gpoa_ebit_ev.py `
  --n 0 --seed 0

# Every available registered signal; BAB variants and beta variants are grouped
.venv\Scripts\python.exe -m experiments.factor_model_runner.run `
  --config src\experiments\factor_model_runner\all_signals_catalogue.py `
  --n 0 --seed 0
```

## Writing a config

Copy `src/experiments/factor_model_runner/example_model.py`. `FIELDS` is required; `SETTINGS` and
`FACTOR_SETTINGS` are optional:

```python
FIELDS = {
    "Quality": ["gpoa"],
    "Value": ["earnings_yield"],
    "Quality + Value": ["gpoa", "earnings_yield"],
}

SETTINGS = {
    "name": "Quality and value catalogue",
    "signal_transform": "rank",
    "composite_transform": "rank",
    "winsor_limits": (0.01, 0.99),
    "construction": "signal_weighted",
    "long_short": True,
    "top_market_cap_fraction": 0.375,
    "exclude_financials": True,
    "cost_bps": 0.0,
    "factor_returns_csv": "path/to/monthly_controls.csv",
    "factor_return_columns": ("SMB", "VAL", "MOM", "BAB", "QMJ"),
}

FACTOR_SETTINGS = {
    "Quality": {"directions": {"gpoa": 1}},
    "Value": {"directions": {"earnings_yield": 1}},
    "Quality + Value": {"directions": {"gpoa": 1, "earnings_yield": 1}},
}
```

Each `FIELDS` entry creates one independent FMP; a multi-signal entry is a composite. Component
directions are construction signs: `+1` means high values score high and `-1` means low values
score high. After construction, the runner applies one factor-level sign so common-window raw
Sharpe is non-negative. That final direction is disclosed in the report and is distinct from the
component signs.

Configs retain the runner's signal/composite transforms, winsorization, component coverage floors,
portfolio construction, long-only/long-short status, beta neutrality, sector/industry-relative
signals, universe filters, date bounds, costs, benchmark and factor controls. Unknown settings,
missing fields and invalid directions stop the run rather than silently changing the model.

Useful CLI overrides:

| Option | Meaning |
|---|---|
| `--data-version <timestamp>` | Pin one gold snapshot instead of using latest-at-start |
| `--stratify-by sector` | Preserve sector composition in a sampled `--n` run |
| `--min-price 5` | Apply an additional point-in-time formation-price floor |
| `--min-dollar-volume 1e6` | Apply an additional trailing 21-day dollar-volume floor |
| `--no-default-liquidity-screen` | Disable the runner's default price/liquidity screen |
| `--no-universe-screen` | Include securities excluded by the shared common-stock screen |

Each run writes a new non-overwriting directory under
`data/outputs/factor_model_runner/runs/<run-id>/`. It contains the PDF, source and normalized
configs, manifest, canonical tables, monthly returns, loadings, PCA coordinates, rank diagnostics,
plots and layout warnings. Factors must independently pass the configured breadth, history,
coverage and volatility gates; a failure stops the run with a factor-specific reason instead of
silently dropping that factor.
