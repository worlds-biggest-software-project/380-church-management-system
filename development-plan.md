# Church Management System (ChMS) — Phased Development Plan

> Project: 380-church-management-system · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three `data-model-suggestion-*.md` files into a concrete, phased build. It adopts **Data Model Suggestion 1 (Entity-Centric Normalized Relational)** as the canonical schema — financial integrity, IRS-compliant contribution statements, FK enforcement, and clear pastoral/financial/engagement separation are core requirements that normalisation serves best. The custom-field flexibility from Suggestion 2 is incorporated via `custom_fields_json` / `config_json` columns. Suggestion 3's event-sourcing approach is deferred — its analytics value is captured by an append-only `audit_log` plus materialised reporting views rather than a full CQRS rebuild.

---

## Core Requirements Summary

**What it does:** A unified, AI-native, self-hostable platform managing the full congregational lifecycle — membership, giving, groups, volunteers, events, children's check-in, and communications — for churches whose small admin teams currently juggle spreadsheets, accounting software, and consumer apps.

**Primary personas:**
- **Church administrator** (often part-time/volunteer, low technical sophistication) — manages people, runs reports, sends communications. Ease of use is the #1 differentiator.
- **Pastoral staff** — needs unified engagement view, at-risk detection, pastoral notes (privacy-controlled).
- **Finance/treasurer** — giving entry, fund accounting, year-end statements (IRS Pub 1771).
- **Volunteer/group leader** — roster management, self-service scheduling.
- **Congregation member** — mobile PWA for giving, event sign-up, group rosters, viewing their giving statement.

**Key differentiators (AI-native):** first-time-visitor follow-up drafting, at-risk member detection, volunteer burnout prediction, natural-language reporting/search, sermon-to-discussion-guide generation, prayer-request auto-categorisation. No incumbent (Breeze, Planning Center, Pushpay, Rock RMS, ChurchCRM) offers a native AI layer; the two open-source options (Rock, ChurchCRM) have significant UX gaps.

**Deployment model:** Self-hostable via Docker Compose; cloud-deployable (single-tenant or multi-tenant SaaS). Mobile-first PWA member portal. Public REST API + webhooks.

**Integration surface:** Stripe (giving), Twilio (SMS), SendGrid/Postmark (email), QuickBooks/Xero (accounting export), Mailchimp/Zapier (ecosystem), LLM provider (OpenAI/Anthropic) for the AI layer, optional MCP server for AI-agent access.

**Standards compliance:** OWASP ASVS 4.0.3 / Top 10, NIST SP 800-63B (auth), PCI DSS SAQ-A (Stripe tokenisation only — no card data stored), IRS Pub 1771 (statements), GDPR/CCPA (consent + pastoral privacy), CAN-SPAM + RFC 8058 (one-click unsubscribe), TCPA (SMS consent), WCAG 2.2 (member UI), OpenAPI 3.1 (API), iCalendar RFC 5545 (event feeds), vCard RFC 6350 (contact export), ISO 4217/8601/3166 (currency/date/country), JWT RFC 7519 + OAuth 2.0/OIDC + WebAuthn (auth).

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript (strict) | One language across REST API, server logic, and the member-facing PWA reduces cognitive load for a community of contributors and avoids the Windows/.NET barrier that hampers Rock RMS adoption. Strong typing protects financial and pastoral data integrity. |
| Runtime | Node.js 22 LTS | Stable LTS with native `fetch`, test runner, and broad library support for Stripe/Twilio/SendGrid SDKs. |
| Web framework | Next.js 15 (App Router) | Server Components for the admin dashboard, Route Handlers for the REST API, and built-in PWA-friendly rendering. Single deployable unit simplifies self-hosting. Server Actions for form-heavy admin flows. |
| API style | REST over Route Handlers, documented with OpenAPI 3.1 | `features.md` and `standards.md` mandate a public REST API + webhooks; OpenAPI 3.1 is the recommended surface. JSON:API conventions optional but REST/JSON chosen for ecosystem familiarity (mirrors Breeze/Planning Center). |
| Database | PostgreSQL 16 | Suggestion 1 is a Postgres-native normalized schema using UUID, JSONB, `TEXT[]` + GIN indexes, partitioned `audit_log`, and partial indexes. Fund-accounting reporting needs relational joins; financial integrity needs FK enforcement. |
| ORM / query layer | Drizzle ORM | Type-safe schema-as-code matching the SQL DDL in Suggestion 1, first-class Postgres feature support (JSONB, arrays, partial indexes, partitions via raw SQL), and lightweight migrations. Avoids the heavier Prisma runtime. |
| Migrations | drizzle-kit + hand-written SQL for partitions | Generated migrations for the bulk; raw SQL for `audit_log` range partitioning and GIN indexes. |
| Background jobs / queue | BullMQ on Redis | Async workloads: webhook processing (Stripe/Twilio/SendGrid), bulk email/SMS sends, AI generation calls, recurring-giving reconciliation, follow-up sequence steps, statement generation. Redis also serves rate-limit and session needs. |
| Auth (staff) | Auth.js (NextAuth) with credentials + OIDC + WebAuthn | NIST SP 800-63B password rules, MFA, passkeys (WebAuthn L3 per standards.md), and SAML/OIDC SSO for multi-site denominational directories. JWT (RFC 7519) session tokens. |
| Auth (API) | OAuth 2.0 (Authorization Code + Client Credentials) + Personal Access Tokens | Mirrors Planning Center/Breeze API auth so third-party developers find it familiar. |
| Payments | Stripe (Payment Intents, Subscriptions, Checkout/Elements) | Tokenisation keeps the system in PCI DSS SAQ-A scope (no card data stored — only `pi_`/`ch_`/`sub_` references per Suggestion 1). Stripe Connect supports multi-site sub-accounts. |
| SMS | Twilio | Industry standard; TCPA consent tracked in `people.sms_opt_in`/`sms_consent_date`. |
| Email | SendGrid (primary), Postmark (alt) | SPF/DKIM/DMARC, one-click unsubscribe (RFC 8058), engagement webhooks (opens/clicks/bounces). |
| LLM provider | Provider-agnostic via Vercel AI SDK (OpenAI + Anthropic) | All AI features (follow-up drafting, at-risk detection, NL query, sermon guides, prayer categorisation) route through one abstraction so churches can choose or self-host a model. |
| Frontend UI | React 19 + Tailwind CSS + shadcn/ui | shadcn/ui gives accessible (WCAG 2.2) primitives and a clean dashboard matching the "easiest ChMS to onboard" bar set by Breeze. |
| PWA | Next.js + Serwist (service worker) + Web Push API | Member portal must be installable and push-capable (W3C Push API in standards.md) without a native app build. |
| Validation | Zod | Runtime validation of API payloads, form input, custom-field JSON Schema (2020-12), and webhook bodies; shared between client and server. |
| Testing | Vitest (unit/integration) + Playwright (E2E) + Testcontainers (real Postgres/Redis) | Vitest for fast unit/integration; Testcontainers spins real Postgres for migration and query tests; Playwright drives the PWA and check-in kiosk flows. |
| Linting / formatting | Biome | Single fast tool for lint + format; replaces ESLint+Prettier. |
| Type checking | `tsc --noEmit` strict | Catch financial/pastoral type errors at build. |
| Package manager | pnpm | Fast, disk-efficient, good monorepo support. |
| Monorepo | pnpm workspaces + Turborepo | Separate `apps/web` (Next.js), `packages/db`, `packages/core`, `packages/ai`, `packages/integrations` for clear boundaries and parallel CI. |
| Containerisation | Docker + Docker Compose | Self-host story: one `docker compose up` brings up app, Postgres, Redis. Multi-stage Dockerfile. |
| PDF generation | `@react-pdf/renderer` | Year-end contribution statements (IRS Pub 1771) and children's check-in labels rendered server-side. |
| Label printing | Browser print + ZPL/EPL templates | Dymo/Zebra check-in labels via configurable `check_in_label_format`. |
| Key libraries | `stripe`, `twilio`, `@sendgrid/mail`, `ical-generator` (RFC 5545), `vcard4` (RFC 6350), `@simplewebauthn/server`, `bullmq`, `zod`, `ai` (Vercel AI SDK), `@modelcontextprotocol/sdk` (optional MCP server) | Domain-specific, all actively maintained. |

### Project Structure

```
church-management-system/
├── package.json                     # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── biome.json
├── tsconfig.base.json
├── docker-compose.yml               # app + postgres + redis
├── Dockerfile                       # multi-stage build of apps/web + worker
├── .env.example
├── README.md
├── openapi/
│   └── chms.openapi.yaml            # generated + hand-curated OpenAPI 3.1 spec
├── packages/
│   ├── db/                          # Drizzle schema + migrations (Suggestion 1)
│   │   ├── src/
│   │   │   ├── schema/
│   │   │   │   ├── churches.ts
│   │   │   │   ├── users.ts
│   │   │   │   ├── people.ts
│   │   │   │   ├── households.ts
│   │   │   │   ├── groups.ts
│   │   │   │   ├── groupMemberships.ts
│   │   │   │   ├── volunteerPositions.ts
│   │   │   │   ├── events.ts
│   │   │   │   ├── attendance.ts
│   │   │   │   ├── donations.ts
│   │   │   │   ├── pledges.ts
│   │   │   │   ├── communications.ts
│   │   │   │   ├── aiSuggestions.ts
│   │   │   │   ├── auditLog.ts
│   │   │   │   └── index.ts
│   │   │   ├── client.ts             # pooled Postgres client
│   │   │   └── seed.ts               # demo church fixture
│   │   ├── migrations/               # drizzle-kit + raw SQL (partitions, GIN)
│   │   └── drizzle.config.ts
│   ├── core/                         # framework-agnostic domain logic
│   │   ├── src/
│   │   │   ├── people/               # member lifecycle, household linking
│   │   │   ├── giving/               # fund accounting, statements, reconciliation
│   │   │   ├── groups/               # roster, membership
│   │   │   ├── volunteers/           # scheduling, burnout calc
│   │   │   ├── events/               # registration, capacity, recurrence (RRULE)
│   │   │   ├── checkin/              # security codes, custody/allergy, labels
│   │   │   ├── comms/                # segmentation, consent, unsubscribe
│   │   │   ├── reporting/            # standard report queries
│   │   │   ├── rbac/                 # role/permission matrix, field-level access
│   │   │   ├── audit/                # audit log writer
│   │   │   └── statements/           # IRS Pub 1771 statement builder
│   │   └── ...
│   ├── ai/                           # LLM abstraction + AI features
│   │   ├── src/
│   │   │   ├── provider.ts           # Vercel AI SDK wrapper
│   │   │   ├── prompts/              # versioned prompt templates
│   │   │   ├── followup.ts
│   │   │   ├── atRisk.ts
│   │   │   ├── burnout.ts
│   │   │   ├── nlQuery.ts            # NL → safe SQL/filter
│   │   │   ├── sermonGuide.ts
│   │   │   └── prayerCategorise.ts
│   │   └── ...
│   ├── integrations/                 # external service adapters
│   │   ├── src/
│   │   │   ├── stripe/
│   │   │   ├── twilio/
│   │   │   ├── sendgrid/
│   │   │   ├── quickbooks/
│   │   │   └── webhooks/             # signature verification per provider
│   │   └── ...
│   └── shared/                       # zod schemas, types, constants
│       └── src/
├── apps/
│   ├── web/                          # Next.js 15 (admin dashboard + member PWA + API)
│   │   ├── app/
│   │   │   ├── (admin)/              # staff dashboard (RBAC-gated)
│   │   │   ├── (portal)/             # member PWA
│   │   │   ├── (checkin)/            # kiosk check-in mode
│   │   │   ├── api/                  # Route Handlers (REST + webhooks + OAuth)
│   │   │   │   ├── v1/
│   │   │   │   └── webhooks/
│   │   │   └── ...
│   │   ├── public/
│   │   │   ├── manifest.webmanifest  # PWA manifest
│   │   │   └── sw.js                 # Serwist service worker
│   │   └── ...
│   ├── worker/                       # BullMQ worker process (jobs, AI, sends)
│   │   └── src/
│   │       ├── index.ts
│   │       └── jobs/
│   └── mcp/                          # optional MCP server (read-only, audited)
│       └── src/
└── tests/
    ├── fixtures/                     # sample churches, people, CSV imports, SARIF-style fixtures
    ├── integration/
    └── e2e/
```

---

## Phase 1: Foundation — Monorepo, Database, Auth, RBAC, Audit

### Purpose
Establish the buildable skeleton every later phase depends on: the pnpm/Turborepo monorepo, the canonical PostgreSQL schema (Suggestion 1), Drizzle migrations, the staff authentication system (NIST SP 800-63B), the role-based access control matrix with pastoral/financial field-level gating, and the append-only audit log. After this phase a developer can run `docker compose up`, log in as a seeded admin, and the database enforces tenancy and access control.

### Tasks

#### 1.1 — Monorepo & toolchain bootstrap

**What**: Stand up the pnpm workspace, Turborepo pipeline, Biome, TypeScript strict config, Docker Compose, and CI.

**Design**:
- `pnpm-workspace.yaml` declares `packages/*` and `apps/*`.
- `tsconfig.base.json`: `"strict": true`, `"noUncheckedIndexedAccess": true`, `"moduleResolution": "bundler"`, path aliases `@chms/db`, `@chms/core`, `@chms/ai`, `@chms/integrations`, `@chms/shared`.
- `turbo.json` tasks: `build`, `lint`, `typecheck`, `test`, `test:e2e` with dependency graph (`build` depends on `^build`).
- `docker-compose.yml` services: `postgres:16` (volume-backed), `redis:7`, `web` (Next.js), `worker`. Healthchecks on Postgres/Redis.
- `.env.example` enumerates every required env var: `DATABASE_URL`, `REDIS_URL`, `NEXTAUTH_SECRET`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `TWILIO_*`, `SENDGRID_API_KEY`, `OPENAI_API_KEY`/`ANTHROPIC_API_KEY`, `AI_PROVIDER`, `APP_BASE_URL`.
- Multi-stage `Dockerfile`: deps → build → runtime (non-root user).

**Testing**:
- `Unit: tsconfig resolves @chms/* aliases → import compiles`.
- `Integration: docker compose up → /api/health returns 200 with { status: "ok", db: "up", redis: "up" }`.
- `CI: pnpm lint && pnpm typecheck && pnpm test pass on clean checkout`.

#### 1.2 — Database schema (Drizzle, Suggestion 1)

**What**: Implement the 13-table normalized schema from Suggestion 1 as Drizzle schema modules plus migrations, with a `group_memberships` junction added (Suggestion 1 references group membership but the junction is implied).

**Design**:
- Translate each DDL block from `data-model-suggestion-1.md` into Drizzle: `churches`, `users`, `people`, `households`, `groups`, `volunteer_positions`, `events`, `attendance`, `donations`, `pledges`, `communications`, `ai_suggestions`, `audit_log`.
- Add junction `group_memberships`:
  ```ts
  // group_memberships
  id            uuid pk default gen_random_uuid()
  church_id     uuid not null references churches(id)
  group_id      uuid not null references groups(id)
  person_id     uuid not null references people(id)
  role          text not null check in ('member','leader','co_leader','apprentice')
  joined_date   date not null default now()
  status        text not null default 'active' check in ('active','inactive','left')
  unique (group_id, person_id)
  ```
- Preserve all CHECK constraints, partial indexes, GIN indexes (`tags`, `groups.tags`), and the `audit_log` `PARTITION BY RANGE (created_at)` via raw SQL migration that also creates monthly partitions for the current ± 1 year and a default partition.
- `config_json`/`custom_fields_json` typed as `jsonb` with a Zod-validated `ChurchConfig` and `CustomFieldDefinition[]` shape in `@chms/shared` (incorporates Suggestion 2's custom-field flexibility).
- Money is always `bigint` cents + ISO 4217 `currency_code`; never floats.
- Pooled client in `packages/db/src/client.ts` using `postgres` driver with `max` connections from env.

**Testing**:
- `Integration (Testcontainers): run all migrations against fresh Postgres → 14 tables exist with expected columns`.
- `Integration: insert donation with non-cents float → type error at compile; insert negative amount allowed only for refunds path (asserted in 1.x giving)`.
- `Integration: audit_log partition routing → row with created_at in May 2026 lands in the May partition`.
- `Integration: insert person with household_id FK → enforced; orphan household_id → FK violation`.
- `Unit: ChurchConfig Zod schema rejects unknown fund-rule string, accepts 'irs_pub_1771'`.

#### 1.3 — Staff authentication (Auth.js)

**What**: Email/password + TOTP MFA + WebAuthn passkey login for staff `users`, issuing JWT sessions.

**Design**:
- Auth.js with Credentials provider; passwords hashed with Argon2id (NIST SP 800-63B: ≥8 chars, breached-password check against a local k-anonymity HIBP range API or skip if offline, no forced rotation, no composition rules).
- MFA: TOTP enrolment storing secret encrypted at rest; `mfa_enabled` on `users`.
- WebAuthn via `@simplewebauthn/server`; `webauthn_registered` flag. New `webauthn_credentials` table:
  ```ts
  id, user_id fk, credential_id text unique, public_key bytea,
  counter bigint, transports text[], created_at
  ```
- Session: short-lived JWT (15 min) + rotating refresh token; `last_login_at` updated on success.
- OIDC/SAML provider stubs configured but disabled by default (enabled in Phase 8 multi-site).
- Login rate limiting via Redis (10 attempts / 15 min / IP+email).

**Testing**:
- `Unit: Argon2id verify(correct) → true; verify(wrong) → false`.
- `Integration: login with valid creds, mfa disabled → JWT returned, last_login_at updated`.
- `Integration: login with valid creds, mfa enabled, no TOTP → 401 mfa_required`.
- `Integration: 11th failed login within window → 429`.
- `E2E (Playwright): register passkey, log out, log in with passkey → dashboard reached`.

#### 1.4 — RBAC & field-level access control

**What**: A permission matrix mapping the 13 `users.role` values to resource/action permissions, with special gating for pastoral and financial fields.

**Design**:
- `packages/core/src/rbac`: declarative matrix `Record<Role, Record<Resource, Action[]>>` where `Resource ∈ {people, pastoral_notes, donations, pledges, statements, groups, volunteers, events, checkin, communications, ai, settings, users}` and `Action ∈ {read, create, update, delete, export}`.
- Two cross-cutting field gates:
  - `pastoral_notes` (and `medical_notes`, `custody_alert`) readable only by roles with `pastoral_notes:read` (`senior_pastor`, `associate_pastor`, `campus_pastor`, `children_director` for child medical only).
  - Financial fields (`donations`, `pledges`) readable only by `finance`, `senior_pastor`, `system_admin`.
- `can(user, action, resource, opts?)` returns boolean; `assertCan` throws `ForbiddenError` (→ HTTP 403).
- Serialization layer strips gated fields from API/UI responses based on caller role (defence in depth — not just route guards).
- Multi-tenancy: every query is scoped by `church_id` derived from the session; a `withChurchScope` helper enforces it. Cross-church access only for `system_admin`.

**Testing**:
- `Unit: can(office_staff, 'read', 'pastoral_notes') → false; can(senior_pastor, ...) → true`.
- `Unit: serialize(person, asRole='office_staff') → pastoral_notes/medical_notes absent`.
- `Unit: can(finance, 'read', 'donations') → true; can(volunteer_coordinator, 'read', 'donations') → false`.
- `Integration: GET /v1/people/:id as office_staff → 200 without pastoral_notes; as senior_pastor → includes it`.
- `Integration: query scoped to church A cannot read church B rows (system_admin excepted)`.

#### 1.5 — Audit log writer

**What**: Centralised audit writer flagging pastoral and financial access, used by all later mutations and sensitive reads.

**Design**:
- `audit(ctx, { action, entityType, entityId, changes?, pastoralData?, financialData? })` inserts into `audit_log` with `church_id`, `user_id`, `actor_type`, `ip_address`, `session_id`.
- Reads of pastoral notes and financial records emit `action='read'` audit rows with the appropriate flags (GDPR/data-handling demonstrability).
- Writer is fire-and-forget through BullMQ in production (never blocks the request) but synchronous in tests for assertion.

**Testing**:
- `Integration: update person pastoral_notes → audit row with pastoral_data=true, changes diff captured`.
- `Integration: GET donation as finance → audit row action='read', financial_data=true`.
- `Unit: changes diff excludes unchanged fields`.

---

## Phase 2: People & Households Core

### Purpose
Deliver the heart of any ChMS: member/household profiles with custom fields, tags, relationship mapping, milestones, and the member lifecycle (visitor → member). This is the data foundation every other module references. After this phase, admins can manage the directory through the dashboard and the REST API.

### Tasks

#### 2.1 — Person & household CRUD (core + API)

**What**: Domain services and REST endpoints for creating, reading, updating, listing, merging, and soft-deleting people and households.

**Design**:
- Service signatures (`@chms/core/people`):
  ```ts
  createPerson(ctx, input: NewPersonInput): Promise<Person>
  updatePerson(ctx, id: string, patch: PersonPatch): Promise<Person>
  linkToHousehold(ctx, personId, householdId, role): Promise<void>
  mergePeople(ctx, primaryId, duplicateId): Promise<Person> // re-points all FKs, audits
  listPeople(ctx, filter: PeopleFilter, page): Promise<Paginated<Person>>
  ```
- `PeopleFilter`: `{ memberStatus?, tags?, campusId?, groupId?, hasEmail?, ageRange?, search? }`. `search` does trigram match on name/email.
- REST (OpenAPI 3.1 documented):
  - `POST /v1/people`, `GET /v1/people` (filter+paginate via query), `GET /v1/people/:id`, `PATCH /v1/people/:id`, `DELETE /v1/people/:id` (soft → `member_status='inactive'` unless `system_admin` hard-deletes), `POST /v1/people/:id/merge`.
  - `POST /v1/households`, `GET/PATCH /v1/households/:id`, `POST /v1/households/:id/members`.
- All input validated with Zod; money/dates ISO 8601; phone E.164.
- Custom fields validated against the church's `custom_fields` definitions (JSON Schema 2020-12).
- `member_count`/`last_attendance_date` are derived columns updated by triggers/jobs, never set directly by clients.

**Testing**:
- `Unit: createPerson with invalid email → ValidationError naming 'email'`.
- `Unit: custom field 'Campus Preference' not in church definitions → rejected`.
- `Integration: mergePeople re-points donations, attendance, group_memberships, volunteer_positions from duplicate → primary; duplicate marked merged; audit row written`.
- `Integration: DELETE as office_staff → soft delete (inactive); as system_admin with ?hard=true → row removed`.
- `Integration: listPeople filter tags=['young_adults'] uses GIN index (EXPLAIN contains Bitmap Index Scan)`.

#### 2.2 — Tags, custom fields, milestones

**What**: Tagging, per-church custom field definitions, and milestone tracking (baptism, membership class, salvation, first visit).

**Design**:
- Tag endpoints: `POST/DELETE /v1/people/:id/tags`, `GET /v1/tags` (distinct, with counts).
- Custom field definition CRUD lives under settings (`PATCH /v1/settings/custom-fields`); each definition: `{ key, label, type: 'text'|'number'|'select'|'multiselect'|'date'|'boolean', options?, required?, appliesTo: 'person'|'household' }`.
- Milestones stored as typed columns on `people` (`baptism_date`, `membership_class_date`, `salvation_date`, `first_visit_date`, `membership_date`) plus a free `custom` milestone list in `custom_fields_json`.
- Setting `membership_date` transitions `member_status` to `member` (lifecycle rule in 2.3).

**Testing**:
- `Unit: add duplicate tag → idempotent, no duplicate in array`.
- `Integration: define required select field, create person without it → 422; with valid option → 201`.
- `Integration: set baptism_date → audit row; milestone appears in person timeline (2.4)`.

#### 2.3 — Member lifecycle state machine

**What**: Enforce valid `member_status` transitions and auto-derive engagement fields.

**Design**:
- States: `visitor → first_time_visitor → regular_attender → member → inactive`; plus terminal `transferred`, `deceased`. Allowed transition table enforced in `core/people/lifecycle.ts`.
- `first_time_visitor` set automatically on first attendance row; `regular_attender` after N attendances in M weeks (church-configurable, default 3-in-6); `inactive` after no attendance/giving in `inactive_threshold_days` (default 180) via a daily job.
- Transitions emit audit rows and (later) trigger AI follow-up (Phase 7).

**Testing**:
- `Unit: transition member → first_time_visitor → InvalidTransitionError`.
- `Unit: 3 attendances in 6 weeks → regular_attender`.
- `Integration: daily inactivity job marks person inactive after threshold; sets last_attendance_date correctly`.

#### 2.4 — Admin people UI + member timeline

**What**: Dashboard screens for the directory, profile, household view, and a unified engagement timeline.

**Design**:
- Server Components list with virtualised table, inline tag editing (Breeze-style), filter sidebar matching `PeopleFilter`.
- Profile page assembles, in role-aware sections: identity, household members, groups, giving summary (finance/pastor only), volunteer commitments, attendance, pastoral notes (gated), milestones.
- Timeline aggregates from `attendance`, `donations` (gated), `group_memberships`, milestones, and `communications` into one chronological feed — delivering Suggestion 3's "engagement journey" value without event sourcing.
- WCAG 2.2 AA: keyboard nav, labelled inputs, focus management.

**Testing**:
- `E2E: search 'Smith', open profile, add tag inline → tag persists on reload`.
- `E2E: office_staff profile view shows no giving/pastoral sections`.
- `Unit: timeline merges sources sorted desc by date`.

---

## Phase 3: Giving & Financial Management

### Purpose
Ship the financially load-bearing module: Stripe-backed online giving (one-time + recurring), manual cash/check batch entry, fund allocation, pledges, and IRS Pub 1771-compliant year-end statements. This is regulated functionality — correctness and auditability are non-negotiable. After this phase a church can accept and reconcile gifts and produce tax statements.

### Tasks

#### 3.1 — Stripe online giving (one-time + recurring)

**What**: Donor-facing giving flow using Stripe Elements/Checkout, storing only Stripe references.

**Design**:
- Card + ACH via Stripe Payment Intents; recurring via Stripe Subscriptions (`sub_`).
- No PAN/CVV/bank data ever touches the server (PCI DSS SAQ-A). Server stores `stripe_payment_id`, `stripe_charge_id`, `stripe_subscription_id` only (per Suggestion 1).
- Multi-fund: donor selects fund(s) from `churches.config_json.giving_funds`; one `donations` row per fund allocation.
- Endpoints: `POST /v1/giving/intent` (creates PaymentIntent, returns client secret), `POST /v1/giving/recurring` (creates Subscription), `POST /v1/giving/recurring/:id/cancel`.
- `text_to_give` and `kiosk` flows reuse the same intent endpoint with a `source` tag.
- Stripe Connect: each church's `stripe_account_id` used for destination charges (multi-site sub-accounts in Phase 8).

**Testing**:
- `Integration (Stripe test mode): create intent → confirm with test card → webhook (3.2) creates completed donation`.
- `Unit: giving to fund not in church config → 422`.
- `Integration: cancel recurring → Stripe subscription cancelled, local status updated`.
- `Assert: no card/bank PAN persisted anywhere (grep donations columns)`.

#### 3.2 — Stripe webhook reconciliation

**What**: Verified Stripe webhook handler that is the source of truth for donation status.

**Design**:
- `POST /api/webhooks/stripe` verifies signature with `STRIPE_WEBHOOK_SECRET`; rejects unverified (401) and never trusts client-reported success.
- Idempotent: dedupe on Stripe `event.id` via a `processed_webhook_events` table.
- Handles `payment_intent.succeeded` → donation `completed`; `payment_intent.payment_failed` → `failed`; `charge.refunded` → `refunded`; `invoice.paid` (subscriptions) → recurring `pledge_payment`/`recurring` donation; `customer.subscription.deleted` → mark recurring cancelled.
- Updates `people.last_giving_date` and pledge `amount_fulfilled_cents` where linked.
- Heavy work enqueued to BullMQ; webhook returns 200 fast.

**Testing**:
- `Integration: valid signature payment_intent.succeeded → donation completed, last_giving_date set`.
- `Integration: invalid signature → 401, no donation`.
- `Integration: same event.id twice → second is no-op (idempotent)`.
- `Integration: charge.refunded → status refunded, pledge fulfilled reduced`.

#### 3.3 — Manual batch entry (cash/check)

**What**: Treasurer workflow to enter cash/check gifts in batches with totals reconciliation.

**Design**:
- `batch_id` groups manual donations. Endpoints: `POST /v1/giving/batches` (open), `POST /v1/giving/batches/:id/donations`, `POST /v1/giving/batches/:id/close` (asserts entered total == declared total, else 409).
- `payment_method ∈ {cash, check}`, `check_number` captured for checks.
- Only `finance`/`system_admin` roles.

**Testing**:
- `Integration: close batch with mismatch declared vs entered → 409 with diff`.
- `Integration: matching totals → batch closed, donations completed`.
- `Integration: volunteer_coordinator attempts batch → 403`.

#### 3.4 — Pledges & campaigns

**What**: Pledge tracking against campaigns with fulfilment progress.

**Design**:
- `pledges` per Suggestion 1; `POST /v1/pledges`, progress derived from linked `donations.pledge_id`.
- Job recomputes `amount_fulfilled_cents`, `payments_made`, and flips status to `completed` when target met.
- Pledge vs giving report shows outstanding per campaign.

**Testing**:
- `Unit: fulfilment = sum of completed linked donations`.
- `Integration: donation linked to pledge → fulfilment increments; reaching total → status completed`.

#### 3.5 — Year-end contribution statements (IRS Pub 1771)

**What**: Generate per-person and per-household tax-compliant contribution statements as PDF.

**Design**:
- Statement builder (`core/statements`) aggregates `donations` for a `fiscal_year` where `tax_deductible=true`, grouped by household (IRS requires household aggregation for joint filers) and itemised by fund.
- Required Pub 1771 content: church name/address/EIN, donor name, fiscal year, total deductible amount, statement that "no goods or services were provided in exchange except intangible religious benefits" (configurable per `tax_receipt_rules`), per-gift listing with dates and amounts, quid-pro-quo disclosure for gifts ≥ $75 where applicable.
- Jurisdiction-aware: `config_json.tax_receipt_rules ∈ {irs_pub_1771, uk_gift_aid, ca_cra, au_dgr}` selects the template; MVP implements `irs_pub_1771` fully, others stubbed with TODO.
- `@react-pdf/renderer` template; `POST /v1/statements/generate?fiscalYear=2026` enqueues a batch job; statements optionally emailed (Phase 4) with `receipt_sent`/`receipt_sent_at` set.

**Testing**:
- `Unit: statement aggregates only tax_deductible completed donations in fiscal_year`.
- `Unit: married couple in one household → single combined statement`.
- `Unit: required Pub 1771 boilerplate present; gift ≥ $75 quid-pro-quo line included when configured`.
- `Integration: generate batch → PDF per household, receipt_sent flag updated`.
- `Fixture: golden-file comparison of rendered statement text against committed expected fixture`.

#### 3.6 — Giving dashboard & accounting export

**What**: Giving trends dashboard and QuickBooks/CSV export.

**Design**:
- Materialised reporting view `mv_giving_by_period` (church, campus, period_type, period_key, fund, total_cents, donation_count, unique_givers) refreshed by job — realising Suggestion 3's `rm_giving_dashboard` value relationally.
- Dashboard charts: weekly/monthly/yearly totals, by fund, by method, recurring vs one-time, first-time givers.
- Export: `GET /v1/giving/export?format=qbo|csv&period=...` producing QuickBooks-importable IIF/CSV and generic CSV.

**Testing**:
- `Integration: refresh view → totals match SUM(donations) for period`.
- `Unit: CSV export columns and cent→dollar formatting correct`.
- `Integration: only finance/pastor can access export`.

---

## Phase 4: Communications (Email, SMS, Push) with Compliance

### Purpose
Enable segmented bulk and transactional messaging across email, SMS, and push, with consent and unsubscribe compliance baked in (CAN-SPAM, RFC 8058, TCPA, GDPR). This module powers receipts, reminders, and the follow-up sequences that the AI layer (Phase 7) drives.

### Tasks

#### 4.1 — Segmentation engine

**What**: Build recipient segments from people filters and saved segments.

**Design**:
- Reuses `PeopleFilter` plus group/campus/giving predicates; compiles to a parameterised SQL `WHERE` (never string-concatenated — injection-safe).
- `recipient_filter_json` on `communications` stores the segment; `recipient_count` computed at send time.
- Respects `do_not_contact`, channel opt-in flags, and per-channel suppression lists.

**Testing**:
- `Unit: filter {memberStatus:['member'], tags:['choir']} → expected SQL params`.
- `Integration: segment excludes do_not_contact and email_opt_in=false members`.

#### 4.2 — Email sending (SendGrid) with compliance

**What**: Bulk + transactional email with SPF/DKIM/DMARC and one-click unsubscribe.

**Design**:
- `core/comms/email` builds messages; every bulk email includes `List-Unsubscribe` + `List-Unsubscribe-Post` headers (RFC 8058 one-click) and a footer unsubscribe link (CAN-SPAM).
- Sends batched through BullMQ to respect provider rate limits and Gmail/Yahoo bulk rules (>5k/day).
- Unsubscribe endpoint `POST /unsubscribe/:token` flips `email_opt_in=false`, writes audit + consent record.
- SendGrid event webhook (`/api/webhooks/sendgrid`, signature-verified) updates `delivered/opened/clicked/bounced/unsubscribed` counts.

**Testing**:
- `Unit: bulk email has List-Unsubscribe + List-Unsubscribe-Post headers`.
- `Integration: unsubscribe token → email_opt_in false, audit written`.
- `Integration: sendgrid bounce webhook → bounced_count incremented`.

#### 4.3 — SMS sending (Twilio) with TCPA consent

**What**: SMS broadcast and transactional with enforced prior consent.

**Design**:
- Send blocked unless `sms_opt_in=true` with a recorded `sms_consent_date` (TCPA).
- STOP/HELP handling via Twilio inbound webhook (`/api/webhooks/twilio`): STOP → `sms_opt_in=false`.
- Per-message segment counting and cost estimate surfaced to admin before send.

**Testing**:
- `Unit: send to non-consented number → blocked, reason 'no_sms_consent'`.
- `Integration: inbound STOP → sms_opt_in false, audit row`.

#### 4.4 — Push notifications (Web Push)

**What**: PWA push via W3C Push API + VAPID.

**Design**:
- Subscription stored per member portal device in `push_subscriptions` table; send via web-push library with VAPID keys.
- Used for event reminders, giving receipts, volunteer reminders.

**Testing**:
- `Integration: subscribe → endpoint stored; send → 201 from push service (mocked)`.
- `Integration: expired subscription (410) → pruned`.

#### 4.5 — Communications composer UI + history

**What**: Admin composer with segment preview, scheduling, and delivery analytics.

**Design**:
- Composer: channel pick, segment builder with live recipient count, template variables (`{{first_name}}`), schedule or send now.
- History view with per-campaign delivered/open/click/bounce/unsubscribe.
- Drafts persist as `communications` rows with `status='draft'`.

**Testing**:
- `E2E: build segment, preview count, schedule send → status scheduled; job sends at time → status sent`.
- `E2E: composer warns when segment includes do_not_contact members (count shown as suppressed)`.

---

## Phase 5: Events, Attendance & Registration

### Purpose
Provide event creation (including recurring services via iCal RRULE), public registration with capacity controls, RSVP, and attendance tracking — the backbone for check-in (Phase 6) and attendance reporting. After this phase churches can publish events, take registrations, and record who attended.

### Tasks

#### 5.1 — Event CRUD + recurrence

**What**: Event management with RFC 5545 recurrence and iCal feed.

**Design**:
- `events` per Suggestion 1; recurrence stored as `recurrence_rule` (RRULE), instances materialised on read for a window or persisted as child events (`parent_event_id`) for ones with registrations.
- `GET /v1/events`, `POST /v1/events`, `PATCH/DELETE /v1/events/:id`.
- iCal feed: `GET /calendar/:churchId.ics` (RFC 5545) for public/subscribed calendars, built with `ical-generator`.
- Schema.org Event JSON-LD emitted on public event pages (standards.md).

**Testing**:
- `Unit: weekly RRULE expands to correct instance dates within window`.
- `Integration: iCal feed validates against RFC 5545 parser; contains VEVENTs`.
- `Integration: editing a recurring parent does not corrupt instances with registrations`.

#### 5.2 — Registration & capacity

**What**: Public RSVP/registration with capacity, deadline, waitlist, and optional paid registration.

**Design**:
- `attendance` rows with `status='registered'`; `registered_count` derived.
- Capacity enforcement is transactional (`SELECT ... FOR UPDATE` on event row or atomic count check) to prevent overbooking; waitlist when full.
- Paid registration reuses the Stripe intent flow (Phase 3); registration confirmed only on payment success.
- Registration deadline enforced server-side.

**Testing**:
- `Integration: concurrent registrations at capacity-1 → exactly one succeeds, other waitlisted (no overbooking)`.
- `Integration: registration after deadline → 409`.
- `Integration: paid registration unpaid → remains pending, not counted`.

#### 5.3 — Attendance recording & reporting

**What**: Mark attendance (manual, self, or via check-in) and attendance trend reports.

**Design**:
- `POST /v1/events/:id/attendance` to mark `attended`/`no_show`; bulk mark for service attendance.
- Recording attendance updates `people.last_attendance_date`, `attendance_streak`, and feeds lifecycle (Phase 2.3).
- Materialised `mv_attendance_by_period` for trend dashboards (realising Suggestion 3's `rm_event_attendance`): totals, first-time, children, volunteers, week-over-week and year-over-year deltas.

**Testing**:
- `Unit: attendance_streak increments on consecutive weeks, resets on gap`.
- `Integration: mark attended → last_attendance_date set, lifecycle re-evaluated`.
- `Integration: attendance report deltas vs previous week correct`.

---

## Phase 6: Children's & Family Check-In

### Purpose
Deliver secure children's check-in — the highest-trust workflow in a ChMS — with label printing, allergy and custody alerts, pickup security codes, and volunteer-to-child ratio awareness. Designed from first principles to avoid the unclear patent landscape noted in `standards.md`. After this phase a church can run a check-in kiosk on a Sunday.

### Tasks

#### 6.1 — Check-in session & security codes

**What**: Kiosk check-in that generates matching pickup codes and writes child-safety alerts.

**Design**:
- Check-in extends the `attendance` table (Suggestion 1 design decision #2): `is_child_check_in=true`, generated `security_code` (random 4-char, unique per event+active), `checked_in_by` (guardian), `room_assignment`, `allergy_alert_shown`, `custody_alert_shown`.
- Code generation: cryptographically random, collision-checked among active check-ins for the event.
- Custody alert (`people.custody_alert`) and allergies (`people.allergies`) surfaced to the volunteer at check-in and check-out; check-out requires matching `security_code` and records `checked_out_to`.
- Endpoints (kiosk-scoped token, audited): `POST /v1/checkin` (household → eligible children → rooms → codes), `POST /v1/checkout` (verify code, record pickup).

**Testing**:
- `Unit: security code generation is collision-free among active event check-ins`.
- `Integration: checkout with wrong code → 403, child not released, audit row`.
- `Integration: custody_alert present → checkout blocks unauthorised pickup, requires staff override (audited)`.
- `Integration: allergy alert flagged on check-in payload`.

#### 6.2 — Label printing

**What**: Print child + guardian + security-code labels in configurable formats.

**Design**:
- `check_in_label_format` from `config_json` (e.g., `dymo_30252`, `zebra_zd410`).
- Generate ZPL/EPL or PDF label with child name, room, allergies, security code, date; matching parent claim-check label.
- Browser print path for non-thermal printers.

**Testing**:
- `Unit: ZPL template renders child name, code, allergies for dymo_30252`.
- `Integration: label_printed flag set after print confirmation`.

#### 6.3 — Ratio & roster awareness

**What**: Surface volunteer-to-child ratios per room for compliance.

**Design**:
- Per-room live counts (checked-in children vs serving volunteers from `attendance.is_serving`) with a configurable target ratio; warn when exceeded.

**Testing**:
- `Unit: ratio computed = children / serving volunteers per room; warning when below target`.

---

## Phase 7: AI-Native Layer

### Purpose
Implement the differentiating AI features that no incumbent offers, as first-class capabilities with human-in-the-loop approval and full auditability. All AI suggestions persist in `ai_suggestions` (Suggestion 1) with confidence, explanation, and accept/reject feedback for a learning loop. Pastoral data passed to models is role-gated and consent-aware.

### Tasks

#### 7.1 — LLM provider abstraction & guardrails

**What**: Provider-agnostic AI client with prompt versioning, redaction, and audit.

**Design**:
- `packages/ai/provider.ts` wraps Vercel AI SDK; `AI_PROVIDER` env selects OpenAI/Anthropic/self-hosted.
- Every AI call: (1) checks RBAC for the data it reads, (2) redacts fields the requesting context may not use, (3) records an `ai_suggestions` row and an audit entry, (4) stores `model_id` + prompt version.
- Prompt templates versioned in `packages/ai/prompts/*` with explicit system prompts; structured output via Zod-validated JSON schema (AI SDK `generateObject`).

**Testing**:
- `Unit: provider switch via env → correct client`.
- `Unit: redaction removes pastoral fields for non-pastoral context before prompt`.
- `Integration (mocked LLM): suggestion persisted with model_id, confidence, explanation`.

#### 7.2 — First-time-visitor follow-up drafting & sequences

**What**: AI-drafted, personalised follow-up messages and a multi-step follow-up sequence.

**Design**:
- On `first_time_visitor` lifecycle entry, enqueue follow-up: draft a personalised email/SMS using name, service attended, and church voice config. Output is a `visitor_follow_up` suggestion (`status='pending'`) for staff approval — not auto-sent.
- Approved drafts feed the communications module (Phase 4); sequence state (step, channel, timing) tracked; subsequent steps scheduled by job.
- System prompt template (structure):
  ```
  System: You are a warm, concise church follow-up assistant for {{church_name}}.
          Tone: {{tone_config}}. Never invent facts about the person.
  User:   First-time visitor {{first_name}} attended {{service_name}} on {{date}}.
          Draft a {{channel}} message inviting them to {{next_step}}.
          Constraints: <= {{max_len}} chars, include one clear call to action.
  Output (JSON): { subject?, body, suggested_send_at }
  ```

**Testing**:
- `Integration (mocked LLM): new first_time_visitor → pending visitor_follow_up suggestion created`.
- `Integration: accept suggestion → communication queued; reject → feedback stored, nothing sent`.
- `Unit: sequence advances step on schedule; stops if member replies/visits again`.

#### 7.3 — At-risk member & giving-pattern detection

**What**: Detect declining engagement/giving and surface pastoral-outreach suggestions.

**Design**:
- Scheduled job computes engagement features per member (attendance recency/streak decline, giving lapse vs prior cadence, group participation drop) and produces `at_risk_member`/`giving_pattern` suggestions with `confidence` and `explanation` listing the risk factors.
- A deterministic rules baseline runs first; the LLM adds narrative explanation and prioritisation. Risk surfaced on the member 360 view and a pastoral worklist.
- Pastoral-data access gated; suggestions only visible to pastoral roles.

**Testing**:
- `Unit: member with 6-week attendance gap + lapsed monthly gift → flagged with both factors`.
- `Integration: at_risk suggestion visible to senior_pastor, hidden from office_staff`.

#### 7.4 — Volunteer burnout prediction & slot suggestion

**What**: Predict burnout and recommend rotation-balanced volunteer slots, respecting household coordination.

**Design**:
- Uses `volunteer_positions.consecutive_weeks`/`max_consecutive` (Suggestion 1) + recent serve history to raise `burnout_warning` when approaching the threshold.
- `volunteer_slot` suggestions fill upcoming gaps preferring under-utilised, available, non-burnt-out volunteers; honours `prefer_same_service_as_household` so families serve together.

**Testing**:
- `Unit: consecutive_weeks >= max_consecutive-1 → burnout_warning`.
- `Unit: slot suggestion excludes blackout_dates and burnt-out volunteers; prefers household alignment`.

#### 7.5 — Natural-language query & search

**What**: Let non-technical staff ask questions like "first-time givers from last quarter who haven't been contacted".

**Design**:
- NL → a constrained, parameterised query over an allow-listed schema view (never free-form SQL execution). The model emits a structured `QuerySpec` (entities, filters, date ranges) validated by Zod; the app compiles `QuerySpec` to safe SQL.
- Results render in a table with a "save as segment/report" action. Every NL query audited.
- Hard guardrails: no DML, church-scoped, pastoral/financial fields excluded unless caller authorised.

**Testing**:
- `Unit: NL → QuerySpec for "first-time givers last quarter not contacted" → expected filters`.
- `Unit: QuerySpec compiler rejects any non-allow-listed field`.
- `Integration: NL query as office_staff cannot surface giving amounts`.

#### 7.6 — Sermon-to-discussion-guide & prayer categorisation

**What**: Generate small-group discussion guides from sermon transcripts; auto-categorise prayer requests.

**Design**:
- Sermon transcript (uploaded or from a media URL) → `sermon_discussion_guide` suggestion: summary + discussion questions + scripture references, attachable to small groups.
- Incoming prayer requests → `prayer_categorisation` suggestion with category tags and optional sensitivity flag; auto-tagging assists pastoral triage (never auto-shares private requests).

**Testing**:
- `Integration (mocked LLM): transcript → guide with N questions persisted as suggestion`.
- `Unit: prayer request marked private is never included in shared outputs`.

---

## Phase 8: Multi-Site, Member PWA, Public API & SSO

### Purpose
Open the platform to congregation members and the wider ecosystem, and support multi-campus churches — the mid-market gap incumbents gate behind enterprise pricing. After this phase members self-serve via the PWA, third parties integrate via the documented API + webhooks, and multi-site churches get per-campus security with consolidated reporting.

### Tasks

#### 8.1 — Multi-site campus support

**What**: Per-campus data scoping, security, and consolidated roll-up reporting.

**Design**:
- Uses Suggestion 1's self-referencing `churches` (`parent_church_id`, `campus_type`). `campus_pastor` role scoped to their campus; `system_admin` sees the whole org.
- People have `primary_campus_id` but may attend any campus; giving and attendance reports roll up to parent or drill to campus.
- Stripe Connect sub-accounts per campus for separated payouts; consolidated giving view across campuses.

**Testing**:
- `Integration: campus_pastor sees only own-campus people/events; parent reports aggregate all campuses`.
- `Integration: giving rolls up parent = sum of campuses`.

#### 8.2 — Member PWA portal

**What**: Installable mobile-first member portal for giving, event sign-up, group rosters, and statements.

**Design**:
- `(portal)` route group; member auth (passwordless email link or passkey). Members see only their own data + public directory (where `directory_visible`).
- Features: give now / manage recurring, RSVP to events, view my groups + roster, view/download my giving statement, update contact + communication preferences (consent self-service), receive push.
- Serwist service worker for offline shell + installability; `manifest.webmanifest`. WCAG 2.2 AA.

**Testing**:
- `E2E: member logs in via magic link, gives one-time gift, sees it on statement`.
- `E2E: member toggles sms_opt_in → consent recorded with date`.
- `E2E: PWA installable (manifest + SW registered), offline shell loads`.

#### 8.3 — Public REST API, OAuth, webhooks (OpenAPI 3.1 / AsyncAPI 3.0)

**What**: Stabilise and document the public API with OAuth 2.0, Personal Access Tokens, scopes, rate limiting, and outbound webhooks.

**Design**:
- OAuth 2.0 Authorization Code + Client Credentials; PATs for scripts (mirrors Breeze/Planning Center). Scopes per resource (`people:read`, `giving:read`, etc.) enforced via the RBAC matrix.
- Versioned `/v1` surface fully described in `openapi/chms.openapi.yaml` (3.1); generated client-friendly. Rate limiting via Redis.
- Outbound webhooks for key events (person.created, donation.completed, event.registration, checkin.completed) documented with AsyncAPI 3.0; HMAC-signed payloads, retry with backoff via BullMQ.

**Testing**:
- `Integration: OAuth client_credentials token with people:read can GET /v1/people, cannot POST`.
- `Integration: outbound webhook signed with HMAC; receiver verifies; failed delivery retried`.
- `Contract: every /v1 route present in openapi.yaml (spec-vs-routes test)`.

#### 8.4 — SSO (SAML 2.0 / OIDC) & SCIM provisioning

**What**: Denominational/enterprise SSO and user provisioning.

**Design**:
- Enable the Auth.js OIDC/SAML providers stubbed in Phase 1; map IdP groups → ChMS roles.
- SCIM 2.0 endpoints (`/scim/v2/Users`) for provisioning staff from a denominational directory (RFC 7643/7644).

**Testing**:
- `Integration: OIDC login provisions/links user, maps role from claim`.
- `Integration: SCIM create user → users row; deactivate → is_active=false`.

#### 8.5 — Optional MCP server (read-only, audited)

**What**: Expose church data to AI agents (e.g., a pastor-facing assistant) via a controlled MCP server.

**Design**:
- `apps/mcp` using `@modelcontextprotocol/sdk`; read-only tools (`search_people`, `get_member_360`, `giving_summary`) that go through the same RBAC + audit + pastoral redaction as the REST API.
- Disabled by default; per-church opt-in with a scoped token.

**Testing**:
- `Integration: MCP search_people respects church scope + pastoral redaction; every call audited`.
- `Integration: MCP cannot perform any mutation`.

---

## Phase 9: Data Portability, Reporting, Hardening & Release

### Purpose
Address the cross-cutting concerns that make the platform adoptable and trustworthy: migration in/out (most incumbents lock data in — a key opportunity), the standard reports library, accessibility, security hardening to OWASP ASVS, and a polished self-host release.

### Tasks

#### 9.1 — Import & export (data portability)

**What**: CSV import from competitors and full data export (vCard, iCal, CSV, JSON).

**Design**:
- CSV importers with column mapping + dry-run validation for people, households, giving history (Breeze/Planning Center/ChurchCRM column profiles).
- Exports: people → vCard 4.0 (RFC 6350) + CSV; events → iCal; full church → JSON archive (a candidate "ChMS interchange format" noted as absent in standards.md).
- GDPR/CCPA data-subject export + erasure workflow (per-person export bundle; erasure soft-anonymises while preserving financial audit per legal retention).

**Testing**:
- `Integration: import Breeze CSV → people + households created, dry-run reports row errors first`.
- `Unit: person → vCard validates; church export round-trips via importer`.
- `Integration: GDPR erasure anonymises PII but retains donation amounts/dates for tax audit`.

#### 9.2 — Standard reports library

**What**: The reports churches expect: attendance trends, giving trends, group health, volunteer coverage, new-visitor follow-up funnel.

**Design**:
- Reports backed by the materialised views (`mv_giving_by_period`, `mv_attendance_by_period`) plus group-health and volunteer-coverage aggregations (relational equivalents of Suggestion 3's `rm_group_health` / `rm_volunteer_board`).
- Each report: filters, chart, table, CSV/PDF export, save-as-saved-report. RBAC-gated (giving reports finance/pastor only).

**Testing**:
- `Integration: group health report avg_attendance matches meeting attendance data`.
- `Integration: visitor follow-up funnel counts match lifecycle + communications data`.

#### 9.3 — Security hardening (OWASP ASVS 4.0.3 / Top 10)

**What**: Systematic security pass.

**Design**:
- Address OWASP Top 10: parameterised queries everywhere (A03), authz checks on every route + object-level (A01), secrets via env/secret manager, security headers (CSP, HSTS), CSRF on cookie flows, rate limiting, dependency scanning in CI, encrypted-at-rest pastoral notes option.
- ASVS checklist tracked in repo; PCI SAQ-A re-verified (no card data); audit-log integrity (append-only, partitioned).

**Testing**:
- `Integration: IDOR attempt (person from another church) → 403/404, audited`.
- `Integration: SQLi payload in filter → safely parameterised, no error leak`.
- `CI: dependency audit + SAST gate; CSP/HSTS headers present`.

#### 9.4 — Accessibility, i18n scaffolding & release packaging

**What**: WCAG 2.2 AA conformance, i18n/multi-currency scaffolding, and the self-host release.

**Design**:
- Automated (axe) + manual a11y audit on admin + portal + kiosk; keyboard and screen-reader passes.
- i18n via message catalogues; ISO 4217 multi-currency and ISO 3166 already in schema — wire currency formatting and locale dates (ISO 8601) through the UI (full translations are backlog).
- Release: pinned Docker images, `docker compose up` quickstart, seed/demo data, backup/restore scripts, upgrade/migration guide, `.env` documentation, versioned changelog.

**Testing**:
- `E2E (axe): admin dashboard, member portal, check-in kiosk → no critical violations`.
- `Integration: fresh docker compose up + seed → reachable admin + member login`.
- `Integration: backup → restore → data intact`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (monorepo, DB, auth, RBAC, audit)   ─── required by everything
    │
Phase 2: People & Households Core                       ─── requires Phase 1
    │
    ├── Phase 3: Giving & Finance        ─── requires Phase 2
    │       │
    │       └── (statements/receipts use Phase 4 email)
    │
    ├── Phase 4: Communications          ─── requires Phase 2 (can parallel Phase 3)
    │
    └── Phase 5: Events & Attendance     ─── requires Phase 2 (can parallel Phases 3,4)
            │
            └── Phase 6: Children's Check-In  ─── requires Phase 5
    │
Phase 7: AI-Native Layer                 ─── requires Phases 2,3,4,5 (consumes their data)
    │
Phase 8: Multi-Site, Member PWA, Public API, SSO ─── requires Phases 2-6 (8.1/8.2/8.3/8.4/8.5 parallelisable)
    │
Phase 9: Portability, Reporting, Hardening, Release ─── requires all prior; final gate
```

Parallelism:
- After Phase 2, **Phases 3, 4, and 5 can be built concurrently** by separate developers (distinct modules, shared only via `@chms/core` + `@chms/db`).
- Within Phase 8, **8.1 (multi-site), 8.2 (PWA), 8.3 (API), 8.4 (SSO), 8.5 (MCP)** are largely independent once Phase 7 lands.
- The MVP (per `features.md` "Must-have") is complete after **Phases 1–6 + Phase 8.2 (PWA)**. Phase 7 (AI) and the rest of Phase 8 map to "Should-have (v1.1)". Phase 9 spans cross-cutting concerns drawn into the timeline as features land.

---

## Definition of Done (per phase)

Every phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests pass; new code has meaningful coverage of happy-path and edge cases.
3. `pnpm lint` (Biome) passes with no errors.
4. `pnpm typecheck` (`tsc --noEmit` strict) passes across all touched packages.
5. `docker compose build` succeeds and the stack starts cleanly.
6. The phase's user-facing capability works end-to-end (Playwright E2E where a UI/kiosk/portal flow exists).
7. New configuration options are documented in `.env.example` and the README, with safe defaults.
8. New or changed REST endpoints appear in `openapi/chms.openapi.yaml` (3.1) and pass the spec-vs-routes contract test; outbound webhooks documented in AsyncAPI 3.0.
9. Database changes ship as Drizzle/SQL migrations that apply cleanly forward on a fresh and an existing database (partitions/GIN indexes included).
10. RBAC enforced for every new resource (route-level and serialization-level), with audit logging for pastoral and financial access.
11. No card/bank PAN or other PCI-scoped data is persisted (Stripe references only); pastoral fields remain role-gated.
12. Relevant standards are satisfied and noted in the PR (e.g., RFC 8058 headers on bulk email, IRS Pub 1771 statement content, RFC 5545 iCal validity, WCAG 2.2 AA for new UI).
```
