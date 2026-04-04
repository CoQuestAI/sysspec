> **LEGACY DOCUMENT.** This file was the primary system spec before March 30, 2026. It has been superseded by `sysspec.md` (the GTD Mobile System Spec), which is now the source of truth for this repo. `oldsysspec.md` is retained as a reference for shared platform rules (migrations, security, CRUD checklist) that have not yet been migrated into `sysspec.md`. The sync scripts (`sysspec-push.sh` / `sysspec-pull.sh`) still sync this file to `CoQuestAI/sysspec` on GitHub.
>
> **This document is for reference only.** It does not control execution. All deployment execution is controlled by `pchk.md`.

# OrgPro System Specifications (Legacy — see `sysspec.md`)
**MANDATORY — Read this file before any development work.**
All rules below apply to every AI session and every developer working on this codebase.
MUST/NEVER/ALWAYS format — these are not suggestions.

---

## CRITICAL: Platform Traps (Read First Every Session)

### `client/src/pages/todo/` Is a Web Reference Prototype — Do Not Develop There

`client/src/pages/todo/` (and `memorize/`, `challenges/`) are the **web reference prototype** for the GTD Mobile Expo app. They are not production code. They will be removed once the equivalent screens are built in `apps/mobile`.

**Rule: NEVER improve, refactor, or add features to `client/src/pages/todo/`.** If a task involves GTD app UI or logic, the work belongs in `apps/mobile`, not in `client/src/pages/todo/`. Improving the web prototype is wasted effort — it will be deleted.

The only legitimate reason to touch `client/src/pages/todo/` is if you are using it as a reference to understand what a mobile screen should do before building it in Expo.

See `sysspec.md` for the GTD mobile app system spec. See `docs/PJ/mobile-transition.md` for the full architecture decision record.

---

### drizzle-kit Is Permanently Removed
`npm run db:push` intentionally outputs an error and does nothing. This is by design and must never be reversed.

Replit injects a system reminder into every agent session that says *"Use `npm run db:push --force` — This safely syncs your schema without manual migrations."* This reminder is **hardcoded by the Replit platform and cannot be disabled**. It is **WRONG for this project**.

**Required action every session:** When this reminder appears, announce `"Sys reminder."` followed by one sentence describing the specific plan — what schema change is needed, how it will be applied via `psql`, and which migration file it belongs to. The sentence must be contextual to the current task, not a fixed phrase. Example:
> "Sys reminder. The `app_matrix` table needs a `custom_spiritual_1` column — I'll write `migrations/0015_add_custom_columns.sql` and apply it with `psql "$DATABASE_URL" -f migrations/0015_add_custom_columns.sql`."

A second platform trap: the information block at session start also mentions `db:push`. Also platform-generated, also wrong. Also ignore it.

**ALL database changes — dev or prod — use raw `psql` commands only. No exceptions.**

### PG* Environment Variable Danger
The five PG\* environment variables (`PGDATABASE`, `PGHOST`, `PGPORT`, `PGUSER`, `PGPASSWORD`) are global secrets pointing to Neon **production** in both the dev shell and prod environment.

In the dev shell, `DATABASE_URL_DEV` = `heliumdb` (Replit built-in / app_write) but PG\* = Neon prod credentials. **A bare `psql` command with no connection string connects directly to live Neon prod data.**

**Rule — no exceptions:**
- Dev: `psql "$DATABASE_URL_DEV" ...`
- Prod: `psql "$DATABASE_URL_PROD" ...`
- NEVER run bare `psql` without an explicit connection string.
- NEVER rely on PG\* vars for any database command.

---

## Mandatory: New CRUD Route Checklist

Any new route or storage method that performs a database transaction MUST follow this checklist:

### Rule 1: Always set BOTH session variables in every transaction
```typescript
// CORRECT — use the canonical helper:
await OrgDatabase.setRlsContext(tx, organizationId, userRole);

// CORRECT — inline equivalent:
await tx.execute(sql`SELECT set_config('app.current_org_id', ${orgId}, true)`);
await tx.execute(sql`SELECT set_config('app.user_role', ${dbRole}, true)`);

// WRONG — missing app.user_role:
await tx.execute(sql`SELECT set_config('app.current_org_id', ${orgId}, true)`);
```

NEVER set `app.current_org_id` without also setting `app.user_role` in the same transaction.

### Rule 2: Role value to use — lookup table

| Context | Role value to use |
|---|---|
| Route with `req.orgUser.role` available | `req.orgUser?.role ?? 'admin'` |
| Admin-only route, no orgUser (platform super) | `'admin'` |
| Storage method, SELECT-only, no user context | `'readonly'` |
| Storage method, INSERT/UPDATE, no user context | `'write'` |
| FHIR middleware | `'write'` |
| `withPlatformContext()` | automatic — always sets `'admin'` |

### Rule 3: mapToDbRole() mapping
Defined in `server/orgDb.ts`. Maps the 7-value app `roleEnum` to 3 DB tiers:

| App role | DB tier |
|---|---|
| `platform_super`, `admin` | `'admin'` |
| `manager`, `supervisor`, `sales`, `marketing` | `'write'` |
| `user` or unknown | `'readonly'` |

### Rule 4: Helper reference
```typescript
// Transaction-based routes (most common):
await OrgDatabase.setRlsContext(tx, orgId, role);

// Connection-level (withOrgContext):
await OrgDatabase.withOrgContext(orgId, operation, userRole);

// Platform super user (automatically sets 'admin'):
await OrgDatabase.withPlatformContext(operation, reason);
```

All helpers defined in `server/orgDb.ts`.

### DEPRECATED helpers — NEVER call these

| Method | Reason | Replacement |
|--------|--------|-------------|
| `OrgDatabase.withOrgContext(orgId, fn)` | Pool isolation bug: `SET LOCAL` runs on a different connection than `fn()`. Returns unfiltered data silently. **Throws unconditionally as of March 04, 2026.** | `db.transaction(async (tx) => { await OrgDatabase.setRlsContext(tx, orgId, role); ... })` |
| `OrgDatabase.withPlatformContext(fn)` | Same pool isolation bug. No active call sites. **Throws unconditionally as of March 04, 2026.** | `db.transaction(async (tx) => { await OrgDatabase.setPlatformContext(tx); ... })` |

Both methods throw `Error` immediately — they cannot silently fail. Any remaining call site will surface as an unhandled exception during development.

---

## Mandatory: Post-Change Quality Check

After any change that touches more than one file, or that adds new code calling an existing API, run the LSP diagnostic check before declaring work complete:

```javascript
const result = await getLatestLspDiagnostics();
const files = Object.keys(result.diagnostics);
console.log(`Files with diagnostics: ${files.length}`);
for (const [path, errors] of Object.entries(result.diagnostics)) {
  console.log(`\n=== ${path} (${errors.length} issues) ===`);
  for (const e of errors) {
    console.log(`  Line ${e.startLine}: [${e.severity}] ${e.message}`);
  }
}
```

This queries the same TypeScript language server the editor uses. **Zero errors expected before work is delivered.**

### Before writing new code that calls an existing API

Grep the relevant directory first to find the established pattern:

```bash
grep -r "db\.execute" server/services/        # e.g. before using db.execute()
grep -rn "apiRequest\|useMutation" client/src/ # e.g. before adding a mutation
```

Copying an existing call site prevents type errors that only surface in the editor (e.g. `(result as any).rows` on `db.execute()` results — the Drizzle return type does not expose `.rows` in a typed way; the cast is always required).

### When to run

| Trigger | Run check? |
|---|---|
| New file created | Yes |
| Existing file with >10 lines changed | Yes |
| Import added or changed | Yes |
| Single-line fix or comment only | No |

---

## Mandatory: RLS Policy Rules

### TO clause — always public
ALL RLS policies MUST use `TO public` in the TO clause.
NEVER use custom role names (`app_admin`, `app_write`, `app_readonly`) in `TO`.

**Reason:** Replit reads `pg_policies` directly from Neon prod to generate provision SQL for each deployment. Custom role names in `TO` cause `"role does not exist"` on the fresh deployment database at provision time → deployment fails.

`TO public` is a built-in PostgreSQL keyword — always present on any database.

### Role differentiation — session variable in USING clause
When a policy must restrict by role, put the check in the `USING` clause:
```sql
-- CORRECT:
CREATE POLICY "policy_name" ON "table_name"
  AS PERMISSIVE FOR DELETE TO public
  USING (
    (organization_id = (current_setting('app.current_org_id'::text, true))::uuid)
    AND (current_setting('app.user_role'::text, true) = 'admin'::text)
  );
```

### Current policy state (as of migration 0005)
- Both dev (`$DATABASE_URL_DEV`) and Neon prod (`$DATABASE_URL_PROD`) have **38 policies**.
- All 38 use `TO public`.
- `medical_records_admin_only` (DELETE on `patient_medical_records`) additionally checks `app.user_role = 'admin'`.
- Dev/prod policy diff must be **DIFF CLEAN** before every publish (pchk.md Step 9a).

---

## Mandatory: Migration Process

NEVER use drizzle-kit. ALL migrations are manual SQL via psql.

> Current migration counts (Dev/Prod) and applied migration history are tracked in `deliverables.md`.

> **Dev and prod databases intentionally diverge during development. This is expected
> and correct. Prod is brought in sync exclusively as a pchk.md activity.**

### 5-step process — dev migration (during development)

**Step 1:** Create `migrations/NNNN_descriptive_name.sql` with a header comment explaining why the migration exists.

**Step 2:** Apply to dev:
```bash
psql "$DATABASE_URL" -f migrations/NNNN_name.sql
```
Note: use `$DATABASE_URL` (Replit owner connection) for DDL; `$DATABASE_URL_DEV` is app_write and cannot CREATE TABLE.

**Step 3:** Compute hash:
```bash
sha256sum migrations/NNNN_name.sql
```

**Step 4:** Insert tracking row on dev:
```bash
psql "$DATABASE_URL" -c "INSERT INTO drizzle.__drizzle_migrations (hash, created_at) VALUES ('<hash>', <unix_millis>);"
```

**Step 5:** Update `migrations/meta/_journal.json` — add new entry:
```json
{
  "idx": N,
  "version": "7",
  "when": <unix_millis>,
  "tag": "NNNN_descriptive_name",
  "breakpoints": true
}
```

> **STOP HERE during development.** Applying a migration to prod during active
> development is prohibited. Prod migration is a pchk.md activity only — it
> happens when the feature is complete and the full pre-publish checklist is run.

---

## Mandatory: Pre-Publish Checklist

Read `pchk.md` before EVERY publish. Run log goes in `publishlog.md`.

**Critical invariants to verify every time:**
- Step 0: Confirm both DB connections live and pointing to correct databases before running any diff steps.
- Step 9a: Dev vs prod policy diff → must be **DIFF CLEAN**
- Step 9b: Dev vs prod `rowsecurity` diff → must be **DIFF CLEAN**
- Step 9c: No custom role references in TO clause → `grep app_admin/app_write/app_readonly` must return nothing

**If any step fails:** stop, do not publish, diagnose and fix first.

---

## Mandatory: UI Rules

- **EVERY user-facing message goes through `notify()` — no exceptions.** This includes errors, success confirmations, warnings, and info toasts. Direct calls to `toast()`, `useToast()`, or any other notification primitive are forbidden in component and page code. `notify()` is the single public API. The internal toaster is private to the notification handler. This rule applies to all new code immediately and to existing code when the `notify()` system is built (see `future.md` — Unified Notification & Error Handling System).

- **Minimum font size: 16px (`text-base`).** NEVER use `text-sm` in content areas. `text-xs` is permitted only for badges and purely decorative elements, or when overridden by the developer.
- **Minimum font size for non-UX display text: 18pt (18px / `text-lg`).** Textareas, accordion content body, and MDX editor content must use at least `text-lg`. The 16px minimum applies to UX chrome; this rule is a more specific override for readable content areas.
- **Minimum icon size for clickable icons: `h-6 w-6` (24px).** All icons inside `<Button>`, `<DropdownMenuTrigger>`, `<button onClick>`, or any element with an `onClick` handler must have minimum dimensions `h-6 w-6`. Purely decorative / status icons (section headers, sync indicators, lock icons, completion dots) are exempt.
- **Mobile-first, offline-first (Capacitor iOS/Android WebView).** No external links. No network-dependent routing. All assets stored within the app.
- **Single scroll owner per page.** Use `overflow-hidden` on parent flex containers. Use `overflow-y-auto` exclusively on the innermost content wrapper (Layer 5). Require `min-h-0` on flexible children. This prevents competing scroll containers on mobile.
- **Accordion mutual exclusion — mandatory.** Whenever two or more `<AccordionItem>`s appear in the same visual group, they MUST share a single `<Accordion type="single" collapsible>` parent. Never wrap each item in its own `<Accordion>` — doing so makes items independent and breaks the one-open-at-a-time contract. Non-accordion siblings (plain cards, toggles) may be placed as direct children of the shared `<Accordion>` root; they render normally without joining the accordion state.
- **Radix `<Select>` controlled value — mandatory optimistic pattern.** When a `<Select>`'s `value` prop is derived from server state (`useQuery`), the trigger display lags the full network round-trip after a tap. On a Capacitor native device this looks like nothing happened even though the PATCH succeeds. **Every such Select MUST use a local `useState` so the display updates instantly:** `const [localVal, setLocalVal] = useState<string | null>(null)` and compute the effective value as `localVal ?? serverData?.field ?? "fallback"`. In `onValueChange`, call `setLocalVal(v)` before `mutation.mutate(v)`. In `onError`, call `setLocalVal(null)` to revert to the server value. **Exemption:** Selects that call `updateSetting(key, v)` or a similar synchronous local-state handler (e.g. `useJournalSettings`) are already backed by `useState` in the hook — no additional local state is needed in those components.

### Z-Index Layer System

> **Mobile work:** This table governs `client/src` (the web admin / web reference prototype). For the GTD mobile app (`apps/mobile`), the same layer values apply but as `zIndex: <number>` in React Native StyleSheet — see `sysspec.md` Section 10.
>
> **`client/src/pages/todo/` is a web reference prototype** for the Expo mobile screens. It is being superseded by `apps/mobile` and will eventually be removed. Do NOT invest effort in todo-page improvements; build the equivalent feature in `apps/mobile` instead. The component names listed in the table below are reference mappings from the web prototype only; the canonical implementation lives in `apps/mobile`.

All stacking contexts must use one of the canonical layers below. **No new z-index value may be introduced outside these layers without updating this table first.**

| Value | Tailwind class | Purpose | Reference mappings in web prototype (`client/src`) — not canonical |
|-------|---------------|---------|-------------------|
| 0 | `z-0` | Background video / image | — |
| 5 | `z-[5]` | Current page bg color overlay (translucent) | — (not yet built in web prototype) |
| 10 | `z-10` | Home page carousel container | `DomainCarousel` container |
| 15 | `z-[15]` | Home page carousel panels and badges | `DomainCarousel` notification badge |
| 20 | `z-20` | Current page content (scrolls underneath header) | Page content wrapper in `App.tsx`, loading overlay in `App.tsx` |
| 30 | `z-[30]` | Header bg color overlay (translucent) | — (not yet built in web prototype) |
| 32 | `z-[32]` | Header chrome (back arrow, title, squirrel, kabab) + floating bottom tab bar | `AppHeader`, `AppBottomBar` wrapper in `App.tsx` |
| 34 | `z-[34]` | Header progress bar | — (not yet built in web prototype) |
| 36 | `z-[36]` | Header achievement dots | — (not yet built in web prototype) |
| 40 | `z-40` | Full-panel menus / settings (solid bg, slides in from left) | — (not yet built in web prototype) |
| 45 | `z-[45]` | Inline popovers, dropdowns, tooltips, selects, context menus | `PopoverContent`, `SelectContent`, `ContextMenuContent`, `ContextMenuSubContent`, `DropdownMenuContent`, `DropdownMenuSubContent`, `TooltipContent`, `MenubarContent`, `MenubarSubContent`, search dropdown in `AppMainView.tsx`, tag suggestion dropdown in `Jot.tsx` |
| 50 | `z-50` | Modal background overlay (translucent backdrop) | `DialogOverlay`, `AlertDialogOverlay`, `DrawerOverlay`, `SheetOverlay` |
| 52 | `z-[52]` | Modal form content | `DialogContent`, `AlertDialogContent`, `DrawerContent`, `SheetContent` |
| 58 | `z-[58]` | Toast notifications | `ToastViewport` |
| 60 | `z-[60]` | Celebration bg color overlay (translucent) | — (not yet built in web prototype) |
| 65 | `z-[65]` | Celebrations stack (multi-layer, Pomodoro) | `Celebration.tsx` outer fixed container, `PomodoroTimer.tsx` celebration overlay |

- **NEVER use the Tailwind `prose` class anywhere in the GTD app.** It overrides font sizes and conflicts with the 16px minimum rule and `gtd-font-big/bigger/biggest` scaling. (The `.mdx-inline-editor .prose` / `.mdx-content-editor .prose` rules in `index.css` are platform editor areas, not the GTD app, and are exempt.)
- **`alert-dialog.tsx` is customized.** Do NOT revert to shadcn defaults.

---

## Mandatory: Security Rules

- **TOTP MFA is mandatory** for all org user sessions. Never bypass or weaken.
- **PHI data** uses AES-256-GCM application-level encryption. Medical records are PHI.
- **Admin-only operations** MUST be gated at BOTH levels:
  1. Application level: route guard checking `req.orgUser.role === 'admin'`
  2. Database level: `app.user_role = 'admin'` in transaction context
- **Never display, log, or expose** actual values of secrets or API keys.
- **Dev/prod secret isolation:** All secrets are Replit global (not env-scoped). See `devprodseparation.md` for the full risk table. Keys requiring `_DEV`/`_PROD` variants: Object Storage, encryption keys, session secret, OAuth RSA keys, Stripe keys, access codes.

---

## Mandatory: HIPAA & HL7 FHIR R4 Compliance

### What This Application Is

This is a **HIPAA-covered entity application** handling Protected Health Information (PHI) for healthcare practices. Every feature that touches patient data is subject to the HIPAA Security Rule and Privacy Rule — not as a future goal, but right now, for every line of code written today.

The patient record layer is structured as **HL7 FHIR R4 resources** (Patient, Observation, Condition, Encounter, etc.) served via the FHIR API at `server/services/fhir/`. All FHIR resources are PHI.

**This compliance context applies to every AI session and every developer. Read it before touching any feature that involves users, organizations, or medical data.**

---

### What Counts as PHI in This System

The following tables and data types are PHI. Apply all PHI rules below to anything in this list:

- `patient_medical_records` — primary PHI table; admin-only DELETE enforced via RLS
- All FHIR R4 resources (Patient, Observation, Condition, Encounter, etc.)
- Any field containing: patient name, date of birth, diagnosis, medication, treatment, provider notes, contact information linked to health data, insurance details, or any combination of fields that could identify a patient alongside their health status

PHI is not limited to these tables. When in doubt, treat it as PHI.

---

### PHI Handling — NEVER

- **NEVER log PHI.** `console.log`, server logs, error messages, and stack traces must never contain PHI values. Log record IDs and org IDs only. A patient object printed to the server log is a HIPAA violation.
- **NEVER include PHI in error messages** returned to the client. Return a generic error code and log only the incident ID server-side.
- **NEVER write PHI into source code**, test fixtures, seed files, or migration SQL. All dev/test data must be fabricated (synthetic names, fake DOBs, nonsense diagnoses).
- **NEVER send raw PHI to external AI APIs** (Gemini, OpenAI, or any future integration). If an AI feature needs to process patient data, de-identify it first under HIPAA Safe Harbor (remove name, DOB, MRN, contact info, geographic data below state level, dates more specific than year for patients over 89).
- **NEVER expose PHI in URL query parameters or route path segments.** Use opaque IDs only.
- **NEVER cache PHI in browser storage** (localStorage, sessionStorage, IndexedDB) without AES-256-GCM encryption applied at the application layer.
- **NEVER approve a Replit-generated migration** without reading the SQL first — a destructive migration touching PHI tables could corrupt live patient records.

---

### PHI Handling — MUST

- **MUST encrypt PHI** at the application level with AES-256-GCM before storing. Use `ENCRYPTION_KEY_V1_PROD` / `ENCRYPTION_KEY_V2_PROD` in prod and `ENCRYPTION_KEY_V1_DEV` / `ENCRYPTION_KEY_V2_DEV` in dev. `server/encryptionService.ts` selects the correct variant automatically via `NODE_ENV`. The old unscoped `ENCRYPTION_KEY_V1` / `ENCRYPTION_KEY_V2` remain as fallbacks only — the scoped variants take precedence. **Set the 4 scoped secrets before the next publish** (see `devprodseparation.md`). Resolved March 04, 2026.
- **MUST log PHI access events** (audit trail): record THAT a record was accessed, by whom (user ID + org ID), and when — but never log the PHI content itself.
- **MUST gate all PHI access behind TOTP MFA.** No bypass for any role including `platform_super`.
- **MUST apply the minimum necessary standard:** queries must only SELECT the fields needed for the current operation. Do not `SELECT *` on PHI tables when a subset of columns is sufficient.
- **MUST scope all PHI queries to a single organization** via `organization_id`. Cross-org PHI access is a HIPAA breach.

---

### Dev Database Rules — HIPAA Critical

- The dev database (`heliumdb` / Replit built-in PostgreSQL) **MUST NEVER contain real patient data.** Use only fabricated synthetic data in dev.
- **NEVER copy production data to dev** for debugging, even temporarily. This is a HIPAA violation regardless of intent. Generate synthetic equivalents instead.
- The Replit deploy setting **"Set up prod DB with dev data" MUST remain unchecked at all times.** This is enforced in `pchk.md`. The reason is PHI: enabling it risks copying prod PHI into dev and dev data into prod.
- If a prod `psql` query is run for debugging and the output contains PHI rows, **do NOT save that output to any file in this repository.** Close the terminal session when finished.

---

### HL7 FHIR R4 Rules

- **ALL FHIR endpoints MUST require authentication.** No FHIR resource may be returned to an unauthenticated request under any circumstance.
- **ALL FHIR resources MUST be org-scoped** via `organization_id`. The FHIR middleware uses `'write'` role in `setRlsContext()` — do not change this.
- **FHIR R4 resource structure must not be modified** to remove required fields that support patient safety (e.g., `subject` reference, `status`, `effectiveDateTime` on Observations).
- **SMART on FHIR `ISSUER_URL`** must use `_DEV`/`_PROD` variants. A wrong issuer in cross-environment tokens causes silent auth failures on FHIR endpoints (see `devprodseparation.md`).
- FHIR R4 resources are the authoritative patient record. Any code that writes to FHIR resources is writing to PHI — apply all PHI rules above.

---

### AI Integration Restrictions (HIPAA)

- Gemini and any future AI integrations **MUST NOT receive raw PHI.**
- If an AI feature summarizes or analyzes patient data: (1) de-identify first, (2) use only the de-identified version, (3) do not store the AI response back into PHI tables.
- The current Gemini config uses `_DUMMY_API_KEY_` in prod (AI features are non-functional in prod — see `devprodseparation.md`). When fixing this, create `AI_INTEGRATIONS_GEMINI_API_KEY_DEV` and `AI_INTEGRATIONS_GEMINI_API_KEY_PROD` as separate secrets. The prod key must never be accessible from the dev environment.

---

### Breach Protocol

If credentials, PHI, or encryption keys are believed to have been exposed:

1. **Immediately rotate** the compromised secret in Replit's secrets panel.
2. **If PHI was exposed:** treat as a HIPAA breach. Document: what data, how many patients affected, how the exposure occurred, and the timestamp of discovery.
3. **Do NOT attempt to recreate or debug** the breach in the dev environment using prod data.
4. **Flag to the user immediately** — do not attempt to silently remediate a potential PHI exposure.

---

## DailyTasks — Scheduled Maintenance

`server/services/dailyTasks.ts` — runs once per UTC calendar day, triggered by `/api/health`.

**Heartbeat:** An external cronjob pings `GET /api/health` every **10 minutes** to keep the server alive. Two pings fall inside the preferred 00:00–00:15 UTC window (at :00 and :10), so tasks fire in-window on a healthy server. If the server is down for the full 15-minute window, tasks run on the first ping after restart (catch-up mode). The `app_daily_task_log` sentinel ensures tasks run at most once per UTC day.

**Task files (all driven by `dailyTasks.ts`):**

| File | Scope | Current tasks |
|---|---|---|
| `dailyPlatTasks.ts` | Platform | `taskPurgeOldErrorLogs` — DELETE `error_logs` > 90 days |
| `dailyOrgTasks.ts` | Org | _(empty — placeholder for org-level maintenance)_ |
| `dailyAppTasks.ts` | App | `taskPurgeTodayVerses` — DELETE `bible_verses` WHERE `is_today = true`; `taskArchiveJournals` — move journal/memory rows > 1 year old from `app_matrix` → `app_matrix_archive` |

**Adding a new task:** Add an `async function taskXxx(): Promise<string>` to the appropriate file and push its result inside that file's `runDailyXxxTasks()`. Return a human-readable summary string. The sentinel and logging in `dailyTasks.ts` are already handled.

---

## Reference Documents

| File | Purpose |
|---|---|
| `pchk.md` | Full pre-publish checklist (225 lines) |
| `devprodseparation.md` | Dev/prod secret separation audit (406 lines) |
| `userpref.md` | User communication and workflow preferences |
| `deliverables.md` | Completed work log — migration state, resolved tasks, architecture phases |
| `future.md` | Deferred decisions and future features |
| `migrations/meta/_journal.json` | Migration history and idx tracking |
| `server/orgDb.ts` | OrgDatabase class: setRlsContext, mapToDbRole, withOrgContext, withPlatformContext |
| `shared/schema/policies.ts` | All 38 RLS policy definitions |
