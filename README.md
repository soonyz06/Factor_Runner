# Factor Runner

Reports from the configurable factor model runner. Each PDF is one self-contained study of a
candidate factor model: how it was built, what it earned, and whether it survives against the
standard factor controls (VAL, MOM, BAB, QMJ).

The runner lives in the private research repo; this repo holds only its rendered output.

## How to run

You never edit the runner to test an idea. `run.py`, `spec.py` and `reporting.py` stay untouched.
You add one config file and point `--config` at it:

```bash
.venv/Scripts/python.exe src/experiments/factor_model_runner/run.py --config src/experiments/factor_model_runner/capital_discipline.py --n 300 --seed 0
```

Each run prints a run id and a PDF path under
`data/outputs/factor_model_runner/runs/<timestamp>_<slug>_<hash>/`. Runs never overwrite each
other, so re-running is always safe. Start at `--n 150` to confirm a config renders before paying
for a bigger run.

Useful flags:

| Flag | What it does |
| --- | --- |
| `--n` | Sample this many **symbols**, each keeping full history. `--n 0` = whole universe, slow. |
| `--seed` | Seed for `--n` sampling. |
| `--stratify-by sector` | Keep the universe's sector mix in a small sample. |
| `--data-version <ts>` | Pin a gold snapshot so a multi-stage run reads one consistent view. |
| `--min-price` / `--min-dollar-volume` | Extra point-in-time row screens, e.g. `5` and `1e6`. |

List the 72 usable signal columns:

```bash
.venv/Scripts/python.exe src/experiments/factor_model_runner/list_signals.py
```

## Writing a config

Pure declaration, no logic. Four top-level names, of which only `FIELDS` is required:

```python
FIELDS = {
    "Quality": ["gpoa"],
    "Value": ["earnings_yield"],
}

SETTINGS = {
    "name": "gpoa plus EBIT-EV",
    "signal_transform": "rank",
    "construction": "signal_weighted",
    "long_short": True,
    "beta_neutral": False,
    "top_market_cap_fraction": 0.375,
    "exclude_financials": True,
    "standard_factor_columns": ("VAL", "MOM", "BAB", "QMJ"),
}

FACTOR_SETTINGS = {
    "Quality": {"directions": {"gpoa": 1}},
    "Value": {"directions": {"earnings_yield": 1}},
}

COMPARISONS = [
    {"candidate": "Quality", "baseline": "QMJ", "label": "gpoa versus existing QMJ"},
]
```

- **`FIELDS`** — named sleeves mapped to their component signals.
- **`SETTINGS`** — construction, universe screens, benchmark and controls.
- **`FACTOR_SETTINGS`** — per-sleeve overrides. `directions` sets each signal's sign; `-1` means
  low values score high.
- **`COMPARISONS`** — candidate sleeve versus the incumbent factor it would replace.

Configs are validated strictly on load, so an unknown key, an unknown signal or a bad direction
fails immediately instead of quietly producing the wrong book.

Two settings that change results the most:

`beta_neutral` — turn it on for a book that is structurally long-safe and short-risky. Otherwise
its alpha against SPY is largely a short market position wearing a factor label.

`top_market_cap_fraction` — a fraction, never a fixed top-k: the universe grows from 1,427 names
in 2007 to 4,773 in 2026, so a fixed count would drift from "large cap" to "mid cap" mid-sample.
`0.375` is the AQR VME convention.
