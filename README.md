# Rooster

> Predictive employee rosters from historical attendance data.
> Interpretable rule-based predictor in production, gradient-boosted ML benchmark for evaluation.

[![CI](https://github.com/EigenAx2Pi/rooster/actions/workflows/ci.yml/badge.svg)](https://github.com/EigenAx2Pi/rooster/actions/workflows/ci.yml)
[![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Code style: ruff](https://img.shields.io/badge/code%20style-ruff-000000.svg)](https://github.com/astral-sh/ruff)

---

## The problem

Workforce-management teams forecast next month's office presence from historical
swipe / booking data. The data is messy (Excel, merged headers, holiday
calendars per city), the constraints are real (≥3 days/week hybrid mandates,
holiday blackouts), and the consumers are non-technical (HR ops, facilities).
"Black-box ML" is a bad fit when the output gets challenged by an employee on a
Wednesday morning.

Rooster ships an **interpretable rule-based predictor as the production
default** and a **gradient-boosted ML model as a benchmark** so we can
quantify what we'd be giving up — or gaining — by switching.

## Results

Held-out evaluation on synthetic data that includes weekday preferences,
mid-history drift (~20% of employees), and a Thursday→Friday correlation
that pure rules cannot capture (50 employees, 6 months, seed=42):

<!-- METRICS_BLOCK_START -->
| Model | Accuracy | Precision | Recall | F1 | AUC |
|---|---|---|---|---|---|
| Rule-based (baseline) | 0.624 | 0.604 | 0.771 | 0.677 | n/a |
| Gradient-boosted (ML) | 0.622 | 0.596 | 0.812 | 0.687 | 0.675 |

> ML lifts F1 from 0.677 → 0.687 and recall from 0.771 → 0.812 — the gain
> comes from features the rule cannot express (recency drift,
> Thursday→Friday correlation). Accuracy is comparable; the rule remains
> calibrated, explainable per-cell, and zero-retrain.
<!-- METRICS_BLOCK_END -->

The rule baseline is **calibrated**, **explainable per-cell**, and survives
small data — that's why it's the production default. The ML model serves as a
**ceiling reference**: if its lift exceeds a meaningful margin on real data,
that's the trigger to consider promoting it.

Reproduce locally:

```bash
pip install -e ".[dev]"
python -m scripts.benchmark
```

## Architecture

```
                 ┌──────────────────────┐
  Excel uploads  │  FastAPI (app/main)  │   downloadable
  ───────────►   │   /api/predict       │ ──────────►  Excel
                 │   /api/download      │              roster
                 └──────────┬───────────┘
                            │
                ┌───────────┴───────────┐
                ▼                       ▼
      ┌──────────────────┐    ┌──────────────────┐
      │  app.core        │    │  app.ml          │
      │  rule-based      │    │  GBM benchmark   │
      │  (production)    │    │  (evaluation)    │
      └──────────────────┘    └──────────────────┘
                ▲                       ▲
                └─────────┬─────────────┘
                          │
                ┌─────────┴─────────┐
                │  app.eval         │  held-out month,
                │  app.synth        │  leakage-safe split,
                └───────────────────┘  reproducible synthetic data
```

| Module | Role |
|---|---|
| `app/core.py` | Rule-based predictor: weekday-frequency threshold + min-days-per-week top-up. Deployed. |
| `app/ml.py` | Gradient-boosted classifier. Leakage-safe per-(employee, weekday) features. |
| `app/synth.py` | Reproducible synthetic roster generator — non-trivial signal (drift, Thu→Fri correlation). |
| `app/eval.py` | Held-out month evaluation. Aligns predictions and reports precision/recall/F1/AUC. |
| `app/main.py` | FastAPI service: upload Excel → return populated roster Excel. |
| `scripts/benchmark.py` | One-command end-to-end benchmark on synthetic data. |
| `scripts/make_samples.py` | Regenerate the synthetic Excel files in `samples/`. |
| `samples/` | Ready-to-upload `.xlsx` files so you can try the web UI without your own data. |

## Stack

**Backend** Python 3.10+ · FastAPI · pandas · openpyxl · scikit-learn
**Frontend** vanilla HTML / CSS / JS (no framework, no build step)
**Tooling** ruff · pytest · GitHub Actions · Docker

## Run it

### Web app (Docker)

```bash
docker build -t rooster:latest .
docker run --rm -p 8000:8000 rooster:latest
# open http://localhost:8000
```

### Web app (local)

```bash
pip install -e .
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

### Try it without your own data

The repo ships sample Excel files in `samples/` so you can exercise the web
UI immediately:

```bash
uvicorn app.main:app --port 8000
# in another terminal — verify against the live API:
curl -X POST http://localhost:8000/api/predict \
    -F "roster_file=@samples/roster_sample.xlsx" \
    -F "holiday_file=@samples/holidays_sample.xlsx" \
    -F "month=5" -F "year=2025" \
    -F "threshold=0.6" -F "min_days_per_week=3"
```

Or upload them through the browser at `http://localhost:8000`. Set
**month=5, year=2025** — that's the held-out month for the synthetic
4-month history. To regenerate the samples (e.g. with different
parameters): `python -m scripts.make_samples`.

### Benchmark

```bash
pip install -e ".[dev]"
python -m scripts.benchmark
# or with custom synthetic-data parameters:
python -m scripts.benchmark --employees 100 --months 8 --seed 7
```

### Tests

```bash
pytest -v
ruff check .
```

## How the predictor works

### Rule-based (production)

For each `(employee, weekday)` pair, the predictor computes booking
frequency over the historical window. If the frequency meets `--threshold`
(default 0.6), the cell is marked `Y`. After thresholding, each
`(employee, week-of-month)` is checked against `--min-days-per-week` (default 3);
underbooked weeks are topped up using the highest-frequency days.

The intent is **calibration**: if an employee has been in 4 of 5 Mondays for
six months, predict Monday. If you challenge the prediction you get a number
back: "you booked Monday 80% of the time."

### ML benchmark

Per-`(employee, target-workday)` binary classification with leakage-safe
features computed only from a strictly-historical feature window:

- weekday one-hot, week-of-month, month
- employee overall booking rate
- employee per-weekday booking rate
- employee recent (last-30-days) booking rate
- previous Thursday's booking outcome (when predicting Friday)

Trained on month *N-1* labels with features from months *1..N-2*; evaluated on
month *N* labels with features from months *1..N-1*. The synthetic data
generator embeds a Thursday→Friday correlation specifically to give the ML
model a non-rule-derivable signal to find.

## Why rules, not ML?

A few honest reasons:

1. **Calibration challenge tomorrow morning.** When an employee asks why
   they're booked Wednesday, "your historical Wednesday rate is 84%" beats
   "the model said so."
2. **Data scarcity per employee.** 6 months × 5 days/week = 130 observations
   per employee. The rule's bias is *exactly* the inductive bias the data
   supports.
3. **The ML lift is real but small.** The benchmark above shows the ceiling.
   Until that lift translates into an SLA-relevant gain on *real* data, the
   complexity is not worth it.
4. **Operability.** The rule has two tunable knobs (`threshold`,
   `min_days_per_week`) and zero retrain cost. Ops likes that.

The ML model stays in the repo as a benchmark and as a forcing function: if
the rule ever underperforms on real data, the comparison is one command away.

## Project layout

```
rooster/
├── app/
│   ├── core.py        # rule-based predictor (production)
│   ├── ml.py          # gradient-boosted benchmark
│   ├── synth.py       # synthetic data generator
│   ├── eval.py        # held-out evaluation harness
│   └── main.py        # FastAPI service
├── scripts/
│   └── benchmark.py   # one-command rule-vs-ML comparison
├── static/            # vanilla web UI
├── tests/             # pytest unit tests
├── .github/workflows/ # CI: pytest + ruff + benchmark on every push
├── Dockerfile
└── pyproject.toml
```

## What I learned building this

- **Leakage in panel-data forecasting is subtle.** Computing a feature like
  "employee historical rate" from the *full* history seems harmless until you
  notice the feature for an October row was computed using October labels.
  The fix is a strict feature window that ends before any label being predicted.
- **A rule baseline is not a strawman.** On data where the underlying signal
  is `P(book | employee, weekday)`, a simple frequency threshold *is* the
  Bayes-optimal classifier minus noise. The ML lift only shows up where the
  rule's independence assumption breaks.
- **Operational constraints (`min ≥ 3 days/week`) are the cheap part of
  scheduling, but easy to mis-implement.** Rooster enforces it after the
  threshold pass, not during, so the constraint never silently overrides a
  high-confidence "no".

## License

MIT — see [`LICENSE`](LICENSE).
