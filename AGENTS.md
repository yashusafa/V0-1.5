# AGENTS.md

## Purpose
Build **V0 through V1.5 only** of a Databento-based intraday equity-options research pipeline.

This repository is for **predictive research and translation audit only**.
It is **not** a trading bot and **not** a backtesting/live-trading repo yet.

## Scope lock
Allowed in this repo:
- Databento historical ingestion
- local parquet caching
- metadata inspection
- point-in-time option-chain discovery
- deterministic candidate-row generation
- feature engineering approved in the spec
- V1 fixed-horizon bar-based labeling
- chronological train/validation/test splitting
- train-only inner CV
- model comparison
- hyperparameter search
- calibration
- validation-based selection
- one-time untouched test evaluation
- V1.5 fixed-horizon quote-aware translation audit

Forbidden in this repo:
- TP/SL path-dependent backtesting
- P&L backtesting
- strategy simulation
- broker APIs
- paper trading
- live trading
- order routing
- execution automation

If a task would create any of the forbidden items above, stop and say it is out of scope for V0–V1.5.

## Source of truth
Read these files before doing work:
1. `AGENTS.md`
2. `docs/spec_v0_v1_5.md`
3. `docs/PLAN.md`

If they conflict:
- `AGENTS.md` wins on scope and safety
- `docs/spec_v0_v1_5.md` wins on technical detail
- `docs/PLAN.md` wins on execution order

## Engineering rules
- Python only
- type hints required
- docstrings required
- config-driven paths and experiments
- deterministic random seeds where practical
- no notebook-only logic
- save outputs/artifacts/config snapshots
- keep modules small and testable
- prefer pure functions where practical
- do not invent unavailable fields
- if a field is assumed, mark it explicitly and make it configurable

## Data rules
Use only the data sources defined in `docs/spec_v0_v1_5.md`.

Do not substitute:
- midpoint fantasy fills for quote-aware audit
- OHLC high/low for bid/ask
- future information for present-time features
- static option-chain assumptions for point-in-time definitions

## Modeling rules
- All splits must be chronological
- No shuffling in the main pipeline
- Validation may drive design choices
- Test may not drive any design changes
- Calibration must use held-out data or CV-consistent procedures
- Final untouched test is evaluated once after configuration is frozen

## Workflow rules
For each task:
1. inspect the repo
2. restate assumptions
3. update or create a plan if needed
4. implement only the requested phase
5. run tests
6. update docs/reports
7. summarize exact changed files
8. stop

Do not skip directly to later phases.

## Done criteria
A phase is not done until:
- code is written
- tests are written and passing
- docs/reports are updated
- assumptions are listed
- commands to run are listed
- changed files are listed
- outputs are reproducible from config
