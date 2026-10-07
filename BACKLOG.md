# Battery Twin backlog

Product backlog for the battery digital twin, built on the
[NASA PCoE Li-ion battery aging data set](https://www.nasa.gov/intelligent-systems-division/discovery-and-systems-health/pcoe/pcoe-data-set-repository/).
The NASA data is used to prove the whole architecture end to end without
needing real vehicle data:

```
raw battery data → Parquet → features → SOH → RUL → anomaly → API → dashboard → AI agent
```

Repos involved:

| Repo | Role |
|---|---|
| [`bdt_pipeline`](https://github.com/rdcsung/bdt_pipeline) | Airflow DAGs: ingestion, silver/gold layers, feature and training jobs |
| [`bdt`](https://github.com/rdcsung/bdt) | FastAPI backend + React dashboard (currently on synthetic data) |
| [`bdt-gitops`](https://github.com/rdcsung/bdt-gitops) | Argo CD manifests for the app and platform services |
| [`bdt-infra`](https://github.com/rdcsung/bdt-infra) | Ansible for the k3s cluster, runbooks |

Status: `[x]` done · `[ ]` open · `[~]` in progress

## Use cases at a glance

| # | Use case | What you build | Difficulty | Value |
|---|---|---|---:|---:|
| 1 | SOH estimation | Predict SOH from recent voltage/current/temperature data | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| 2 | Remaining useful life (RUL) | Predict how many cycles remain before end of life (EOL) | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 3 | Degradation prediction | Predict the future capacity/SOH trajectory | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 4 | Anomaly detection | Detect abnormal degradation or cycling behaviour | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| 5 | Explainable SOH | Show *why* the model estimates a given SOH | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 6 | Battery digital twin | Current state + predicted future state, served by API and dashboard | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 7 | Battery AI agent | Ask an LLM questions about battery data | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

**MVP = Phases 1–3:** SOH + RUL + degradation visualisation, not just a
generic neural network. No LLM until the ML models work.

---

## Phase 1: NASA data → SOH → Battery Twin dashboard

Goal: the dashboard shows real NASA batteries with measured SOH and its history.

- [x] **BT-101 Ingest NASA data to bronze.** `nasa_battery_ingest` DAG in
  `bdt_pipeline`: download → extract → `.mat` to Parquet (`cycles`,
  `timeseries`, `impedance`) in `s3://bdt-lake/bronze/nasa_pcoe_battery/`.
- [ ] **BT-102 Silver layer.** Clean and deduplicate bronze (B0025–B0028
  appear twice in the archive), normalise units/types, one row per battery
  per cycle plus the per-cycle timeseries. Write to `silver/`.
- [ ] **BT-103 SOH ground truth.** From the discharge capacity per cycle,
  compute `SOH = capacity / initial capacity`. Output dataset:
  `battery_id, cycle, capacity, soh, ambient_temp, ...` in `gold/`.
  - Acceptance: per-battery SOH curve starts at ~1.0, EOL cycle derivable
    at the 70 % / 1.4 Ah threshold used by NASA.
- [ ] **BT-104 Backend reads NASA data.** Add a data source to `bdt` backend
  that reads the gold SOH dataset from MinIO alongside (or instead of) the
  synthetic generator. Endpoints: list batteries, SOH history per battery.
- [ ] **BT-105 Battery Twin card in the dashboard.** Per battery: current
  SOH, SOH bar, trend (%/cycle over the last N cycles), degradation
  history chart.

```
┌──────────────────────────────────┐
│        BATTERY TWIN              │
│ SOH              89.0 %          │
│ Remaining life   ~180 cycles     │
│ Current trend    ↓ 0.12%/cycle   │
│ ██████████████████░░ 89%         │
│ [ Degradation history ]          │
└──────────────────────────────────┘
```

## Phase 2: Features → SOH ML model → actual vs predicted

Goal: estimate SOH from raw measurements instead of using the measured
capacity directly. The NASA capacity measurement is the ground truth.

- [ ] **BT-201 Feature engine.** Per cycle from the V/I/T timeseries:
  voltage curve features, current curve, temperature curve (max/mean/rise),
  charge time, discharge time, energy, cycle number. Write to `gold/features/`.
- [ ] **BT-202 Train/validation split by battery.** Hold out whole batteries
  (e.g. leave-one-battery-out), not random cycles, to avoid leakage.
- [ ] **BT-203 Baseline SOH model.** Gradient boosting / linear baseline on
  BT-201 features. Report MAE/RMSE per held-out battery.
- [ ] **BT-204 Deep learning SOH model.** 1D CNN / LSTM on the raw curves;
  compare against BT-203.
- [ ] **BT-205 Training as a pipeline.** Airflow DAG that builds features,
  trains, evaluates and stores the model artefact + metrics in MinIO.
- [ ] **BT-206 Serve predictions.** Backend endpoint returning predicted SOH
  per cycle; dashboard chart of actual vs predicted SOH.

## Phase 3: RUL and future degradation

Goal: answer "how long will this battery last?", not just "what is the SOH now?".

- [ ] **BT-301 RUL labels.** For each cycle, cycles remaining until EOL
  threshold.
- [ ] **BT-302 RUL model.** Predict remaining cycles from current state and
  history. Example output: current cycle 650, SOH 82 %, predicted EOL
  cycle 920, RUL ~270 cycles.
- [ ] **BT-303 Degradation trajectory forecast.** Given cycles 1..N, predict
  the SOH trajectory for N+1..EOL. Evaluate at several cut-off points.
- [ ] **BT-304 Uncertainty.** Prediction intervals for RUL and trajectory.
- [ ] **BT-305 Dashboard: forecast view.** Actual SOH vs forecast trajectory
  with interval band, EOL line and predicted EOL cycle; "Remaining life"
  on the twin card.

Automotive mapping: BMS measurements → Battery Twin → SOH → degradation
model → RUL → maintenance / fleet decision.

## Phase 4: Anomaly detection

Goal: flag batteries that behave differently from the normal degradation pattern.

- [ ] **BT-401 Degradation-rate change detection.** Detect breakpoints where
  the degradation slope accelerates. Example: "Anomaly detected around
  cycle 620. Capacity degradation accelerated by 2.8× compared with the
  previous trend."
- [ ] **BT-402 Cross-battery anomaly detection.** Compare each battery's
  V/I/T/capacity behaviour with the fleet; flag outliers.
- [ ] **BT-403 Anomaly events API + dashboard markers.** Store anomaly
  events; show them on the SOH chart and in a battery list.

## Phase 5: Explainable SOH

Goal: no black-box SOH number; show what drove the estimate.

- [ ] **BT-501 Feature attribution.** SHAP (or similar) on the SOH model,
  grouped into voltage profile, discharge capacity, temperature, charge
  characteristics.
- [ ] **BT-502 Confidence score.** Per-prediction confidence derived from
  model uncertainty.
- [ ] **BT-503 Explanation view.** Dashboard panel: SOH, contributing
  factors with weights, confidence, and the actual voltage curve the
  prediction was based on.

```
SOH = 87.3%
Main contributing factors:
  Voltage profile          42%
  Discharge capacity       27%
  Temperature              18%
  Charge characteristics   13%
Confidence: 94%
```

## Phase 6: Battery digital twin

Goal: one service that holds each battery's current state and predicts its future state.

```
NASA dataset → ingestion (MAT → Parquet) → feature engine (V/I/T/time)
            → SOH model + RUL model → Battery Twin API (FastAPI) → React dashboard
```

- [ ] **BT-601 Twin state model.** Per battery: latest measurements, SOH
  (measured + predicted), RUL, trend, anomalies, explanation.
- [ ] **BT-602 Twin API.** Consolidated FastAPI endpoints for twin state and
  forecasts, authenticated via Keycloak like the existing API.
- [ ] **BT-603 Replay mode.** Replay a NASA battery cycle by cycle to
  simulate live telemetry and watch the twin update.
- [ ] **BT-604 Deploy.** Model serving and new jobs added to `bdt-gitops`
  (dev → qa → prod).

## Phase 7: Battery AI agent

Goal: natural-language analysis on top of the twin. Only after Phases 2–4 work.

- [ ] **BT-701 Agent tools.** Tools over the twin API: cycle data, voltage
  curves, temperature, capacity history, SOH/RUL predictions, anomalies.
- [ ] **BT-702 Comparison questions.** E.g. "Why did B0005 degrade faster
  than B0006?" → answer citing temperature exposure, capacity-loss onset,
  discharge characteristics, degradation-rate difference.
- [ ] **BT-703 Chat panel in the dashboard.**
- [ ] **BT-704 Evaluation set.** A fixed set of questions with expected
  facts, to check the agent's answers against the data.

---

## Ideas / later

- Other data sets beside NASA under the same `bdt-lake` layout (other PCoE
  sets, own cell tests).
- Fleet view: rank batteries by RUL / anomaly score.
- Link to a longer-term battery verification agent.
