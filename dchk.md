> **This document is the daily session close-out process.** The agent runs all steps autonomously at the end of every work session and appends a structured entry to `dailylog.md`. No user action is required except reading the result.

# Daily Close-Out Checklist

**Execution order: Steps 1–9 → Step 10 (task consolidation) → Step 11 (dailylog.md entry last).**

---

## STEP 1 — Dev Server Health

Run:
```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:5000/api/server-time
```
Expected: `200`.

If not 200 — note the failure in the log entry and investigate before closing out. Do not proceed to Step 11 until the server is healthy or the failure is explained.

---

## STEP 2 — Checkpoint List (last 24 hours)

Run:
```bash
git log --oneline --since="24 hours ago"
```

Collect all SHAs and messages. Filter out automated env/toolchain commits (messages containing "Update environment variables", "Update environment and toolchain", "toolchain configuration", "agent state") — these are Replit platform noise. Record only meaningful work commits.

---

## STEP 3 — Changed Files Summary

Run using the oldest meaningful commit SHA from Step 2:
```bash
git diff --name-only <oldest-meaningful-sha> HEAD
```

Filter out `.cache/`, `.local/state/`, `attached_assets/` — platform noise. Record only source files: `client/`, `server/`, `shared/`, `migrations/`, and root-level `.md` files.

---

## STEP 4 — Migration State Verification

Run:
```bash
psql "$DATABASE_URL_DEV" -c "SELECT COUNT(*) FROM drizzle.__drizzle_migrations;" -t
```

Compare to the dev count in `deliverables.md` Migration State table.

- **No change:** record "dev=N — no change".
- **Count increased:** record the new count, update the `deliverables.md` Migration State table, and note which migration(s) were applied.

**Step 4b — Spot-check most recent migration's DDL** *(always run, regardless of count change)*

The tracking table can be ahead of the actual schema — a hash can be recorded without the DDL ever executing (as happened with Task #49: `narrative_arc` hash tracked, column never created). Always verify the most recently applied migration file's DDL is reflected in dev:

```bash
# Identify the most recently tracked migration file
ls -t migrations/*.sql | head -1
```

Read that file, identify its key structural changes (`ADD COLUMN`, `CREATE INDEX`, `ADD CONSTRAINT`, etc.), and verify each:

```bash
psql "$DATABASE_URL_DEV" -t -c "
  SELECT column_name FROM information_schema.columns
  WHERE table_name='<table>' AND column_name='<column>';"
# Empty = column missing despite being tracked → DDL was never applied
```

- PASS: all claimed DDL confirmed present in dev — record "DDL spot-check: PASS".
- FAIL: any claim unverified — apply the migration manually before closing out:
  ```bash
  psql "$DATABASE_URL" -f migrations/NNNN_name.sql
  ```
  Use `$DATABASE_URL` (owner connection) for DDL. Do **not** insert a new tracking row — it is already recorded. Record the fix in the log entry.

---

## STEP 5 — Merge Log Scan

Read `mergelog.md`. Check whether any new entries are dated today.

- **No merges today:** record "None".
- **Merge(s) today:** record task name(s).

---

## STEP 6 — Publish Log Scan

Read `publishlog.md`. Check whether any publish runs are dated today.

- **No publish today:** record "None".
- **Published today:** record a summary (TSX-only / schema changes / migration count applied to prod).

---

## STEP 7 — Environment Variable Audit

Scan the changed source files from Step 3 for new `process.env.*`, `import.meta.env.*`, or Replit Secrets references that did not exist in previous sessions.

- **No new refs:** record "None".
- **New refs found:** confirm each is documented in `devprodseparation.md`. If not, add them before closing out.

---

## STEP 8 — Open Issues

List any known bugs, regressions, or incomplete items discovered or noted during this session. These carry forward as "Next session priorities" in the log entry and as context for the next session's opening.

If none, record "None".

---

## STEP 9 — Autonomous File Updates

The agent checks each file below and updates it if warranted. No user prompt required — the agent decides based on today's work and acts.

| File | Update condition |
|---|---|
| `deliverables.md` | Migration count changed; security task resolved; architectural phase completed |
| `replit.md` | New modules added; key file locations changed; new dependencies introduced; architectural decisions made |
| `future.md` | Deferred items identified or discussed during session |
| `sysspec-push.sh` *(action, not a file update)* | Any of the 6 shared docs (`oldsysspec.md`, `userpref.md`, `mchk.md`, `pchk.md`, `dchk.md`, `devprodseparation.md`) were edited this session → run `bash sysspec-push.sh` and record the result. Also note in the log entry whether a `sysspec-pull.sh` run is recommended at the start of the **next** session (i.e., if another Repl is likely to have pushed shared docs in the interim). Note: `sysspec.md` (the GTD Mobile System Spec) is local-only and is NOT synced to GitHub. |

If none of the conditions apply, record "No file updates required this session. Sysspec push: N/A." in Step 9 of the log entry.

---

## STEP 10 — Completed Task Consolidation

Run **before** Step 11 so the log entry can capture the consolidation result.

### 10a — Consolidate `✓` PROPOSED tasks

Scan the project task list for PROPOSED tasks with a `✓` prefix — these are Build-mode-completed tasks with no formal close-out path in the current platform.

```javascript
const tasks = await listProjectTasks({ state: "PROPOSED" });
const done = tasks.filter(t => t.title.startsWith("✓"));
```

- **0 or 1 `✓` tasks:** nothing to do — record "No ✓-task consolidation needed".
- **2 or more `✓` tasks:** consolidate:
  1. Pick the lowest task ref as the **master**.
  2. Update master title to `✓ Completed sessions (consolidated)` and append each other task's title + key points into the master description.
  3. Update all other `✓` tasks to title `(archived — see #N)` where N is the master task ref, with a minimal description.
  4. Record in the log entry: "Consolidated #X, #Y into master #N."

### 10b — Sweep MERGED tasks into master

Tasks that complete through the task-agent merge flow end up in MERGED state and are invisible to 10a. Sweep them into the same master summary task so no work is lost from the daily log.

```javascript
const merged = await listProjectTasks({ state: "MERGED" });
const today = new Date().toISOString().slice(0, 10);
const mergedToday = merged.filter(t => t.updatedAt && t.updatedAt.slice(0, 10) === today);
```

- **0 MERGED tasks updated today:** nothing to do — record "No MERGED sweep needed".
- **1 or more MERGED tasks updated today:**
  1. Identify the master `✓` task (lowest-ref `✓`-prefixed PROPOSED task from 10a). If none exists yet, pick the lowest existing `✓` task from the full PROPOSED list; if still none, create one titled `✓ Completed sessions (consolidated)`.
  2. Build a fresh description block containing only **this session's** MERGED tasks:
     ```
     ### [Date] — MERGED #[ref]: [title]
     [One-line summary]
     ```
  3. **Overwrite** the master task's description with this block using the positional-argument form — the object form fails validation:
     ```javascript
     await updateProjectTask(masterRef, "✓ Completed sessions (consolidated)", newDescriptionString);
     ```
     `updateProjectTask` takes `(taskRef, title, description)` as three positional strings. Do **not** pass an object as the second argument. Each call fully replaces the description — this is intentional. Historical MERGED task info is permanently captured in `dailylog.md`; master #2 only needs to hold the current session.
  4. Record in the log entry's "Task consolidation" line: "Overwrote master #N with MERGED #X, #Y."

### Log entry format

The "Task consolidation" line in the Step 11 log entry should capture both actions:

```
**Task consolidation:** [None needed | Consolidated ✓ #X, #Y → master #N | Overwrote master #N with MERGED #A, #B | Both: consolidated ✓ #X, #Y and overwrote master #N with MERGED #A, #B]
```

Net result: always exactly one active summary task showing the most recent session's MERGED work. Archived stubs accumulate slowly but their titles make them ignorable. See `future.md` → "Project Task System: No Cancel/Archive from PROPOSED State" for context.

**MERGED task accumulation (confirmed March 17, 2026):** MERGED tasks accumulate permanently and cannot be pruned. `updateProjectTask` silently no-ops on MERGED state — the call succeeds but the state does not change. When listing tasks for any purpose in this checklist, always filter for PROPOSED and IN_PROGRESS only. The MERGED list is silent historical noise.

---

## STEP 11 — Append to `dailylog.md`

Always run **last** — after Step 10 so the task consolidation result is known. Append a new entry at the **top** of `dailylog.md` (most recent first) using this exact template:

```
## Session: [Date] — [Short description of main focus]

**Checkpoints:**
- `[SHA]` [commit message]
- (env/toolchain commits omitted)

**Files changed:**
- [group by area: dialogs/, journal/, server/, etc.]

**Migration state:** dev=[N] (idx 0–N-1) — [No change | Changed: was M, now N — migration `NNNN_name.sql` applied]

**Merges today:** [None | Task name — see mergelog.md]

**Publish today:** [None | Yes — [brief description] — see publishlog.md]

**Env var changes:** [None | List new refs and confirmation of devprodseparation.md status]

**File updates this session:** [None | List files updated in Step 9]

**Sysspec sync:** [N/A — no shared docs edited | Pushed: [file list] — commit [SHA] | Pull recommended next session: yes/no]

**Task consolidation:** [None needed | Consolidated ✓ #X, #Y → master #N | Overwrote master #N with MERGED #A, #B | Both: consolidated ✓ #X, #Y and overwrote master #N with MERGED #A, #B]

**Open items carried forward:**
- [item | None]

**Next session priorities:**
- [item]
```

---

## Quick Reference — File Ownership

| File | Updated by |
|---|---|
| `dailylog.md` | This checklist — Step 11, every session |
| `deliverables.md` | This checklist — Step 9, when migration/architecture changes occur |
| `replit.md` | This checklist — Step 9, when architecture/modules change |
| `publishlog.md` | `pchk.md` — every publish run |
| `mergelog.md` | `mchk.md` — every subtask merge |
| `future.md` | Any session where deferred items arise |
