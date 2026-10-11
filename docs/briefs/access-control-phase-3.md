# Access control brief: authorization phase 3

Backlog items: **AZ-301 – AZ-303** (section "Access control" in
[`BACKLOG.md`](../../BACKLOG.md)); AZ-302 also closes **PL-3**. Builds on
[phases 1–2](access-control.md). Read [`AGENTS.md`](../../AGENTS.md) first.

## Goal

Know who changed access and who was refused, and stop sharing powerful
credentials: each workload gets its own key with only what it needs, kept
in Vault.

## Decisions (agreed 2026-10-11)

- **D1 Append-only audit log:** a database trigger rejects `UPDATE`,
  `DELETE` and `TRUNCATE` on `audit_events`. The backend keeps its one DB
  role. This stops app bugs and mistakes; a DB owner could still drop the
  trigger. (A separate runtime role with insert-only rights is the
  stronger option, left for later.)
- **D2 What is recorded:** every membership change (add, role change,
  remove), and refused requests from **signed-in** users: non-member or
  unknown project (404), role too low (403), not a platform admin (403),
  no BDT platform role (403). Requests with a missing or invalid token
  (401) go only to the server log, so anonymous traffic can't grow the
  table. Dataset deletion and model deployment are recorded once those
  features exist.
- **D3 Who reads it:** platform admins see all events; project admins see
  their project's events. Shown in the dashboard's Admin tab.
- **D4 Retention:** keep everything for now (low volume); revisit if the
  table grows.
- **D5 Airflow credentials (PL-3):** each Airflow connection is stored (as
  Airflow's JSON form) in Vault at `secret/k8s/airflow/<name>`, synced by External Secrets like
  the other Airflow secrets and given to Airflow as an `AIRFLOW_CONN_<ID>`
  environment variable from the Helm values. No connection is kept in
  Airflow's metadata DB, and Airflow never logs in to Vault itself.
- **D6 One MinIO key per DAG**, replacing the shared `airflow` key
  (`s3:*` on all of `bdt-lake`):

  | Key / connection | May | Prefixes in `bdt-lake` |
  |---|---|---|
  | `nasa-ingest` / `minio_nasa_ingest` | list bucket; read, write, delete | `landing/nasa_pcoe_battery/`, `raw/nasa_pcoe_battery/`, `bronze/nasa_pcoe_battery/` |
  | `nasa-transform` / `minio_nasa_transform` | list bucket; read | `bronze/nasa_pcoe_battery/` |
  | | read, write, delete | `silver/nasa_pcoe_battery/`, `gold/nasa_pcoe_battery/` |
  | `bdt-dashboard` (exists) | read | `gold/` |

  Neither DAG key can delete the bucket, change policies or touch the
  other DAG's output. The `airflow` key and the `minio_s3` connection are
  removed once both DAGs have run green with the new keys.

## Plan

### AZ-301 Audit events (`bdt`)

- Migration `0002`: table `audit_events` (`id`, `occurred_at`,
  `actor_user_id`, `actor_sub`, `actor_email`, `action`, `project_key`,
  `target`, `outcome`, `detail` JSON) with indexes on `occurred_at` and
  `project_key`, plus the append-only trigger (Postgres; SQLite tests use
  an equivalent trigger).
- `app/audit.py`: `record(...)`. Refusals are written in their own
  transaction, because the request's session is rolled back when it
  fails.
- Calls from `permissions.py` (404/403), `auth.py` (no platform role) and
  `routers/members.py` (add, role change, remove).
- `GET /api/projects/{key}/audit` (project admin) and `GET
  /api/admin/audit` (platform admin), newest first, paged.
- Admin tab: an **Activity** section.
- Tests: each event type is written; refusals are written even though the
  request fails; updates and deletes are rejected; who can read what.

### AZ-302 Pipeline workload identity (`bdt-infra`, `bdt-gitops`, `bdt_pipeline`)

1. `bdt-infra/scripts/setup-pipeline-keys.sh` (run by the owner on odin):
   creates the two MinIO users and policies from D6, checks each key can
   do only what the table says, and stores each connection URI in Vault.
2. `bdt-gitops`: ExternalSecrets for the two connections in
   `platform/airflow`, and `AIRFLOW_CONN_MINIO_NASA_INGEST` /
   `AIRFLOW_CONN_MINIO_NASA_TRANSFORM` env vars in the Airflow Helm values
   (needs owner approval: change to `main` of bdt-gitops).
3. `bdt_pipeline`: each DAG uses its own connection id; README setup
   section updated.
4. Run both DAGs; then remove the `airflow` MinIO user, the `bdt-lake-rw`
   policy and the `minio_s3` connection.

### AZ-303 Least-privilege review (`bdt-infra`)

An inventory in `bdt-infra/docs/service-accounts.md` of every credential a
workload uses (MinIO keys, Keycloak clients, Postgres roles, JFrog pull
credentials): who uses it, what it may do, where it's stored. Anything
broader than needed gets a backlog item or is fixed in place.

## Done when

- [x] AZ-301: membership changes and refusals appear in the Activity
      section in dev; an update or delete on `audit_events` fails.
- [x] AZ-302: both DAGs run green with their own keys; each key is
      refused outside its prefixes; `airflow` key and `minio_s3`
      connection removed.
- [ ] AZ-303: inventory written; every credential is per workload, minimal
      and in Vault, or has a backlog item.
- [ ] Platform map updated with each change.
