# meshbay-sample-native-seeded

The **no-`meshbay.toml`** variant of
[meshbay-sample-seeded](https://github.com/renakaagusta/meshbay-sample-seeded): the same Node.js +
Prisma + Postgres app, where every deploy migrates and then seeds the database, but **nothing in the
repository tells Meshbay how to deploy it**. Railpack detects Node, the start command comes from
`package.json` (`npm start` → `node src/server.js`), and the migrate + seed step is set in the
dashboard as the service's **Pre-deploy command**.

```
     deploy ──► Pre-deploy command (set in the dashboard, runs in the new revision's image)
                    │
                    ├─ npx prisma migrate deploy   pending migrations in prisma/migrations
                    │       │  fails ───────────────► command exits non-zero
                    │       ▼                            │
                    ├─ npx prisma db seed          upsert 3 plans + 1 demo admin, count the run
                    │       │  FAIL_SEED=1 ─────────────┤
                    │       ▼                            ▼
                    └─ exit 0                     rollout stops, old revision keeps serving
                            │
                            ▼
                    new revision starts on $PORT
                    /health · /plans · /meta · /api/items
```

## Dashboard setup (replaces meshbay.toml)

| Setting | Where | Value |
|---|---|---|
| Repository | New project → GitHub repo | `renakaagusta/meshbay-sample-native-seeded`, branch `main` |
| Database | Canvas → **+ New → Database → Postgres** | attach it to the app service |
| `DATABASE_URL` | app service → **Variables** | `${{postgres.DATABASE_URL}}` (reference to the attached database) |
| **Pre-deploy command** | app service → **Settings → Deploy commands** | `npx prisma migrate deploy && npx prisma db seed` |

Build and start commands stay empty: Railpack detects them (the Deploy commands section labels them
"Detected"). Without the Pre-deploy command the app still starts, but `/health` reports the database
tables as missing because nothing migrated it.

## Repository layout

```
meshbay-sample-native-seeded
├── prisma
│   ├── schema.prisma           Plan, User, Item, SeedState
│   ├── migrations             0001 init tables, 0002 items.note + created_at index
│   └── seed.js                 idempotent upsert seeder, FAIL_SEED=1 exits 1
├── src
│   ├── server.js               node:http router (health, plans, meta, items CRUD)
│   └── database.js             PrismaClient
└── scripts/smoke.sh            health, seeded plans, meta, CRUD; prints OK
```

## Endpoints

| Method | Path | Description |
|---|---|---|
| GET | `/health` | `200` `{"status":"ok","database":"ok"}`; `503` when Postgres is unreachable |
| GET | `/plans` | JSON array of the seeded plans, `priceCents` included |
| GET | `/meta` | `lastMigration`, `lastMigrationAt`, `seedRuns`, `lastSeedAt`, `plans` |
| GET | `/api/items` (also `/items`) | JSON array of items |
| POST | `/api/items` | `{"name":"...","note":"..."}` → `201`; `note` optional |
| GET | `/api/items/{id}` | `200`, `404` when missing |
| PUT | `/api/items/{id}` | `{"name":"...","note":"..."}` → `200`, `404` when missing |
| DELETE | `/api/items/{id}` | `204`, `404` when missing |

The `/api/items` path is deliberate: `meshbay/scripts/smoke-crud.sh` drives it unchanged.

## Environment

| Variable | Used by | Required | Default | Description |
|---|---|---|---|---|
| `DATABASE_URL` | app, migrate, seed | yes | none | Postgres URL, e.g. `postgres://user:password@host:5432/app` |
| `PORT` | app | no | `8080` | Injected by Meshbay from the container port |
| `FAIL_SEED` | seed | no | empty | `1` makes the seeder exit non-zero on purpose |

`DATABASE_URL` is a service variable that references the Postgres database attached to the project.

## Migrations and seeding

`prisma/migrations/` holds two plain-SQL migrations:

* `20261116090000_init` — `plans`, `users`, `items` and `seed_state`.
* `20261116090100_items_note` — adds `items.note` and an index on `items.created_at`.

`prisma migrate deploy` records each applied migration in `_prisma_migrations`, and `/meta` reports
the newest `migration_name` from that table.

`prisma/seed.js` upserts fixed reference data:

* 3 plans — `starter` (0), `growth` (2900), `scale` (9900) — keyed by `code`;
* 1 demo admin — `admin@example.com`, keyed by `email`, **with no password column at all**;
* one `seed_state` row whose `run_count` grows by one per seeder invocation.

Because every write is an upsert keyed on a natural column, redeploying leaves `/plans` at exactly 3
while `/meta` shows `seedRuns` one higher. That is the idempotency proof.

## Test scenario (what the coordinator runs on the jkt cluster)

1. Create a project from `https://github.com/renakaagusta/meshbay-sample-native-seeded`, branch `main`.
2. Add a **Postgres** database and attach it, then set `DATABASE_URL` and the **Pre-deploy command** as in
   [Dashboard setup](#dashboard-setup-replaces-meshbaytoml).
3. Deploy. Expected deploy log:
   * `prisma migrate deploy` applies `20261116090000_init` and `20261116090100_items_note`;
   * `seed` logs `seeded 3 plans and 1 demo admin`;
   * the new revision logs `listening on port 8080`.
4. Smoke the deployment:

   ```sh
   bash scripts/smoke.sh https://<project>-api.renakaagusta.dev
   ```

   Ends with `OK plans=3 seedRuns=1 migration=20261116090100_items_note on https://...`.
5. Redeploy `main` (no new commit needed — trigger a deploy). Expected: the log shows no migration
   applied, one seed execution, and the smoke script stays at `plans=3` while `seedRuns` becomes 2:

   ```sh
   bash scripts/smoke.sh https://<project>-api.renakaagusta.dev 2
   ```

6. CRUD is covered by the same script; the repository's `/api/items` contract is also the one
   `meshbay/scripts/smoke-crud.sh` exercises.
7. **Failure toggle.** Add the project variable `FAIL_SEED=1` and deploy. Expected:
   * `prisma migrate deploy` succeeds (or reports nothing pending);
   * `prisma db seed` fails with `FAIL_SEED=1: refusing to seed (failure simulation)` and the deploy
     is marked failed;
   * the previous revision keeps serving: the smoke script against the same URL is still green and
     `/meta` reports the same `seedRuns` as before the failed deploy.
   * Remove `FAIL_SEED` (or set it to anything but `1`) and deploy again to recover.

## Run locally

`prisma validate` only needs `DATABASE_URL` to be *set*, not reachable:

```sh
npm install
DATABASE_URL=postgres://user:password@localhost:5432/app npx prisma validate
npm run check
```

The full flow needs a reachable Postgres:

```sh
DATABASE_URL=postgres://postgres:postgres@localhost:5432/app npx prisma migrate deploy
DATABASE_URL=postgres://postgres:postgres@localhost:5432/app npx prisma db seed
DATABASE_URL=postgres://postgres:postgres@localhost:5432/app PORT=8080 node src/server.js
bash scripts/smoke.sh http://localhost:8080
```

`FAIL_SEED=1 npx prisma db seed` exits 1 without touching the database.

## Notes and known Meshbay gaps

* `prisma` is a **production** dependency, not a dev one: the Pre-deploy command runs inside the runtime image,
  after Railpack has pruned dev dependencies, so the CLI must survive the prune.
* Pre-deploy semantics (once per deploy or once per replica) are still undocumented in Meshbay.
  This sample is safe either way: migrations are ordered and locked by Prisma, seeding is upsert-only,
  and `run_count` simply counts the invocations.
* A seeder failure is not transactional across rows. A mid-run crash can leave some plans written;
  the next deploy's upsert converges them, which is the point of the fixture.
