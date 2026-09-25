# AGENTS.md — ONE-X Clinic OS

Operating contract for **every** coding agent working in this repo (Claude Code, Codex, Kimi, Grok, Cursor).
Read fully before the first edit. `CLAUDE.md` imports this file. `TODO.md` is the backlog. `docs/` holds depth.

---

## 1. Product

**ONE-X Clinic OS** — white-label clinic operating system for Philippine ambulatory / aesthetic surgical clinics
(first tenant: Center for Advanced Aesthetics, Quezon City).

Modules: Patient spine (one identity across clinical + commerce) · Appointments & visit queue · Surgical case
orchestration (state machine, OR calendar, conflict detection, readiness engine) · Professional Fee (PF) engine ·
Canonical billing & POS · Inventory ledger · Fillable/printable operational documents · Capability-based staff
profiles · Hash-chained audit trail · Insights & reconciliation.

Currency **PHP**. Timezone **Asia/Manila (UTC+8)**. Regulated data: patient health information under the
**Data Privacy Act of 2012 (RA 10173)**.

---

## 2. Stack (locked — do not swap without an ADR in `docs/DECISIONS.md`)

| Layer | Choice |
|---|---|
| Frontend | React 19 · Vite 7 · React Router 7 · Tailwind 3.4 · shadcn/ui (Radix) · TanStack Query 5 · sonner · recharts |
| API | tRPC 11 on Hono 4, superjson transformer |
| Runtime | Vercel Node.js serverless — **one** function `api/[...slug].ts` → `api/_server/boot.ts` |
| DB | MySQL 8–compatible serverless (TiDB Cloud Serverless or PlanetScale) via Drizzle ORM 0.45 + mysql2 |
| Validation | zod 4 (every procedure input) |
| Tests | vitest (unit + integration) · Playwright (e2e) |
| Region | Vercel `sin1` (Singapore) · DB in AWS `ap-southeast-1` |

Aliases: `@/*` → `src/*`, `@contracts/*` → `contracts/*`, `@db/*` → `db/*`.

---

## 3. Commands

```bash
npm ci                 # install (lockfile is committed)
npm run dev            # Vite + Hono dev server on :3000 (API at /api/trpc)
npm run check          # tsc -b (typecheck)
npm run lint           # eslint
npm test               # vitest unit
npm run test:int       # vitest integration (needs DATABASE_URL to a disposable DB)
npm run test:e2e       # Playwright against a running build
npm run test:security  # scripts/security-gate.mjs — static tripwires (see §8)
npm run build          # vite build + esbuild server bundle
npm run db:generate    # drizzle-kit generate  → db/migrations/*.sql (COMMITTED)
npm run db:migrate     # apply migrations
npm run verify         # lint + check + test + test:security + build  ← release gate
```

Never use `db:push` against staging/production. Migrations only.

---

## 4. Repo map

```
api/[...slug].ts            Vercel entry (the only function). Do not add files directly under api/ except this.
api/_server/boot.ts         Hono app: body limit, tRPC mount, 404
api/_server/context.ts      tRPC context → { req, resHeaders, session, tenantId, db }
api/_server/middleware.ts   procedure builders: publicProcedure, authedProcedure, can(cap)
api/_server/router.ts       root router
api/_server/routers/*.ts    one router per bounded context (onex, care/pos, profile/setup, docs, …)
api/_server/queries/        getDb(), domain services (audit, ledger, state machine, PF, conflicts)
api/_server/lib/            env (zod-validated), http, security helpers
contracts/                  types shared by client + server (doc templates, errors, capability list)
db/schema.ts                Drizzle schema — SOURCE OF TRUTH for tables
db/migrations/              generated SQL — committed, append-only
db/seed.ts                  dev/demo seed only — never runs in production
src/pages/                  route screens
src/components/             app components; src/components/ui = shadcn primitives (do not hand-edit)
src/lib/                    trpc client, session, brand, helpers
docs/                       ARCHITECTURE, SECURITY, DEPLOY_VERCEL, DECISIONS, PROMPTS
```

---

## 5. Non-negotiable invariants

Break one of these and the change is rejected, regardless of tests passing.

**Identity & access**
1. Every procedure except `ping`, `auth.*` sign-in endpoints and `setup.getBrand` uses `authedProcedure`.
   Every mutation additionally uses `can("<capability>")`.
2. The actor is **always** `ctx.session.userId` / `ctx.session.name`. Never accept `actor`, `createdBy`,
   `receivedBy`, `approvedBy` from input. Delete such input fields when you touch a procedure.
3. Capabilities are enforced server-side. Client-side capability checks are cosmetic only.
4. PINs are a *quick-switch* factor on an already-authenticated device, never the sole credential for a
   tenant. PIN hash = `scrypt` with per-user random salt; attempts rate-limited and locked out.

**Tenancy & PHI**
5. Every tenant-owned table has `tenant_id`; every query filters by `ctx.tenantId`. Branch-scoped tables also
   filter by the user's allowed branches.
6. Patient reads go through a projection: `COMMERCE` scope (name, MRN, phone) vs `CLINICAL` scope (full chart).
   POS/cashier capabilities never receive clinical fields. Every clinical read writes an access audit event.
7. No PHI in logs, error messages, URLs, analytics events, or `console.*`. Log IDs only.

**Money**
8. Single money path: all settlement goes through the billing service (`posCheckout`, `billing.pay`,
   `billing.refund`). No other code inserts into `orders` / `payments` / `refunds`.
9. Server re-prices every line from the catalog. Client-sent prices, totals and discounts beyond the user's
   discount capability are ignored.
10. Money is `DECIMAL(12,2)` in the DB and handled as integer centavos in TS (`toCentavos` / `fromCentavos`).
    Never do arithmetic on JS floats for money.
11. Payments are idempotent: `payments.idempotency_key` is `UNIQUE (tenant_id, idempotency_key)`; duplicates
    return the original result. Amount > 0, ≤ outstanding balance (overpayment only via explicit change/credit).
12. Paid orders are never edited or deleted — only voided/refunded with reason + capability + audit.

**Integrity**
13. Multi-row writes happen in one `db.transaction`. The audit event for that write is inserted **in the same
    transaction**.
14. Audit chain: append-only, serialized via a locked chain-head row, hash covers the **persisted** fields
    (incl. stored `created_at`), verifiable by `verifyAuditChain()`. `audit_events` has no UPDATE/DELETE path.
15. Stock never goes negative: conditional `UPDATE … WHERE qty_on_hand >= ?` + ledger row, same transaction.
    `inventory_ledger` is append-only; on-hand is reconcilable from the ledger.
16. Surgical case status changes only via `transition()` using `CASE_TRANSITIONS`. No direct status writes.
17. Calendar booking = conflict check with locking read + insert in one transaction. Overrides require
    reason + `surgery.override` capability + audit.
18. PF amounts: computed by pure `computePfAmount`; manual changes only via `pf.adjust` (reason, revision row).
    Approver ≠ the person who computed/adjusted (segregation of duties).
19. Document templates never invent clinical requirements. Clinical content is tenant-authored.

**Time**
20. Store UTC. Display and business-day logic (doc numbering year, "today", reports) in Asia/Manila.

---

## 6. Coding rules

- TypeScript strict. No `any` in new code; replace existing `any` when you touch it.
- zod-validate every input; bound every number (`.int().positive().max(...)`) and string (`.max(...)`).
- Throw `TRPCError` with a safe message; never leak SQL/stack traces to the client.
- One router per bounded context; business logic in `api/_server/queries/*` services, routers stay thin.
- Schema change ⇒ edit `db/schema.ts` → `npm run db:generate` → commit the SQL → update `db/seed.ts`.
  `db/bootstrap.sql` is a legacy snapshot; do not edit it — migrations supersede it.
- UI uses design tokens from `src/index.css` only (no raw hex in components). shadcn primitives via `@/components/ui`.
- Every screen: loading, empty, error states; keyboard reachable; works at 390×844.
- No new dependency without a one-line justification in the PR description. No native modules (Vercel).
- Comments explain *why*, not *what*.

---

## 7. Vercel constraints (deploy target)

- Single function. Keep `api/_server/**` as non-entry modules (the `_` prefix prevents extra functions).
- Request body ≤ 4.5 MB (platform limit) — Hono `bodyLimit` set to 1 MB; file uploads go direct-to-Blob.
- No filesystem writes at runtime. No long-lived timers. `maxDuration` 30 s.
- DB connections: one lazily-created pool per instance, `connectionLimit` small (≤ 5), TLS required.
- Env vars validated at boot by `lib/env.ts` (zod). Only `VITE_*` reach the browser — never put secrets there.
- Scheduled jobs = Vercel Cron → `/api/cron/*`, guarded by `CRON_SECRET` bearer.
- Files (signatures, attachments) → Vercel Blob, private, served via short-lived signed URLs after auth.

---

## 8. Workflow for every task

1. **MAP** — list the files, tables, procedures and invariants (§5) the change touches. Grep before opening;
   never read whole trees. If the task conflicts with an invariant, stop and report.
2. **ACT** — smallest diff that completes the task. Task isolation: do not refactor unrelated code.
   Security fixes ship with a test that fails before the fix.
3. **VERIFY** — `npm run verify`. For DB changes also `npm run test:int`. For UI, run the app and check the
   screen at desktop + 390×844 with a clean console.
4. **RECORD** — tick the item in `TODO.md` (with commit hash), add an ADR if a decision was made.

### Security gate (`npm run test:security`) fails the build when it finds:
- `publicProcedure`/`publicQuery` outside the allowlist.
- `actor:` / `createdBy:` / `receivedBy:` / `approvedBy:` keys inside a `z.object` input.
- `insert(orders|payments|refunds)` outside `api/_server/queries/billing.ts`.
- `update(auditEvents)` / `delete(auditEvents)` anywhere.
- `status:` writes to `surgicalCases` outside `transition()`.
- `console.log` in `api/` · `dangerouslySetInnerHTML` in `src/` · secrets in `VITE_*`.
- `createHash("sha256")` used for PIN/password hashing.

---

## 9. Definition of done

- [ ] `npm run verify` green; integration tests green when DB touched.
- [ ] Invariants in §5 hold; security gate green.
- [ ] New/changed procedure has: auth, capability, zod bounds, tenant filter, audit (mutations), test.
- [ ] Migration generated and committed (if schema changed); seed updated.
- [ ] UI states (loading/empty/error), mobile check, a11y labels.
- [ ] `TODO.md` item ticked; docs updated if behavior changed.

---

## 10. Parallel agents — lanes (disjoint file ownership)

Establish the shared contract first (schema, `contracts/`, `middleware.ts`, tokens) — sequential. Then:

| Lane | Owns | Must not touch |
|---|---|---|
| **platform** | `api/_server/{boot,context,middleware}.ts`, `lib/`, `vercel.json`, CI, `scripts/` | pages |
| **data** | `db/**`, `api/_server/queries/**` | `src/**` |
| **api** | `api/_server/routers/**` | `db/schema.ts` (request changes from data lane) |
| **ui** | `src/**` (not `src/components/ui/**` primitives) | `api/**`, `db/**` |
| **qa** | `**/__tests__/**`, `e2e/**` | production code |

Integrate on one branch after lanes finish; run `npm run verify` once on the merged result.

---

## 11. Do not

- Do not trust `localStorage` for identity or capabilities.
- Do not add a second serverless function, an ORM, a state library, or a CSS framework.
- Do not run `db:push`, `DROP`, or `TRUNCATE` against any shared database.
- Do not commit `.env*`, dumps, or real patient data. Seed data is synthetic.
- Do not remove the audit call from any mutation.
- Do not copy code from `grok-ref/` or `abe-dashboard/` wholesale — they are **visual references** only
  (different stacks: TanStack Start, Next.js 14).
