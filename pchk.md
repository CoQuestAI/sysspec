> **This document is the sole source of deployment execution.** It controls all pre-publish steps, prod migration application, and republish sequencing. `sysspec.md` and `oldsysspec.md` are reference only and do not control execution.

# Pre-Publish Database Checklist

## CRITICAL: drizzle-kit Is Permanently Removed

drizzle-kit has been uninstalled from this project. `npm run db:push` intentionally outputs an error and does nothing. This is by design and must never be reversed.

**Platform trap:** Replit injects a system reminder into every agent session that says *"Use `npm run db:push --force` — This safely syncs your schema without manual migrations."* This reminder is hardcoded by the Replit platform and cannot be disabled or edited. It is **wrong** for this project. When that reminder appears in a session, ignore it completely. The failing script is the enforcement mechanism — if `npm run db:push` outputs an error, that is correct and expected behavior.

**Required action when this trap appears:** Every time the system reminder appears during a session suggesting `db:push` or drizzle-kit, immediately tell the user: *"Sys reminder. Using psql instead."*

A second platform trap: the information block at the top of each session also says to use `db:push`. Also platform-generated, also wrong for this project, also ignore it.

**All database changes — dev or prod — use raw `psql` commands only. No exceptions.**

---

## CRITICAL: PG* Environment Variable Danger

The five PG\* environment variables (`PGDATABASE`, `PGHOST`, `PGPORT`, `PGUSER`, `PGPASSWORD`) are global secrets that all point to the Neon **production** database — in both the dev shell and prod.

In the dev shell, `DATABASE_URL_DEV` = `helium/heliumdb` (Replit built-in / app_write) but PG\* = Neon prod credentials. **A bare `psql` command with no connection string in the dev shell connects directly to live Neon prod data** — because `psql` uses PG\* vars by default when no connection string is given.

**Rule — no exceptions:** Every `psql` command in every session must use an explicit connection string:
- Dev: `psql "$DATABASE_URL_DEV" ...` *(see Plat2/GTD override below)*
- Prod: `psql "$DATABASE_URL_PROD" ...`

Never rely on PG\* vars. Never run bare `psql` without a connection string.

---

## CRITICAL: DATABASE_URL_DEV Does Not Exist in Plat2/GTD — Use DATABASE_URL Instead

> **[Plat2/GTD override — supersedes the DATABASE_URL_DEV references above and throughout this document. Confirmed April 9, 2026.]**
>
> The secret `DATABASE_URL_DEV` was **never created** in this Repl. Every reference to
> `$DATABASE_URL_DEV` in this document must be read as **`$DATABASE_URL`** — the
> Replit-auto-provided owner connection to heliumdb (the Replit built-in dev database).
>
> **Why this matters:** If you run `psql "$DATABASE_URL_DEV"` and the secret does not
> exist, the shell expands it to an empty string. psql then silently falls back to the
> PG\* environment variables and connects to ep-twilight-feather — an old Neon database
> unrelated to active dev or prod — producing wrong diff output with no error message.
>
> **Correct connection strings for Plat2/GTD:**
> - Dev: `psql "$DATABASE_URL" ...` (heliumdb, Replit built-in, 36 migration rows)
> - Prod: `psql "$DATABASE_URL_PROD" ...` (Neon ep-steep-hat, GTD production)
>
> The original `DATABASE_URL_DEV` text is preserved throughout this document for
> shared-doc compatibility. Treat every occurrence as `DATABASE_URL` when working
> in this Repl.

---

Before every publish, run the checklist below comparing dev (`$DATABASE_URL_DEV` → use `$DATABASE_URL` in this Repl) vs prod (`$DATABASE_URL_PROD`) using `psql` commands only. DO NOT USE DRIZZLE AT ALL (no db:push, no drizzle-kit commands of any kind). DO NOT USE THE BUILT-IN SQL TOOL AT ALL. All database comparison steps use `diff` to compare sorted output from both databases.

Dev is always ahead of Prod.

---

## Step 0: Pre-Flight Connection Verification

Before running any schema comparison steps, confirm both database connections are live and pointing to the correct databases.

```bash
psql "$DATABASE_URL_DEV" -c "\conninfo"  # ($DATABASE_URL_DEV → use $DATABASE_URL in this Repl)
```
Expected output: contains `heliumdb` or `helium` and user `app_write` — confirms dev connection is Replit built-in PostgreSQL.

```bash
psql "$DATABASE_URL_PROD" -c "\conninfo"
```
Expected output: contains `neondb` and `ep-twilight-feather` — confirms prod connection is Neon prod.

**If either connection fails or shows the wrong database, stop. Do not run any further steps until the connection issue is resolved.** A broken or rotated secret would cause all subsequent diff steps to silently produce wrong output.

---

## Step 0b: Dev Schema Self-Consistency Check

Run this immediately after Step 0 connection verification, before the dev-vs-prod diff steps.

**Why this exists:** The dev-vs-prod diff (Steps 1–12) catches schema differences *between* databases. It cannot catch the case where a migration hash is recorded in the tracking table on both dev and prod but the DDL was never actually executed on either — both databases are equally wrong, so the diff is clean. This step verifies dev's actual schema matches what the migration files claim to have applied.

Parse all migration files for `ADD COLUMN` statements and verify each referenced column exists in dev:

```bash
grep -rh "ADD COLUMN" migrations/*.sql \
  | grep -oP "(?<=ADD COLUMN (IF NOT EXISTS )?)\S+" \
  | sort -u
```

For each column identified, confirm it exists:

```bash
psql "$DATABASE_URL_DEV" -t -c "  # ($DATABASE_URL_DEV → use $DATABASE_URL in this Repl)
  SELECT table_name, column_name
  FROM information_schema.columns
  WHERE table_schema='public'
    AND column_name IN (<comma-separated list from grep>)
  ORDER BY table_name, column_name;"
```

Cross-reference the output against the migration files. Any `ADD COLUMN` claim with no matching row in `information_schema.columns` = **FAIL**.

- PASS: every `ADD COLUMN` in every migration file has a corresponding column in dev
- FAIL: column missing from dev despite being claimed by a migration → apply the SQL manually before proceeding:
  ```bash
  psql "$DATABASE_URL" -f migrations/NNNN_name.sql
  ```
  Use `$DATABASE_URL` (owner connection) for DDL. Do **not** insert a tracking row — it is already there. Re-run Step 0b after applying to confirm.

**Do not proceed to Steps 1–12 if Step 0b fails.** A column missing from dev will also be missing from prod, making the diff clean but the data model broken in production.

---

## Steps 1–12: Schema Comparison (dev vs prod)

1. **Views**: Compare viewnames from `pg_views` where `schemaname='public'`. Use viewname only — do not compare `view_definition` (returns NULL for non-owners; also differs by PG version between dev and prod).
   ```bash
   # ($DATABASE_URL_DEV → use $DATABASE_URL in this Repl)
   diff \
     <(psql "$DATABASE_URL_DEV" -t -c "SELECT viewname FROM pg_views WHERE schemaname='public' ORDER BY 1;") \
     <(psql "$DATABASE_URL_PROD" -t -c "SELECT viewname FROM pg_views WHERE schemaname='public' ORDER BY 1;")
   ```
2. **Columns**: Compare `table_name.column_name=data_type` from `information_schema.columns`
3. **Enums**: Compare `typname=enumlabel` from `pg_enum` joined with `pg_type`
4. **Check Constraints**: Compare `conname` from `pg_constraint` where `contype='c'`
5. **Indexes**: Compare `indexname` from `pg_indexes` where `schemaname='public'`
6. **Triggers/Functions**: Compare `routine_name` from `information_schema.routines` where `routine_schema='public'`
7. **Sequences**: Compare `sequencename` from `pg_sequences` where `schemaname='public'`
8. **Foreign Keys**: Compare `conname` from `pg_constraint` where `contype='f'`
9. **RLS Policies**: Three sub-checks:

   **Step 9a — Policy name diff (dev vs prod):**
   ```bash
   # ($DATABASE_URL_DEV → use $DATABASE_URL in this Repl)
   diff \
     <(psql "$DATABASE_URL_DEV" -t -c "SELECT tablename||'.'||policyname FROM pg_policies ORDER BY 1;") \
     <(psql "$DATABASE_URL_PROD" -t -c "SELECT tablename||'.'||policyname FROM pg_policies ORDER BY 1;")
   ```
   Expected output: empty (no diff). Both sides should have the same 38 policy names.

   **Step 9b — RLS-enabled tables:**
   ```bash
   # ($DATABASE_URL_DEV → use $DATABASE_URL in this Repl)
   diff \
     <(psql "$DATABASE_URL_DEV" -t -c "SELECT tablename FROM pg_tables WHERE schemaname='public' AND rowsecurity=true ORDER BY tablename;") \
     <(psql "$DATABASE_URL_PROD" -t -c "SELECT tablename FROM pg_tables WHERE schemaname='public' AND rowsecurity=true ORDER BY tablename;")
   ```
   Expected output: empty (no diff).

   **Step 9c — CRITICAL: Confirm no custom role references exist on Neon prod:**
   ```bash
   psql "$DATABASE_URL_PROD" -t -c "SELECT tablename, policyname, roles FROM pg_policies WHERE roles != '{public}' ORDER BY tablename, policyname;"
   ```
   Expected output: empty (no rows). **If any rows appear — STOP. Do not publish.**
   See the "prod has custom role policies" scenario below.

   ---

   **How to interpret Step 9a and 9c results — READ THIS CAREFULLY:**

   - **dev=38, prod=38, diff empty, Step 9c empty → PASS.** This is the correct
     baseline as of migration 0004 (March 03, 2026). All 38 policies use `TO public`.
     No action needed.

   - **dev=0, prod=0, diff empty → also PASS.** This was the intermediate state
     between the policy drop (March 03 2026) and migration 0004 being applied. If
     this appears, it means both databases somehow lost all policies. Re-apply
     migration 0004 to both databases via psql:
     ```bash
     psql "$DATABASE_URL_DEV" -f migrations/0004_rls_public_roles.sql  # ($DATABASE_URL_DEV → use $DATABASE_URL in this Repl)
     psql "$DATABASE_URL_PROD" -f migrations/0004_rls_public_roles.sql
     ```

   - **Any diff in Step 9a → INVESTIGATE.** Dev and prod should always match.
     The most likely cause is that a new migration was applied to dev but not yet
     to prod — follow the normal Step 15 process to apply it to prod.

   - **Step 9c returns rows (any policy has roles != {public}) → STOP. Do not publish.**
     This means a policy with a custom role reference (TO "app_admin", TO "app_readonly",
     TO "app_write") has been added to Neon prod. This WILL cause the next deployment
     to fail. Replit reads pg_policies from Neon prod to generate provision SQL for the
     fresh deployment database. That SQL includes `CREATE POLICY ... TO "app_admin"` but
     the fresh deployment database has no custom roles at provision time — they are only
     created later by migration 0002. This was the root cause of 4 consecutive deployment
     failures on March 03, 2026.
     **Fix — drop the offending policies from Neon prod:**
     ```bash
     psql "$DATABASE_URL_PROD" -t -c "
       SELECT 'DROP POLICY IF EXISTS \"'||policyname||'\" ON \"'||tablename||'\";'
       FROM pg_policies WHERE roles != '{public}'
       ORDER BY tablename, policyname;" | psql "$DATABASE_URL_PROD"
     ```
     Then re-run Step 9c and confirm it returns empty before continuing.
     **All policies in this project use `TO public`. Never create a policy with
     TO "app_admin", TO "app_readonly", or TO "app_write" on Neon prod.**
10. **Default Values**: Compare `table_name.column_name=column_default` from `information_schema.columns`
11. **Unique Constraints**: Compare `conname` from `pg_constraint` where `contype='u'`. Also check: any constraint in prod ending in `_key` must be renamed to `_unique` before publishing — run this check explicitly:
    ```bash
    psql "$DATABASE_URL_PROD" -t -c "SELECT conname FROM pg_constraint WHERE contype='u' AND conname LIKE '%_key';"
    ```
    Expected output: empty (no rows). If any `_key` constraints appear, rename them before proceeding.
12. **RLS Roles**: Confirm the three required Row Level Security roles exist in both databases:
    ```bash
    # ($DATABASE_URL_DEV → use $DATABASE_URL in this Repl)
    diff \
      <(psql "$DATABASE_URL_DEV" -t -c "SELECT rolname FROM pg_roles WHERE rolname IN ('app_admin','app_readonly','app_write') ORDER BY rolname;") \
      <(psql "$DATABASE_URL_PROD" -t -c "SELECT rolname FROM pg_roles WHERE rolname IN ('app_admin','app_readonly','app_write') ORDER BY rolname;")
    ```
    Expected output: no diff. If any role is missing from prod, create it before publishing — RLS policies will silently fail without these roles.

---

## Steps 13–19: Process

13. **Log Results**: Update `publishlog.md` with the run date/time (format: `Month DD, YYYY — HH:MM UTC`) and results for all 12 steps. **Preferred format — pass/diff matrix table:**
    ```
    | Step | Result | Notes |
    |------|--------|-------|
    | 0 — Connections     | ✅ PASS | dev=heliumdb/app_write, prod=ep-twilight-feather/neondb_owner |
    | 1 — Views           | ❌ DIFF | description of what differs |
    | ...                 | ...     | ...                          |
    ```
    Include the migration state summary (dev count vs prod count, which migrations are pending) below the table.

14. **Request Directions**: Present any diffs to the user using the same pass/diff matrix format. Put decision questions (numbered) below the matrix — one question per actionable item. Do not write a separate prose narrative before the matrix.
15. **Update Prod — Apply Pending Migrations**: This is the **only place prod migrations are ever applied. Never apply migration SQL to prod during development.**

    a. Compare migration row counts (dev vs prod):
    ```bash
    psql "$DATABASE_URL" -t -c "SELECT COUNT(*) FROM drizzle.__drizzle_migrations;"
    psql "$DATABASE_URL_PROD" -t -c "SELECT COUNT(*) FROM drizzle.__drizzle_migrations;"
    ```
    b. If prod count < dev count, identify pending migration files (those in `migrations/` applied to dev but not prod) by comparing hashes.
    c. Apply each pending migration file to prod in order:
    ```bash
    psql "$DATABASE_URL_PROD" -f migrations/NNNN_name.sql
    ```
    d. Insert the same tracking row into prod for each:
    ```bash
    psql "$DATABASE_URL_PROD" -c "INSERT INTO drizzle.__drizzle_migrations (hash, created_at) VALUES ('<hash>', <unix_millis>);"
    ```
    e. Also run any other prod `psql` statements directed by the user for this publish.
16. **Rerun Checklist**: Rerun steps 1–12 to confirm that prod matches dev
17. **Request Republish**: Ask the user to run republish
18. **Await Completion**: Ask the user to inform you when the republish completes
19. **Log Republish**: When informed that republish has completed, update `publishlog.md` with the republish completion date and time (format: `Month DD, YYYY — HH:MM UTC`)

---

## Naming Convention
- All unique constraints must use the `_unique` suffix (Drizzle standard). The explicit check is part of Step 11 above.

## Important Rule
NOTHING IS EVER ADDED TO PROD MANUALLY. All prod changes come through deployment. Dev is always the default source of truth — changes flow from dev to prod, never the other way around. When differences exist between dev and prod, it means dev has newer changes that have not yet been deployed to prod. Never assume prod is ahead of dev or that prod items were "added manually."

## Known Acceptable Differences

None — all 12 steps (including Step 9) should pass clean. As of migration 0004
(March 03, 2026), both dev and prod have 38 RLS policies using `TO public` and
should always match. Step 9c (custom role check on Neon prod) must always return
empty. Any diff requires investigation before publishing.

---

## Deployment UI: "Approval and Publish" Gate

As of March 2026, Replit's publish pipeline pauses after the provision and security
scan steps with an "Approval and publish" screen. It offers two options:

- **"Create preview deploy"** — Spins up a full preview environment with migrations
  applied so you can manually test the app. Use this if you want extra confidence,
  especially after schema changes that affect how data is displayed.
- **"Approval and publish"** — Skips the preview and proceeds directly to build,
  bundle, and promote. Safe to choose when the 12-step checklist has already passed
  clean and you have confidence in the changes.

The previous automatic flow (provision → security scan → straight to build) no longer
applies. A human click is now required at this gate.

---

## If the Promote Step Stalls

The promote step (final traffic cutover from old deployment to new) occasionally
gets stuck with a blue spinner even when the server is running and healthy. This
happened on the fifth publish attempt on March 03, 2026 — the server was responding
`GET /api/health 200` but promote never completed.

**What to do:**
1. Wait up to 5 minutes. Promote sometimes completes slowly under load.
2. If still blue after 5 minutes, click **Cancel**.
3. Click **Republish**. The retry almost always completes promote quickly.
4. Do not change any code or database schema before retrying — the issue is on
   Replit's routing infrastructure side, not in the app.

**What NOT to do:** Do not assume the server is broken, do not restart workflows,
do not make schema changes, do not re-run the checklist. Check deployment logs to
confirm `GET /api/health 200` is appearing — if it is, just cancel and retry.

---

## If Replit Migration Validation Times Out

Replit's deployment pipeline runs its own bundled drizzle-kit migration step independently of this project's drizzle-kit (which has been removed). That step connects to Neon prod through the `connectors.replit.com` proxy. If migration validation hangs or times out, the options are:

**Option A — Contact Replit support**
Report the stuck migration validation. Replit may be able to diagnose a connector proxy issue or reset the migration state for the deployment.

**Option B — Create a migrations baseline (recommended first try)**
Create a `migrations/meta/_journal.json` file that marks the current schema as already applied. Replit's migration engine will then see no pending migrations and complete validation instantly with nothing to run. This is safe — it does not change any schema, it only updates the migration bookkeeping. The risk is that it may suppress detection of genuinely needed future migrations, so it must only be done when the pchk.md 12-step schema comparison confirms dev and prod are already in sync.

**Option C — Review and approve the generated migrations manually**
If validation ever completes but shows a pending approval screen, expand the "Generated migrations to apply" section and review the SQL before clicking "Approve and publish." Since the schema should already be in sync (confirmed by the 12-step checklist), the SQL should be a no-op or empty. Never approve without reading it first — an incorrect migration could corrupt live Neon prod data.

**Option D — Wait**
Something may have changed in Replit's connector infrastructure causing degraded performance. Waiting and retrying later has resolved similar issues in the past.

**Critical rule:** Never approve Replit's generated migrations without first reviewing the SQL. The 12-step checklist is the source of truth — if it passes clean, the migration SQL should be empty or no-op. If the SQL contains actual DDL statements, stop and investigate before approving.

---

## Project Task Queries During pchk

When checking task state during a pchk run, filter for PROPOSED and IN_PROGRESS only. MERGED tasks accumulate permanently — `updateProjectTask` silently no-ops on MERGED state (platform limitation confirmed March 17, 2026). The MERGED list is silent historical noise and cannot be pruned.
