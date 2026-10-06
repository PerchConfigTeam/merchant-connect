# CLAUDE.md — Merchant Connect Cloud

## What this project is

Merchant Connect Cloud is a UK B2B partner portal. Independent **partners** (resellers/introducers) hand off merchant card-processing leads to **Merchant Connect HQ** (the admin). HQ qualifies the leads, marks them Sold when the merchant goes live, and the partner earns a flat commission per sale. Partners then request payouts to their verified UK bank account. Leads can also arrive through a public REST API (website forms, Zapier), and HQ can push events to its CRM via signed webhooks.

The goal of this repo is to rebuild the working browser demo (`reference/demo.html`) as a **live production app**. The demo is the functional spec: its in-browser "demo backend" (`demoRoute()`) defines every endpoint, validation message and business rule. Port its behaviour faithfully, then harden it as described in "Production hardening" below.

## Source of truth: `reference/demo.html`

Keep the original demo in the repo at `reference/demo.html` and read it before building any feature. It is a single file; useful landmarks (line numbers approximate):

| Lines | What |
|---|---|
| 44–106 | Constants: lead statuses, API scopes, webhook events, UK providers/volumes/business types, UK postcode/phone/Companies House validators |
| 108–949 | UK bank **modulus checking** (Pay.UK/VocaLink spec): weight patterns, compact rules table, substitutions, `modulusCheck()` with exceptions 1–14 |
| 951–995 | Security: PBKDF2 hashing, OTP generation, password rules, lockout constants |
| 997–1032 | Email helper, partner profile sections, completeness, masking (`profileOut`) |
| 1034–1121 | `seedDB()`: the data model by example (users, leads, remittances, apiKeys, webhooks, deliveries, settings, emails) |
| 1129–1168 | Output shapes, high-volume rule, `sellerBalance`, `emit` (webhooks), `authenticate` |
| 1170–1658 | **`demoRoute()` — the API spec.** Every route, auth requirement, validation and side effect |
| 1660–1692 | Client `api()` wrapper (401/423 handling) |
| 1750–1880 | Demo-only bar and fake inbox — **do not port** |
| 1880–2045 | Auth screens: sign in, apply, create password, forgot/reset |
| 2046–3150 | Views: Dashboard (metrics, leaderboard, chart), Leads, Payouts, Integrations, My account, Partners, Settings |

When the demo and this file disagree, this file wins for security/infra; the demo wins for UX copy and business rules unless noted.

## Target stack

The demo was written to mirror a Cloudflare Worker API (`src/index.js`). Default to:

- **Runtime:** Cloudflare Workers (TypeScript), router such as Hono
- **Database:** Cloudflare D1 (SQLite) with SQL migrations in `migrations/`
- **Background work:** Cloudflare Queues for webhook delivery and email sending; Cron Triggers for retries, OTP/session cleanup
- **Secrets:** Wrangler secrets (`wrangler secret put`), never committed
- **Frontend:** Vite + TypeScript SPA (React or vanilla, keep the demo's layout), Tailwind compiled at build time, Chart.js and Font Awesome bundled (no CDN scripts in production), served from Workers static assets
- **Email:** a transactional provider (e.g. Postmark, Resend or SES) behind an `EmailSender` interface
- **Tests:** Vitest + `@cloudflare/vitest-pool-workers` for API; Playwright for E2E

If a different stack is chosen, keep the API contract and rules identical.

## Repo layout (proposed)

```
reference/demo.html          # original demo, read-only
migrations/                  # D1 SQL migrations, numbered
src/
  index.ts                   # Worker entry, router, error handler
  routes/                    # auth, leads, remittances, users, account, settings, keys, webhooks, stats
  domain/                    # pure business logic, no I/O (commission, balance, leaderboard, lifecycle)
  lib/
    modulus/                 # UK modulus check + valacdos/scsubtab data loader
    validation/              # ukPostcode, ukPhone, companyNumber, email, password rules
    crypto.ts                # PBKDF2, OTP, HMAC, constant-time compare, AES-GCM field encryption
    email/                   # EmailSender interface + templates
    webhooks.ts              # signing, enqueue, delivery
  queue.ts                   # Queue consumer
  cron.ts                    # scheduled handler
web/                         # SPA source
test/
wrangler.toml
```

## Commands

Fill these in once scaffolded and keep them accurate:

```
npm run dev          # wrangler dev + vite
npm test             # unit + API tests
npm run e2e          # Playwright
npm run db:migrate   # wrangler d1 migrations apply <db> --local / --remote
npm run deploy       # wrangler deploy (staging first)
```

## Conventions

- British English in all UI copy and emails. Keep the demo's wording; it is deliberately plain and GOV.UK-style.
- Currency is GBP. **Store money as integer pence** in the database (the demo uses pounds as floats). Format with `en-GB` / `GBP`.
- Store timestamps as UTC ISO 8601. Display and bucket by month in **Europe/London**.
- IDs: prefixed random IDs as in the demo (`lead_…`, `u_…`, `rem_…`, `key_…`, `wh_…`, `whd_…`, `evt_…`), generated with `crypto.getRandomValues`, never `Math.random`.
- Errors: JSON `{ "error": "<human message>" }` with the demo's status codes (400, 401, 403, 404, 409, 423). Reuse the demo's messages verbatim.
- All user text is HTML-escaped on render (the demo's `esc()`); never inject raw HTML.

## Domain model

Derive tables from `seedDB()`. Minimum:

**users** — id, name (trading name shown in UI), email (unique, lowercased), role (`admin` | `seller`), company, status (`pending` | `awaiting_password` | `active` | `deactivated` | `rejected`), password_hash, password_set_at, failed_attempts, locked_at, approved_at, last_sign_in_at, created_at.

**otps** (or columns on users) — user_id, purpose (`activation` | `reset`), code_hash, issued_at, expires_at, attempts.

**partner_profiles** — user_id, full_name, company_name, address (line1, line2, town, county, postcode), phones (primary, primary_type, secondary, secondary_type), bank (account_name, bank_name, sort_code, account_number **encrypted**, check_status, check_detail, checked_at, updated_at), next_of_kin (name, relationship, phone, email), right_to_work (declared, basis, share_code **encrypted**, signed_name, declared_at), updated_at.

**leads** — id, seller_id, external_id, source (`portal` | `api` | `admin`), business_name, contact_name, phone, email, postcode, business_type, company_number, current_provider, monthly_volume, notes, contact_consent_at, status, commission_pence, created_at, updated_at, sold_at. Unique index on (seller_id, external_id) where external_id is not null.

**remittances** (payouts) — id, seller_id, amount_pence, method, status (`Pending` | `Paid` | `Rejected`), reference, note, requested_at, paid_at.

**api_keys** — id, user_id, name, key_prefix, key_hash, scopes, last_used_at, created_at, revoked_at.

**webhooks** — id, url, secret (encrypted), events, active, created_at.
**webhook_deliveries** — id, webhook_id, event, payload, status_code, ok, error, attempt, created_at.

**settings** — single row: flat_commission_pence (default 20000 = £200), high_volume_bonus_pence (default 7500 = £75), auto_qualify_leads (false), require_partner_approval (true).

**sessions** — id (random, hashed in DB), user_id, kind (`full` | `setup`), created_at, expires_at, last_seen_at.

**audit_log** — actor_id, action, target_type, target_id, metadata, ip, created_at. Write an entry for every admin action, bank-detail change, password change, lockout and unlock.

## Business rules (port exactly)

### Partner lifecycle

1. **Apply** (`POST /auth/register`): full name, company, email, UK phone. Creates a `pending` seller with no password. Emails the applicant ("We've received your partner application") and support ("New partner application"). Duplicate email → 409.
2. **Approve** (admin): `pending` → `awaiting_password`; issue an **activation OTP** valid 72 hours and email it.
3. **Reject** (admin): `pending` → `rejected`; email with optional reason.
4. **First sign-in**: partner signs in with email + OTP → a `setup` session that can only call `/auth/set-password`. Setting a password → `active`, OTP cleared, full session started, confirmation email.
5. **Resend OTP** only while `awaiting_password`.
6. **Deactivate** only `active`; **reactivate** returns to `active` if a password exists, otherwise `awaiting_password`.
7. Admins cannot perform these actions on their own account.
8. `requirePartnerApproval` exists in settings but the demo always requires approval. Keep approval mandatory unless the product owner confirms otherwise.

### Authentication and security

- Password rules: at least 12 characters, an uppercase letter, a lowercase letter, a number; not in the common-password list; must not contain the email local part (if ≥ 4 chars). Use a larger breached-password list in production (e.g. k-anonymity check against HIBP).
- **Lockout:** 5 consecutive failures (wrong password, wrong OTP, wrong current password on change-password or bank-detail change) → `locked_at` set, sessions revoked, emails to partner and support, 423 with the "contact support" message. Only an admin can unlock. Failure messages say how many attempts are left.
- **Forgot password:** always returns the same message whether or not the account exists. Active, unlocked accounts get a reset code valid 30 minutes; locked accounts get an email telling them to contact support. Reset code allows 5 wrong attempts, then is invalidated.
- OTP format: 8 chars from `ABCDEFGHJKLMNPQRSTUVWXYZ23456789`, shown as `XXXX-XXXX`, normalised (uppercase, strip non-alphanumerics) before hashing.
- Password changes and resets send a "Your password was changed" email.
- Deactivated, pending, rejected and locked users cannot authenticate by session **or** API key.

### Leads

- Statuses: `Pending Review` → `Qualified` → `Sold` | `Rejected` (admin may set any status).
- New leads start `Pending Review`, or `Qualified` if `autoQualifyLeads` is on.
- Required: businessName, contactName, and phone or email. `contactConsent` must be true (UK GDPR); store `contact_consent_at`.
- Optional and validated: UK phone, UK postcode (normalised to `LS1 6AZ`), businessType from the fixed list, Companies House number (8 digits zero-padded, or 2 letters + 6 digits), email.
- **Commission** is set at creation: `flatCommissionRate` + `highVolumeBonus` if monthly volume ≥ £50,000 (`isHighVolume()`). House leads (admin-created with no `sellerId`) earn 0.
- Admin may create on behalf of a partner via `sellerId`.
- **Idempotency:** same `externalId` for the same seller returns the existing lead with `duplicate: true` and creates nothing.
- Partners see only their own leads (404, not 403, for others). Admin sees all.
- Only admin may PATCH (status, commission, notes). Status change sets/clears `sold_at`. Emits `lead.updated`, plus `lead.status_changed` and `lead.sold` as appropriate.
- `GET /leads` supports `status`, `updatedSince`, `limit` (1–500, default 100), `offset`; sorted newest first; returns `hasMore`.

### Payouts (remittances)

- `earned` = sum of commission on the seller's Sold leads. `committed` = sum of non-Rejected remittances. `available` = max(0, earned − committed).
- Partners request payouts only with a session (not API key), only with bank details **and** a right-to-work declaration on file, amount > 0 and ≤ available. Method is "Faster Payments" or "BACS" "to account ending NNNN".
- Admin marks a Pending payout Paid (optional reference, else generated) or Rejected (optional reason). Only Pending payouts can change (409 otherwise).
- Do the balance check and insert in **one transaction** so two concurrent requests cannot overdraw.
- Use one balance function everywhere (the demo's `/stats` recomputes it inline; unify).

### Partner account ("My account")

Six sections, each saved independently via `PUT /account/<section>`; completeness is "N of 6":

1. **details** — full name (≥ 2 chars), company name (also updates the user's display name).
2. **address** — line1, town, valid UK postcode required.
3. **phones** — UK primary required; optional secondary must differ; type Mobile/Landline/Work.
4. **bank** — requires the user's **current password** (counts towards lockout). Validate with `modulusCheck`; a fail is rejected, an uncheckable sort code is accepted with status `not_checkable`. Email the partner on add/change. Never return the full account number: always masked (`••••1234`), sort code formatted `20-00-00`. `POST /account/bank/check` lets the form pre-check.
5. **next-of-kin** — name, relationship from list, UK or `+` international phone, optional email.
6. **right-to-work** — declaration ticked, basis from list; share code (9 alphanumerics) required unless "British or Irish citizen"; signed name must match full name (case-insensitive). Share code is masked on output.

Admins do not have account sections (403).

### Settings (admin)

`flatCommissionRate`, `highVolumeBonus` (≥ 0), `autoQualifyLeads`, `requirePartnerApproval`. Changes affect **new** leads only.

### Dashboard

- `/stats`: lead counts (total, sold, qualified, pending review), commissions earned, payouts pending/paid, partner `available`, admin counts of pending and locked partners, partner account completeness, monthly submitted/sold for the last 6 calendar months (Europe/London).
- `/leaderboard` (admin): period `30` | `90` | `365` | all. Per seller: sales (Sold in period by `sold_at`), commission, leads submitted in period, conversion %, last sale. Hide inactive partners with no activity. Sort by sales, then commission, then leads, then name; equal sales **and** commission share a rank.

### API keys

- Any signed-in user can create, list and revoke their own keys. Scopes: `leads:read`, `leads:write`, `remittances:read` (default: leads read + write).
- Format `plk_` + 32 hex chars. **Show once**; store only a SHA-256 hash plus a display prefix. Update `last_used_at` on use.
- API keys authenticate with `Authorization: Bearer <key>` and are limited by scope. Session-only endpoints return 403 for key auth.

### Webhooks (admin)

- HTTPS URLs only. Events: `lead.created`, `lead.updated`, `lead.status_changed`, `lead.sold`, `remittance.requested`, `remittance.paid`, `remittance.rejected`, plus `ping` for "Send test".
- Signing secret `whsec_…` from a CSPRNG, shown once.
- Payload: `{ id: "evt_…", event, createdAt, data }`. Headers: `X-MerchantConnect-Event`, `X-MerchantConnect-Timestamp`, `X-MerchantConnect-Signature: v1=<hex HMAC-SHA256(secret, timestamp + "." + body)>`. Document verification for receivers.
- Deliver asynchronously via Queue with exponential backoff (e.g. 6 attempts over ~24h), 10s timeout, record each attempt (status, error). Keep the last 50 per webhook visible in the delivery log.
- Block private/loopback/link-local destinations (SSRF).

## API surface

All routes under `/api`. "S" = session only, "A" = admin, scope = required for API-key callers.

| Method | Path | Auth |
|---|---|---|
| POST | /auth/register | public |
| POST | /auth/login | public |
| POST | /auth/set-password | setup session |
| POST | /auth/forgot-password | public |
| POST | /auth/reset-password | public |
| POST | /auth/change-password | S |
| POST | /auth/logout | S |
| GET | /auth/me | S |
| GET | /stats | S or key |
| GET | /leaderboard?period= | S, A |
| GET | /leads | `leads:read` |
| POST | /leads | `leads:write` |
| GET | /leads/:id | `leads:read` |
| PATCH | /leads/:id | S, A |
| GET | /remittances | `remittances:read` |
| POST | /remittances | S, partner only |
| POST | /remittances/:id/pay \| reject | S, A |
| GET | /users, /users/:id | S, A |
| POST | /users/:id/approve \| reject \| resend-otp \| unlock \| deactivate \| reactivate | S, A |
| GET | /account | S |
| POST | /account/bank/check | S |
| PUT | /account/details \| address \| phones \| bank \| next-of-kin \| right-to-work | S, partner only |
| GET, PUT | /settings | S, A |
| GET, POST | /keys | S |
| DELETE | /keys/:id | S |
| GET, POST | /webhooks | S, A |
| DELETE | /webhooks/:id | S, A |
| POST | /webhooks/:id/test | S, A |
| GET | /webhooks/:id/deliveries | S, A |

Response shapes: match the demo (`{ lead }`, `{ leads, limit, offset, hasMore }`, `{ remittances, balance }`, `{ user }`, `{ profile }` etc.). Publish an OpenAPI spec at `/api/openapi.json`.

## Transactional emails

Port every `sendEmail(...)` call as a template (subject + paragraphs + optional code): application received, new application (to support), application approved (with OTP), application rejected, password created, reset code, password changed (reset and change variants), reset requested while locked, account locked (partner and support), account unlocked, bank details added/changed. Send from a verified domain with SPF/DKIM/DMARC. Never put OTPs in subject lines.

## Production hardening (differences from the demo)

Must-do before go-live:

1. **Remove demo-only code:** `/auth/demo-login`, the demo bar, fake inbox, "Use this code" shortcut, `demo-reset`, `localStorage` DB, the seeded hashes/keys/passwords and `plk_demo_*` keys. Seed data only in a dev-only script.
2. **Sessions:** random 256-bit token in an `HttpOnly; Secure; SameSite=Lax` cookie, hashed in DB, idle and absolute expiry, revoked on lockout, password change/reset and deactivation. CSRF protection on state-changing session routes (SameSite + Origin check or token).
3. **Rate limiting** (per IP and per email) on login, register, forgot/reset password, OTP entry, and per API key on `/api/leads`.
4. **Passwords:** PBKDF2-SHA256 at 100,000 iterations is the Workers WebCrypto maximum; keep the `pbkdf2$<iter>$<salt>$<hash>` format so iterations can be raised later, or move to Argon2id via WASM if acceptable.
5. **Encrypt at rest** (AES-GCM, key in a secret, key versioning): bank account number, right-to-work share code, webhook secrets. API keys and OTPs stored as hashes only.
6. **Secrets from CSPRNG only.** The demo uses `Math.random` for webhook secrets and payout references; replace.
7. **Money in pence**, and **transactions** for payout requests and status changes.
8. **Commission integrity:** decide (with the product owner) whether commission on a Sold lead can be edited or the lead un-sold after a payout has been paid against it; at minimum audit-log it and never let `available` go negative silently.
9. **Modulus check data:** load `valacdos.txt` and `scsubtab.txt` from the latest Pay.UK release (they are updated several times a year) via a build script, rather than the hand-compacted table in the demo. Add the published Pay.UK test cases as unit tests. A pass means the details *could* exist; consider Confirmation of Payee before first payout.
10. **Security headers:** strict CSP (no inline scripts; bundle Tailwind, Chart.js, Font Awesome), HSTS, `X-Content-Type-Options`, `Referrer-Policy`, `frame-ancestors 'none'`.
11. **UK GDPR:** privacy notice, retention policy (leads, rejected applications, right-to-work records, audit logs), data subject export/erase for partners, lawful basis recorded via `contact_consent_at`. Restrict who can view partner bank and right-to-work data, and log access.
12. **Observability:** structured logs without PII or secrets, error tracking, alerting on webhook failures and lockout spikes.
13. **Environments:** separate dev, staging and production D1 databases, secrets and email senders.

## UI and brand

- Dark slate theme (`slate-900` background, `slate-800/80` cards, rounded-2xl). Brand blue replaces Tailwind `indigo` (palette in the demo `<head>`), accent `peach` 100–400.
- Wordmark: "Merchant Connect Cloud" with gradient initials (`.mcc-m`, `.mcc-c1`, `.mcc-c2`). The logo is a base64 WebP in the demo (`LOGO_SRC`, line 43); extract it to `web/public/logo.webp`.
- Tabs: Dashboard, Leads, Payouts, Partners (admin) / My account (partner), Integrations, Settings (admin). Mobile tab bar under the header.
- Keep: toasts, modal pattern, live password checklist, status badges, "Try the API" playground (pointed at the real API with the user's key), delivery log with payload viewer.
- Accessibility: labelled inputs, `aria-label` on icon buttons, visible focus ring (`#8cbfe5`), WCAG 2.2 AA contrast.

## Testing requirements

- Unit tests for: every validator (postcode, phone, Companies House, email, password rules), `modulusCheck` (Pay.UK vectors plus each exception), `isHighVolume`, commission calculation, balance, leaderboard ranking and ties, OTP normalisation.
- API tests for every route: happy path, each validation message, auth (no auth, wrong role, key missing scope, session-only route via key), lockout after 5 failures, idempotent `externalId`, payout overdraw (including concurrent requests), webhook emission per event.
- E2E: apply → approve → OTP sign-in → set password → complete account → admin adds and sells lead → partner requests payout → admin pays.
- Run tests before every commit; do not mark work done with failing tests.

## Suggested build order

1. Scaffold Worker, D1, migrations, error handling, test harness.
2. Validators and modulus check module with tests.
3. Auth: register, approve, OTP, set password, login, lockout, forgot/reset, sessions, rate limits.
4. Leads + commission + settings.
5. Account sections (with encryption) and payouts.
6. Stats and leaderboard.
7. API keys and public API; OpenAPI spec.
8. Webhooks with Queue delivery, signing and retries.
9. Email provider and templates.
10. Frontend port, view by view, against the real API.
11. Hardening checklist, staging deploy, E2E, go-live.

## Open questions for the product owner

- Confirm the Cloudflare stack and email provider.
- Should `requirePartnerApproval = false` allow self-serve activation?
- Can commission be edited, or a lead un-sold, after it has been paid out?
- Are payouts executed manually (admin marks Paid) or via a payments API?
- Retention periods for leads, rejected applications and right-to-work records.
- Production domain (the demo shows `https://your-domain.co.uk`) and support mailbox (demo: `admin@merchantconnect.demo`).
