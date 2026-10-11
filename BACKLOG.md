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

Agents work from the phase briefs in [`docs/briefs/`](docs/briefs/) under the
rules in [`AGENTS.md`](AGENTS.md).

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
- [x] **BT-102 Silver layer.** Clean and deduplicate bronze (B0025–B0028
  appear twice in the archive), normalise units/types, one row per battery
  per cycle plus the per-cycle timeseries. Write to `silver/`.
  *`nasa_battery_transform` DAG plus data-quality flags. Ran on the
  cluster 2026-10-09 on `ingest_date=2026-10-07`: 37/37 tasks succeeded.*
- [x] **BT-103 SOH ground truth.** From the discharge capacity per cycle,
  compute `SOH = capacity / rated capacity (2.0 Ah)` (decision D1).
  Output: `gold/…/soh` and `gold/…/batteries`, see
  [data dictionary](docs/data-dictionary.md).
  - Acceptance: EOL derivable at NASA's thresholds (1.4 Ah = 70 %,
    1.6 Ah = 80 %). *Revised: curves start at measured/rated (B0005 ≈ 93 %),
    not ~1.0.* Gold written on the cluster with BT-102's run.
- [x] **BT-104 Backend reads NASA data.** `/api/batteries` routes in `bdt`
  reading the gold tables from MinIO with a read-only key. *Deployed to
  dev (cf19aef); the dev pod serves 34 batteries from the cluster's gold
  snapshot; unauthenticated requests get 403.*
- [~] **BT-105 Battery Twin card in the dashboard.** Per battery: current
  SOH, SOH bar, trend (%/cycle over the last N cycles), degradation
  history chart. *"NASA test cells" view, deployed to dev (cf19aef).
  Checked in headless renders locally; a logged-in check in dev is still
  pending. "Remaining life" waits for Phase 3 (RUL).*

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
- [ ] **BT-204 Stronger SOH model.** Interpretable first (random forest /
  gradient boosting) and compared against the baseline. Deep learning
  (1D CNN / LSTM on the raw curves) only later, if the simpler model
  falls short.
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

## Platform

- [x] **PL-1 Secrets in Vault.** Hand-made Kubernetes Secrets (15) stored
  in Vault and synced by External Secrets Operator (`bdt-gitops` README,
  `bdt-infra/docs/vault-secrets.md`). Done 2026-10-09: fingerprints
  unchanged, no pod restarts.
- [ ] **PL-2 Vault backups.** Vault (file storage, local-path PVC on odin)
  is now the source of truth for those Secrets and has no backup. Move to
  integrated storage (Raft) and take nightly `vault operator raft snapshot`
  to MinIO or thor; test a restore.
- [x] **PL-3 Airflow connections from Vault.** The `minio_s3` connection
  (read-write MinIO key) lives in Airflow's DB; use Airflow's Vault
  secrets backend instead. Done with AZ-302 another way (decision D5):
  connections are kept in Vault and given to Airflow as `AIRFLOW_CONN_*`
  env vars through External Secrets; `minio_s3` removed 2026-10-11.

## Access control (authorization)

Goal: a valid login no longer means access to everything. Keycloak says
*who* the user is; the `bdt` backend decides *what* they may do, using
project memberships stored in Postgres. All data access stays behind the
backend; the browser never gets S3 credentials.

Two levels:

- **Platform roles in Keycloak** (realm `bdt`): `bdt-platform-admin`,
  `bdt-user`, `bdt-service-account` (only if needed). Platform admin is
  kept separate from project admin.
- **Project roles in Postgres:** `viewer`, `engineer`, `project_admin`
  per user per project. Project memberships are not Keycloak roles.

Proposed policy (adjust as needed):

| Action | viewer | engineer | project_admin |
|---|:-:|:-:|:-:|
| View battery datasets | ✓ | ✓ | ✓ |
| Run analysis | – | ✓ | ✓ |
| Upload a dataset | – | ✓ | ✓ |
| Delete a dataset | – | – | ✓ |
| Manage project members | – | – | ✓ |

Today (2026-10-10): `bdt/backend/app/auth.py` checks signature, issuer
and expiry but not audience (`verify_aud: False`, public client
`bdt-frontend` has no audience mapper), and reads no roles. There is no
app database yet; `bdt-gitops/platform/postgres` is a general-purpose
instance.

Decided (2026-10-10):

- **Database:** a `bdt` database on the shared `postgres` instance
  (`bdt-gitops/platform/postgres`, ns `postgres`), with its own login role.
- **Projects:** NASA PCoE data (`/api/batteries`) is one project, the
  synthetic vehicle fleet (`/api/vehicles`) is another.
- **Migrations:** SQLAlchemy + Alembic in `bdt` (installed with `uv`);
  `alembic upgrade head` runs before the backend starts (init container
  or Job).
- **Denied access:** `404` for a project the user is not a member of
  (doesn't reveal it exists); `403` for a member whose role is too low.
- **Routes:** project-scoped routes `/api/projects/{key}/...` replace
  `/api/batteries` and `/api/vehicles`; the old routes are removed in the
  same change as the dashboard switch (home lab, no other consumers).

### AZ phase 1: minimum viable authorization

Brief: [access control](docs/briefs/access-control.md). AZ-101 – AZ-104,
AZ-201 – AZ-203 and AZ-205 are deployed to dev (47f061a, 2026-10-10);
logged-in check done: the platform admin sees both projects and their
data, and their bdt_users row was created on first login. Non-admin
check done 2026-10-11 (see AZ-204). qa/prod not promoted.

- [x] **AZ-101 API audience.** Add a `bdt-api` audience (client or
  mapper) in realm `bdt` so frontend tokens carry `aud: bdt-api`; turn
  audience verification on in `auth.py`.
- [x] **AZ-102 Platform roles.** Create `bdt-platform-admin` and
  `bdt-user` in the realm, map them into the token, and expose them on the
  current user in the backend. New users get `bdt-user` by default.
- [x] **AZ-103 BDT database.** Create the `bdt` database and login role
  on the shared `postgres`, with credentials (Vault → External Secrets),
  wire the backend to it via SQLAlchemy, add Alembic migrations run
  before the backend starts. Tables: `bdt_users` (unique `keycloak_issuer` +
  `keycloak_sub`), `projects` (`project_key`, `name`),
  `project_memberships` (`user_id`, `project_id`, `role` ∈ viewer /
  engineer / project_admin). No passwords or identity data beyond what
  is needed.
- [x] **AZ-104 Resolve the current user.** `get_current_user` maps the
  validated token's `iss` + `sub` (never email or username) to a
  `bdt_users` row, creating it on first login.

### AZ phase 2: project-level access

- [x] **AZ-201 Projects for existing data.** Create the two first
  projects: NASA PCoE (the `nasa_pcoe_battery` gold data) and the
  synthetic vehicle fleet. Move their routes to
  `/api/projects/{key}/...`, switch the dashboard, and delete the old
  `/api/batteries` and `/api/vehicles` routes.
- [x] **AZ-202 Enforce on every protected route.** A
  `require_project_role(...)` dependency on all project-scoped routes;
  identity only from the token, never from the request. Non-member →
  404, member with too low a role → 403. Platform admins pass every
  project check.
- [x] **AZ-203 Allowed/denied tests.** For each route and role, a test
  that the permitted role gets through, lower roles get 403 and
  non-members get 404, plus no membership, expired token, wrong issuer, wrong audience.
- [x] **AZ-204 Membership management.** API for a project admin to add,
  change and remove members of their project; platform admin manages all
  projects.
  *API `/api/projects/{key}/members` (+ `/api/admin/users` for platform
  admins) and an Admin tab in the dashboard. Deployed to dev (af47230,
  2026-10-10). Checked live 2026-10-11: the platform admin added a second
  account as viewer of NASA PCoE from the Admin tab; that account then saw
  only the NASA tab and no Admin tab.*
- [x] **AZ-205 Dashboard follows permissions.** Project switcher shows
  only the user's projects; actions the role doesn't allow are hidden.
  Usability only, the backend stays the security boundary.
- [x] **AZ-206 Account page.** Every signed-in user can see their
  Keycloak identity (token claims, realm roles, sign-in and session
  expiry) and their BDT access (platform roles, projects, role and where
  it comes from).
  *Dashboard `#account` and a richer `/api/me` (bdt #4). Deployed to dev
  as 78fcce4 and checked with both accounts, 2026-10-11.*

### AZ phase 3: data and operational security

Brief: [access control phase 3](docs/briefs/access-control-phase-3.md),
decisions D1–D6 agreed 2026-10-11.

- [x] **AZ-301 Audit events.** `audit_events` table (actor, action,
  resource, time, outcome) written for membership changes, dataset
  deletion, model deployment and denied access; append-only for the app
  role. Done for membership changes and refusals (dataset deletion and
  model deployment once those routes exist); Activity section in the Admin
  tab. Verified in dev (f6fd31a) 2026-10-11.
- [x] **AZ-302 Pipeline workload identity.** Airflow uses its own
  credentials scoped to the buckets/prefixes each DAG needs, never a human
  user's token. Goes with PL-3. MinIO keys `nasa-ingest` and
  `nasa-transform` (bdt-infra `scripts/setup-pipeline-keys.sh`), one
  Airflow connection per DAG from Vault; old `airflow` key, `bdt-lake-rw`
  policy and `minio_s3` removed. Both DAGs ran green 2026-10-11.
- [ ] **AZ-303 Least-privilege service accounts.** Review every MinIO key
  and Keycloak client: one per workload, minimal policy, stored in Vault.
  Inventory: bdt-infra `docs/service-accounts.md` (bdt-infra PR #4).
  Everything broader than needed, or still unknown, is AZ-304 – AZ-311;
  done once that inventory is merged.

#### AZ-303 follow-ups (from the inventory, 2026-10-11)

Findings (F) and open questions (Q) refer to `docs/service-accounts.md`
in bdt-infra; update its table when an item is done.

- [ ] **AZ-304 One dashboard MinIO key per environment (F1).**
  `bdt-dashboard` is a single user; `setup-nasa-gold-key.sh <namespace>`
  recreates it, so qa/prod would share dev's key and a rotation breaks the
  others. Make it `bdt-dashboard-<env>`. Before qa/prod get the NASA key.
- [ ] **AZ-305 Separate runtime database role for the backend (F2).**
  The backend runs as `bdt_<env>`, the database owner, so it could drop
  the audit trigger. Add `bdt_<env>_app` with `SELECT, INSERT, UPDATE,
  DELETE` (only `SELECT, INSERT` on `audit_events`) for the backend; keep
  the owner for the `migrate` init container. Script in bdt-infra, Secret
  in Vault, wiring in bdt-gitops `components/bdt-db`. (Decision D1's
  stronger option.)
- [ ] **AZ-306 Non-superuser Postgres logins for Airflow, Keycloak, JFrog
  (F3).** Each uses its instance's superuser; each instance holds only its
  own database. Low priority.
- [ ] **AZ-307 Delete `argocd-initial-admin-secret` (F4).** After checking
  the Argo CD admin password was changed (or SSO is used):
  `kubectl -n argocd delete secret argocd-initial-admin-secret`.
- [ ] **AZ-308 Scoped Vault token for the setup scripts (F5).** A Vault
  policy that can write `secret/k8s/*` only, and a token from it for
  `setup-*.sh`; keep root for `vault-eso-setup.sh` and emergencies.
- [ ] **AZ-309 Registry credentials (Q1, Q2).** Find which JFrog user is
  behind `jfrog-pull-secret` (dev, qa, prod) and behind CI's
  `JFROG_USERNAME`, and their permissions (Administration → User
  Management). Target: `k8s-pull` with Read on `battery-docker-local`
  only, in Vault for the three environments; a separate CI user with
  Deploy on that repo only. A push test with the pull login must be
  refused.
- [ ] **AZ-310 Git credentials scope (Q3, Q4, Q5).** `GITOPS_REPO_TOKEN`
  should be a fine-grained token for `rdcsung/bdt-gitops` (contents
  read/write) only; Argo CD's `gitops-repo-creds` and the bdt_pipeline
  deploy key (`airflow-git-ssh`) should be read-only. Check in GitHub
  (Settings → Developer settings → tokens; each repo's Settings → Deploy
  keys) and replace any that are broader.
- [ ] **AZ-311 Keycloak bootstrap admin (Q6).** Check the first-boot admin
  from `keycloak-admin-creds` was disabled or deleted after the named
  admin was created (master realm → Users); if not, disable it.

### AZ phase 4: only when a real use case needs it

- [ ] **AZ-401 Finer-grained access.** Dataset-level permissions,
  temporary access, approval workflows or attribute-based policies; S3
  bucket policies or catalog authorization (if Iceberg is adopted) as a
  second enforcement layer.

## Ideas / later

- Other data sets beside NASA under the same `bdt-lake` layout (other PCoE
  sets, own cell tests).
- Fleet view: rank batteries by RUL / anomaly score.
- Link to a longer-term battery verification agent.
