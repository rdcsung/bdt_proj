# Agent instructions: Battery Digital Twin (BDT)

Standing rules for any AI agent working on the BDT repos. They apply to
every task. The work itself is described in a **phase brief** under
[`docs/briefs/`](docs/briefs/); the product backlog is
[`BACKLOG.md`](BACKLOG.md). Always work from one brief at a time.

## The repos

All live under `github.com/rdcsung/`, checked out side by side.

| Repo | Owns | Does not own |
|---|---|---|
| `bdt_pipeline` | Airflow DAGs; bronze/silver/gold layers in MinIO; feature engineering; model training and evaluation; data schemas; pipeline tests | Serving, UI, cluster config |
| `bdt` | FastAPI backend (routers + `data_store.py`, Keycloak auth); React/Vite dashboard; model inference; API and frontend tests | Training, data transformation |
| `bdt-gitops` | Argo CD Applications; Kustomize `base/` + `overlays/{dev,qa,prod}` for the app; `platform/` (Airflow via the official chart, JFrog, Keycloak, Postgres, Vault) | Anything that isn't a manifest |
| `bdt-infra` | Ansible for the k3s nodes; MinIO on odin; the full rebuild runbook | Anything running inside the cluster |
| `bdt_proj` | Backlog, phase briefs, architecture decisions (ADRs), data dictionary, these instructions | Code |

Don't move a responsibility to another repo because it's easier there. If
a task seems to need that, stop and ask.

## What already exists (don't rebuild it)

- **Bronze is done.** `bdt_pipeline`'s `nasa_battery_ingest` DAG downloads
  the NASA PCoE battery zip, records a manifest (URL, ETag, SHA-256,
  time), unzips to `raw/`, and converts every `.mat` to three Parquet
  tables in `s3://bdt-lake/bronze/nasa_pcoe_battery/ingest_date=<date>/`:
  `cycles/`, `timeseries/`, `impedance/`. Schemas are in
  `dags/nasa_battery/convert.py`. Silver and gold don't exist yet.
- **The app runs on synthetic data.** `bdt/backend/app/data_store.py`
  serves vehicle-level tables (`vehicles`, `soh`, `driving`, `charging`)
  from CSVs or a generator; routes are `/api/vehicles/...`.
- **CI/CD:** `bdt` builds and tests on GitHub Actions (`uv run pytest`,
  `npm test`, `npm run build`), pushes images to JFrog, and promotes a
  built tag to an environment by committing to `bdt-gitops`
  (`promote.yml`). Argo CD syncs from there.

Read the README of every repo you touch before changing it.

## Conventions to follow

- **Python:** `uv` with `pyproject.toml` + `uv.lock`. Don't add another
  package manager.
- **Airflow:** Airflow 3 TaskFlow (`airflow.sdk`), stock
  `apache/airflow:3.2.2` image, DAGs deployed by git-sync from
  `bdt_pipeline`. `bdt_pipeline`'s dev dependencies are pinned to what the
  image ships; anything not in the image is a deployment decision, not a
  `pyproject.toml` edit.
- **DAGs stay thin.** Logic goes in importable modules under `dags/`
  (like `nasa_battery/convert.py`) with unit tests; the DAG file only wires
  tasks.
- **Storage:** MinIO via the Airflow connection `minio_s3`, bucket
  `bdt-lake`, layout `<layer>/<dataset>/ingest_date=<date>/...`. Every task
  is idempotent: re-running a date replaces that date's output.
- **Config, not constants:** bucket, prefixes, dataset version and model
  location come from params/config, never hard-coded paths.
- **Secrets** never go in Git: no passwords, keys, tokens or credentials,
  in code, manifests or docs. Use the existing Airflow connections,
  Kubernetes secrets created out of band, or Vault.

## Honesty rules

These matter more than finishing.

- Never fabricate data, metrics, test results or deployment status.
- Never swap in synthetic data to make something pass.
- Run what you claim works: tests, the API, the DAG. Report real output.
- A model that trains is not a model that works. Look at the metrics
  per battery before calling it done; weak results are fine if reported.
- If something fails or is skipped, say what failed, why, what you tried,
  and what's left.

## When to stop and ask

Stop and ask the owner (Gordon) before:

- making any decision a brief lists under **Decisions needed**;
- pushing to `bdt-gitops` (Argo CD deploys it to the live cluster) or
  promoting to qa/prod;
- changing `bdt-infra` or anything on the cluster nodes;
- changing an existing API contract the dashboard depends on;
- deleting data in MinIO outside the date you're re-running.

Pushing branches and opening PRs on `bdt_pipeline`, `bdt` and `bdt_proj`
is fine.

## Keep the platform map current

[`docs/architecture.html`](docs/architecture.html) is the interactive map
of every component, host, URL, Secret, Vault path and repo relation. Its
facts live in the `MODEL` block at the top of its script.

- Any change that adds, removes or reconfigures a component, host, URL,
  port, Secret or Vault path, pipeline step, or repo relation **updates
  `MODEL` in the same change** (or the matching `bdt_proj` PR if the
  change is in another repo).
- Bump `MODEL.updated` and add a `MODEL.changelog` line.
- Facts only, checked against the repos or the cluster. Never secret
  values.

## Git

- Small commits, one logical change each, Conventional Commit style:
  `feat(pipeline): add silver battery cycles`,
  `test(api): cover unknown battery id`, `docs: add SOH data dictionary`.
- Don't reformat or refactor unrelated code.
- Mark a `BACKLOG.md` item `[x]` only once it has been run and verified.

## Reporting back

End each brief with a short report: what changed per repo, commands run
and their real results (tests, metrics), what's deployed where, open
issues, and suggested next step. Update the brief's checklist and the
backlog in `bdt_proj` in the same change.
