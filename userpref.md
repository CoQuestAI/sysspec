# OrgPro User Preferences
Preferences and working style for all AI sessions on this project.

---

## Communication Style
- Use simple, everyday language. Match the user's language (English or French).
- User is an App Architect and BA. Technical jargon is fine.
- Calm, professional tone. Acknowledge specific points — no blanket praise or flattery.
- Keep responses measured. Do not over-explain or repeat information unnecessarily.

## Code Change Notifications
When making any code changes, always state:
- Which files were updated
- Which line numbers were modified

## Browser Testing
The user always does a **hard refresh (Ctrl+F5)** after changes. Do NOT assume HMR or soft refresh is the issue if changes are not visible.

## Screenshot Annotations
Pink or colored markings on attached screenshots are the **user's annotations** highlighting areas of interest. They are NOT part of the app UI. Never treat them as UI elements.

## Target Platform
Mobile app — **offline-first, Capacitor iOS/Android WebView**.
- All navigation is internal. No external links.
- No network-dependent routing.
- All assets stored within the app.
- See `sysspec.md` for mandatory UI rules that follow from this.

## Database Operations
ALL production database operations use direct `psql` commands via bash:
```bash
psql "$DATABASE_URL_PROD" -c "SELECT ..."
```
This applies to: pre-publish checklist, schema checks, production syncs, and all other prod queries. The built-in SQL tool is unstable with prod.

For dev-only operations, any tool (built-in SQL tool, psql, etc.) is fine.

See `sysspec.md` for the PG\* environment variable danger rules — mandatory reading.

## Plan Execution
When the user approves a proposed session plan, execute it exactly as described in the proposal — no silent deviations. If what was proposed and what gets coded differ, that is a violation of the approved plan. Re-proposing a modified plan is encouraged when scope changes; any new proposal requires fresh user approval before execution. 

## Diffs Discovered During Plan Execution
User makes code modifications which may cause diffs. If a clarification is needed mid-execution, ask concisely and wait for the answer before continuing.

## Pre-Publish Process
Always read `pchk.md` at the project root before running the checklist.
Run log is stored in `publishlog.md`.
`drizzle-kit` is permanently disabled — `npm run db:push` intentionally fails.

## Dev/Prod Environment Separation
Full audit of all shared secrets between dev and prod is documented in `devprodseparation.md`. Replit secrets are global (not env-scoped), so all secrets are shared unless deliberately split into `_DEV`/`_PROD` variants.

## File Operations
- **Searching** (content or filename patterns): always use the search tool (ripgrep); never use bash `grep` or `find`.
- **All other file operations** (move, copy, rename, delete, run scripts): use bash.

## Project Task Queries
When listing tasks for planning, dchk sweeps, or mchk scans, always filter for PROPOSED and IN_PROGRESS only. MERGED tasks accumulate permanently — `updateProjectTask` silently no-ops on MERGED state (platform limitation confirmed March 17, 2026). The MERGED list is silent historical noise and cannot be pruned. Never query MERGED state for actionable task work.
