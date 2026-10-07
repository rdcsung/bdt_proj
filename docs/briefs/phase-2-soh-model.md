# Phase 2 brief: SOH model, actual vs predicted

Backlog items: **BT-201 – BT-206**. Read [`AGENTS.md`](../../AGENTS.md)
first. Requires Phase 1 (gold SOH table) to be done.

## Goal

Estimate SOH from cycle measurements (voltage, current, temperature,
timing) instead of reading measured capacity, and show actual vs predicted
SOH in the dashboard. The aim is a reliable, reproducible ML pipeline, not
a benchmark score.

## Work

### BT-201 Features (`bdt_pipeline`)

- Feature module computing per-discharge-cycle features from silver
  timeseries: voltage/current/temperature statistics, discharge duration,
  energy, temperature rise, cycle number, and the latest Re/Rct from the
  most recent impedance test **before** the cycle.
- **No capacity-derived features** (capacity, capacity delta, rolling
  capacity): the target is derived from capacity, so they leak it.
- Only past or same-cycle information; test this explicitly
  (`test_feature_no_future_leakage`).
- Output `gold/nasa_pcoe_battery/features/` with a `feature_version`.
  Every feature documented in `bdt_proj/docs/data-dictionary.md`.

### BT-202 Split

- Hold out whole batteries (grouped by test group so each split has
  comparable conditions); report per-battery results. Write the reasoning
  to `bdt_proj/docs/ml/split-strategy.md`.
- Expect noisy metrics: 34 batteries, uneven cycle counts and conditions.
  Report that honestly rather than tuning it away.

### BT-203 Baseline and BT-204 model

- Baseline first: SOH as a function of cycle number only.
- Then one interpretable model (linear / random forest / gradient
  boosting). No deep learning in this phase.
- Metrics: MAE, RMSE, R², max absolute error, per battery, and error vs
  cycle number. Compare against the baseline; if the model doesn't beat
  it, say so.

### BT-205 Training pipeline (`bdt_pipeline`)

- DAG: features → dataset validation → train → evaluate → write artefact.
- Artefact in MinIO under a model id, containing the model file plus a
  manifest: feature list and version, target definition, training data
  `ingest_date`, hyperparameters, seed, metrics, git commit, timestamp.
  Seeds fixed; re-running gives the same metrics.

### BT-206 Serve and show (`bdt`)

- Backend loads a named model artefact and serves predicted SOH per
  cycle and model info (id, metrics, data version). No training code in
  `bdt`.
- Dashboard: actual vs predicted on the SOH chart; model info panel.
- API tests: prediction for a known battery, unknown battery, model not
  available.

## Decisions needed (ask before implementing)

- **D4 Where training runs.** The stock Airflow image has no ML
  libraries. Options: (a) custom Airflow image built via JFrog with the
  ML dependencies; (b) training as its own container run from Airflow
  (`KubernetesPodOperator`), image built from `bdt_pipeline`; (c)
  `_PIP_ADDITIONAL_REQUIREMENTS` at startup (quick but slow and fragile).
  Proposal: (b), keeps Airflow stock and the training env versioned.
- **D5 The shared feature/model contract.** Training (`bdt_pipeline`)
  and inference (`bdt`) must agree on feature computation and artefact
  format. Options: a small shared package, or `bdt` only reads
  precomputed features + predictions from gold and never computes
  features itself. Proposal: the latter for this phase; batch predictions
  are written to gold by the pipeline, the API serves them.
- **D6 Library.** scikit-learn vs LightGBM/XGBoost, and the artefact
  format (joblib vs ONNX).

## Done when

- [ ] Training DAG runs end to end in Airflow; artefact + manifest in
  MinIO; re-run reproduces the metrics.
- [ ] Real metrics reported per battery, baseline vs model.
- [ ] Leakage test and feature unit tests pass; `uv run pytest` green in
  both repos.
- [ ] API and dashboard show actual vs predicted, verified locally, then
  in dev after owner approval.
- [ ] Split strategy and feature docs written; BT-201–BT-206 updated.
