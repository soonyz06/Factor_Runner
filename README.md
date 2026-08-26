# Factor Runner

Rendered output from a configurable factor-model runner. Each PDF in `outputs/` is one
self-contained study: how a candidate model was built, what it earned, and whether it survives the
market benchmark and the standard four factors (VAL, MOM, BAB, QMJ). Every report opens with an
abstract written to be read *instead of* the report.

The runner lives in a private research repo; this repo holds only its output.

## Running it

You never edit the runner to test an idea. You add one declarative config and point `--config` at
it:

```bash
python src/experiments/factor_model_runner/run.py --config <config>.py --n 0 --seed 0
```

`--n 0` is the full screened universe, about 55 seconds. `--n 300` samples 300 symbols and is
worth it only while checking that a new config renders at all. Sampling is the only thing `--n`
controls: the universe screen, price floor, market-cap fraction and financial exclusion always
apply. Each run writes its own timestamped directory and never overwrites another.

## Writing a config

Four names, of which only `FIELDS` is required:

```python
FIELDS = {"Quality": ["gpoa"], "Value": ["earnings_yield"]}
SETTINGS = {"name": "gpoa plus EBIT-EV", "top_market_cap_fraction": 0.375,
            "exclude_financials": True}
FACTOR_SETTINGS = {"Quality": {"directions": {"gpoa": 1}}}
COMPARISONS = [{"candidate": "Quality", "baseline": "QMJ", "label": "gpoa versus QMJ"}]
```

`FIELDS` names the sleeves and their signals. `SETTINGS` covers construction, screens and controls.
`FACTOR_SETTINGS` holds per-sleeve overrides, where a direction of `-1` means low values score
high. `COMPARISONS` names the standard factor each candidate would replace. Configs are validated
on load, so an unknown key or a bad direction fails immediately rather than quietly producing the
wrong book.

Three settings move results more than the rest:

- `top_market_cap_fraction` — a fraction, never a fixed top-k, since the universe grows from 1,427
  names in 2007 to 4,773 in 2026. `0.375` is the AQR VME convention; the attention- and
  liquidity-driven books widen it.
- `model_combine` — `score` (default) pools sleeve scores and sorts once, giving one portfolio;
  `returns` equal-weights the sleeve return series. Different books, not two spellings of one.
- `min_components` — how many of a sleeve's signals a row needs to be scored. 60% by default, so a
  sparse signal degrades a factor instead of deleting most of its cross-section.
