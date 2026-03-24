# Dev / Prod Environment Separation

## Overview

This document records the full audit of shared state between the development and
production environments, conducted March 2026. The root issue: **Replit secrets are
global and not environment-scoped.** Every secret is identical in both dev and prod
unless it is deliberately split into separate named secrets (e.g. `_DEV` / `_PROD`
variants). Environment variables (non-secrets) can be scoped, but this project uses
no env vars — only secrets.

---

## What Is Already Correctly Separated

| Item | Dev value | Prod value | Status |
|------|-----------|------------|--------|
| PostgreSQL databases | `DATABASE_URL_DEV` → app_write/heliumdb (Replit built-in) | `DATABASE_URL_PROD` → app_write/Neon prod | ✅ Confirmed separated — `server/db.ts` reads `DATABASE_URL_DEV ?? DATABASE_URL` in dev, `DATABASE_URL_PROD ?? DATABASE_URL` in prod |
| `NODE_ENV` | Not set in shell; start script injects `development` into the node process | `production` — confirmed in deployment logs: `NODE_ENV=production node dist/index.js` | ✅ Confirmed different |
| `REPL_ID` | `9d5c547d-c680-43a4-a01c-fb2c4e78c89d` | Not logged in prod — almost certainly **the same** | ⚠️ Likely SHARED — identifies the Repl workspace, not the environment. Do not use this to distinguish dev from prod. |
| `REPLIT_DOMAINS` | `9d5c547d-...picard.replit.dev` (dev tunnel) | Not logged in prod — expected to be the `.replit.app` domain | ⚠️ Likely different but **unverified** |
| `REPLIT_DEV_DOMAIN` | `9d5c547d-...picard.replit.dev` (same as above) | Not logged in prod — likely absent or a different value | ⚠️ Likely different/absent in prod but **unverified** |

> **Note:** To confirm `REPLIT_DOMAINS`, `REPLIT_DEV_DOMAIN`, and `REPL_ID` in
> production, the app would need to log these values at startup. Currently it
> does not. Add a startup log line if verification is needed.

---

## Key Principles Established

- **Dev data is always corrupt/test data.** It is never synced to prod.
- **Schema changes** flow dev → prod via the `pchk.md` psql process.
- **Row data** never flows dev → prod. Production data is entered directly in prod
  (via the app UI or deliberate prod-targeted psql commands).
- **Supabase and Object Storage files** are cloud-hosted resources. The URL stored
  in a database column is what matters. If dev and prod point to the same bucket,
  dev test files are visible to prod users.
- **Stripe key mode** (test `sk_test_...` vs live `sk_live_...`) must be confirmed
  before any payment testing. Live keys in dev = real money at risk.

---

## Object Storage Isolation Plan

### Problem
`DEFAULT_OBJECT_STORAGE_BUCKET_ID`, `PUBLIC_OBJECT_SEARCH_PATHS`, and
`PRIVATE_OBJECT_DIR` are single global secrets — dev and prod share the same bucket.
Test files uploaded in dev appear in prod.

### Solution
Create `_DEV` and `_PROD` variants for all three secrets.

**Secrets to create:**
- `DEFAULT_OBJECT_STORAGE_BUCKET_ID_DEV` — copy of current value
- `DEFAULT_OBJECT_STORAGE_BUCKET_ID_PROD` — new prod bucket ID (from Replit UI)
- `PUBLIC_OBJECT_SEARCH_PATHS_DEV` — copy of current value
- `PUBLIC_OBJECT_SEARCH_PATHS_PROD` — new prod bucket paths
- `PRIVATE_OBJECT_DIR_DEV` — copy of current value
- `PRIVATE_OBJECT_DIR_PROD` — new prod bucket private dir

**Secrets to delete (after cutover):**
- `DEFAULT_OBJECT_STORAGE_BUCKET_ID`
- `PUBLIC_OBJECT_SEARCH_PATHS`
- `PRIVATE_OBJECT_DIR`

### Steps (must be done atomically — do not delete originals until all steps complete)

1. Create new prod bucket via Replit UI (Object Storage tool pane) → note new bucket ID
2. Populate all `_PROD` secrets with new bucket values
3. Populate all `_DEV` secrets with current (copied) values
4. Update code in two files to select `_DEV` vs `_PROD` based on `NODE_ENV`:
   - `server/objectStorage.ts`
   - `server/replit_integrations/object_storage/objectStorage.ts`
   - Logic: `const isProd = process.env.NODE_ENV === 'production';`
   - Then read `PUBLIC_OBJECT_SEARCH_PATHS_DEV` or `PUBLIC_OBJECT_SEARCH_PATHS_PROD`, etc.
   - The `Storage` client init may need explicit bucket ID passed rather than
     relying on SDK auto-discovery of `DEFAULT_OBJECT_STORAGE_BUCKET_ID` by exact name
5. Restart workflow and verify dev Object Storage still works
6. Deploy to prod and verify prod Object Storage works with new bucket
7. Delete the three original (unscoped) secrets

### Note on existing files
The current bucket contains dev test files. These are dev data and should remain
in the dev bucket. The new prod bucket starts empty — correct behavior.

---

## Confirmed Dev vs Prod Comparison — March 03, 2026

Live comparison performed using `printenv` in the dev shell and the full prod
secrets list exported from the Replit deployment settings.

| Secret | Same or Different | Notes |
|--------|------------------|-------|
| `DATABASE_URL` | ✅ **Auto-injected** | NOT user-set — Replit auto-injects it (built-in PostgreSQL). Setting it as a user secret triggers "External database detected" and blocks publishing. App code prefers `DATABASE_URL_DEV` in dev and `DATABASE_URL_PROD` in prod; `DATABASE_URL` is only used as fallback. |
| `CONNECTORS_HOSTNAME` | ✅ **Deployment-only** | Not set in dev; `connectors.replit.com` in prod only |
| `REPLIT_CONNECTORS_HOSTNAME` | ✅ **Deployment-only** | Same as above — not set in dev |
| `SESSION_SECRET` | ✅ **RESOLVED (March 04, 2026)** | `server/sessionManager.ts`, `server/replitAuth.ts`, and `server/services/fhir/standalone.ts` now read `SESSION_SECRET_DEV` in dev and `SESSION_SECRET_PROD` in prod, falling back to the unscoped `SESSION_SECRET`. `SESSION_SECRET_PROD` is a new random value — dev sessions are no longer cryptographically valid in prod. |
| `ENCRYPTION_KEY_V1` | ✅ **RESOLVED (March 04, 2026)** | `server/encryptionService.ts` now reads `ENCRYPTION_KEY_V1_PROD` in prod and `ENCRYPTION_KEY_V1_DEV` in dev. Old `ENCRYPTION_KEY_V1` kept as fallback only. Set `ENCRYPTION_KEY_V1_PROD` = current value (preserves prod PHI decryption); set `ENCRYPTION_KEY_V1_DEV` = new random key (dev cannot decrypt prod PHI). **Action required: set the 4 new secrets in Replit before next publish.** |
| `ENCRYPTION_KEY_V2` | ✅ **RESOLVED (March 04, 2026)** | Same as V1 — reads `ENCRYPTION_KEY_V2_PROD` in prod, `ENCRYPTION_KEY_V2_DEV` in dev. Set `ENCRYPTION_KEY_V2_PROD` = current value; `ENCRYPTION_KEY_V2_DEV` = new random key. |
| `STRIPE_SECRET_KEY` | 🟡 **IDENTICAL** | Both = `sk_test_51SIes...` — test-mode, safe pre-launch; must split before go-live |
| `VITE_STRIPE_PUBLIC_KEY` | 🟡 **IDENTICAL** | Both = `pk_test_51SIes...` — same |
| `INITIAL_ACCESS_CODE_1` | 🔴 **IDENTICAL** | Both = `A1!asdfasdfasdf` — placeholder value, shared across environments |
| `INITIAL_ACCESS_CODE_2` | 🔴 **IDENTICAL** | Both = `A2!asdfasdfasdf` — same |
| `AI_INTEGRATIONS_GEMINI_BASE_URL` | 🔴 **IDENTICAL** | Both = `http://localhost:1106/modelfarm/gemini` — localhost only works in dev; AI is broken in prod |
| `AI_INTEGRATIONS_GEMINI_API_KEY` | 🔴 **IDENTICAL** | Both = `_DUMMY_API_KEY_` — AI calls fail in prod |
| `DEFAULT_OBJECT_STORAGE_BUCKET_ID` | ✅ **SPLIT** | `_DEV` = original dev bucket; `_PROD` = `replit-objstore-0e30edf9-abf3-4241-bf29-2d8864c64a81` (bucket "Prod"). Code reads scoped variant per `NODE_ENV`. Resolved March 04, 2026. |
| `PUBLIC_OBJECT_SEARCH_PATHS` | ✅ **SPLIT** | `_DEV` = dev bucket paths; `_PROD` = `/replit-objstore-0e30edf9-abf3-4241-bf29-2d8864c64a81/public`. Resolved March 04, 2026. |
| `PRIVATE_OBJECT_DIR` | ✅ **SPLIT** | `_DEV` = dev bucket private dir; `_PROD` = `/replit-objstore-0e30edf9-abf3-4241-bf29-2d8864c64a81/.private`. Resolved March 04, 2026. |
| `DATABASE_URL_DEV` | 🟡 **IDENTICAL** | Both = app_write/heliumdb — expected; this is the dev database connection string used by the running app (NODE_ENV=development); same value in both environments by design |
| `DATABASE_URL_PROD` | 🟡 **IDENTICAL** | Both = Neon prod — expected; same value by design |
| `VITE_APP_VERSION` | ✅ **SAFE — OPTIONAL** | Not currently set as a secret. Optional build-time string (e.g. `1.0.0`) embedded via Vite. If set, same value acceptable in both environments. Documented in `.env.example`. |
| `VITE_BUILD_TIMESTAMP` | ✅ **SAFE — OPTIONAL** | Not currently set as a secret. Optional ISO timestamp injected at build/deploy time for error log enrichment. If set, naturally differs per deployment. Documented in `.env.example`. |
| `INSTANCE_ID` | ✅ **SAFE — OPTIONAL** | Not currently set as a secret. Optional identifier for the server instance (falls back to `os.hostname()`). No isolation requirement. Documented in `.env.example`. |
| `PGDATABASE` | ⚠️ **IDENTICAL** | Both = `neondb` — points to Neon prod; see PG* risk note below |
| `PGHOST` | ⚠️ **IDENTICAL** | Both = `ep-twilight-feather-afmxpyty...` — Neon prod host |
| `PGPORT` | ⚠️ **IDENTICAL** | Both = `5432` |
| `PGUSER` | ⚠️ **IDENTICAL** | Both = `neondb_owner` — Neon prod user |
| `PGPASSWORD` | ⚠️ **IDENTICAL** | Both = `npg_XtKrW90VMGnd` — Neon prod password |

### ⚠️ PG* Variables Risk — Dev Shell Connects to Neon Prod

The five PG\* environment variables (`PGDATABASE`, `PGHOST`, `PGPORT`, `PGUSER`,
`PGPASSWORD`) are global secrets that all point to the Neon **production** database.

In the dev shell:
- `DATABASE_URL_DEV` = `helium/heliumdb` (Replit built-in / app_write) — used by the running app
- `PG*` = Neon prod credentials — **NOT** the Replit built-in PostgreSQL

This means any tool, script, or library that uses `PG*` env vars instead of
`DATABASE_URL_DEV` will silently connect to live Neon prod data from within the dev
environment. Critically: **`psql` uses PG\* vars by default when no connection
string is provided.** A bare `psql` command in the dev shell (without specifying
a connection) goes directly to Neon prod.

**Rule:** All `psql` commands in the dev shell and in `pchk.md` must always
use an explicit connection string — `psql "$DATABASE_URL_DEV"` for dev or
`psql "$DATABASE_URL_PROD"` for prod — and never rely on PG\* vars.

---

## Full Secrets Audit

### CRITICAL — Must Separate (Active Security Risk)

| Secret | Risk | Action |
|--------|------|--------|
| `ENCRYPTION_KEY_V1` | Same PHI encryption keys in both environments. HIPAA requires key isolation. Dev key leak exposes prod encryption scheme. | Create `_DEV` and `_PROD` variants |
| `ENCRYPTION_KEY_V2` | Same as above | Create `_DEV` and `_PROD` variants |
| ~~`OAUTH_RSA_PRIVATE_KEY`~~ | ~~JWT tokens signed in dev are cryptographically valid in prod (and vice versa).~~ | ✅ **RESOLVED (March 05, 2026)** — Unique RSA-2048 key pairs generated and stored as `OAUTH_RSA_PRIVATE_KEY_DEV` / `OAUTH_RSA_PUBLIC_KEY_DEV` and `OAUTH_RSA_PRIVATE_KEY_PROD` / `OAUTH_RSA_PUBLIC_KEY_PROD`. `jwtKeyService.ts` reads the scoped variant based on `NODE_ENV`. Dev JWTs are no longer valid in prod. |
| ~~`OAUTH_RSA_PUBLIC_KEY`~~ | ~~Same as above~~ | ✅ **RESOLVED (March 05, 2026)** — See above. |
| ~~`SESSION_SECRET`~~ | ~~Session cookies from dev are valid in prod.~~ | ✅ **RESOLVED (March 05, 2026)** — `SESSION_SECRET_DEV` / `SESSION_SECRET_PROD` variants created and wired in `sessionManager.ts`, `replitAuth.ts`, `fhir/standalone.ts`. Dev sessions no longer valid in prod. |
| `INITIAL_ACCESS_CODE_1` / `INITIAL_ACCESS_CODE_2` | ✅ **RESOLVED (March 05, 2026)** — Seed logic updated to read `_DEV`/`_PROD` scoped variants. Dev DB: "Initial Setup" revoked, "PJ" active (confirmed Feb 16, 2026). Prod DB: identical state confirmed via `gtdpro.replit.app` March 05, 2026. Both environments secured. | No secrets to create — running databases already have custom active pairs. |
| ~~`STRIPE_SECRET_KEY`~~ | ~~**Confirmed test-mode** (`sk_test_51SIes...`) — correct for pre-launch. Must switch to `sk_live_...` before accepting real payments.~~ | ✅ **CODE COMPLETE (March 05, 2026)** — `server/routes/audiobook.ts`, `server/routes/billing.ts`, `server/routes/platform/billing.ts` all read `STRIPE_SECRET_KEY_PROD` in prod and `STRIPE_SECRET_KEY_DEV` in dev, falling back to `STRIPE_SECRET_KEY`. **Secrets to create:** `STRIPE_SECRET_KEY_DEV` (copy current test key), `STRIPE_SECRET_KEY_PROD` (live key at launch). |
| `VITE_STRIPE_PUBLIC_KEY` | **Confirmed test-mode** (`pk_test_51SIes...`) — correct for pre-launch. Must switch to `pk_live_...` at same time as `STRIPE_SECRET_KEY`. **DEFERRED** — Vite bakes this value at compile time; cannot be split via `_DEV`/`_PROD` pattern until frontend API-injection approach is implemented at go-live. | At go-live: expose via server endpoint and load client-side dynamically instead of `import.meta.env`. |
| ~~`STRIPE_WEBHOOK_SECRET`~~ | ~~Dev webhook events route to same signing secret as prod~~ | ✅ **CODE COMPLETE (March 05, 2026)** — `server/routes/billing.ts` and `server/routes/platform/billing.ts` now read `STRIPE_WEBHOOK_SECRET_PROD` in prod and `STRIPE_WEBHOOK_SECRET_DEV` in dev. **Secrets to create:** `STRIPE_WEBHOOK_SECRET_DEV`, `STRIPE_WEBHOOK_SECRET_PROD`. |
| `PGDATABASE` / `PGHOST` / `PGPORT` / `PGUSER` / `PGPASSWORD` | All point to Neon prod. In dev shell, `PG*` = Neon prod while `DATABASE_URL` = helium. Any tool using `PG*` vars in dev (including bare `psql`) silently connects to live prod. | Add awareness; enforce explicit connection strings in all psql commands |

### IMPORTANT — Should Separate (Operational Contamination)

| Secret | Risk | Action |
|--------|------|--------|
| ~~`AI_INTEGRATIONS_GEMINI_BASE_URL`~~ | ~~**LIVE DEFECT IN PROD:** localhost URL only resolves in dev workspace; AI broken in prod.~~ | ✅ **CODE COMPLETE (March 05, 2026)** — `server/replit_integrations/chat/routes.ts`, `server/replit_integrations/image/client.ts`, `server/speechService.ts` all read `AI_INTEGRATIONS_GEMINI_BASE_URL_PROD` in prod and `AI_INTEGRATIONS_GEMINI_BASE_URL_DEV` in dev. **Secrets to create:** `AI_INTEGRATIONS_GEMINI_BASE_URL_DEV` (copy `http://localhost:1106/modelfarm/gemini`), `AI_INTEGRATIONS_GEMINI_BASE_URL_PROD` (real Gemini API endpoint). ⚠️ **Risk:** These files live in `server/replit_integrations/` and may be overwritten if the Gemini AI integration is reinstalled from Replit. Re-apply this pattern after any integration reinstall. |
| ~~`AI_INTEGRATIONS_GEMINI_API_KEY`~~ | ~~**LIVE DEFECT IN PROD:** `_DUMMY_API_KEY_` — AI calls fail with auth error in prod.~~ | ✅ **CODE COMPLETE (March 05, 2026)** — Same three files read `AI_INTEGRATIONS_GEMINI_API_KEY_PROD` in prod, `AI_INTEGRATIONS_GEMINI_API_KEY_DEV` in dev. **Secrets to create:** `AI_INTEGRATIONS_GEMINI_API_KEY_DEV` (copy `_DUMMY_API_KEY_`), `AI_INTEGRATIONS_GEMINI_API_KEY_PROD` (real Gemini API key). Same reinstall risk as above. |
| ~~`APP_URL`~~ | ~~Used in Stripe checkout redirect URLs. Dev and prod have different domains.~~ | ✅ **CODE COMPLETE (March 05, 2026)** — `server/routes/billing.ts` reads `APP_URL_PROD` in prod and `APP_URL_DEV` in dev, falling back to `APP_URL`. **Secrets to create:** `APP_URL_DEV`, `APP_URL_PROD`. |
| ~~`ISSUER_URL`~~ | ~~Used in OIDC discovery for JWT issuer claims. Wrong issuer in cross-env tokens causes auth failures.~~ | ✅ **CODE COMPLETE (March 05, 2026)** — `server/replitAuth.ts` reads `ISSUER_URL_PROD` in prod and `ISSUER_URL_DEV` in dev, falling back to `ISSUER_URL` then `https://replit.com/oidc`. **Secrets to create:** `ISSUER_URL_DEV`, `ISSUER_URL_PROD`. |
| ~~`SEARCH_SALT`~~ | ~~Consistent hash salt makes search indexes predictable and cross-comparable.~~ | ✅ **CODE COMPLETE (March 05, 2026)** — `server/encryptionService.ts` `generateSearchHash()` reads `SEARCH_SALT_PROD` in prod and `SEARCH_SALT_DEV` in dev. **Secrets to create:** `SEARCH_SALT_DEV`, `SEARCH_SALT_PROD`. |
| ~~`CRON_API_KEY`~~ | ~~Dev cron triggers could fire against prod endpoints if the key is shared.~~ | ✅ **CODE COMPLETE (March 05, 2026)** — `server/routes/todo/content.ts` reads `CRON_API_KEY_PROD` in prod and `CRON_API_KEY_DEV` in dev at module level. **Secrets to create:** `CRON_API_KEY_DEV`, `CRON_API_KEY_PROD`. |

### Verify Existence and Isolation

| Item | Status | Action |
|------|--------|--------|
| Resend API key | Not visible in secrets list — may be managed by Resend integration internally | Confirm: dev emails should not go through the prod email account |
| Twilio credentials | Not visible in secrets list — may not be configured yet | When configured, must use separate dev/prod credentials |

### Already Correctly Separated

| Secret | Status |
|--------|--------|
| `DATABASE_URL_DEV` / `DATABASE_URL_PROD` | ✅ Properly isolated — separate Neon databases |
| `DATABASE_URL` | ✅ Isolated via deployment-only override: dev = Replit built-in PostgreSQL (`helium/heliumdb`); prod = Neon prod (`neondb_owner`) — different values per environment by design |
| `NODE_ENV` | ✅ Confirmed different: dev = `development` (injected by start script), prod = `production` (confirmed in deployment logs) |
| `REPL_ID` | ⚠️ Likely SHARED (workspace identifier, not environment-scoped) — do not use to distinguish dev from prod |
| `REPLIT_DOMAINS` | ⚠️ Likely different (dev = picard.replit.dev tunnel, prod = .replit.app) — unverified; app does not log this at startup |
| `REPLIT_DEV_DOMAIN` | ⚠️ Likely absent in prod — unverified |

### Lower Risk / Informational

| Secret | Notes |
|--------|-------|
| `PGDATABASE` | `neondb` — individual Neon connection component, auto-synced. Consistent with DATABASE_URL prod value. |
| `PGHOST` | `ep-twilight-feather-afmxpyt...` — Neon host. Auto-synced. |
| `PGPORT` | `5432` — standard PostgreSQL port. No isolation concern. |
| `PGUSER` | `neondb_owner` — Neon prod user. Auto-synced. |
| `PGPASSWORD` | `npg_XtKrW90VMGnd` — Neon prod password. Auto-synced. These PG* vars are a decomposed form of DATABASE_URL prod — all consistent. |
| `CONNECTORS_HOSTNAME` | `connectors.replit.com` — Replit's internal connector proxy. Deployment-only secret. Identical to `REPLIT_CONNECTORS_HOSTNAME`. |
| `REPLIT_CONNECTORS_HOSTNAME` | `connectors.replit.com` — duplicate of `CONNECTORS_HOSTNAME`. Both deployment-only. Redundant but harmless. |
| `GCS_BAA_VERIFIED` / `GCS_BAA_EXPIRY` | Compliance markers, not credentials. Acceptable to share. |
| `NEON_BAA_VERIFIED` / `NEON_BAA_EXPIRY` | Same as above. |
| `REDIS_URL` | Redis is currently disabled (not configured). When enabled, must be separate. |
| `WEB_REPL_RENEWAL` | Replit-specific. No isolation concern. |
| `ENABLE_CLUSTERING` | Configuration flag. No isolation concern. |
| `GITHUB_TOKEN` | Tooling-only. Used by `sysspec-push.sh` / `sysspec-pull.sh` in the dev shell to sync shared docs to `CoQuestAI/sysspec`. Never referenced by application code; never deployed to prod. Requires `repo` scope on `CoQuestAI/sysspec`. Single token, not split DEV/PROD — no isolation concern. Added March 24, 2026 (Task #142). |

---

## Implementation Priority

1. **Stripe** — confirm test vs live mode immediately. Highest financial risk.
2. **Object Storage** — plan above, requires new bucket creation first.
3. **Encryption keys** — HIPAA obligation. Separate before next HIPAA audit.
4. **Session + OAuth RSA keys** — Security best practice.
5. **Access codes** — Prevents dev testers from accessing prod.
6. **AI, APP_URL, ISSUER_URL, SEARCH_SALT, CRON** — Operational hygiene.

---

## How to Execute Each Split

For each secret being split:
1. Note the current value (user reads it from the Secrets pane)
2. Create `SECRET_NAME_DEV` with the current (dev) value
3. Create `SECRET_NAME_PROD` with the new (prod-specific) value
4. Update server code to read `SECRET_NAME_DEV` or `SECRET_NAME_PROD` based on
   `process.env.NODE_ENV === 'production'`
5. Verify dev still works, deploy, verify prod works
6. Delete `SECRET_NAME` (the original unscoped secret)

---

## Access Code Replacement

### Design

The access gate (`client/src/pages/AccessGate.tsx`) is a **two-person control** —
the user must enter both `code1` and `code2` simultaneously, and they must both
match the same database record (`code1Hash` + `code2Hash` as a pair). Neither
code works alone.

The verify endpoint is `POST /api/access-code/verify` in
`server/routes/platform/accessCodes.ts`. It iterates all active code records and
checks both codes against `bcrypt.compare`. On success it sets
`req.session.accessCodeVerified = true` and calls `storage.markAccessCodeUsed`.

The platform admin API (super user only) supports:
- `GET /api/platform/access-codes` — list all active records (hashes stripped)
- `POST /api/platform/access-codes` — create a new pair (code1, code2, label)
- `DELETE /api/platform/access-codes/:id` — revoke a record

### Current Problem

`INITIAL_ACCESS_CODE_1` = `A1!asdfasdfasdf` and
`INITIAL_ACCESS_CODE_2` = `A2!asdfasdfasdf` — obvious placeholder values,
identical in dev and prod, seeded into both databases at first startup.

### Critical: Env Vars Are Now Inert

`server/index.ts` seeds `INITIAL_ACCESS_CODE_1` / `INITIAL_ACCESS_CODE_2` into
the database **only when there are zero active access code records**
(`if (existing.length === 0)`). Since both dev and prod already have 1 active
record (confirmed in startup logs: `found 1 active codes`), changing the env vars
has **no effect on either environment**. The database records are what the verify
route compares against — not the env vars.

Changing `INITIAL_ACCESS_CODE_1` / `INITIAL_ACCESS_CODE_2` (or creating `_DEV`
/ `_PROD` variants of them) will only matter for a **fresh database with zero
codes** — useful for future environment resets, but it does not fix the currently
running dev or prod databases.

### Correct Fix Path

The replacement must be done via the **platform admin API**, not via env vars.
Execute this for **each environment separately** (dev first, then prod after
publish).

**Step 1 — Generate strong replacement codes**

Each code must pass `passwordComplexitySchema` (same schema used for user
passwords — minimum length, uppercase, lowercase, digit, special character
required). Use a password manager or:
```
node -e "const c=require('crypto');console.log(c.randomBytes(16).toString('base64url')+'A1!')"
```
Run twice — once for code1, once for code2. Store both securely; they cannot be
recovered after hashing.

**Step 2 — Add new code pair (while old one is still active)**

```
POST /api/platform/access-codes
{ "code1": "<new-code1>", "code2": "<new-code2>", "label": "Prod 2026-03" }
```

Must be called as a **super user** (platform admin session). Can be done via:
- Replit shell: `curl -X POST http://localhost:5000/api/platform/access-codes ...`
  with the active session cookie, OR
- The platform admin UI if an access codes management screen exists

**Step 3 — Verify new codes work**

Test the access gate manually with the new codes before deleting the old record.

**Step 4 — Delete the placeholder record**

```
DELETE /api/platform/access-codes/<id-of-old-record>
```

The ID of the old record is visible in the `GET /api/platform/access-codes`
response (label: `Initial Setup`).

**Step 5 — Update env vars for future resets**

Create `INITIAL_ACCESS_CODE_1_DEV`, `INITIAL_ACCESS_CODE_2_DEV`,
`INITIAL_ACCESS_CODE_1_PROD`, `INITIAL_ACCESS_CODE_2_PROD` with the new values
(or a separate set of strong values intended for fresh-DB seeding).

Update `server/index.ts` `seedAccessCodes()` to read the scoped variant per
`NODE_ENV === 'production'`.

### Status

✅ **Code change complete (March 05, 2026)** — `server/index.ts`
`seedAccessCodes()` now reads `INITIAL_ACCESS_CODE_1_DEV` / `_PROD` (falls back
to unscoped). Applies only to fresh databases with zero codes.

✅ **Dev DB replacement confirmed (March 05, 2026)** — Live API check
(`GET /api/platform/access-codes`) confirmed:
- Record `208e5da8` label `"Initial Setup"` — `isActive: false` — revoked Feb 16, 2026
- Record `7d97d8ba` label `"PJ"` — `isActive: true` — custom pair, created Feb 16, 2026

The placeholder pair is inactive and cannot be used at the Access Gate. The dev
database is secured. No further action needed for dev. Do NOT create
`INITIAL_ACCESS_CODE_1_DEV` / `INITIAL_ACCESS_CODE_2_DEV` secrets — they serve
no purpose while the DB already has an active pair.

✅ **Prod DB secured (Feb 16, 2026)** — confirmed via prod UI (`gtdpro.replit.app`)
March 05, 2026:
- Record `Initial Setup` — Revoked — Feb 16, 2026, 12:40:56 AM
- Record `PJ` — Active — used March 05, 2026, 11:50:49 AM

The placeholder pair is inactive in prod. Both dev and prod are fully secured.
No `INITIAL_ACCESS_CODE_1_PROD` / `INITIAL_ACCESS_CODE_2_PROD` secrets are
needed — the running databases already have custom active pairs.

---

## Ongoing Verification — What Replit Can Silently Reset

### Why Active Verification Is Necessary

Replit has been observed resetting or losing configuration in several scenarios:

- **Integration reinstalls** (Object Storage, Gemini, etc.) may restore original
  unscoped secret names (`PUBLIC_OBJECT_SEARCH_PATHS`, etc.), silently undoing the split
- **Workspace resets or Neon branch resets** can lose environment-scoped vars
- **Workspace cloning** does not carry secrets to the new workspace
- **Code in `server/replit_integrations/`** may be overwritten by integration
  updates, restoring hardcoded original env var names
- **Agent sessions** that don't read this document may re-introduce bare secret
  names when adding new features

Any of these events would silently re-merge dev and prod — undetectable without
explicit checks at every publish.

---

## Required Pre-Publish Separation Checks (Step 0)

These checks must be added to `pchk.md` **after all splits are complete**.
Until then, this section is the authoritative verification guide.

Run these as **Step 0** before the existing Steps 1–11, using `viewEnvVars` in
the code execution sandbox.

---

### Step 0a — All `_DEV` secrets must exist

The following secrets must be present (existence check only — values are never read):

```
DEFAULT_OBJECT_STORAGE_BUCKET_ID_DEV
PUBLIC_OBJECT_SEARCH_PATHS_DEV
PRIVATE_OBJECT_DIR_DEV
ENCRYPTION_KEY_V1_DEV
ENCRYPTION_KEY_V2_DEV
SESSION_SECRET_DEV
OAUTH_RSA_PRIVATE_KEY_DEV
OAUTH_RSA_PUBLIC_KEY_DEV
INITIAL_ACCESS_CODE_1_DEV
INITIAL_ACCESS_CODE_2_DEV
STRIPE_SECRET_KEY_DEV
VITE_STRIPE_PUBLIC_KEY_DEV
STRIPE_WEBHOOK_SECRET_DEV
AI_INTEGRATIONS_GEMINI_API_KEY_DEV
AI_INTEGRATIONS_GEMINI_BASE_URL_DEV
APP_URL_DEV
ISSUER_URL_DEV
SEARCH_SALT_DEV
CRON_API_KEY_DEV
DATABASE_URL_DEV
```

Any missing entry = **BLOCK publish, fix before proceeding.**

---

### Step 0b — All `_PROD` secrets must exist

Same list with `_PROD` suffix:

```
DEFAULT_OBJECT_STORAGE_BUCKET_ID_PROD
PUBLIC_OBJECT_SEARCH_PATHS_PROD
PRIVATE_OBJECT_DIR_PROD
ENCRYPTION_KEY_V1_PROD
ENCRYPTION_KEY_V2_PROD
SESSION_SECRET_PROD
OAUTH_RSA_PRIVATE_KEY_PROD
OAUTH_RSA_PUBLIC_KEY_PROD
INITIAL_ACCESS_CODE_1_PROD
INITIAL_ACCESS_CODE_2_PROD
STRIPE_SECRET_KEY_PROD
VITE_STRIPE_PUBLIC_KEY_PROD
STRIPE_WEBHOOK_SECRET_PROD
AI_INTEGRATIONS_GEMINI_API_KEY_PROD
AI_INTEGRATIONS_GEMINI_BASE_URL_PROD
APP_URL_PROD
ISSUER_URL_PROD
SEARCH_SALT_PROD
CRON_API_KEY_PROD
DATABASE_URL_PROD
```

Any missing entry = **BLOCK publish, fix before proceeding.**

---

### Step 0c — Original unscoped secrets must be GONE

The following secrets must **NOT** exist (their presence means a Replit reset
has re-introduced them and dev/prod are now sharing a single value again):

```
DEFAULT_OBJECT_STORAGE_BUCKET_ID
PUBLIC_OBJECT_SEARCH_PATHS
PRIVATE_OBJECT_DIR
ENCRYPTION_KEY_V1
ENCRYPTION_KEY_V2
SESSION_SECRET
OAUTH_RSA_PRIVATE_KEY
OAUTH_RSA_PUBLIC_KEY
INITIAL_ACCESS_CODE_1
INITIAL_ACCESS_CODE_2
STRIPE_SECRET_KEY
VITE_STRIPE_PUBLIC_KEY
STRIPE_WEBHOOK_SECRET
AI_INTEGRATIONS_GEMINI_API_KEY
AI_INTEGRATIONS_GEMINI_BASE_URL
APP_URL
ISSUER_URL
SEARCH_SALT
CRON_API_KEY
```

Any surviving unscoped secret = **BLOCK publish. Delete it or re-execute the split.**

---

### Step 0d — Code must reference only `_DEV` / `_PROD` variants

Run this grep check. Zero matches is the passing state:

```bash
grep -rn "process\.env\.SESSION_SECRET[^_]" server/ --include="*.ts"
grep -rn "process\.env\.ENCRYPTION_KEY_V[12][^_]" server/ --include="*.ts"
grep -rn "process\.env\.OAUTH_RSA_PRIVATE_KEY[^_]" server/ --include="*.ts"
grep -rn "process\.env\.OAUTH_RSA_PUBLIC_KEY[^_]" server/ --include="*.ts"
grep -rn "process\.env\.INITIAL_ACCESS_CODE_[12][^_]" server/ --include="*.ts"
grep -rn "process\.env\.STRIPE_SECRET_KEY[^_]" server/ --include="*.ts"
grep -rn "process\.env\.STRIPE_WEBHOOK_SECRET[^_]" server/ --include="*.ts"
grep -rn "process\.env\.PUBLIC_OBJECT_SEARCH_PATHS[^_]" server/ --include="*.ts"
grep -rn "process\.env\.PRIVATE_OBJECT_DIR[^_]" server/ --include="*.ts"
grep -rn "process\.env\.DEFAULT_OBJECT_STORAGE_BUCKET_ID[^_]" server/ --include="*.ts"
```

Any match = **BLOCK publish. Update the code to use the `_DEV`/`_PROD` variant.**

---

### Step 0e — Stripe key mode confirmation

Cannot read the secret value, but verify the dev app uses a Stripe **test** key
by confirming no live charges appear in the Stripe dashboard during dev testing.
If any doubt, check `STRIPE_SECRET_KEY_DEV` in the Replit Secrets pane — the
value must start with `sk_test_`, not `sk_live_`.

---

### Step 0f — Object Storage bucket isolation spot-check

Confirm that the dev and prod bucket IDs differ:

```bash
# Compare the bucket ID the running dev server resolves to
# (cannot read secrets directly, but can check which bucket the app reports using)
# Look for any files uploaded during dev testing appearing in the prod app — if so,
# buckets are still shared. Immediately block publish and re-execute the OS split.
```

---

## Timing Note

Steps 0a–0f should be formally added to `pchk.md` only after all splits
are complete. Adding them before the splits would cause every run to fail on Step 0c
(unscoped secrets still exist by design while splits are in progress).

---

*Document created: March 03, 2026*
*Author: Agent session — based on full environment audit*
*Ongoing verification section added: March 03, 2026*
*Code-complete sweep (Gemini, Stripe, APP_URL, ISSUER_URL, SEARCH_SALT, CRON_API_KEY): March 05, 2026*

---

## Secrets Still Requiring Creation in Replit (March 05, 2026)

All code changes are done. The app falls back to unscoped secrets until the following
`_DEV` / `_PROD` scoped variants are created in Replit Secrets.

### Must Create Now (Gemini — Active Prod Defect)

| Secret | DEV value | PROD value |
|--------|-----------|------------|
| `AI_INTEGRATIONS_GEMINI_API_KEY_DEV` | `_DUMMY_API_KEY_` (copy current) | — |
| `AI_INTEGRATIONS_GEMINI_API_KEY_PROD` | — | Real Gemini API key |
| `AI_INTEGRATIONS_GEMINI_BASE_URL_DEV` | `http://localhost:1106/modelfarm/gemini` (copy current) | — |
| `AI_INTEGRATIONS_GEMINI_BASE_URL_PROD` | — | Real Gemini API base URL (e.g. `https://generativelanguage.googleapis.com`) |

### Must Create Before Go-Live (Stripe)

| Secret | DEV value | PROD value |
|--------|-----------|------------|
| `STRIPE_SECRET_KEY_DEV` | `sk_test_51SIes...` (copy current test key) | `sk_live_...` at launch |
| `STRIPE_SECRET_KEY_PROD` | — | Live Stripe secret key at launch |
| `STRIPE_WEBHOOK_SECRET_DEV` | Current webhook secret (test endpoint) | — |
| `STRIPE_WEBHOOK_SECRET_PROD` | — | Live Stripe webhook signing secret |

### Should Create (Operational Isolation)

| Secret | DEV value | PROD value |
|--------|-----------|------------|
| `APP_URL_DEV` | Dev tunnel URL (e.g. `https://9d5c547d-...picard.replit.dev`) | — |
| `APP_URL_PROD` | — | `https://gtdpro.replit.app` (or custom domain) |
| `ISSUER_URL_DEV` | `https://replit.com/oidc` (same as current) | — |
| `ISSUER_URL_PROD` | — | `https://replit.com/oidc` (likely same — confirm with FHIR SMART team) |
| `SEARCH_SALT_DEV` | Any random string | — |
| `SEARCH_SALT_PROD` | — | Stable random string (do not change after prod data exists) |
| `CRON_API_KEY_DEV` | Any random string | — |
| `CRON_API_KEY_PROD` | — | Strong random string for prod cron endpoint |

### Encryption Keys (Action Required Before Next Publish)

| Secret | DEV value | PROD value |
|--------|-----------|------------|
| `ENCRYPTION_KEY_V1_DEV` | New random 64-char hex | — |
| `ENCRYPTION_KEY_V1_PROD` | — | Current `ENCRYPTION_KEY_V1` value (preserves prod PHI decryption) |
| `ENCRYPTION_KEY_V2_DEV` | New random 64-char hex | — |
| `ENCRYPTION_KEY_V2_PROD` | — | Current `ENCRYPTION_KEY_V2` value |

Generate a new key: `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"`

### VITE_STRIPE_PUBLIC_KEY — Deferred

Cannot be split via `_DEV`/`_PROD` pattern — Vite bakes this into the frontend
bundle at compile time using global secrets. **Action at go-live:** Replace
`import.meta.env.VITE_STRIPE_PUBLIC_KEY` with a server-injected API call so the
correct key is served at runtime for each environment.
