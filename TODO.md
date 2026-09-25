# TODO.md — ONE-X Clinic OS build backlog

Legend: `P0` blocker · `P1` launch-required · `P2` launch-desirable · `P3` post-launch ·
`[S]` <½ day · `[M]` ½–2 days · `[L]` >2 days · `blocked-by: ID`.
Tick with commit hash: `- [x] SEC-01 … (a1b2c3d)`.

Baseline (from spec audit, 2026-09-25): `security-gate.mjs` reports **84 violations** — 57 public procedures, 19 client-supplied actor inputs, 5 money-path bypasses;
identity lives in `localStorage`; no tenant/branch scoping; audit chain unverifiable. Phases 0–2 are mandatory
before any real patient data enters the system.

---

## Phase 0 — Repo & deploy baseline

- [ ] **OPS-01** `P0 [S]` Rename package `my-app` → `onex-clinic-os`; set `"engines": {"node": "22.x"}`.
- [ ] **OPS-02** `P0 [S]` Commit `package-lock.json`: remove it from `.gitignore`, run `npm install`, commit. Switch
      Vercel install to `npm ci`. *Accept:* reproducible build from clean clone.
- [ ] **OPS-03** `P0 [S]` Stop ignoring migrations: remove `db/migrations/*.sql` from `.gitignore`.
- [ ] **OPS-04** `P0 [M]` Schema drift: add `doc_counters` (used in `routers/docs.ts:47`) to `db/schema.ts`
      with `(tenant_id, yr)` PK. Generate baseline migration `0000_baseline.sql` from `db/schema.ts`; diff against
      `db/bootstrap.sql`; resolve every difference. Mark `bootstrap.sql` legacy. *Accept:* fresh DB from
      migrations == schema.ts; `drizzle-kit check` clean.
- [ ] **OPS-05** `P0 [S]` `lib/env.ts` → zod schema (`DATABASE_URL`, `SESSION_SECRET` ≥32 chars, `CRON_SECRET`,
      `APP_URL`, `BLOB_READ_WRITE_TOKEN?`, `RESEND_API_KEY?`, `HITPAY_*?`). Fail fast with names, never values.
- [ ] **OPS-06** `P0 [S]` `boot.ts:11` body limit 50 MB → 1 MB (Vercel caps at 4.5 MB).
- [ ] **OPS-07** `P0 [S]` Replace `vercel.json` with kit version (region `sin1`, security headers, `no-store` on
      `/api`, crons, function config).
- [ ] **OPS-08** `P0 [S]` DB connection for serverless: mysql2 pool `connectionLimit: 5`, `enableKeepAlive`,
      TLS `rejectUnauthorized: true`, reuse across invocations. Keep `mode: "planetscale"` for TiDB/PlanetScale.
- [ ] **OPS-09** `P0 [M]` Provision DB (see `docs/DEPLOY_VERCEL.md §2`): prod + preview branches/instances in
      `ap-southeast-1`. Separate credentials per environment.
- [ ] **OPS-10** `P0 [S]` Add `.github/workflows/ci.yml` (kit version): verify + integration tests on MySQL service.
- [ ] **OPS-11** `P1 [S]` Scripts: `test:int`, `test:e2e`, `test:security`, `verify`. Remove `db:push` from docs.
- [ ] **OPS-12** `P1 [S]` Move `public/hero-doctor.png` (1.6 MB) → optimized WebP/AVIF ≤150 KB; lazy-load.
- [ ] **OPS-13** `P2 [S]` Self-host Inter + Plus Jakarta Sans (`@fontsource`) — drops Google Fonts from CSP and
      avoids third-party requests from a clinical app.

## Phase 1 — Security foundation  *(blocked-by: OPS-04, OPS-05)*

- [ ] **SEC-01** `P0 [L]` **Authentication.** Tables `users` (email, name, password_hash scrypt, status,
      tenant_id, mfa_secret?), `sessions` (id = random 32 B, user_id, tenant_id, device_label, expires_at,
      last_seen_at, revoked_at). Endpoints `auth.signIn`, `auth.signOut`, `auth.me`, `auth.changePassword`.
      Session cookie `__Host-onex_sid`: HttpOnly, Secure, SameSite=Lax, Path=/, 12 h idle / 7 d absolute.
      *Accept:* no procedure reachable without a valid session except allowlist.
- [ ] **SEC-02** `P0 [M]` `context.ts` resolves session → `ctx.session {userId, name, tenantId, branchIds,
      capabilities}`. `middleware.ts`: `publicProcedure`, `authedProcedure`, `can(cap)`. Convert **every**
      procedure (onex, care, pos, profile, setup, docs). *Accept:* security gate lists zero unexpected public procedures.
- [ ] **SEC-03** `P0 [M]` Remove `actor`/`createdBy` inputs everywhere (`care.ts`, `onex.ts`, `profile.ts`,
      `docs.ts`); derive from `ctx.session`. Test: forged `actor` in payload is rejected by zod `.strict()`.
- [ ] **SEC-04** `P0 [M]` **PIN quick-switch** re-scoped: profiles become `users`; PIN unlocks a user *on a device
      that already holds a tenant device session*. Hash: `scrypt(pin, per-user 16 B salt, N=2^15)`; replace
      `createHash("sha256")` + static `SALT` (`profile.ts:9-10`). 5 failed attempts → 15 min lockout per
      (user, device); audit every failure.
- [ ] **SEC-05** `P0 [S]` Capability catalog in `contracts/capabilities.ts` (single source): `patients.read`,
      `patients.clinical.read`, `patients.write`, `appointments.manage`, `visits.manage`, `surgery.schedule`,
      `surgery.override`, `surgery.transition`, `pf.rules`, `pf.adjust`, `pf.approve`, `billing.charge`,
      `billing.pay`, `billing.refund`, `billing.discount`, `pos.sell`, `inventory.adjust`, `inventory.receive`,
      `docs.write`, `docs.finalize`, `insights.view`, `audit.view`, `users.manage`, `settings.manage`.
      Role presets: Owner, Admin, Surgeon, Anesthesiologist, Nurse, Front Desk, Cashier, Inventory, Accounting.
- [ ] **SEC-06** `P0 [S]` Guard admin mutations (`setCapabilities`, `resetPin`, `setActive`, `setBrand`,
      `addBranch/Room/Provider/Service`) with `users.manage` / `settings.manage`. Users cannot grant capabilities
      they don't hold. Keep last-admin protection.
- [ ] **SEC-07** `P0 [M]` **Audit chain fix** (`queries/domain.ts:10-12`): (a) `audit(tx, …)` takes the caller's
      transaction; (b) serialize with `SELECT … FROM audit_chain_head WHERE tenant_id=? FOR UPDATE`; (c) hash over
      canonical JSON of persisted columns incl. stored `created_at` (set explicitly, ms precision) — drop
      unpersisted `Date.now()`; (d) `verifyAuditChain(tenantId)` + test that tampering one row is detected;
      (e) DB user for app has no UPDATE/DELETE grant on `audit_events` where the DB supports it.
- [ ] **SEC-08** `P0 [S]` `payments.idempotency_key` → `NOT NULL`, `UNIQUE(tenant_id, idempotency_key)`; catch
      duplicate-key and return the original result (race-safe). Test: 10 concurrent identical checkouts → 1 order.
- [ ] **SEC-09** `P0 [M]` Harden `onex.billing.pay` (`onex.ts:452`): amount `positive()`, ≤ outstanding; reject
      `VOID`/`PAID` orders; payment + status update + audit in one tx with `SELECT … FOR UPDATE` on the order;
      idempotency key required.
- [ ] **SEC-10** `P0 [S]` Bound all zod numbers/strings (qty ≤ 999, ids `.int().positive()`, text ≤ 10 k,
      arrays ≤ 100). `.strict()` on all input objects.
- [ ] **SEC-11** `P0 [M]` **PHI projection**: `projectPatient(row, scope)` with `COMMERCE` / `CLINICAL`
      allowlists; `onex.patients.list`, `onex.search`, `pos.catalog` → COMMERCE; Patient360/visit/case → CLINICAL
      + `patients.clinical.read`. Log every CLINICAL read to `phi_access_log` (user, patient, purpose, ts).
- [ ] **SEC-12** `P0 [S]` Client: `lib/session.ts` stores only non-sensitive display data; source of truth is
      `auth.me`. Sign-out clears Query cache. Idle auto-lock (5 min, configurable) returns to PIN screen.
- [ ] **SEC-13** `P1 [S]` Rate limiting on `auth.signIn`, PIN unlock, `onex.search`: DB-backed sliding window
      (`rate_limits` table) or Upstash Redis from Vercel Marketplace.
- [ ] **SEC-14** `P1 [S]` CSRF: tRPC mutations require `content-type: application/json` + `Origin` == `APP_URL`
      check in Hono middleware.
- [ ] **SEC-15** `P1 [S]` Error hygiene: tRPC `errorFormatter` strips stack/SQL in production; server logs
      request id + error class only.
- [ ] **SEC-16** `P1 [M]` Optional TOTP MFA for Owner/Admin (`otplib`); enforce for `users.manage`.
- [ ] **SEC-17** `P1 [S]` `scripts/security-gate.mjs` implementing AGENTS.md §8 tripwires + its own tests.

## Phase 2 — Tenancy, branches, data integrity  *(blocked-by: SEC-02)*

- [ ] **DAT-01** `P0 [L]` Add `tenant_id` to every tenant-owned table + composite indexes
      `(tenant_id, …)`. Backfill tenant 1. `settings` PK → `(tenant_id, key)`. Every query filters `ctx.tenantId`
      (helper `tenantScope(table)`). Integration test: tenant A cannot read/write tenant B by id.
- [ ] **DAT-02** `P0 [M]` Branch scoping: `user_branches` join; appointments, cases, calendar blocks, stock,
      orders filtered by allowed branches. Owner = all branches.
- [ ] **DAT-03** `P0 [M]` Stock per branch: `inventory_stock (tenant_id, branch_id, item_id, qty_on_hand)`;
      `inventory_items` becomes catalog. `decrementStock` and POS checkout use branch row.
- [ ] **DAT-04** `P0 [M]` Transactions: wrap `care.checkIn`, `care.chargeVisit`, `care.closeVisit`,
      `onex.surgery.schedule`, `transition`, `recordSupply`, `pf.compute`, `pf.adjust`, `billing.assemble`,
      `docs.create` — each with its audit inside the tx.
- [ ] **DAT-05** `P1 [S]` Money helpers `contracts/money.ts` (`toCentavos`, `fromCentavos`, `sum`, `pct`);
      replace float math in `care.ts` checkout and `onex.ts` billing. Property test: no rounding drift.
- [ ] **DAT-06** `P1 [S]` Manila time helpers `contracts/time.ts`; fix doc-number year (`docs.ts:45`),
      "today" queries, report day boundaries.
- [ ] **DAT-07** `P1 [S]` `computeReadiness` (`domain.ts:372`): readiness = READY only when all required items
      complete; override → `NEEDS_REVIEW` with reason; remove "≤2 missing = NEEDS_REVIEW" heuristic or make it
      tenant-configurable. Persist result on every checklist change.
- [ ] **DAT-08** `P1 [S]` Unique constraints: `(tenant_id, mrn)`, `(tenant_id, sku)`, `(tenant_id, code)` on
      services/branches, `(tenant_id, doc_number)`.
- [ ] **DAT-09** `P1 [S]` MRN generator: atomic per-tenant counter (`MRN-YYYY-000001`), same pattern as docs.
- [ ] **DAT-10** `P2 [M]` Soft-delete + `updated_at`/`updated_by` on master data; history on patient demographics.

## Phase 3 — Clinical core completeness

- [ ] **CLN-01** `P1 [L]` Appointments module: create/reschedule/cancel/no-show, provider + room availability,
      conflict check (reuse `bookBlockTx`), day/week calendar, drag-to-reschedule, walk-ins.
- [ ] **CLN-02** `P1 [M]` Visit queue board (checked-in → in-room → done) with live refresh (Query polling 15 s).
- [ ] **CLN-03** `P1 [L]` Structured SOAP note per visit (S/O/A/P, vitals, diagnosis ICD-10 free-text + code),
      signing = lock + addendum-only edits after sign.
- [ ] **CLN-04** `P1 [M]` Patient 360: demographics, allergies/alerts banner, visits, cases, documents, orders
      (COMMERCE fields only for cashier).
- [ ] **CLN-05** `P1 [L]` Surgical case UI end-to-end: schedule wizard (procedure → team from protocol →
      room/time with conflicts → checklist seeded from `services.protocol`), readiness panel, transition bar,
      supplies used, PF panel, billing panel, documents tab.
- [ ] **CLN-06** `P1 [M]` Consent & document signing: signature pad → PNG → Vercel Blob (private);
      `FINAL` docs immutable; void requires reason; print view with tenant letterhead.
- [ ] **CLN-07** `P1 [M]` PF workflow: compute → review → approve (SoD: approver ≠ computer/adjuster) →
      billable → paid; PF statement per provider per period (printable).
- [ ] **CLN-08** `P2 [M]` Follow-up auto-scheduling from `protocol.followUpDays` on DISCHARGED.
- [ ] **CLN-09** `P2 [M]` Quality/adverse event log (case-linked, CAPA status).
- [ ] **CLN-10** `P2 [M]` Clinical attachments (photos before/after) → Blob private, signed URL 5 min, audit.

## Phase 4 — Billing, POS, inventory

- [ ] **BIL-01** `P1 [M]` Consolidate money path into `queries/billing.ts`: `posCheckout`, `assembleCaseBill`,
      `pay`, `void`, `refund`. Routers call service only. *blocked-by: SEC-08, SEC-09, DAT-05*
- [ ] **BIL-02** `P1 [M]` Refunds table + flow (partial/full, reason, `billing.refund`, reverses stock via
      `RETURN` ledger for products).
- [ ] **BIL-03** `P1 [M]` Discounts: line/order discount with cap per capability (e.g. front desk ≤10 %,
      admin any), senior/PWD discount type with ID reference field.
- [ ] **BIL-04** `P1 [M]` Receipt/acknowledgement print + email. **Compliance:** confirm with the clinic's
      accountant which receipts must be BIR-registered (CAS/POS accreditation) before labeling any printout an
      "Official Receipt"; until confirmed label as "Acknowledgement Receipt".
- [ ] **BIL-05** `P1 [M]` Cashier shift: open float → sales by method → close with count → variance → Z-report.
- [ ] **BIL-06** `P2 [L]` QR/e-wallet payments (HitPay or PayMongo): server creates intent bound to order +
      amount; webhook `/api/webhooks/pay` verifies signature, idempotent on gateway id, settles via
      `billing.pay`. Secrets server-only.
- [ ] **BIL-07** `P2 [M]` HMO/split billing: payer split on order, HMO receivable aging.
- [ ] **INV-01** `P1 [M]` Purchase orders → receiving (PURCHASE_RECEIPT ledger), suppliers.
- [ ] **INV-02** `P1 [M]` Branch transfers (TRANSFER_OUT/IN pair, in-transit state).
- [ ] **INV-03** `P1 [M]` Stock count / audit (AUDIT_VARIANCE with approval).
- [ ] **INV-04** `P1 [M]` Service BOM (`service_supply_rules`): auto-consume on treatment/case completion;
      fail closed when BOM missing for stock-consuming services.
- [ ] **INV-05** `P2 [S]` Lot/expiry tracking + expiry alerts; reorder points + low-stock alerts.

## Phase 5 — Communications & automation

- [ ] **COM-01** `P1 [M]` Email via Resend: appointment confirmation/reminder, receipt, staff invite.
      Templates per tenant; `communication_log`; opt-out per patient.
- [ ] **COM-02** `P2 [M]` SMS (Semaphore / Twilio) for reminders; PH mobile normalization (+63).
- [ ] **COM-03** `P1 [S]` Vercel Cron: `/api/cron/reminders` (hourly), `/api/cron/audit-verify` (daily),
      `/api/cron/reconcile` (daily). `CRON_SECRET` bearer. *Note:* Hobby cron = daily only.
- [ ] **COM-04** `P3 [L]` Patient portal (booking, documents, receipts) on sub-route with separate auth.

## Phase 6 — Insights & reporting

- [ ] **RPT-01** `P1 [M]` Dashboard KPIs (Grok ref layout): active patients, today's appointments, revenue
      MTD, case pipeline — from real queries, Manila day boundaries.
- [ ] **RPT-02** `P1 [M]` Reconciliation center: orders vs payments vs ledger vs PF; mismatches listed.
- [ ] **RPT-03** `P1 [S]` CSV export (sales, PF statements, inventory movements) — capability-gated, audited.
- [ ] **RPT-04** `P2 [M]` Audit viewer with chain status badge + filters.
- [ ] **RPT-05** `P3 [L]` GeneSys AI insights hook (server-side, no PHI leaves tenant without DPA review).

## Phase 7 — UX & design system

- [ ] **UX-01** `P1 [M]` Tokens: merge ABE canvas plan (`plan.md`) + ONE-X brand (navy / amber / teal from
      `grok-ref` screenshots) into `src/index.css` light + dark. No raw hex in components.
- [ ] **UX-02** `P1 [M]` Shell: sidebar w/ section subtitles (Grok ref), top bar (search ⌘K, notifications,
      user/branch switcher, Manila clock), mobile bottom nav <768 px.
- [ ] **UX-03** `P1 [M]` Access screen: tenant sign-in → device → profile tiles → PIN pad; lockout messaging.
- [ ] **UX-04** `P1 [S]` Route-level code splitting (`React.lazy`) — recharts + calendar off the main chunk.
      Target: initial JS ≤ 250 KB gz.
- [ ] **UX-05** `P1 [M]` Accessibility: WCAG 2.2 AA contrast in both themes, focus rings, labels, `aria-live`
      toasts, keyboard for calendar + POS.
- [ ] **UX-06** `P2 [M]` PWA: manifest, icons, offline shell (read-only), install prompt for tablets at front desk.
- [ ] **UX-07** `P2 [S]` Print CSS for all documents (A4 + 80 mm thermal receipt).
- [ ] **UX-08** `P2 [S]` White-label: logo upload (Blob), colors per tenant applied as CSS vars at runtime.

## Phase 8 — Testing & QA

- [ ] **QA-01** `P0 [M]` Integration harness: vitest + MySQL (CI service / local Docker), migrate fresh DB per
      run, factories for tenant/user/patient/case.
- [ ] **QA-02** `P0 [M]` Security tests: unauthenticated call → 401 for every non-allowlisted procedure
      (generated from router); cross-tenant id access → 404; cashier → no clinical fields.
- [ ] **QA-03** `P1 [M]` Money tests: concurrent checkout idempotency, stock never negative under 20 parallel
      sales, overpayment rejected, refund reverses ledger.
- [ ] **QA-04** `P1 [M]` Audit tests: concurrent writes keep a single valid chain; tamper detected.
- [ ] **QA-05** `P1 [L]` Playwright e2e: sign-in → book → check-in → SOAP → charge → pay → receipt;
      surgical case PLANNING → CLOSED; desktop + 390×844.
- [ ] **QA-06** `P2 [S]` Load test checkout + calendar (k6): p95 < 800 ms at 20 rps from Manila.

## Phase 9 — Compliance & operations

- [ ] **CMP-01** `P1 [M]` Data Privacy Act (RA 10173): privacy notice + consent capture at registration;
      DPO contact in settings; data subject requests (access/correction/erasure-where-lawful) workflow;
      retention policy per record type (confirm periods with the clinic's counsel/DPO).
- [ ] **CMP-02** `P1 [S]` Breach runbook in `docs/` (detect → contain → assess → notify NPC/subjects within
      the statutory window — confirm current NPC rules with counsel).
- [ ] **CMP-03** `P1 [S]` Backups: DB provider PITR on; weekly logical export to encrypted storage; quarterly
      restore drill.
- [ ] **OBS-01** `P1 [S]` Observability: Vercel logs + Speed Insights; Sentry (server + client) with PHI scrubbing
      (`beforeSend` drops request bodies).
- [ ] **OBS-02** `P1 [S]` `/api/health` (DB ping, migration version) + uptime monitor.
- [ ] **OBS-03** `P2 [S]` Alerts: 5xx rate, audit-verify failure, reconcile mismatch → email.

## Phase 10 — Launch

- [ ] **LCH-01** `P0 [S]` Vercel **Pro** plan (Hobby is for non-commercial use — confirm current terms).
- [ ] **LCH-02** `P0 [S]` Custom domain + HTTPS, HSTS preload after 2 weeks stable.
- [ ] **LCH-03** `P0 [S]` Production seed: tenant, branches, rooms, providers, services, PF rules,
      role presets, first Owner (via one-time CLI script, not a public endpoint).
- [ ] **LCH-04** `P0 [S]` Go-live checklist (`docs/DEPLOY_VERCEL.md §9`) signed off.
- [ ] **LCH-05** `P1 [S]` Staff training scripts per role + 1-week hypercare.

---

## Discovered
<!-- Agents append out-of-scope findings here: - [ ] **DISC-NN** description (file:line) -->
