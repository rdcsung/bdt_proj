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
  discharge current, cut-off voltage). Several groups stop before reaching
  end of life; some have only a few dozen cycles. The 1.4 Ah (30 % fade)
  end-of-life threshold in NASA's README applies to the first group
  (B0005–B0018), not to all.
- Capacity can rise for a few cycles (recovery after rest periods). This
  is physical behaviour, not bad data.

Verify each of these against the data; correct this brief if any are wrong.

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
  `battery_id, discharge_index, cycle_index, capacity_ah,
  initial_capacity_ah, soh, ambient_temperature_c, test_group`.
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

## Decisions needed (ask before implementing)

- **D1 Initial capacity.** First discharge cycle, mean of the first N, or
  rated 2.0 Ah? Proposal: mean of the first 3 discharge cycles, stored
  as a parameter.
- **D2 API shape.** New `/api/batteries/...` routes beside
  `/api/vehicles`, or map NASA batteries onto the vehicle model?
  Proposal: new routes; the data doesn't fit the vehicle shape
  (no odometer, trips or charging sessions).
- **D3 How the backend gets gold data.** Read from MinIO at startup with
  a read-only key (new secret in `bdt-gitops`), or export a snapshot into
  the image at build time? Proposal: MinIO, read-only credentials scoped to
  `gold/`.

## Done when

- [ ] Silver and gold DAG runs succeed in Airflow on the current bronze
  snapshot; data-quality task output visible; re-run gives identical output.
- [ ] `uv run pytest` passes in `bdt_pipeline` and `bdt/backend`; new
  logic has unit tests (SOH calculation, dedup, quality checks).
- [ ] API returns real NASA SOH for every battery; verified with real
  requests against a locally running backend.
- [ ] Dashboard shows it; verified in a browser locally.
- [ ] Deployed to **dev only** after the owner approves the `bdt-gitops`
  change (D3 adds config there).
- [ ] Data dictionary written; backlog items BT-102–BT-105 updated.
