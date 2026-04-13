# docs/PLAN.md

# Execution Plan for V0 through V1.5

## Ground rule
Do not work on later phases until the current phase is complete, tested, documented, and summarized.

## Phase 0 — bootstrap and guardrails
Deliver:
- repo scaffold
- `AGENTS.md`
- `docs/spec_v0_v1_5.md`
- `docs/PLAN.md`
- README
- requirements.txt
- `.env.example`
- CLI skeleton
- test skeleton

Exit criteria:
- project structure exists
- commands import successfully
- tests pass for scaffolding
- docs explain scope lock clearly

## Phase 1 — Databento client, metadata, caching, raw pulls
Deliver:
- Historical client wrapper
- metadata helpers
- parquet cache layer
- underlying bar loader
- option definition loader
- option bar loader
- option quote loader
- tests for client, metadata, cache, and symbology

Exit criteria:
- metadata-check works
- raw pulls write to cache
- parent symbology discovery works
- README usage examples updated

## Phase 2 — session calendar, event flags, candidate-row generation
Deliver:
- ET session handling
- event-flag merge
- deterministic contract filters
- candidate-row builder
- row-count summary reports
- tests for session logic and filters

Exit criteria:
- reproducible candidate rows exist
- experiment session/day filters work
- reports show counts by key slices

## Phase 3 — feature engineering
Deliver:
- approved feature families only
- model-ready feature tables
- data dictionary
- tests for null handling and feature exclusion
- feature diagnostics report

Exit criteria:
- feature parquet exists
- forbidden feature families absent
- required features documented

## Phase 4 — V1 label generation and split manifests
Deliver:
- fixed-horizon bar labels
- regression target
- session-level split manifests
- inner train-only TimeSeriesSplit helper with gap
- tests for labels and split chronology

Exit criteria:
- label tables exist
- split manifests exist
- no leakage in split logic

## Phase 5 — V1 model training, search, calibration, validation selection
Deliver:
- preprocessing pipelines
- four model families
- hyperparameter search
- calibration comparisons
- saved artifacts
- validation reports

Exit criteria:
- all four model families run
- calibration outputs saved
- final frozen V1 config selected using validation only

## Phase 6 — untouched V1 test evaluation
Deliver:
- one-time test evaluation for frozen config
- metrics and slice reports
- calibration plots on test
- final V1 report

Exit criteria:
- untouched test evaluated once
- result artifacts saved
- report explicitly states that this is not a backtest

## Phase 7 — V1.5 quote-aware translation audit
Deliver:
- quote-aware fixed-horizon label builder
- quote-return calculations
- score-decile lift tables
- V1 vs V1.5 comparison report
- tests for quote-return logic

Exit criteria:
- V1.5 report generated
- no TP/SL logic introduced
- no P&L backtest introduced

## Phase 8 — hardening
Deliver:
- stronger tests
- end-to-end smoke path
- CLI polish
- reproducibility review
- known limitations updated

Exit criteria:
- docs coherent
- tests pass
- repo is ready for human review before any V2 backtest work

## Non-negotiable stage gates

### Before leaving Phase 5:
- feature list frozen
- splits frozen
- model family/hyperparameter/calibration decisions made from validation only

### Before leaving Phase 6:
- test evaluated once
- no design changes made because of test

### Before leaving Phase 7:
- V1.5 remains fixed-horizon quote-aware audit only
- still no backtesting logic

## If Codex gets stuck
When a phase stalls:
1. ask for a short retrospective
2. identify the repeated mistake
3. patch `AGENTS.md` only if the mistake is durable and general
4. do not broaden scope to “solve” the blockage
