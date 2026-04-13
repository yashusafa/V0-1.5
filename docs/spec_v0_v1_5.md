# docs/spec_v0_v1_5.md

# Databento Intraday Equity-Options Research Pipeline
## V0 through V1.5 only

## 1. Objective

Build a research-grade, reproducible pipeline that answers two questions:

1. **V1**: Is there a useful fixed-horizon predictive signal for short-horizon option outcomes using approved bar-based information?
2. **V1.5**: Does that V1 signal translate in the right direction when evaluated against a quote-aware fixed-horizon outcome using historical bid/ask data?

This repo intentionally stops before true backtesting.
Do not implement:
- TP/SL path-dependent simulation
- trade-level P&L backtests
- paper trading
- broker connectivity
- live trading

## 2. Why the project is structured this way

This design is intentionally adversarial.
The goal is to reduce self-deception before any later V2 backtest work.

This means:
- deterministic candidate selection before scoring
- point-in-time chain discovery
- strict chronological splitting
- validation-driven selection
- untouched test evaluated once
- quote-aware translation audit before any backtest logic

## 3. Repo layout

project/
  AGENTS.md
  README.md
  requirements.txt
  .env.example
  .codex/
    config.toml
  config/
    datasets.yaml
    experiments.yaml
    features.yaml
    models.yaml
  data/
    raw/
      equs_ohlcv_1s/
      opra_definition/
      opra_ohlcv_1s/
      opra_cbbo_1s/
    interim/
    processed/
      candidates/
      labels_v1/
      labels_v15/
      features/
      splits/
      artifacts/
  reports/
    v0/
    v1/
    v15/
  docs/
    spec_v0_v1_5.md
    PLAN.md
  src/
    __init__.py
    settings.py
    logging_utils.py
    databento_client.py
    metadata_utils.py
    cache_store.py
    session_calendar.py
    event_flags.py
    equity_loader.py
    option_definition_loader.py
    option_bar_loader.py
    option_quote_loader.py
    contract_filters.py
    candidate_builder.py
    feature_builder.py
    label_builder_v1.py
    label_builder_v15.py
    splitters.py
    preprocess.py
    model_registry.py
    train.py
    calibrate.py
    evaluate.py
    audit_v15.py
    cli.py
  tests/
    test_databento_client.py
    test_chain_filters.py
    test_candidate_builder.py
    test_feature_builder.py
    test_labels_v1.py
    test_labels_v15.py
    test_splitters.py
    test_train_pipeline.py

## 4. Databento data scope

Use Databento Historical only.

Approved datasets/schemas:

### Underlying data
- dataset: `EQUS.MINI`
- schema: `ohlcv-1s`

### Option definitions
- dataset: `OPRA.PILLAR`
- schema: `definition`

### Option bars
- dataset: `OPRA.PILLAR`
- schema: `ohlcv-1s`

### Option quotes for V1.5 only
- dataset: `OPRA.PILLAR`
- schema: `cbbo-1s`

### Optional metadata helper
- `metadata.list_schemas`
- `metadata.get_dataset_range`
- `metadata.get_cost`

### Optional helper schema
- `statistics` may be implemented as a helper only if needed, but it is not a required modeling input in V1.

## 5. Databento usage rules

- Read API key from environment in production code: `DATABENTO_API_KEY`
- Example docs/tests may show placeholder `111111`
- Always check metadata before large pulls
- Cache all raw pulls locally to parquet
- Never re-pull the same range if it already exists in cache unless `force_refresh=true`
- Cache key must include at least:
  - dataset
  - schema
  - symbol family or symbols hash
  - start
  - end
  - stype_in
  - stype_out if used

## 6. Symbology and chain discovery

Use Databento parent symbology and point-in-time definitions.

Rules:
- use parent symbology like `SPY.OPT`
- use `stype_in=parent`
- pull definitions for the relevant session date/range
- derive candidate raw symbols from point-in-time definitions
- only then request option bars or quotes for shortlisted raw symbols

Do not brute-force the whole OPRA chain on every pass.

## 7. Initial universe

Allowed underlyings:
- SPY
- AAPL
- NVDA
- AMD

Default starting underlying for smoke and first end-to-end path:
- SPY

## 8. Session and timezone rules

Timezone:
- America/New_York

Regular trading session:
- 09:30:00 to 16:00:00 ET

Decision cadence:
- every 5 seconds

Default session filters to support:
- `all_rth`
- `open_early` = 09:35–10:30
- `morning` = 10:30–12:00
- `midday` = 12:00–14:00
- `afternoon` = 14:00–15:30

## 9. Event-day handling

Use a local event file at minimum:
- columns: `date`, `major_event_flag`

Support day filters:
- `all_days`
- `exclude_event_days`

Later extension may add richer official calendars, but V0–V1.5 only needs the local event map plus clean merge logic.

## 10. Candidate-row philosophy

Each row is:
- one option contract
- at one decision timestamp
- with only information available at that timestamp

Do **not** ask the model to choose directly from the whole option chain.
That is a mistake.

Correct flow:
1. point-in-time chain discovery
2. deterministic filtering
3. build candidate rows
4. engineer approved features
5. score candidates

## 11. Deterministic candidate filters

Default rules:
- allowed expiries: 0DTE and 1DTE
- calls and puts both allowed
- absolute strike distance from spot <= 2.0%
- contract must have valid option bar coverage over full lookback window
- contract must have valid current observation at decision timestamp
- `premium_lt_1` is a feature, not a hard exclusion by default

Do not define moneyness by a hard `$2 OTM` rule across all names.
That is unstable across symbols and volatility regimes and should not be treated as a universal contract bucket. This is one of the places people fool themselves. :contentReference[oaicite:5]{index=5}

## 12. Required candidate-row columns

Each candidate parquet should include at least:
- underlying_symbol
- raw_symbol
- instrument_id if available
- timestamp
- call_put
- strike
- expiry
- days_to_expiry
- seconds_to_expiry
- underlying_spot
- strike_distance_abs
- strike_distance_pct
- abs_strike_distance_pct
- session_date_et
- seconds_since_open
- seconds_to_close
- minute_of_day
- session_bucket
- major_event_flag
- experiment_day_filter
- experiment_session_filter

## 13. Approved feature families for V1

### A. Underlying bar features from EQUS.MINI / ohlcv-1s
For windows:
- 300s
- 600s
- 900s
- 1800s

Per window compute:
- cumulative return
- realized volatility of 1-second returns
- high-low range divided by current close
- average volume
- volume z-score vs trailing 30m
- fraction of up bars
- fraction of down bars
- last-bar return
- max drawup
- max drawdown

### B. Option bar features from OPRA.PILLAR / ohlcv-1s
For the same windows, where data exist:
- cumulative option return
- realized volatility
- range / close
- average option volume
- fraction up bars
- fraction down bars
- last-bar return
- max drawup
- max drawdown

### C. Current contract/state features
- current option close
- log option premium
- `premium_lt_1`
- premium bucket
- days_to_expiration
- seconds_to_expiration
- strike distance absolute
- strike distance percent
- abs strike distance percent
- moneyness bucket
- underlying current close
- underlying current log price
- underlying symbol as categorical
- call_put as categorical

### D. Time-of-day features
Always include:
- seconds_since_open
- seconds_to_close
- minute_of_day
- session_bucket
- is_opening_30m
- is_midday
- is_last_30m
- tod_sin
- tod_cos

### E. Event feature
- major_event_flag

## 14. Explicitly forbidden feature families in V1/V1.5 training

Do not add these unless later scope explicitly changes:
- prior-day node / volume-profile features
- unusual options-flow features
- IV/Greeks
- quote-side spread features in V1 training
- L1 quote features as model inputs in V1
- features that require future information
- features built using globally normalized full-dataset statistics before split

Your earlier instinct to add “unusual options flow” is exactly the kind of thing that can create timestamp-alignment leakage if the feed timing is not proven. That concern was already in your pasted file and it is valid. :contentReference[oaicite:6]{index=6}

## 15. V1 labels

V1 is **bar-based predictive research only**.

Horizons:
- 60s
- 120s
- 240s

For each horizon h:
- `future_return_h = close_{t+h} / close_t - 1`

Binary classification targets:
- `y_up_05_h`
- `y_up_10_h`
- `y_up_15_h`

Meaning:
- 1 if `future_return_h >= threshold`
- 0 otherwise

Auxiliary regression target:
- `y_return_h`

Important:
- these are **not** trade-level TP/SL labels
- these are **not** quote-aware execution labels
- these are **not** backtest labels

## 16. V1.5 labels

V1.5 is a **quote-aware fixed-horizon translation audit only**.

Use `OPRA.PILLAR / cbbo-1s`.

For each horizon h in {60s, 120s, 240s}:
- `entry_ask_t` = first valid ask at decision timestamp
- `exit_bid_t+h` = first valid bid at or after the horizon
- `quote_return_h = exit_bid_t+h / entry_ask_t - 1`

Binary quote-aware labels:
- `y_quote_up_05_h`
- `y_quote_up_10_h`

Important:
- no TP/SL
- no path-dependent logic
- no P&L backtest
- no trade simulation

This phase exists because translating a bar-based signal to an actual quote-aware object is where many naive options ideas fail. Your own earlier concern about label honesty was exactly right. :contentReference[oaicite:7]{index=7}

## 17. Split discipline

Split by complete sessions only.

Default session split:
- earliest 70% sessions = train
- next 15% sessions = validation
- latest 15% sessions = untouched test

Rules:
- no shuffling ever
- validation may drive design choices
- test may not drive any design changes
- save split manifest with exact dates and row counts

Inside train only:
- use `TimeSeriesSplit`
- `n_splits = 5`
- `gap = max_horizon_samples`

Compute `max_horizon_samples` from:
- decision cadence
- max prediction horizon

Reason:
time-ordered data require time-aware splitting, and scikit-learn explicitly states `TimeSeriesSplit` is for time-ordered data where ordinary CV would train on the future and evaluate on the past. Its `gap` parameter is specifically useful when adjacent samples overlap in time. :contentReference[oaicite:8]{index=8}

## 18. Overfitting discipline

Validation set is for:
- model family selection
- feature-set selection within approved families
- horizon selection
- threshold selection
- session-filter selection
- day-filter selection
- hyperparameter selection
- calibration selection

Test set is for:
- final locked evaluation once

If any decision changes after seeing test:
- that test is contaminated
- create a new untouched holdout before claiming anything

That exact discipline was a central concern in your pasted file, and it is correct. Validation can still be overfit if you iterate recklessly, so keep experiment tracking explicit. :contentReference[oaicite:9]{index=9}

## 19. Model families

Implement exactly these V1 model families:
1. LogisticRegression pipeline
2. HistGradientBoostingClassifier
3. RandomForestClassifier
4. MLPClassifier

## 20. Preprocessing

Use sklearn preprocessing with explicit pipelines.

Requirements:
- `ColumnTransformer`
- `SimpleImputer`
- `StandardScaler` for numeric features where appropriate
- `OneHotEncoder` for categorical features where appropriate

Persist:
- column lists
- preprocessors
- fitted artifacts
- config snapshots

## 21. Hyperparameter search

### LogisticRegression
Use `GridSearchCV`

Search space:
- solver: `lbfgs`, `saga`
- penalty:
  - `l2` for `lbfgs`
  - `l1`, `l2`, `elasticnet` for `saga`
- C: `0.01`, `0.1`, `1.0`, `10.0`, `100.0`
- l1_ratio: `0.0`, `0.5`, `1.0` for elasticnet
- max_iter: `500`, `1000`
- class_weight: `None`, `balanced`

### HistGradientBoostingClassifier
Use `RandomizedSearchCV`

Search space:
- learning_rate: `0.01`, `0.03`, `0.05`, `0.1`
- max_iter: `100`, `200`, `400`
- max_leaf_nodes: `15`, `31`, `63`, `127`
- max_depth: `None`, `3`, `5`, `7`
- min_samples_leaf: `20`, `50`, `100`, `200`
- l2_regularization: `0.0`, `0.01`, `0.1`, `1.0`
- class_weight: `None`, `balanced`
- randomized search `n_iter=25`

### RandomForestClassifier
Use `RandomizedSearchCV`

Search space:
- n_estimators: `200`, `400`, `800`
- max_depth: `None`, `6`, `12`, `20`
- min_samples_leaf: `1`, `5`, `20`, `50`
- max_features: `sqrt`, `0.25`, `0.5`, `0.75`
- class_weight: `None`, `balanced`, `balanced_subsample`
- randomized search `n_iter=25`

### MLPClassifier
Use `RandomizedSearchCV`

Search space:
- hidden_layer_sizes: `(64,)`, `(128,)`, `(64,64)`, `(128,64)`, `(128,128)`
- activation: `relu`, `tanh`
- alpha: `1e-5`, `1e-4`, `1e-3`, `1e-2`
- learning_rate_init: `1e-4`, `1e-3`, `1e-2`
- batch_size: `256`, `512`, `1024`
- solver: `adam`
- early_stopping: `True`
- max_iter: `200`, `400`
- randomized search `n_iter=20`

Primary search metric:
- `neg_log_loss`

Secondary metrics to log:
- ROC AUC
- Average Precision
- Precision
- Recall
- F1
- Balanced Accuracy

## 22. Calibration

Evaluate for each model family:
- uncalibrated
- sigmoid
- isotonic

Probability calibration belongs here because scikit-learn is explicit that some classifiers produce poor probability estimates, and calibration should be done on data not used for fitting. Calibration curves are reliability diagrams, and Brier score is a proper probabilistic loss where lower is better. :contentReference[oaicite:10]{index=10}

Required calibration outputs:
- calibration curve / reliability diagram
- Brier score
- log loss
- decile table of predicted probability vs empirical frequency

Selection rule:
1. lowest validation Brier score
2. tie-break by highest validation Average Precision
3. tie-break by highest validation ROC AUC

## 23. Untouched test evaluation

After validation chooses the final frozen configuration, evaluate once on untouched test.

Required outputs:
- ROC AUC
- Average Precision
- Precision
- Recall
- F1
- Balanced Accuracy
- Log loss
- Brier score
- calibration curve
- confusion matrix at default threshold 0.5
- confusion matrix at top-decile score threshold

Slice analyses:
- underlying symbol
- call vs put
- DTE bucket
- moneyness bucket
- session bucket
- event vs non-event day

## 24. V1.5 translation audit report

Using the frozen V1 model only, report:
- quote-aware ROC AUC
- quote-aware Average Precision
- quote-aware Brier if calibrated probabilities are used
- score-decile lift table
- empirical positive rate by score decile
- comparison of bar-based positive rate vs quote-aware positive rate
- monotonicity check of score deciles

Required conclusion language:
- do not imply tradability
- do not call it a backtest
- state whether higher V1 scores correspond to better quote-aware fixed-horizon outcomes

## 25. Logging, artifacts, and reproducibility

For every experiment save:
- config snapshot
- split manifest
- feature schema
- training metrics
- search results
- calibration results
- chosen final config
- artifact paths
- random seed
- report markdown/json

Outputs must be reproducible from config and cached raw data.

## 26. Testing requirements

Minimum test coverage areas:
- environment variable handling
- metadata helper behavior
- cache hit / miss logic
- chain discovery from parent symbology
- raw symbol extraction
- ET session-date handling
- event-flag merge
- contract-filter correctness
- decision cadence logic
- feature null handling
- feature family presence/absence
- label correctness on toy data
- split chronology and no leakage
- gap behavior in train-only CV
- quote-return calculations for V1.5
- training pipeline smoke test

## 27. CLI surface

Required commands:
- `metadata-check`
- `pull-data`
- `build-candidates`
- `build-features`
- `build-labels-v1`
- `build-labels-v15`
- `train-v1`
- `audit-v15`

Every command must support:
- `--config`
- `--help`

## 28. What this repo is not allowed to claim

This repo may not claim:
- profitable strategy discovered
- live tradability proven
- path-dependent execution validated
- real backtest completed
- broker-ready system exists

At most, after V1.5 it may claim:
- a bar-based signal exists or does not exist
- the signal does or does not translate directionally to a quote-aware fixed-horizon outcome

## 29. Known limitations section to include in README

- V1 uses bar-based labels, not trade-execution labels
- V1.5 uses fixed-horizon quote-aware translation, not TP/SL simulation
- there is no backtest yet
- there is no paper-trading support yet
- there is no live-trading support yet
- option-chain quality and quote coverage can still limit interpretability
