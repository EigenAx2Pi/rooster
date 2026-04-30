# rooster — Status

> Last updated: 2026-04-30

## Current focus

**Shipped.** v0.1.0 graduated from playfield to `~/repo/prod/rooster/` and
pushed to public `https://github.com/EigenAx2Pi/rooster`. CI green. No
active work — see "Next up" if you want to pick something.

## In flight

- (none)

## Next up

- Replace synthetic-data references with a real anonymised demo screenshot
  of the web UI in the README.
- Optional: add a `/api/benchmark` endpoint that runs the held-out
  evaluation on uploaded data and returns the metrics table inline.
- Optional: ONNX export of the GBM for cold-start latency.
- Optional: containerise the benchmark into a `make demo` target.

## Blockers

- (none)

## Recently completed (v0.1.0)

- Graduated from `playfield/mindforge/rooster/` to `~/repo/prod/rooster/`,
  fresh git history, public GitHub repo `EigenAx2Pi/rooster`, tagged
  `v0.1.0`, CI green on first push.
- Old public `EigenAx2Pi/ROOSTER` repo deleted (case-collision avoided).
- Added `samples/{roster,holidays,template}_sample.xlsx` so visitors can
  exercise the web UI without their own data, plus
  `scripts/make_samples.py` to regenerate them.
- Renamed legacy `app/rooster.py` → `app/core.py`; dropped duplicate
  `ML RULE BASED ROOSTER.py`.
- Added `app/ml.py` (sklearn `GradientBoostingClassifier` with leakage-safe
  features), `app/synth.py` (deterministic synthetic 6-month roster
  generator with drift + Thu→Fri correlation), `app/eval.py` (held-out
  evaluation, both models scored on the same labels).
- Added `scripts/benchmark.py` — one-command rule-vs-ML comparison.
- `pyproject.toml` (PEP 621), ruff config, pytest config, MIT `LICENSE`.
- Tests: 9 passing in `tests/test_{synth,core,ml}.py` (incl. explicit
  no-leakage assertion).
- `Dockerfile` (multi-stage, non-root user, healthcheck) + GitHub Actions
  CI matrix (Python 3.10 / 3.11 / 3.12). All three runs green.
- Rewrote README for recruiter / CTO audience: problem framing, live
  benchmark results table, architecture diagram, "why rules, not ML"
  section, "try it without your own data" walkthrough.

## Live links

- Repo: https://github.com/EigenAx2Pi/rooster
- v0.1.0 tag: https://github.com/EigenAx2Pi/rooster/releases/tag/v0.1.0
- CI: https://github.com/EigenAx2Pi/rooster/actions
- Samples: https://github.com/EigenAx2Pi/rooster/tree/main/samples
