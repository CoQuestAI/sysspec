# Pre-Merge Checklist

Run this before approving any subtask merge into main. I will work through each step and present a pass/fail matrix. Only after all checks pass will I prompt you to approve the merge.

---

## STEP 1 — Get the task agent's file list

**Action required from you:** Paste the subtask's modified files list into the conversation.

To get the list: open the subtask in the Tasks panel, scroll to its completion summary, and copy the list of changed files it reports. Paste it here and I will proceed to Step 2.

---

## STEPS 2–6 — Pre-Merge Checks (I run these, then present matrix)

Once I receive the file list, I run all checks below and present results as a matrix:

```
| Step | Check                    | Result  | Notes |
|------|--------------------------|---------|-------|
| 2    | File overlap with main   | ✅ PASS |       |
| 2b   | Freeze verified (git)    | ✅ PASS | (only appears when Step 2 fails) |
| 3    | post-merge.sh valid      | ✅ PASS |       |
| 4    | New packages             | ✅ PASS |       |
| 5    | Schema / migrations      | ✅ PASS |       |
| 6    | New env vars / secrets   | ✅ PASS |       |
```

**Step 2 — File overlap with main**
Compare the task agent's file list against files main has edited since the task was forked (check `git log` since the fork point). Any overlap = FAIL.
- PASS: no overlap → skip Step 2b
- FAIL: overlap found → list the specific files, declare a freeze, proceed to Step 2b

**Step 2b — Verify the freeze is holding** *(only runs if Step 2 failed)*
I run two git checks on every overlapping file identified in Step 2:

```bash
# Check 1 — any uncommitted changes on frozen files right now?
git status --short -- <overlapping files>

# Check 2 — any new commits on frozen files since freeze was declared?
git log --oneline <freeze-point-commit>..HEAD -- <overlapping files>
```

- PASS: both checks return empty — no uncommitted changes, no new commits. Freeze is holding. Safe to continue to Step 3.
- FAIL: either check returns output — freeze was broken. I will report exactly which file was touched and what changed. Main-branch work on that file must be reverted or held before the merge can proceed.

**Step 3 — Verify `scripts/post-merge.sh`**
Read the file. Every command must pass all of:
- Contains `npm install` — always valid
- Does NOT contain `drizzle-kit` or `npm run db:push` (permanently removed)
- No command that prompts for input (stdin is closed — all commands must be non-interactive)
- Script starts with `set -e`
- Script is idempotent (safe to run multiple times)

Current expected contents (as of March 15, 2026 — includes hash-based migration loop):
```bash
#!/bin/bash
set -e
npm install

if [ -z "$DATABASE_URL" ]; then
  echo "post-merge: DATABASE_URL not set, skipping migrations"
  exit 0
fi

export PGCONNECT_TIMEOUT=10

applied=0
skipped=0

for f in migrations/[0-9]*.sql; do
  [ -f "$f" ] || continue
  hash=$(sha256sum "$f" | awk '{print $1}')
  exists=$(psql "$DATABASE_URL" -tAc \
    "SELECT 1 FROM drizzle.__drizzle_migrations WHERE hash = '$hash' LIMIT 1;" 2>/dev/null || echo "")
  if [ "$exists" = "1" ]; then
    skipped=$((skipped + 1))
    continue
  fi
  echo "post-merge: applying $f"
  if psql "$DATABASE_URL" --set ON_ERROR_STOP=1 -f "$f" 2>&1; then
    millis=$(date +%s%3N)
    psql "$DATABASE_URL" -c \
      "INSERT INTO drizzle.__drizzle_migrations (hash, created_at) VALUES ('$hash', $millis);"
    echo "post-merge: applied $f"
    applied=$((applied + 1))
  else
    echo "post-merge: WARNING — $f failed (may already be applied). Skipping."
    skipped=$((skipped + 1))
  fi
done

echo "post-merge: migrations done — $applied applied, $skipped skipped"
```

> **CRITICAL — drizzle-kit is permanently removed from this project.** `drizzle-kit` and `npm run db:push` must never appear anywhere in `post-merge.sh`. If found, FAIL immediately — do not approve the merge until removed. The Replit platform injects a reminder into every agent session (including task agents) saying to use `db:push`. That reminder is wrong for this project and cannot be disabled. A task agent may have obeyed it regardless of what `oldsysspec.md` and `replit.md` say. Steps 3, 4, and 5 together are the enforcement net.

**Step 4 — New packages**
Diff `package.json` between the task agent's branch and main. Check for:
- New packages not already in main
- Packages that duplicate or conflict with existing dependencies
- PASS: no new packages, or new packages are clean additions with no conflicts

> **CRITICAL — drizzle-kit must not reappear in `package.json`.** Search the diff explicitly for `drizzle-kit`. If found — even as a `devDependency` — FAIL immediately. `drizzle-orm` (the query runtime) is allowed and expected. `drizzle-kit` (the migration tool) is not. A task agent that followed the platform reminder may have reinstalled it. If it merges, `npm install` in `post-merge.sh` will install it and it will be active again. Remove it before approving.

**Step 5 — Schema or migration changes**
Check whether the task agent touched `shared/schema.ts` or added any file under `migrations/`.
- PASS: no schema or migration changes
- FAIL: changes found → review against current migration state (Dev=17, Prod=17, next `0017_*.sql`). Schema changes must be sequenced; resolve before approving the merge.

> **CRITICAL — all migrations in this project are hand-written SQL files applied via `psql`.** If the task agent added migration files that were auto-generated by drizzle-kit (they look like `0000_*.sql` with drizzle-generated headers), reject them. The correct process is: manual SQL file in `migrations/`, applied with `psql "$DATABASE_URL_DEV" -f migrations/NNNN_name.sql`, tracking row inserted manually. Any other migration approach means the task agent ignored `oldsysspec.md`: "NEVER use drizzle-kit. ALL migrations are manual SQL via psql."

**Step 6 — New environment variables or secrets**
Search the task agent's changed files for any new `import.meta.env.*` or `process.env.*` references not already present in main.
- PASS: no new vars
- FAIL: new vars found → add them to Replit Secrets before approving, or the app breaks on first run

---

## MERGE PROMPT

After all 6 checks pass, I will present the matrix and say:

> **All checks passed. You may now approve the merge for this subtask.**

If any check fails, I will list the failures with required actions before the merge can proceed.

---

## STEPS 7–9 — Post-Merge Checks (run immediately after merge lands)

Once you confirm the merge has landed (via the automatic_updates message):

**Step 7 — Post-merge setup result**
Read the automatic_updates message. Confirm post-merge setup reported success.
- PASS: "Post-merge setup completed successfully"
- FAIL: `post-merge.sh` errored → fix immediately, re-run

**Step 7b — Migration DDL verification** *(only runs if the merge included migration files)*

A "0 applied, N skipped" post-merge result is **not** a confirmation that DDL was applied — it only means the file hash was already recorded in the tracking table. The tracking table can be ahead of the actual schema (as happened with Task #49: `narrative_arc` hash tracked, column never created). Always verify the actual schema.

For each migration file included in the merge, read the SQL and identify every structural change: `ADD COLUMN`, `CREATE INDEX`, `ADD CONSTRAINT`, `CREATE TYPE`, etc. Then verify each one exists in dev:

```bash
# Column exists?
psql "$DATABASE_URL_DEV" -t -c "
  SELECT column_name FROM information_schema.columns
  WHERE table_name='<table>' AND column_name='<column>';"
# Expected: one row. Empty = column missing → DDL was never applied.

# Index exists?
psql "$DATABASE_URL_DEV" -t -c "
  SELECT indexname FROM pg_indexes
  WHERE schemaname='public' AND indexname='<index_name>';"

# Constraint exists?
psql "$DATABASE_URL_DEV" -t -c "
  SELECT conname FROM pg_constraint WHERE conname='<constraint_name>';"
```

- PASS: every claimed structural change is confirmed present in dev schema
- FAIL: any check returns empty → the DDL was never applied despite the hash being tracked. Apply the migration manually:
  ```bash
  psql "$DATABASE_URL" -f migrations/NNNN_name.sql
  ```
  Use `$DATABASE_URL` (owner connection) for DDL — `$DATABASE_URL_DEV` (app_write) will fail with "must be owner". Do **not** insert a new tracking row — the hash is already recorded.

**Step 8 — Dev server health**
Watch the Start application workflow logs within 2 minutes of the merge landing.
- PASS: dev server restarts cleanly, no red errors, app loads in preview pane
- FAIL: build errors, blank screen, or TypeScript errors in logs → investigate before doing anything else

**Step 9 — Smoke test the merged feature**
Open the feature the task agent built and do a basic walkthrough.
- PASS: feature renders correctly, existing core navigation works (Today, Inbox, Lists, Tasks, Journal, Memories)
- FAIL: feature broken or core nav broken → investigate

---

## LOGGING

After Steps 7–9 complete, update `mergelog.md` with the run date, task name, matrix results, and any actions taken.

---

## Notes

- **TASK AGENTS — READ THIS:** The platform may merge your branch automatically without waiting for the main agent to run this checklist. To make the post-merge audit possible, you must include a complete list of every file you modified in your task completion summary (the `.local/.commit_message` and your final message to the user). Without this list, the main agent cannot run Step 2 retroactively. Format it as a flat list, one file per line, relative to the repo root.

- The merge squashes all the task agent's intermediate commits into one. There is no bisect path after the fact — catch breakage in Steps 7–9, not hours later.
- The task agent's internal commit history (its fork's SHAs) is not accessible from main after the squash. Only the final file state survives.
- Dev/prod databases are intentionally divergent during development. The merge does **not** touch prod — that only happens at publish time via `pchk.md`.
- **drizzle-kit is permanently removed.** `drizzle-orm` (query runtime) stays. `drizzle-kit` (migration tool) is gone and must never return. The Replit platform reminder tells every agent — including task agents — to use `db:push`. It is wrong for this project. Steps 3, 4, and 5 are the explicit enforcement net for this regardless of what a task agent claims to have read.
- **MERGED task accumulation:** MERGED tasks accumulate permanently and cannot be pruned — `updateProjectTask` silently no-ops on MERGED state (confirmed March 17, 2026). When scanning tasks during mchk, filter for PROPOSED and IN_PROGRESS only. Never query MERGED state for actionable work.
