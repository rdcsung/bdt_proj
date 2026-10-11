# Access control brief: authorization phases 1–2

Backlog items: **AZ-101 – AZ-104, AZ-201 – AZ-203, AZ-205** (section
"Access control" in [`BACKLOG.md`](../../BACKLOG.md)). Read
[`AGENTS.md`](../../AGENTS.md) first.

## Goal

A valid login no longer means access to everything. Keycloak says who the
user is; the `bdt` backend decides what they may do from project
memberships in the BDT database.

## Decisions (agreed 2026-10-10)

- **Database:** one database per environment, `bdt_dev`, `bdt_qa`,
  `bdt_prod`, on the shared `postgres` (`bdt-gitops/platform/postgres`),
  each owned by a role of the same name. (Per environment so dev can run
  a newer schema than prod.)
- **Migrations:** SQLAlchemy + Alembic in `bdt`; an init container runs
  `alembic upgrade head` before the backend starts.
- **Projects:** `nasa-pcoe` (NASA PCoE gold data) and `vehicle-fleet`
  (synthetic fleet), seeded by the first migration.
- **Denied access:** `404` for a non-member (same as an unknown project),
  `403` for a member whose role is too low.
- **Routes:** `/api/projects/{key}/...` replace `/api/batteries` and
  `/api/vehicles`; the old routes are removed (no other consumers).

## What was built

**`bdt`** (branch `feat/authorization`)
- `app/auth.py`: token must have `aud` `bdt-api` (`KEYCLOAK_AUDIENCE`)
  besides signature/issuer/expiry, and a platform realm role (`bdt-user`
  or `bdt-platform-admin`, else 403). The user is resolved by `iss` +
  `sub` into `bdt_users`, created on first login.
- `app/db.py`, `app/models.py`, `alembic/`: `bdt_users`, `projects`,
  `project_memberships` (role check constraint); migration `0001` seeds
  the two projects.
- `app/permissions.py`: `require_project(key, min_role)` and
  `require_project_role(min_role)`; platform admins pass every check.
- Routes: `/api/me`, `/api/projects`, `/api/projects/{key}`,
  `/api/projects/vehicle-fleet/vehicles/...`,
  `/api/projects/nasa-pcoe/batteries/...`. Old routes removed.
- Dashboard: loads `/api/projects`, shows only the tabs for the user's
  projects, and a "no access yet" message if there are none.
- `docker-compose.yml`: local Postgres plus a `migrate` step.

**`bdt-gitops`** (branch `feat/bdt-db-authz`)
- `components/bdt-db`: `DATABASE_URL` from Secret `bdt-db` and the
  `migrate` init container, appended with JSON patches (the overlays
  patch env by index). Included by dev only; qa/prod add it when they
  are promoted to an image with authorization.
- `bdt-db` ExternalSecret in all three overlays.

**`bdt-infra`** (branch `feat/authorization-scripts`)
- `scripts/keycloak-bdt-authz.sh [email]`: `bdt-api` client, audience
  mapper on `bdt-frontend`, realm roles, `bdt-user` as default role,
  optional `bdt-platform-admin` grant; prints an example token's claims.
- `scripts/setup-bdt-db.sh [env...]`: roles and databases on the shared
  Postgres, URL into Vault at `secret/k8s/battery-dashboard-<env>/bdt-db`.

## Verification so far (2026-10-10, on odin)

- `uv run pytest`: 39 passed, on SQLite and on a local Postgres 16
  container (`BDT_TEST_DATABASE_URL`). Covers good/bad tokens (wrong or
  missing audience, wrong issuer, expired, foreign key, no platform
  role), user creation by `iss`+`sub`, 404 for non-members, 403 per role,
  platform admin, project list, old routes gone.
- `alembic upgrade head`, `alembic check` (models match migration),
  downgrade and upgrade again: fine on Postgres 16; also run inside the
  built backend image.
- Frontend: `npm test` 18 passed, `npm run build` ok. Not rendered in a
  browser yet.
- `keycloak-bdt-authz.sh` run twice against a local Keycloak 26.0 with the
  committed realm JSON: second run changes nothing; example token has
  `aud: [bdt-api, account]` and the roles.
- End to end with real tokens from that local Keycloak and the backend on
  local Postgres: admin sees both projects; a new user gets `bdt-user`,
  an empty project list and 404s; after a `viewer` row for
  `vehicle-fleet` they read the fleet and still get 404 on `nasa-pcoe`.
- `setup-bdt-db.sh`'s SQL run twice against local Postgres, then the
  migrations as the created role: fine. The full script has not been run
  on the cluster.
- `kubectl apply --dry-run=server -k overlays/dev`: accepted.

## Rollout (order matters)

1. Run `bash scripts/keycloak-bdt-authz.sh <your-email>` on odin. Safe for
   the running backend: it ignores `aud` today.
2. Run `bash scripts/setup-bdt-db.sh` (needs the Vault token).
3. Merge the `bdt-gitops` PR. The new dev pod's init container fails on
   the old image (no Alembic) while the old pod keeps serving.
4. Merge the `bdt` PR. CI builds and bumps dev; the new pod migrates and
   starts.
5. Log in to https://battery-dev.k3s.home: both tabs visible (platform
   admin). Check `/api/me`.

Done 2026-10-10: steps 1–4. Keycloak checked (mapper set, example token
for sungang76@gmail.com has `aud: [bdt-api, account]` and
`bdt-platform-admin`); `bdt-db` synced in dev; CI run for 47f061a green;
the `migrate` init container applied `0001` to `bdt_dev` (projects
`nasa-pcoe`, `vehicle-fleet`). Through the dev ingress: `/api/health`
200, `/api/me` 403 without a token and 401 with a bad one, old
`/api/batteries` 404. Step 5: logged in as sungang76@gmail.com (platform admin), both tabs
show data; the `bdt_users` row was created at first login.

## Done when

- [x] Steps 1–5 done; dev serves the project routes with a real login.
- [x] A non-admin Google account sees no projects, then only the project
      it was given (2026-10-11, added as NASA PCoE viewer via the Admin tab).
- [ ] qa/prod: promoted together with `components/bdt-db` in their overlay.

## AZ-204: member management (2026-10-10)

- API: `GET/POST /api/projects/{key}/members`, `PUT/DELETE
  /api/projects/{key}/members/{user_id}` for project admins (and platform
  admins); `GET /api/admin/users` for platform admins. Add by email once
  the person has logged in; a project admin can't remove or demote the
  last project admin (409).
- Dashboard: Admin tab (`#admin`) for anyone who is project admin
  somewhere.
- Tests: backend 51 passed (SQLite and Postgres 16), frontend 25 passed;
  Admin tab rendered headless with stubbed data. Merged (bdt #3) and
  deployed to dev as af47230; `/api/health` ok, member routes 403 without
  a token.

## Next

AZ phase 3 (audit events, workload identities).
