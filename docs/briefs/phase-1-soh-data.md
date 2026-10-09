# Phase 1 brief: NASA SOH data → dashboard

Backlog items: **BT-102 – BT-105**. Read [`AGENTS.md`](../../AGENTS.md) first.

## Goal

The dashboard shows real NASA batteries with their measured SOH history.
No ML in this phase: SOH comes straight from measured discharge capacity.

```
bronze (exists) → silver → gold SOH table → bdt API → dashboard
```

## Facts about the data

From the bronze schemas in `bdt_pipeline/dags/nasa_battery/convert.py`:

- `cycles`: one row per cycle, `cycle_type` ∈ charge / discharge /
  impedance. `capacity_ah` is set **only on discharge cycles**. `re_ohm`
  and `rct_ohm` (electrolyte and charge-transfer resistance) only on
  impedance cycles, which run every few cycles, not every cycle. There is
  no per-cycle "internal resistance" column.
- `timeseries`: per-sample V/I/T with unit-suffixed names
  (`voltage_measured_v`, `current_measured_a`, `temperature_measured_c`,
  …). Charge cycles fill `*_charge_*`, discharge cycles `*_load_*`.
- `start_time` is lab local time with no time zone.
- 38 files, 34 batteries: **B0025–B0028 appear twice** with identical
  sizes; keep one copy in silver and log which.
- Test conditions differ between battery groups (ambient temperature,
  discharge current/profile, cut-off voltage), and **within** some
  batteries: B0038–B0044 switch between 4/22/24/44 °C and 1/2/4 A loads.
  Capacity measured at 4 °C or 4 A is much lower without the cell being
  more aged, so only cycles at the same conditions are comparable.
- NASA defines end of life against the **rated 2.0 Ah**: 1.4 Ah (30 %
  fade) for B0005–B0018, B0041–B0048, B0053–B0056; 1.6 Ah (20 % fade) for
  B0033–B0040. B0025–B0032 and B0049–B0052 have none (49–52 ended when
  the test software crashed).
- Bad discharges are common outside the first group: capacities of
  0.0, ~0.05 Ah, or null (B0052 has a value for 4 of 25), early cycles far
  below the rest (B0033 starts 0.07, 0.69, 1.16 Ah), and values above
  rated (B0036 2.44 Ah, B0049–B0051 up to 2.64 Ah). NASA's README: "several
  discharge runs where the capacity was very low. Reasons … not fully
  analyzed."
- Clean single-condition runs: B0005/6/7/18 (132–168 discharges), B0036
  (197), B0025–B0028 (28 each), B0029–B0032 (40 each, 43 °C).
- Capacity can rise for a few cycles (recovery after rest periods). This
  is physical behaviour, not bad data.

Checked against the data on 2026-10-09 (bronze rebuilt locally from the
NASA zip, plus the per-group README files in `raw/`).

## Work

### BT-102 Silver (`bdt_pipeline`)

- New DAG (or tasks downstream of ingest) reading one bronze
  `ingest_date` and writing `silver/nasa_pcoe_battery/ingest_date=<date>/`.
- Tables: `cycles` (deduplicated, typed, one row per battery × cycle) and
  `timeseries` (per-sample, joined to cycle keys). Keep impedance as its
  own table.
- Data-quality checks as a separate, testable module, run as their own
  task so failures show in Airflow. Classify findings as invalid /
  missing / outlier / expected behaviour; never drop rows silently, log
  counts of rejected rows with reasons.
- Log per run: batteries, cycles, rows in, rows out, rows rejected.

### BT-103 Gold SOH table (`bdt_pipeline`)

- `gold/nasa_pcoe_battery/soh/`: one row per battery × discharge cycle with
  `battery_id, discharge_index, cycle_index, capacity_ah, soh,
  ambient_temperature_c, mean_current_a, test_group, capacity_flag,
  soh_valid`, plus a per-battery table (test group, conditions, EOL
  threshold, counts, latest valid SOH). All batteries, flagged per D7.
- SOH definition lives in one function with its parameters explicit (see
  decision D1); not inside any later model code.
- Write the data dictionary to `bdt_proj/docs/data-dictionary.md`:
  every column's meaning, units, source columns and calculation.

### BT-104 Backend serves NASA data (`bdt`)

- Extend `data_store.py` with a NASA source reading the gold table (see
  D3 for where from). Keep the synthetic vehicle data working.
- Routes per decision D2. Pydantic models in `schemas.py`, auth as the
  existing routes. Tests for: list, one battery, SOH history, unknown id
  → 404, no data available → clear error.

### BT-105 Dashboard (`bdt`)

- Battery list and a Battery Twin card: battery id, cycle count, initial
  and latest capacity, latest SOH, SOH bar, trend (%/cycle over the last
  N cycles), SOH-vs-cycle chart with the EOL threshold where it applies.
- Reuse the existing chart components and theme. `npm test` and
  `npm run build` must pass.

## Decisions (agreed 2026-10-07, D1 revised and D7 added 2026-10-09)

- **D1 SOH basis:** `SOH = capacity_ah / 2.0 Ah` (rated capacity), as a
  parameter of the SOH function. Matches NASA's EOL definitions (1.4 Ah =
  70 %, 1.6 Ah = 80 %); new cells start below 100 % (B0005 ≈ 93 %).
  *Originally "mean of the first 3 discharges"; dropped because bogus
  early discharges give SOH far above 100 % for about half the batteries.*
- **D2 API shape:** new `/api/batteries/...` routes beside
  `/api/vehicles`, which stay as they are. NASA data doesn't fit the
  vehicle shape (no odometer, trips or charging sessions).
- **D3 Gold data access:** the backend reads the gold table from MinIO
  with read-only credentials scoped to `gold/`. The credentials are a
  Kubernetes secret created out of band (never in Git); `bdt-gitops`
  only references it. Bucket/prefix/endpoint are config. Creating the
  MinIO user and policy is a manual step on odin; document it in
  `bdt-infra/docs/minio.md`, and ask the owner before running it.
- **D7 Scope:** all batteries, with quality flags. Each discharge cycle
  is flagged (ok, missing, zero, very low, above rated, inconsistent,
  off the battery's reference condition); charts plot valid cycles only,
  and each battery shows whether it's a clean single-condition run.

## Done when

- [x] Silver and gold DAG runs succeed in Airflow on the current bronze
  snapshot; data-quality task output visible; re-run gives identical output.
  *Done in the `apache/airflow:3.2.2` image against a local Silo/MinIO
  loaded with bronze rebuilt from the NASA zip (2026-10-09): all 37 tasks
  succeeded twice, outputs identical. Cluster run 2026-10-09 on
  `ingest_date=2026-10-07`: 37/37 tasks succeeded.*
- [x] `uv run pytest` passes in `bdt_pipeline` (24) and `bdt/backend` (20);
  new logic has unit tests (SOH calculation, dedup, quality checks).
  `npm test` (14) and `npm run build` pass in `bdt/frontend`.
- [x] API returns real NASA SOH for every battery; verified with real
  requests against a locally running backend. *Done (34 batteries, over
  S3 with the read-only policy); auth bypassed locally since login needs
  Keycloak/Google.*
- [~] Dashboard shows it; verified in a browser locally. *Headless Chromium
  renders of B0005, B0042, B0049 with real data; Keycloak stubbed.*
- [x] Deployed to **dev only** after the owner approves the `bdt-gitops`
  change (D3 adds config there). *dev runs cf19aef with the read-only
  key (now synced from Vault); qa/prod unchanged.*
- [x] Data dictionary written; backlog items BT-102–BT-105 updated.
