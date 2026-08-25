# Factor

Reports from the configurable **factor model runner**. Each PDF is one self-contained study of a
candidate factor model: how it was built, what it earned, and whether it survives against the
factors that already exist (VAL, MOM, BAB, QMJ).

The runner itself lives in the private research repo; this repo holds only its rendered output.

## Reports

| Report | Model | Sleeves |
| --- | --- | --- |
| `2026-08-25_gpoa-ebit-ev.pdf` | Magic-Formula-style quality + value | `gpoa`, `earnings_yield` (EBIT/EV) |
| `2026-08-25_low-accruals-sloan.pdf` | Sloan (1996) low-accruals anomaly | `accruals` (direction −1) |
| `2026-08-25_lottery-arbitrage-risk.pdf` | Lottery aversion + limits to arbitrage | `lottery_max_5_21d`, `skew_63d`, `ivol_21d` (all −1) |

The first two are fundamentals books on the AQR VME universe (top 37.5% by market cap, financials
excluded). The third is deliberately price/volume-based instead: beta-neutral, financials kept in,
and a wider 75% cap fraction, because lottery demand lives outside the mega-cap tier.

## How to run it

You need the research repo and its gold data layer. From the repo root:

```bash
.venv/Scripts/python.exe src/experiments/factor_model_runner/run.py --config src/experiments/factor_model_runner/gpoa_ebit_ev.py --n 300 --seed 0
```

Swap `--config` for the model you want:

```bash
.venv/Scripts/python.exe src/experiments/factor_model_runner/run.py --config src/experiments/factor_model_runner/lottery_arbitrage_risk.py --n 300 --seed 0
```

Each run prints a run id and the path to its PDF, written to a fresh
`data/outputs/factor_model_runner/runs/<timestamp>_<slug>_<hash>/` directory. Runs never overwrite
each other.

### Useful flags

| Flag | What it does |
| --- | --- |
| `--n` | Sample this many **symbols** (each keeps full history). `--n 0` = the whole universe, and it is slow. |
| `--seed` | Seed for `--n` sampling. |
| `--stratify-by sector` | Keep the universe's sector mix in a small sample. |
| `--data-version <ts>` | Pin a gold snapshot so a multi-stage run reads one consistent view. |
| `--min-price` / `--min-dollar-volume` | Extra point-in-time row screens, e.g. `5` and `1e6`. |

Start small (`--n 150`) to confirm a config renders before paying for a large run.

### To list the signals you can use

```bash
.venv/Scripts/python.exe src/experiments/factor_model_runner/list_signals.py
```

## Writing a new model

A config is a plain Python file with up to four top-level names. `FIELDS` is the only required one:

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
- **`SETTINGS`** — model-wide construction, universe screens, benchmark and controls.
- **`FACTOR_SETTINGS`** — per-sleeve overrides; `directions` sets each signal's sign, where `-1`
  means low values score high.
- **`COMPARISONS`** — candidate sleeve versus an existing factor, head to head.

The config is validated strictly on load, so an unknown key or a bad direction fails immediately
rather than silently producing a wrong book.

## What each report contains

Research objective · configuration and sample · portfolio returns · benchmark fit and
benchmark-orthogonalized returns · attribution to the configured factor controls · signal
persistence · candidate comparisons and replacement · conclusion · reproducibility.

The distinction that matters: **benchmark fit** asks whether the book earns anything SPY does not
already deliver; **attribution** asks which existing factors the return is made of. Both are
reported, separately.

Returns are gross of costs (`cost_bps = 0.0`) in all three reports, so the comparisons are like
for like. The lottery/arbitrage book turns over far faster than the two fundamentals books, so
zero-cost numbers flatter it more.
