# Vael Creative Dashboard — API Security Audit

**Date:** 2026-02-26
**Scope:** All API routes in `dashboard/app/api/` of the `vael-creative` repository
**Severity:** CRITICAL — 46.6% of API routes have zero authentication

---

## Executive Summary

The Vael Creative dashboard exposes **305 API route handlers**. Of these, **142 routes (46.6%) have no authentication whatsoever** — any unauthenticated request from the public internet is processed. An additional 79 routes check login status but fail to enforce client-scoped access control, allowing any authenticated user to access any client's data.

There is **no `middleware.ts`** at the application level. Each route must individually call an auth function, and nearly half fail to do so.

---

## Protection Breakdown

| Protection Level | Count | % | Description |
|---|---|---|---|
| **UNPROTECTED** | **142** | **46.6%** | No auth check. Public internet access. |
| Auth-only (`requireAuth`) | 79 | 25.9% | Verifies login, does NOT check client access or role. |
| Client-scoped (`requireClientAccess`) | 39 | 12.8% | Verifies login + client assignment. Correct. |
| Session-only (`getServerSession`) | 22 | 7.2% | Checks session exists, no role/permission check. |
| Super admin (`checkSuperAdmin`) | 12 | 3.9% | Correct for admin routes. |
| Video auth (`requireVideoAuth`) | 8 | 2.6% | Dual auth (API key or session). |
| Permission-based (`requireAuthWithPermission`) | 2 | 0.7% | Full RBAC. Only 2 routes use it. |
| Custom (cookie check) | 1 | 0.3% | — |

---

## Auth Infrastructure

### `requireAuth()` — `lib/auth/core.ts`
- Calls `getServerSession(authOptions)`, checks `session?.user` exists
- Returns `{ authorized, session, response }`
- **Only checks login status** — does NOT check role, permissions, or client access
- 79 routes rely on this alone

### `requireClientAccess(clientId)` — `lib/auth/index.ts`
- Calls `requireAuth()`, then checks client assignment
- `super_admin` and `founder` bypass client access checks
- `member` must have explicit entry in `user_client_assignments`
- **This is the correct pattern for client-scoped routes.** Only 39 routes use it.

### `requireAuthWithPermission(permission)` — `lib/auth/permissions.ts`
- Checks session AND verifies role has the specific permission
- 3-tier: `super_admin` (all), `founder` (all except admin.users/settings), `member` (limited)
- **Only 2 routes use this.** Massively underutilized.

### `checkSuperAdmin()` — `lib/admin.ts`
- Checks session AND `role === 'super_admin'`
- 12 admin routes use this. Correct.

### No middleware.ts
- **No blanket auth middleware exists.** Every route must self-protect.

---

## Attack Scenarios (No Authentication Required)

An unauthenticated attacker can currently:

1. **Exfiltrate the entire CRM** — `GET /api/crm/leads?limit=10000` returns all PII (emails, phones, names, addresses, companies)
2. **Export CRM as CSV** — `GET /api/crm/leads/export`
3. **Wipe the CRM** — `DELETE /api/crm/leads/bulk` with target IDs
4. **Send emails as the brand** — `POST /api/email/sequences/trigger` sends from `brian@vaelcreative.com` to any address
5. **Send fake proposals to real clients** — `POST /api/crm/proposals/[id]/send`
6. **Burn paid API credits** — Apollo enrichment, Google Places, lead gen endpoints all trigger paid API calls
7. **Read proprietary trading data** — `GET /api/alpha-gen/positions` exposes portfolio positions, performance, and risk metrics
8. **Read internal strategy** — `GET /api/okrs` exposes company OKRs and key results
9. **Disconnect client integrations** — `POST /api/integrations/ga4/disconnect` or `DELETE /api/integrations/shopify/connection`
10. **Create Stripe checkout sessions** — `POST /api/checkout`
11. **Run database migrations** — `POST /api/migrate` (if `ALLOW_MIGRATIONS=true` in production)
12. **Modify or delete blog content** — `PATCH/DELETE /api/draft-insights/[slug]`

---

## CRITICAL Unprotected Routes — Full List

### CRM & Lead Data (PII Exposure + Data Destruction)

| Route | Methods | Risk | Impact |
|---|---|---|---|
| `/api/crm/leads` | GET, POST | CRITICAL | Full lead database: emails, phones, names, addresses. Create fake leads. |
| `/api/crm/leads/[id]` | GET, PATCH, DELETE | CRITICAL | Read, modify, or delete any lead. |
| `/api/crm/leads/bulk` | DELETE, PATCH | CRITICAL | Bulk delete/modify leads. Could wipe entire CRM. |
| `/api/crm/leads/export` | GET | CRITICAL | CSV export of all leads with full PII. |
| `/api/crm/leads/[id]/convert` | POST | CRITICAL | Convert lead to client, creating entries in `clients` table. |
| `/api/crm/leads/[id]/notes` | GET, POST | HIGH | Read/write private notes on any lead. |
| `/api/crm/leads/[id]/notes/[noteId]` | PATCH, DELETE | HIGH | Modify/delete notes. |
| `/api/crm/leads/[id]/tasks` | GET, POST | HIGH | Read/create tasks on any lead. |
| `/api/crm/leads/[id]/proposals` | GET, POST | HIGH | Read/create proposals for any lead. |
| `/api/crm/leads/filter-options` | GET | MEDIUM | Exposes lead database schema metadata. |
| `/api/crm/leads/discover-websites` | POST | HIGH | Triggers web scraping, consuming API credits. |
| `/api/crm/leads/enrich-apollo` | POST | HIGH | Triggers Apollo API calls (paid). |
| `/api/crm/leads/enrich-emails` | POST | HIGH | Triggers email enrichment (paid). |
| `/api/crm/leads/enrich-places` | POST | HIGH | Triggers Google Places API calls (paid). |
| `/api/crm/leads/import-apollo` | POST | HIGH | Import leads from Apollo (paid). |
| `/api/crm/leads/search-apollo` | POST | HIGH | Search Apollo database (paid). |
| `/api/crm/import` | POST, GET | CRITICAL | Bulk import leads. Attacker injects thousands of fake records. |
| `/api/crm/import/preview` | POST | MEDIUM | Preview import data. |

### CRM Pipeline & Outreach

| Route | Methods | Risk | Impact |
|---|---|---|---|
| `/api/crm/pipeline` | GET | HIGH | Full pipeline with all leads by stage. |
| `/api/crm/activities` | GET | HIGH | All CRM activity history. |
| `/api/crm/proposals/[id]` | GET, PATCH, DELETE | HIGH | Read/modify/delete proposals with financial terms. |
| `/api/crm/proposals/[id]/send` | POST | CRITICAL | Send proposals to clients. Fake proposals under your brand. |
| `/api/crm/outreach/sequences` | GET, POST | HIGH | Read/create outreach email sequences. |
| `/api/crm/outreach/sequences/[id]` | GET, PATCH, DELETE | HIGH | Modify/delete sequences. |
| `/api/crm/outreach/sequences/bulk-approve` | POST | HIGH | Mass approve outreach for sending. |
| `/api/crm/outreach/sequences/export` | GET | HIGH | Export all outreach data. |
| `/api/crm/outreach/stats` | GET | MEDIUM | Outreach performance metrics. |
| `/api/crm/generate/[jobId]` | GET | MEDIUM | Lead gen job status. |
| `/api/crm/generate/[jobId]/stream` | GET | MEDIUM | Streaming job updates. |
| `/api/crm/generate/auto` | POST | HIGH | Trigger automated lead generation. |
| `/api/crm/generate/queue` | GET, POST | HIGH | Manage lead gen queue. |
| `/api/crm/generate/send-to-crm` | POST | HIGH | Push generated leads to CRM. |
| `/api/crm/generate/usage` | GET | MEDIUM | API usage statistics. |

### Email System

| Route | Methods | Risk | Impact |
|---|---|---|---|
| `/api/email/sequences/trigger` | POST, GET | CRITICAL | Send emails to any address via Resend as `brian@vaelcreative.com`. |

### Billing & Payments

| Route | Methods | Risk | Impact |
|---|---|---|---|
| `/api/billing/portal` | POST | HIGH | Create Stripe billing portal sessions for any client. |
| `/api/checkout` | POST, GET | HIGH | Create Stripe checkout sessions. |

### Integrations (OAuth Flows)

| Route | Methods | Risk | Impact |
|---|---|---|---|
| `/api/integrations/ga4/connect` | GET | HIGH | Initiate Google OAuth for any clientId. |
| `/api/integrations/ga4/callback` | GET | HIGH | OAuth callback — token theft. |
| `/api/integrations/ga4/disconnect` | POST | HIGH | Disconnect GA4 for any client. |
| `/api/integrations/ga4/metrics` | GET | HIGH | Read GA4 metrics for any client. |
| `/api/integrations/ga4/properties` | GET | MEDIUM | List Google Analytics properties. |
| `/api/integrations/ga4/select-property` | POST | HIGH | Change GA4 property for any client. |
| `/api/integrations/ga4/status` | GET | MEDIUM | Check integration status. |
| `/api/integrations/ga4/sync` | POST | HIGH | Trigger data sync. |
| `/api/integrations/shopify/oauth` | GET | HIGH | Initiate Shopify OAuth for any clientId. |
| `/api/integrations/shopify/callback` | GET | HIGH | Shopify OAuth callback. |
| `/api/integrations/shopify/connection` | GET, DELETE | HIGH | View/delete Shopify connections. |
| `/api/integrations/shopify/analytics` | GET | HIGH | Shopify sales data. |
| `/api/integrations/shopify/sync` | POST | HIGH | Trigger Shopify data sync. |
| `/api/integrations/shopify/webhook` | POST | MEDIUM | Shopify webhook (may have HMAC internally). |

### Content & Blog

| Route | Methods | Risk | Impact |
|---|---|---|---|
| `/api/draft-insights` | GET, POST | HIGH | Full access to all drafts. Create/edit. |
| `/api/draft-insights/[slug]` | GET, PATCH, DELETE | HIGH | Modify/delete any article. |
| `/api/draft-insights/[slug]/generate-image` | POST | HIGH | Generate images (burns AI credits). |
| `/api/draft-insights/generate-images-bulk` | POST | HIGH | Bulk image generation (burns significant credits). |
| `/api/draft-insights/auto-schedule` | POST | HIGH | Auto-schedule content. |
| `/api/draft-insights/bulk-status` | PATCH | HIGH | Bulk change statuses. |
| `/api/draft-insights/cleanup` | POST | HIGH | Delete/cleanup drafts. |
| `/api/draft-insights/import` | POST | HIGH | Import content. |

### Alpha-Gen (Proprietary Trading Strategy)

| Route | Methods | Risk | Impact |
|---|---|---|---|
| `/api/alpha-gen/overview` | GET | HIGH | Strategy overview. |
| `/api/alpha-gen/performance` | GET | HIGH | Trading performance metrics. |
| `/api/alpha-gen/positions` | GET | HIGH | Current portfolio positions. |
| `/api/alpha-gen/rebalances` | GET | HIGH | Rebalance history. |
| `/api/alpha-gen/risk` | GET | HIGH | Risk analytics. |
| `/api/alpha-gen/sectors` | GET | MEDIUM | Sector allocation. |
| `/api/alpha-gen/stats` | GET | MEDIUM | Strategy statistics. |

### Internal Operations

| Route | Methods | Risk | Impact |
|---|---|---|---|
| `/api/okrs` | GET, POST | HIGH | Company OKRs. |
| `/api/okrs/[id]` | GET, PATCH, DELETE | HIGH | Modify/delete OKRs. |
| `/api/okrs/[id]/key-results` | GET, POST | HIGH | Key results CRUD. |
| `/api/okrs/key-results/[id]` | PATCH, DELETE | HIGH | Modify/delete key results. |
| `/api/okrs/migrate` | POST | HIGH | OKR migration. |
| `/api/mission` | GET, POST | HIGH | Company mission data. |
| `/api/mission/tasks` | GET, POST | HIGH | Internal task tracking. |
| `/api/mission/activities` | GET | MEDIUM | Internal activity logs. |
| `/api/mission/btodo` | GET, POST | HIGH | Personal todo list. |
| `/api/moto/tasks` | GET, POST | HIGH | AI agent task tracking. |
| `/api/ops/activity` | GET | MEDIUM | Operations activity log. |
| `/api/ops/deliver` | GET, POST | HIGH | Asset delivery. |
| `/api/ops/members` | GET | HIGH | Team member listing. |
| `/api/ops/scheduled-tasks` | GET | MEDIUM | Scheduled tasks. |
| `/api/outreach/campaigns` | GET, POST | HIGH | Campaign management. |
| `/api/outreach/queue` | GET, POST | HIGH | Outreach queue. |
| `/api/costs` | GET | HIGH | Cost tracking. |

### Infrastructure

| Route | Methods | Risk | Impact |
|---|---|---|---|
| `/api/migrate` | GET, POST | CRITICAL | Run database migrations. Env-gated but catastrophic if misconfigured. |
| `/api/setup` | GET, POST | CRITICAL | Create admin account. Env-gated. |
| `/api/setup/import-leads` | POST | HIGH | Import leads during setup. |
| `/api/sync` | POST | HIGH | Data synchronization. |
| `/api/graph` | POST | HIGH | Brand intelligence graph queries. |
| `/api/graph/sync` | POST | HIGH | Graph sync. |
| `/api/search` | GET | MEDIUM | Global search. |

### Cron Jobs

| Route | Methods | Risk | Impact |
|---|---|---|---|
| `/api/cron/daily-deploy-summary` | POST, GET | MEDIUM | Trigger daily email to founders. Spam vector. |
| `/api/cron/lead-gen` | GET, POST | HIGH | Trigger automated lead generation. Burns API credits. |

### Asset Operations (Missing Client Scoping)

| Route | Methods | Risk | Impact |
|---|---|---|---|
| `/api/assets/[assetId]/caption` | POST | MEDIUM | Generate captions for any asset (AI credits). |
| `/api/assets/[assetId]/metrics` | GET | MEDIUM | Read metrics for any asset. |
| `/api/assets/[assetId]/score` | POST | MEDIUM | Score any asset. |

---

## Auth-Only Routes Missing Client Scoping

These 79 routes use `requireAuth()` but do NOT call `requireClientAccess(clientId)`. Any authenticated user — including `member` role accounts — can access any client's data.

Routes under `/api/clients/[clientId]/` that need `requireClientAccess`:

- `assets/bulk-approve`, `assets/bulk-delete`, `assets/create`
- `autonomy/eligibility`, `autonomy/emergency`, `autonomy/tier`
- `brand-assets/*`, `brand-dna`
- `briefs`, `competitors/*`, `experiments/*`
- `export`, `posts/*`, `products/*`, `reports/*`, `templates/*`

---

## Admin Routes Missing Role Check

Six routes under `/api/admin/` use `getServerSession` but do NOT verify `super_admin` role:

| Route | Type | Risk |
|---|---|---|
| `/api/admin/check-crm-count` | READ | Any logged-in user sees lead counts |
| `/api/admin/check-crm-imports` | READ | Any logged-in user sees import data |
| `/api/admin/check-crm-table` | READ | Schema information leak |
| `/api/admin/enable-lead-gen` | WRITE | Any user can toggle lead gen |
| `/api/admin/import-existing-leads` | WRITE | Any user can bulk import |
| `/api/admin/reset-stuck-queue` | WRITE | Any user can reset queue |

These should use `checkSuperAdmin()`.

---

## Recommended Fix Strategy

### Phase 1: Blanket Middleware (Day 1)

Create `middleware.ts` at the app root that requires authentication for ALL `/api/**` routes by default, with an explicit allowlist for genuinely public routes:

**Allowlist (routes that must remain public):**
- `/api/auth/*` — NextAuth flows
- `/api/webhooks/stripe` — Stripe signature-validated webhook
- `/api/webhooks/deploy`, `/api/webhooks/chatbot` — Webhook receivers
- `/api/portal/[token]/*` — Token-gated client portal
- `/api/review/[token]/*` — Token-gated review
- `/api/proposals/[token]` — Token-gated proposals
- `/api/invite/[token]` — Token-gated invites
- `/api/health`, `/api/health-crm`, `/api/status` — Health checks
- `/api/insights`, `/api/insights/[slug]` — Public blog content
- `/api/authors`, `/api/authors/[slug]` — Public author profiles
- `/api/benchmarks/*` — Public benchmarks
- `/api/leads/inbound` — Public lead capture form
- `/api/inngest` — Inngest handler (has own auth)
- `/api/docs` — API documentation
- `/api/feedback` — Public feedback form

This inverts the security model: **deny by default, allow by exception.** No new route can be accidentally exposed.

### Phase 2: Client Access Scoping (Week 1)

Upgrade all 79 `requireAuth()`-only client routes to `requireClientAccess(clientId)`. This ensures tenant isolation.

### Phase 3: Role-Based Enforcement (Week 2)

- Upgrade 6 admin routes to `checkSuperAdmin()`
- Expand use of `requireAuthWithPermission()` for granular access control
- Add `CRON_SECRET` validation to cron routes

### Phase 4: Audit & Hardening (Week 3)

- Review token-based routes for entropy and expiration
- Add rate limiting to public endpoints
- Add request logging/alerting for sensitive operations
- Verify `ALLOW_MIGRATIONS` and `ALLOW_SETUP` are `false` in production

---

## Routes That Should Remain Public

For reference, these routes are correctly public and should stay on the allowlist:

| Route | Reason |
|---|---|
| `/api/auth/*` | Authentication flows must be accessible pre-login |
| `/api/webhooks/stripe` | Stripe sends webhooks; validated by signature |
| `/api/portal/[token]/*` | Client portal uses cryptographic tokens, not sessions |
| `/api/review/[token]/*` | Review access via token |
| `/api/proposals/[token]` | Proposal access via token |
| `/api/invite/[token]` | Invite acceptance via token |
| `/api/health*`, `/api/status` | Monitoring and uptime checks |
| `/api/insights/*` | Public blog content |
| `/api/authors/*` | Public author profiles |
| `/api/benchmarks/*` | Public benchmark data |
| `/api/leads/inbound` | Public form submission (has honeypot spam protection) |
| `/api/inngest` | Background job handler with its own auth layer |
| `/api/docs` | API documentation |
| `/api/feedback` | Public feedback form |
