# Data dictionary: NASA battery silver and gold

Tables written by `bdt_pipeline`'s `nasa_battery_transform` DAG under
`s3://bdt-lake/{silver,gold}/nasa_pcoe_battery/ingest_date=<date>/`.
Bronze columns are documented in the
[`bdt_pipeline` README](https://github.com/rdcsung/bdt_pipeline#bronze-tables).
Code: `dags/nasa_battery/{metadata,silver,quality,gold}.py`.

Conventions: units are in the column name suffix (`_ah`, `_a`, `_v`,
`_c`, `_s`, `_ohm`); `start_time` is lab local time with no time zone, as
in the source; "discharge only" columns are null on other cycle types.

## Constants

| Name | Value | Source |
|---|---|---|
| Rated capacity | 2.0 Ah | Every NASA README measures fade against 2 Ah |
| EOL capacity | 1.4 Ah (B0005–B0018, B0041–B0048, B0053–B0056); 1.6 Ah (B0033–B0040); none (B0025–B0032, B0049–B0052) | Per-group READMEs in the archive |
| Cut-off voltage, discharge profile, ambient | per battery | Per-group READMEs → `metadata.py` |

## silver/cycles

One row per battery × cycle (charge, discharge, impedance). All bronze
`cycles` columns, with one copy per battery, plus:

| Column | Type | Unit | Definition | Source |
|---|---|---|---|---|
| `test_group` | string | | NASA test group, e.g. `B0005-B0018` | `metadata.py` |
| `duration_s` | double | s | Last minus first `time_s` of the discharge | `timeseries.time_s` |
| `integrated_ah` | double | Ah | Charge removed: `-∫ current_measured_a dt / 3600` (trapezoid) | `timeseries.current_measured_a`, `time_s` |
| `mean_current_a` | double | A | `integrated_ah × 3600 / duration_s`: average discharge current, including the off phases of pulsed profiles | derived |
| `min_voltage_v` | double | V | Lowest terminal voltage during the discharge | `timeseries.voltage_measured_v` |
| `max_temperature_c` | double | °C | Highest cell temperature during the discharge | `timeseries.temperature_measured_c` |

## silver/timeseries, silver/impedance

Bronze tables unchanged, one file per battery (`<battery_id>.parquet`).

## silver/cycle_quality

One row per discharge.

| Column | Type | Unit | Definition |
|---|---|---|---|
| `battery_id`, `cycle_index` | | | Key, joins to `silver/cycles` |
| `discharge_index` | int32 | | n-th discharge of the battery, 0-based (`type_index`) |
| `ambient_temperature_c` | double | °C | From `cycles` |
| `load_current_a` | int32 | A | `mean_current_a` rounded to whole amps |
| `condition` | string | | `"<ambient>C/<load>A"`, e.g. `24C/2A` |
| `is_reference_condition` | bool | | `condition` equals the battery's reference condition: the most common condition among discharges whose capacity passes the value checks (ties: first seen) |
| `capacity_flag` | string | | First matching rule below |

`capacity_flag` rules, in order (thresholds in `QualityConfig`):

| Flag | Meaning | Meant as |
|---|---|---|
| `missing` | no capacity recorded | missing data |
| `zero` | capacity ≤ 0 (NASA stores integer `0` placeholders) | missing data |
| `very_low` | capacity < 0.25 × rated | outlier (NASA: cause not analysed) |
| `above_rated` | capacity > 1.10 × rated | outlier |
| `inconsistent` | `integrated_ah / capacity_ah` < 0.8 or > 2.0 | invalid measurement |
| `off_reference` | not at the battery's reference condition | expected physical behaviour, not comparable |
| `outlier` | > 15 % from the median of up to 4 neighbouring reference-condition discharges (at least 2) | outlier |
| `ok` | none of the above | valid |

Structural problems (null keys, duplicate `(battery_id, cycle_index)`,
unknown `cycle_type`, negative `n_samples`/`duration_s`) are not flags:
they fail the run.

`silver/_quality.json`: run summary with the config, batteries, cycles by
type, discharges, flag counts overall and per battery.

## gold/soh

One row per discharge, ordered by `battery_id`, `discharge_index`.

| Column | Type | Unit | Definition |
|---|---|---|---|
| `battery_id` | string | | |
| `discharge_index` | int32 | | n-th discharge, 0-based |
| `cycle_index` | int32 | | Position among all the battery's cycles |
| `start_time` | timestamp(ms) | | Discharge start, lab local time |
| `capacity_ah` | double | Ah | As recorded by NASA (discharge to 2.7 V) |
| `soh` | double | ratio | `capacity_ah / rated capacity (2.0 Ah)`; null without a capacity. Can exceed 1. |
| `ambient_temperature_c` | double | °C | |
| `mean_current_a` | double | A | See `silver/cycles` |
| `condition` | string | | See `silver/cycle_quality` |
| `test_group` | string | | |
| `capacity_flag` | string | | See `silver/cycle_quality` |
| `soh_valid` | bool | | `capacity_flag == "ok"`. Charts and trends use only these rows. |

## gold/batteries

One row per battery.

| Column | Type | Unit | Definition |
|---|---|---|---|
| `battery_id`, `test_group` | string | | |
| `ambient`, `discharge_profile` | string | | Test conditions as NASA describes them |
| `cutoff_voltage_v` | double | V | Discharge cut-off for this battery |
| `rated_capacity_ah` | double | Ah | 2.0 |
| `eol_capacity_ah` | double | Ah | NASA's EOL threshold; null if the test had none |
| `eol_soh` | double | ratio | `eol_capacity_ah / rated_capacity_ah` |
| `reference_condition` | string | | Most common good condition, e.g. `24C/2A` |
| `single_condition` | bool | | Every discharge that passed the value checks (`ok`, `off_reference`, `outlier`) ran at one condition |
| `n_discharges`, `n_valid` | int32 | | All discharges; those with `soh_valid` |
| `first_valid_soh`, `latest_valid_soh` | double | ratio | SOH of the first / last valid discharge |
| `latest_valid_discharge_index` | int32 | | |
| `reached_eol` | bool | | Any valid SOH ≤ `eol_soh`; null without an EOL threshold or valid rows |

`gold/_latest.json`: `{"ingest_date", "prefix", "written_at"}` of the most
recent successful run. Readers should resolve the snapshot through it.

## Known limitations

- SOH at 4 °C or 43–44 °C isn't comparable with SOH at 24 °C: cold cells
  deliver less capacity without being more aged. B0053 starts at SOH ≈ 0.53
  and so "reaches EOL" from its first discharge. NASA's 1.4 Ah criterion
  for the 4 °C groups has the same problem.
- `load_current_a` is an average, so pulsed profiles show the mean (the
  4 A square wave of B0025–B0028 averages ≈ 1 A).
- The flag thresholds were chosen by inspecting this data set (2026-10-09)
  and aren't validated beyond it.
