# Factor Runner

Reports from the configurable factor model runner. Each PDF in `outputs/` is one self-contained
study of a candidate factor model: how it was built, what it earned, and whether it survives
against the standard model (VAL, MOM, BAB, QMJ).

Every report opens with an abstract written to be read *instead of* the report — what was built,
what it earned, whether it clears the benchmark and the standard factors, and a verdict.

The runner lives in the private research repo; this repo holds only its rendered output.

## How to run

You never edit the runner to test an idea. `run.py`, `spec.py` and `reporting.py` stay untouched.
You add one config file and point `--config` at it:

```bash
.venv/Scripts/python.exe src/experiments/factor_model_runner/run.py --config src/experiments/factor_model_runner/capital_discipline.py --n 0 --seed 0
```

Each run prints a run id and a PDF path under
`data/outputs/factor_model_runner/runs/<timestamp>_<slug>_<hash>/`. Runs never overwrite each
other, so re-running is always safe.

`--n 0` is the full screened universe and takes about 40 seconds. `--n 300` samples 300 symbols
and takes about 25, which is worth it only while you are checking that a new config renders at all.

Useful flags:

| Flag | What it does |
| --- | --- |
| `--n` | Sample this many **symbols**, each keeping full history. `0` = the whole screened universe. |
| `--seed` | Seed for `--n` sampling. |
| `--stratify-by sector` | Keep the universe's sector mix in a small sample. |
| `--data-version <ts>` | Pin a gold snapshot so a multi-stage run reads one consistent view. |
| `--min-price` / `--min-dollar-volume` | Extra point-in-time row screens, e.g. `5` and `1e6`. |

`--n 0` turns off symbol *sampling* only. The universe screen, the price floor, the per-config
market-cap fraction and the financial exclusion all still apply.

List the usable signal columns:

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
    {"candidate": "Quality", "baseline": "QMJ", "label": "gpoa versus standard QMJ"},
]
```

- **`FIELDS`** — named sleeves mapped to their component signals.
- **`SETTINGS`** — construction, universe screens, benchmark and controls.
- **`FACTOR_SETTINGS`** — per-sleeve overrides. `directions` sets each signal's sign; `-1` means
  low values score high.
- **`COMPARISONS`** — candidate sleeve versus the standard factor it would replace. Section 7
  tests each one in, one out: the standard composite rebuilt with the baseline removed and the
  candidate in its place, scored against the unmodified composite on the same months.

Configs are validated strictly on load, so an unknown key, an unknown signal or a bad direction
fails immediately instead of quietly producing the wrong book.

Four settings that change results the most:

`beta_neutral` — off in every config here. Section 4 already regresses on the benchmark and
reports the beta-adjusted alpha, so neutralizing in construction would change the portfolio to
answer a question the report already answers, and would leave the book non-comparable to the
standard factors, which carry their own betas.

`top_market_cap_fraction` — a fraction, never a fixed top-k: the universe grows from 1,427 names
in 2007 to 4,773 in 2026, so a fixed count would drift from "large cap" to "mid cap" mid-sample.
`0.375` is the AQR VME convention used by the fundamentals books; the attention- and
liquidity-driven books widen it, since those effects are weakest in mega-caps.

`model_combine` — how the sleeves become one book. `score` (the default) pools every sleeve's
score into one model score and sorts the cross-section once, giving a single portfolio. `returns`
equal-weights the sleeve return series instead. These are different books, not two spellings of
the same one: a stock mid-rank on every sleeve gets no weight under `score` but carries a partial
position in each of the averaged books. Sleeves with different constructions are rejected on load
unless you set `returns`, since one sorted book cannot be a `paper_bab` book and a `paper_qmj`
book at once.

`min_components` — how many of a sleeve's signals a row needs before it is scored. Defaults to 60%
of them, not all. A signal that is sparse for structural reasons would otherwise delete most of
the cross-section from the factor rather than degrade it.
