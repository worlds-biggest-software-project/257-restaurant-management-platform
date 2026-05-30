# Restaurant Management Platform — Phased Development Plan

> Project: 257-restaurant-management-platform · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four
`data-model-suggestion-*.md` files. It adopts **Data Model 1 (Entity-Centric Normalised Relational, 3NF)**
as the canonical schema — the safest choice for a v1 MVP where correctness, PCI DSS / HACCP / GDPR audit
trails, and BI-tool compatibility matter more than schema flexibility. JSONB columns from that model
(`settings_json`, `layout_json`, `items_json`, etc.) absorb the per-location / per-jurisdiction variability
that Data Model 3 argued for, without forking the schema.

---

## Core Requirements (synthesised)

**What it does.** An AI-native, open-source restaurant operations platform that unifies POS, kitchen
display (KDS), table/floor-plan management, menu & inventory, labour & scheduling, reservations, loyalty,
multi-location control, online ordering / third-party delivery, and reporting — replacing fragmented
proprietary point solutions with a standards-compliant, hardware-agnostic alternative.

**Who uses it.** (1) Independent operators choosing a first/replacement POS; (2) multi-location chain
operations directors standardising tech; (3) ghost-kitchen / multi-brand operators; (4) F&B managers at
hotels, stadiums, and event venues.

**Key differentiators.** Open source; no proprietary hardware lock-in (runs on standard tablets +
Stripe Terminal / EMV terminals); AI built in, not bolted on (menu engineering, demand forecasting,
kitchen throughput sequencing, labour scheduling, guest sentiment); an open, Schema.org-compatible
data model; and an MCP server exposing operations as tools for AI agents.

**MVP feature set (from features.md "Must-have").** Secure PCI-DSS-compliant POS + payment gateway;
KDS with routing, ticket management, course sequencing; menu & inventory management with stock alerts;
multi-location centralised control & sync; mobile/tablet order-taking with offline support; basic
employee timekeeping & labour-cost tracking; sales reporting & daily reconciliation; third-party delivery
integration (Uber Eats, DoorDash, Grubhub).

**Post-MVP (v1.1 / backlog).** AI demand forecasting; advanced analytics; intelligent labour scheduling;
reservations + guest SMS/email; loyalty + CRM; ghost-kitchen multi-brand; dynamic pricing; food-safety
(FSMA/HACCP) workflows; guest sentiment AI; voice ordering; supply-chain intelligence.

**Deployment model.** Cloud-native SaaS (multi-tenant) **and** self-hostable via Docker Compose; mobile/
tablet clients are offline-capable PWAs that sync on reconnect. API-first (OpenAPI 3.1).

**Integration surface.** Stripe (Payments + Terminal/EMV); Uber Eats Marketplace, DoorDash, Deliverect
(delivery aggregation); OpenTable / Resy / Yelp (reservations); 7shifts / Gusto (labour/payroll); Xero
(accounting); LLM providers (OpenAI / Anthropic) via a provider-agnostic gateway; Twilio / SendGrid
(guest comms); MCP clients (AI agents).

**Standards compliance.** PCI DSS v4.0 (tokenised refs only — SAQ A via Stripe-hosted elements);
EMV (Stripe Terminal); OAuth 2.0 / OIDC + JWT (RFC 7519, 6749); OpenAPI 3.1; JSON Schema 2020-12;
CloudEvents 1.0 (audit log + webhooks); Schema.org Restaurant/Menu/MenuItem (JSON-LD export);
FDA allergen guidance (9 allergens) + menu-labelling (calories); ISO 22000 / HACCP (temp logs,
checklists); GDPR / CCPA (consent, retention, erasure); ISO 3166 / 4217 (geo & currency codes);
OWASP ASVS; MCP.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | **TypeScript (Node.js 22 LTS)** | The product is API-, integration-, and real-time-heavy (POS ↔ KDS ↔ floor plan live sync, dozens of webhooks). One language across backend, offline PWA clients, and the MCP server minimises context-switching and lets order/menu types be shared. Square already ships first-class Node SDKs. |
| Language (AI/forecasting) | **Python 3.12 microservice** | Demand forecasting and menu-engineering models use the mature Python data stack (pandas, statsmodels/Prophet, scikit-learn). Isolated as a service so the Node core stays lean; called over HTTP/gRPC. |
| API framework | **NestJS (Fastify adapter)** | Opinionated modular DI architecture maps cleanly onto the domain modules (orders, menu, inventory, labour…). First-class OpenAPI 3.1 generation, validation pipes (class-validator), guards for RBAC, and WebSocket gateways for live KDS/floor-plan sync. |
| Real-time transport | **WebSockets (Socket.IO) + Redis adapter** | KDS tickets, floor-plan status, and order updates must push to many devices per location in <1 s. Redis pub/sub fans out across horizontally-scaled API nodes. |
| Database | **PostgreSQL 16** | Data Model 1 is a 50-table 3NF relational schema with RLS, JSONB, temporal tax rules, and heavy analytical joins — exactly Postgres' sweet spot. Row-Level Security enforces tenant isolation per the data model. |
| ORM / migrations | **Prisma** (+ raw SQL for RLS policies & analytics) | Type-safe queries shared with the TS types; declarative migrations. RLS policies and complex reporting views are managed via committed raw-SQL migration files Prisma applies. |
| Cache / queue / pub-sub | **Redis 7** | Triple duty: WebSocket fan-out, hot-path caching (active menus, table status), and as the BullMQ broker. |
| Task queue | **BullMQ** | Async workloads: delivery webhook processing, SMS/email dispatch, daily-summary rollups, forecast jobs, LLM calls. Retries + dead-letter queues for resilient webhook handling. |
| Payments | **Stripe (Payments + Terminal Connect)** | PCI DSS SAQ A via Stripe-hosted elements / Terminal; EMV card-present; Connect supports franchise multi-party payouts. We never touch raw PAN — only tokens, matching the `payment_transaction` schema. |
| Delivery aggregation | **Deliverect** primary; direct **Uber Eats / DoorDash** adapters behind a common interface | Deliverect unifies 1000+ channels through one API, drastically reducing per-platform integration cost; direct adapters added where commercials demand. |
| Frontend (back-office) | **Next.js 16 (App Router) + React + shadcn/ui + Tailwind** | Server components for dashboards/reporting; rich admin UI for menu, inventory, scheduling, analytics. |
| Frontend (POS / KDS / floor plan) | **React PWA (Vite) with offline support** | Tablet-native, installable, offline-capable. Local persistence via IndexedDB (Dexie); sync engine reconciles on reconnect using UUID PKs + an outbox pattern. |
| Offline sync | **Outbox + last-write-wins-with-vector-clock on `updated_at`** | UUID PKs (per the data model's key decision) allow terminals to generate IDs offline; an outbox queues mutations; server reconciles deterministically. |
| LLM access | **Provider-agnostic gateway (Vercel AI SDK)** | Routes to OpenAI/Anthropic with failover; structured-output + tool-calling power menu engineering and the MCP server. |
| AI tool surface | **MCP server (`@modelcontextprotocol/sdk`)** | Exposes inventory, sales, scheduling, and menu tools so external AI agents can operate the platform — the standards.md-flagged differentiator. |
| Auth | **OAuth 2.0 / OIDC + JWT**, PIN-hash for POS terminals | Multi-tenant SSO (corporate ↔ franchise) with per-tenant token isolation; bcrypt PIN hashes for fast in-venue staff login. |
| API docs | **OpenAPI 3.1** (auto-generated) + **GraphQL** read layer for analytics | REST for transactional writes; GraphQL for flexible multi-outlet reporting (per standards.md). |
| Testing | **Vitest** (unit), **Supertest** (HTTP integration), **Testcontainers** (real Postgres/Redis), **Playwright** (E2E), **pytest** (Python service) | Industry-standard per ecosystem; Testcontainers gives real-dependency integration without mocking the DB. |
| Quality | **ESLint + Prettier + tsc strict**; **ruff + mypy** (Python) | Enforced in CI; strict TS catches schema drift early. |
| Package manager | **pnpm** (monorepo workspaces) + **uv** (Python) | Fast, disk-efficient; workspaces share types across apps/packages. |
| Monorepo | **Turborepo** | Caches builds/tests across apps (api, back-office, pos-pwa, mcp, ai-service) and shared packages. |
| Containerisation | **Docker + docker-compose** | One-command self-host; reproducible CI; per-service images. |
| Observability | **OpenTelemetry → Prometheus/Grafana**, structured JSON logs | Required for an ops-critical system; KDS latency and payment success rates are SLOs. |
| CI/CD | **GitHub Actions** | Lint → typecheck → unit → integration (Testcontainers) → build → Docker. |

### Project Structure

```
restaurant-management-platform/
├── package.json                      # pnpm + turbo root
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml                # postgres, redis, api, back-office, ai-service
├── docker-compose.dev.yml
├── .github/workflows/ci.yml
├── apps/
│   ├── api/                          # NestJS core API (REST + WS + GraphQL)
│   │   ├── src/
│   │   │   ├── main.ts
│   │   │   ├── app.module.ts
│   │   │   ├── common/               # guards, interceptors, pipes, RLS tenant ctx
│   │   │   │   ├── auth/             # JWT, OIDC, RBAC guard, PIN auth
│   │   │   │   ├── tenancy/          # org/location context middleware → SET app.current_org_id
│   │   │   │   ├── audit/            # CloudEvents audit interceptor
│   │   │   │   └── errors/          # problem+json error filter
│   │   │   ├── modules/
│   │   │   │   ├── organisation/
│   │   │   │   ├── location/
│   │   │   │   ├── staff/            # staff, roles, RBAC
│   │   │   │   ├── menu/             # menu, sections, items, modifiers, allergens
│   │   │   │   ├── floorplan/        # floor plans, tables, table sessions (WS)
│   │   │   │   ├── orders/           # orders, line items, discounts, taxes
│   │   │   │   ├── payments/         # Stripe integration, transactions
│   │   │   │   ├── kds/              # kitchen stations, ticket routing (WS)
│   │   │   │   ├── inventory/        # ingredients, recipes, stock, POs, suppliers
│   │   │   │   ├── labour/           # schedules, shifts, time entries
│   │   │   │   ├── reservations/     # reservations + comms
│   │   │   │   ├── guests/           # guests, loyalty
│   │   │   │   ├── delivery/         # online order config, delivery fulfilment, adapters
│   │   │   │   ├── foodsafety/       # checklists, logs, temperature
│   │   │   │   ├── reporting/        # daily summary, analytics, GraphQL resolvers
│   │   │   │   ├── webhooks/         # inbound (Stripe, Deliverect) + outbound (CloudEvents)
│   │   │   │   └── ai/              # forecast/menu-eng proxy to ai-service
│   │   │   └── jobs/                 # BullMQ processors
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   └── migrations/           # incl. raw-SQL RLS + analytics views
│   │   └── test/
│   ├── back-office/                  # Next.js admin dashboard
│   ├── pos-pwa/                      # React PWA: POS, KDS, floor plan (offline)
│   ├── mcp-server/                   # MCP tool server
│   └── ai-service/                   # Python FastAPI: forecasting, menu engineering
│       ├── app/
│       ├── tests/
│       └── pyproject.toml
├── packages/
│   ├── types/                        # shared TS types (Order, MenuItem, …) + zod schemas
│   ├── sdk/                          # generated TS client from OpenAPI
│   ├── sync-engine/                  # offline outbox + reconciliation (shared)
│   ├── events/                       # CloudEvents envelope helpers + event-type registry
│   └── config/                       # shared eslint/tsconfig/tailwind presets
└── docs/
    ├── openapi.json                  # generated
    └── data-model.md
```

The structure is grouped by domain concern (not by phase); every phase adds modules/tables without
restructuring.

---

## Phase 1: Foundation — Monorepo, Database, Tenancy, Auth, Audit

### Purpose
Establish the skeleton everything else builds on: the Turborepo monorepo, a running NestJS API with
Postgres + Redis via Docker, the multi-tenant data foundation (organisation → location with Row-Level
Security), authentication (OIDC/JWT + PIN), RBAC scaffolding, and a CloudEvents audit trail. After this
phase a developer can authenticate, create an organisation and location, and every write is RLS-isolated
and audited.

### Tasks

#### 1.1 — Monorepo & Docker scaffolding

**What**: Stand up the pnpm/Turborepo workspace with the `api` app and a `docker-compose` dev stack.

**Design**:
- `pnpm-workspace.yaml` includes `apps/*` and `packages/*`.
- `turbo.json` pipelines: `build`, `lint`, `typecheck`, `test`, `test:int` (depends on `^build`).
- `docker-compose.dev.yml` services: `postgres:16` (port 5432, volume), `redis:7` (6379).
- `apps/api` NestJS app bootstrapped with the Fastify adapter; health endpoint:
  - `GET /healthz` → `{ status: "ok", db: "up"|"down", redis: "up"|"down" }`
- Shared `packages/config` exports `tsconfig.base.json` (`strict: true`, `noUncheckedIndexedAccess: true`),
  ESLint flat config, Prettier config.
- Env via `@nestjs/config` + zod validation:
  ```ts
  const EnvSchema = z.object({
    NODE_ENV: z.enum(['development','test','production']),
    DATABASE_URL: z.string().url(),
    REDIS_URL: z.string().url(),
    JWT_PUBLIC_KEY: z.string(), JWT_PRIVATE_KEY: z.string(),
    PORT: z.coerce.number().default(3000),
  });
  ```
- Error filter emits RFC 9457 `application/problem+json`.

**Testing**:
- `Unit: EnvSchema with missing DATABASE_URL → throws with "DATABASE_URL" in message`.
- `Integration (Testcontainers): GET /healthz with pg+redis up → 200 {status:"ok"}`.
- `Integration: GET /healthz with redis stopped → 200 {redis:"down"}` (degraded, not 500).
- `E2E: pnpm turbo build succeeds across all workspaces`.

#### 1.2 — Database schema (Data Model 1) & migrations

**What**: Implement the full 51-table 3NF schema from `data-model-suggestion-1.md` in Prisma plus raw-SQL
RLS migrations.

**Design**:
- Translate every table from Data Model 1 into `schema.prisma` (UUID PKs `@default(uuid())`, `@db.Uuid`;
  `Decimal @db.Decimal(p,s)` for money; `Json` for JSONB; `@updatedAt`).
- Phase 1 *physically* creates only the tenancy + staff/RBAC + audit tables; remaining tables are defined
  but introduced in their owning phases via additive migrations (no destructive changes later).
  Phase-1 tables: `organisation`, `location`, `staff_member`, `role`, `staff_role`, `staff_location`,
  `audit_log`.
- Raw-SQL migration `0001_rls.sql` enables RLS and adds the tenant policy on every tenant-scoped table:
  ```sql
  ALTER TABLE location ENABLE ROW LEVEL SECURITY;
  CREATE POLICY location_tenant_isolation ON location
    USING (organisation_id = current_setting('app.current_org_id')::uuid);
  ```
- Seed `allergen` (9 FDA allergens, `is_fda_major=true`), `dietary_tag`, and system `role`s
  (admin, manager, server, bartender, host, cook) with default `permissions` JSON.
- Reference codes validated against ISO 3166-1 alpha-2, ISO 4217, IANA tz at the application layer.

**Testing**:
- `Integration (Testcontainers): prisma migrate deploy → all Phase-1 tables exist; allergen has 9 rows`.
- `Integration: insert location for org A, SET app.current_org_id=B, SELECT → 0 rows (RLS isolates)`.
- `Integration: insert location with country_code='ZZ' → app validation rejects (not ISO 3166-1)`.
- `Unit: migration files apply and roll back cleanly in CI`.

#### 1.3 — Tenancy context & RLS wiring

**What**: Middleware that resolves the active org/location from the JWT and sets the Postgres session var
so RLS applies to every query.

**Design**:
- `TenancyMiddleware` reads `org_id` (+ optional `location_id`) from the validated JWT claims, stores them
  in an AsyncLocalStorage `RequestContext`.
- A Prisma `$extends` / client-extension wraps each transaction with
  `SET LOCAL app.current_org_id = $1` before queries run, guaranteeing RLS scoping even for raw queries.
- `@CurrentOrg()` and `@CurrentLocation()` param decorators expose context to controllers.
- Requests without a resolvable org → 401 (except auth/health endpoints on an allowlist).

**Testing**:
- `Integration: request with org-A JWT querying org-B's location id → 404 (RLS hides the row)`.
- `Integration: request with no org claim to a tenant route → 401`.
- `Unit: TenancyMiddleware extracts org_id from a signed JWT fixture`.

#### 1.4 — Authentication & RBAC

**What**: OIDC/JWT auth for back-office users, PIN auth for POS terminals, and a permission-checking guard.

**Design**:
- `POST /auth/login` (email+password or OIDC code exchange) → `{ accessToken, refreshToken }` (RS256 JWT,
  claims: `sub`, `org_id`, `roles[]`, `locations[]`, `perms[]`, `exp`). Per standards.md: RFC 7519/6749.
- `POST /auth/pin` (location-scoped `{ pin }`) → short-lived terminal JWT; validates against
  `staff_member.pin_hash` (bcrypt) scoped to the device's location.
- `JwtAuthGuard` (verify signature/exp) + `PermissionsGuard` reading `@RequirePermissions('pos.void_order')`
  metadata against the token's `perms[]` (derived from `role.permissions`).
- Refresh-token rotation; tokens isolated per tenant (a token for org A is invalid for org B).
- PCI DSS 4.0: MFA required for accounts with payment-system permissions (enforced at login for
  `perms` containing `payments.*`).

**Testing**:
- `Integration: valid credentials → 200 with JWT containing correct org_id & perms`.
- `Integration: PIN auth with wrong PIN → 401, audit log records auth.failed`.
- `Unit: PermissionsGuard denies request lacking required perm → 403`.
- `Integration: payment-perm user without MFA → login returns mfa_required challenge`.

#### 1.5 — CloudEvents audit interceptor

**What**: Auto-record state-changing operations into `audit_log` in CloudEvents 1.0 shape.

**Design**:
- `AuditInterceptor` on all mutating routes writes `audit_log` rows:
  `event_type` (e.g. `order.created`), `event_source` (`pos|api|webhook|system|backoffice`),
  `entity_type`, `entity_id`, `changes_json` ({field:{old,new}}), `actor_id/actor_type`, `ip_address`,
  `user_agent`. Maps 1:1 to CloudEvents `type`, `source`, `subject`, `data`.
- `packages/events` exports the canonical `EventType` enum (the registry used by both audit + outbound
  webhooks in Phase 8) and a `toCloudEvent(record)` serialiser.

**Testing**:
- `Integration: create a location → audit_log row event_type='location.created' with changes_json`.
- `Unit: toCloudEvent maps fields to spec-compliant envelope (specversion '1.0')`.
- `Integration: failed mutation (validation error) → no audit row written`.

---

## Phase 2: Menu Catalogue & Tax

### Purpose
Build the menu domain — the catalogue every order draws from — plus tax configuration that orders will
need. Includes menus, sections, items, modifier groups/modifiers, allergens, dietary tags, kitchen-station
mapping, Schema.org JSON-LD export, and temporal tax rates. After this phase an operator can model a full
multi-location menu with FDA allergen/calorie data and jurisdiction tax rules.

### Tasks

#### 2.1 — Menu, sections, items, modifiers CRUD

**What**: REST CRUD for the menu hierarchy with org-scoped items assigned to location-scoped menus.

**Design**:
- Migrate menu tables: `menu`, `menu_location`, `menu_section`, `menu_item`, `menu_section_item`,
  `modifier_group`, `menu_item_modifier_group`, `modifier`.
- Endpoints (all org-scoped via RLS):
  - `POST /menus`, `GET /menus`, `GET /menus/:id` (deep — sections→items→modifier groups), `PATCH`, `DELETE`
  - `POST /menu-items`, `PATCH /menu-items/:id`, `GET /menu-items?sku=`
  - `POST /menus/:id/sections`, `POST /menu-sections/:id/items` (assign with `sort_order`)
  - `POST /modifier-groups`, `POST /modifier-groups/:id/modifiers`
- Shared zod/DTO types in `packages/types`:
  ```ts
  type MenuItem = { id: string; name: string; kitchenName?: string; basePrice: string;
    currencyCode: string; costPrice?: string; calories?: number; prepTimeMins?: number;
    isAlcoholic: boolean; isActive: boolean; allergens: AllergenRef[]; dietaryTags: string[] };
  type ModifierGroup = { id: string; name: string; selectionType: 'single'|'multi'|'quantity';
    minSelections: number; maxSelections?: number; isRequired: boolean; modifiers: Modifier[] };
  ```
- Validation: `selection_type='single'` ⇒ `max_selections<=1`; `is_required` ⇒ `min_selections>=1`.

**Testing**:
- `Integration: create item with basePrice → persisted with Decimal precision`.
- `Integration: assign item to section → GET /menus/:id returns nested item in correct sort_order`.
- `Unit: required modifier group with min_selections=0 → ValidationError`.
- `Integration: org-A user fetching org-B item id → 404 (RLS)`.

#### 2.2 — Allergens, dietary tags, kitchen-station routing

**What**: Attach FDA allergens + dietary tags to items and map items to KDS stations.

**Design**:
- Migrate `menu_item_allergen`, `menu_item_dietary_tag`, `kitchen_station`, `menu_item_station`.
- `PUT /menu-items/:id/allergens` body `[{allergenId, severity:'contains'|'may_contain'|'trace'}]`.
- `PUT /menu-items/:id/stations` body `[stationId]` — drives KDS routing in Phase 6.
- Allergen reference is read-only (seeded); custom allergens allowed with `is_fda_major=false`.

**Testing**:
- `Integration: set item allergens to [milk, peanuts] → GET item returns both with severity`.
- `Integration: route item to Grill+Saute stations → mapping persisted`.
- `Unit: severity outside enum → ValidationError`.

#### 2.3 — Tax configuration (temporal)

**What**: Jurisdiction-aware, time-bounded tax rates assignable to locations.

**Design**:
- Migrate `tax_rate`, `location_tax_rate`.
- `POST /tax-rates` `{ name, rate, jurisdiction(ISO 3166-2), appliesTo:'all'|'food'|'alcohol'|'non_food',
  isInclusive, effectiveFrom, effectiveUntil }`; `POST /locations/:id/tax-rates` to assign.
- Resolver `getActiveTaxRates(locationId, at=now)` returns rates where `at ∈ [effective_from, effective_until)`
  and active — consumed by the Phase 4 order tax engine.

**Testing**:
- `Unit: getActiveTaxRates excludes a rate whose effective_until < now`.
- `Integration: alcohol-only rate applied → resolver returns it only for appliesTo in {all,alcohol}`.

#### 2.4 — Schema.org JSON-LD menu export

**What**: Public, cacheable JSON-LD menu feed for SEO and the open data-model goal.

**Design**:
- `GET /public/locations/:slug/menu.jsonld` → Schema.org `Restaurant` with nested `hasMenu` →
  `Menu`/`MenuSection`/`MenuItem`, including `nutrition.calories`, `suitableForDiet` (RestrictedDiet),
  and allergen annotations. No auth; org/location resolved by slug; cached in Redis 5 min.

**Testing**:
- `Integration: export validates against Schema.org Restaurant shape (snapshot test)`.
- `Unit: dietary_tag.schema_org_diet maps to suitableForDiet enum URI`.

---

## Phase 3: Floor Plan & Table Management (Real-Time)

### Purpose
Model the dining room and provide live, multi-device table status — the spatial context for dine-in
orders and reservations. Introduces the WebSocket layer (with Redis fan-out) reused by the KDS in Phase 6.

### Tasks

#### 3.1 — Floor plans & tables CRUD

**What**: Drag-and-drop floor plans with tables (coordinates, capacity, shape, combinable).

**Design**:
- Migrate `floor_plan`, `restaurant_table`. `layout_json` stores canvas metadata; `x_position`/
  `y_position` store per-table coordinates for the editor.
- `POST /locations/:id/floor-plans`, `POST /floor-plans/:id/tables`, `PATCH /tables/:id`
  (move/resize → updates coordinates). `GET /locations/:id/floor-plans?active=true`.

**Testing**:
- `Integration: duplicate table_number within a floor plan → 409 (unique constraint)`.
- `Integration: move table updates x/y_position`.

#### 3.2 — Table sessions & live status (WebSocket)

**What**: Real-time table lifecycle (available → seated → ordering → served → check_presented → cleaning)
synced to all devices at a location.

**Design**:
- Migrate `table_session`. Partial unique index ensures one active session per table
  (`WHERE cleared_at IS NULL`).
- `FloorPlanGateway` (Socket.IO ns `/floor`, room `loc:{locationId}`):
  - emits `table.status_changed { tableId, status, serverId, covers, seatedAt }`
  - Redis adapter fans out across API replicas.
- `POST /tables/:id/seat` `{ covers, serverId }` → creates session, emits event.
- `PATCH /table-sessions/:id` (status transitions, validated by a state machine).
- State machine: invalid transition (e.g. `available`→`served`) → 422.

**Testing**:
- `Integration: seat a table → second seat attempt while active → 409`.
- `Integration (WS): two clients in room; seat table → both receive table.status_changed`.
- `Unit: state machine rejects available→served`.

---

## Phase 4: Orders & Payments (POS Core) — the Heart

### Purpose
The core value proposition: take an order, build the check (line items, modifiers, discounts, taxes, tips,
service charges), and process payment securely via Stripe with PCI-DSS-compliant tokenisation. After this
phase the platform can run a real dine-in / takeout transaction end-to-end.

### Tasks

#### 4.1 — Order & line-item lifecycle

**What**: Create and manage orders with line items, modifiers, snapshotted pricing, and course numbers.

**Design**:
- Migrate `restaurant_order`, `order_line_item`, `order_line_item_modifier`, `order_discount`, `order_tax`.
- Line items **snapshot** `name`, `kitchen_name`, `unit_price`, `cost_price` at creation (per data-model
  decision #4) so later menu price changes don't rewrite history.
- Endpoints:
  - `POST /orders` `{ locationId, orderType, channel, tableSessionId?, serverId?, guestId? }`
  - `POST /orders/:id/items` `{ menuItemId, quantity, seatNumber?, courseNumber?, modifiers:[{modifierId}], notes? }`
  - `PATCH /orders/:id/items/:lineId` (qty, void with reason+actor)
  - `POST /orders/:id/discounts`, `POST /orders/:id/fire` (course → KDS, Phase 6)
- `order_number` generated per location (daily-reset counter in Redis, persisted).
- Order state machine: `open → sent_to_kitchen → in_progress → ready → served → closed`;
  plus `cancelled`/`voided` (void requires `pos.void_order` perm, records `voided_by`/`void_reason`).

**Testing**:
- `Integration: add item with 2 modifiers → line_total = (unitPrice + Σ price_adjustment) × qty`.
- `Integration: void line item without perm → 403; with perm → voided, audit logged`.
- `Unit: order_number increments per location, resets per business date`.
- `Unit: invalid state transition closed→open → 422`.

#### 4.2 — Tax & total calculation engine

**What**: Deterministic money math producing subtotal, per-tax rows, discounts, tips, service charge, grand total.

**Design**:
- `OrderCalculator.recompute(order)` pure function:
  1. `subtotal = Σ line_total`
  2. apply `order_discount` (line-level then order-level)
  3. for each active tax rate (from 2.3) matching `applies_to`, compute `order_tax` row on the taxable base
     (respect `is_inclusive`)
  4. `grand_total = subtotal − discount_total + tax_total + service_charge + tip_total`
- All math in integer minor units internally; persisted as `Decimal`. Rounding: half-up to currency minor unit.
- Recompute on every line/discount/tax mutation; results written to the order's summary columns.

**Testing**:
- `Unit: subtotal 100.00, 8.75% tax → tax_total 8.75, grand 108.75`.
- `Unit: inclusive 10% VAT on 110.00 → tax 10.00, taxable 100.00`.
- `Unit: order-level 10% discount + line discount stack in correct order`.
- `Unit: rounding 0.125 → 0.13 (half-up)`.

#### 4.3 — Stripe payment processing (PCI DSS SAQ A)

**What**: Authorise/capture payments and split checks via Stripe; store only tokens.

**Design**:
- Migrate `payment_transaction` (stores `gateway`, `gateway_txn_id`, `card_brand`, `card_last_four` only —
  never PAN; PCI DSS via Stripe-hosted elements / Terminal → SAQ A).
- `POST /orders/:id/payments` `{ method, amount, tipAmount }`:
  - card → create Stripe `PaymentIntent`; client confirms via Stripe.js / Terminal SDK
  - returns `clientSecret` for client-side confirmation
- `POST /webhooks/stripe` verifies signature → on `payment_intent.succeeded` marks transaction `captured`,
  and `closed` the order if fully paid; on failure marks `failed`.
- Split payments: multiple transactions per order; order closes when Σ captured ≥ grand_total.
- Refunds: `POST /payments/:id/refund` `{ amount, reason }` → Stripe refund, status `refunded`.
- Idempotency keys on all Stripe calls (order/payment id based).

**Testing**:
- `Integration (Stripe mock): create payment → PaymentIntent created, transaction pending, clientSecret returned`.
- `Integration: webhook payment_intent.succeeded → transaction captured, order closed`.
- `Integration: webhook with bad signature → 400, no state change`.
- `Integration: partial payments summing to total → order closes only after last`.
- `Assert: no test ever persists a raw card number (schema has no such column)`.

---

## Phase 5: POS / KDS / Floor-Plan PWA with Offline Sync

### Purpose
Deliver the tablet-native, offline-capable client that staff actually use, plus the sync engine that
reconciles offline mutations on reconnect — a table-stakes requirement (TouchBistro/Aloha parity) and
a hard differentiator for venues with flaky connectivity.

### Tasks

#### 5.1 — Offline sync engine

**What**: Reusable outbox + reconciliation library backing all PWA writes.

**Design**:
- `packages/sync-engine`: IndexedDB (Dexie) stores entities + an `outbox` of pending mutations
  (`{id, entity, op, payload, baseUpdatedAt, createdAt}`). UUID PKs generated client-side (data-model
  decision #1) so creates work offline.
- On reconnect: POST outbox to `POST /sync/batch`; server applies in order, returns per-mutation
  `{id, status:'applied'|'conflict', serverEntity}`. Conflict resolution: last-write-wins on `updated_at`
  with audit of overwrites; voids/payments are server-authoritative (never overwritten by stale client).
- Read model hydrated from server snapshots + applied local mutations.

**Testing**:
- `Unit: offline create then reconnect → mutation POSTed, server id reconciled (matches client UUID)`.
- `Integration: concurrent edits to same line item → newer updated_at wins, older recorded in audit`.
- `Integration: offline payment attempt → queued but flagged requires-online (not auto-applied)`.

#### 5.2 — POS order-taking UI

**What**: Tablet POS for building checks against a location's active menu.

**Design**:
- React PWA (Vite, installable, service worker caches active menu + app shell).
- Screens: floor plan (live status from 3.2) → seat table → menu grid (sections/items/modifiers) →
  check view (line items, course/seat, discounts) → payment (Stripe Terminal / card element).
- Uses `packages/sdk` (generated from OpenAPI) routed through the sync engine.

**Testing**:
- `E2E (Playwright): seat table → add item+modifier → fire course → take card payment (Stripe test) → check closes`.
- `E2E: go offline → add items → reconnect → order syncs, totals match`.

#### 5.3 — KDS display UI

**What**: Kitchen-facing ticket board driven by fired courses.

**Design**:
- Per-station view subscribing to the KDS WebSocket (Phase 6); tickets show item, modifiers, course,
  seat, elapsed timer; bump action marks line `ready`. Colour-coded by elapsed time (Square pattern).

**Testing**:
- `E2E: fire a course → ticket appears on the routed station(s) within 1s`.
- `E2E: bump ticket → line status ready, order advances`.

---

## Phase 6: Kitchen Display System (Routing & Sequencing)

### Purpose
Turn fired orders into routed, sequenced kitchen tickets across stations with course fire-timing — the
operational core that differentiates restaurant-purpose-built systems (Toast/Aloha) from generic POS.

### Tasks

#### 6.1 — Ticket routing & station fan-out

**What**: Route fired line items to their mapped kitchen stations in real time.

**Design**:
- On `POST /orders/:id/fire` (course-scoped), for each line item resolve `menu_item_station` mappings and
  set `fired_at`; `KdsGateway` (Socket.IO ns `/kds`, room `station:{stationId}`) emits
  `ticket.created { lineItemId, orderId, name, kitchenName, modifiers, courseNumber, seatNumber, firedAt }`.
- Items mapped to multiple stations appear on each; an `expo` station aggregates all.
- `POST /kds/lines/:id/bump` → status `ready`, `ready_at` set, emits `ticket.bumped`.

**Testing**:
- `Integration (WS): fire item routed to Grill → grill room receives ticket.created`.
- `Integration: bump line → ready_at set, order recomputes status`.

#### 6.2 — Course sequencing & fire timing

**What**: Hold later courses until released; track per-course timing.

**Design**:
- Courses fired explicitly (`courseNumber`); a course can be auto-released when the prior course's lines
  are all `served`, or manually via `POST /orders/:id/courses/:n/fire`.
- Track `fired_at → ready_at → served_at` per line for Speed-of-Service metrics (feeds Phase 9 reporting).

**Testing**:
- `Integration: fire course 1 only → course 2 items have null fired_at until released`.
- `Unit: SoS = ready_at − fired_at computed correctly`.

---

## Phase 7: Inventory, Recipes & Suppliers

### Purpose
Track ingredients, recipes, and stock with a transaction ledger so the platform can deplete inventory as
items sell, alert on par levels, and compute food cost — the foundation for Phase 10 AI demand forecasting.

### Tasks

#### 7.1 — Ingredients, recipes, stock ledger

**What**: Ingredient catalogue, recipes linking items→ingredients, and a stock + transaction ledger.

**Design**:
- Migrate `ingredient`, `recipe`, `recipe_ingredient`, `inventory_stock`, `inventory_transaction`.
- Depletion: when an `order_line_item` is fired/served, enqueue a BullMQ job that, per the item's recipe,
  writes negative `inventory_transaction` rows (`transaction_type='used'`, applying `waste_factor`) and
  decrements `inventory_stock` (data-model decision #5: stock = current, transactions = ledger).
- `POST /inventory/counts` records `counted` transactions and reconciles `quantity_on_hand`.
- CRUD: `POST /ingredients`, `POST /menu-items/:id/recipe`, `GET /locations/:id/inventory`.

**Testing**:
- `Integration: sell item whose recipe uses 200g flour (10% waste) → stock drops 220g, ledger row written`.
- `Integration: stock count adjusts on-hand and writes a 'counted' transaction`.
- `Unit: theoretical vs actual variance = ledger 'used' vs counts`.

#### 7.2 — Par-level alerts & suppliers / purchase orders

**What**: Low-stock alerts and purchase-order workflow.

**Design**:
- Migrate `supplier`, `purchase_order`, `purchase_order_line`.
- BullMQ scheduled job flags ingredients where `quantity_on_hand < par_level` → emits
  `inventory.low_stock` CloudEvent (back-office notification).
- PO lifecycle `draft→submitted→confirmed→received`; receiving writes `received` transactions and updates stock.

**Testing**:
- `Integration: stock below par → low_stock event emitted once (debounced)`.
- `Integration: receive PO → stock increases by quantity_received, ledger updated`.

---

## Phase 8: Multi-Location Control, Webhooks & Delivery Integration

### Purpose
Make the platform genuinely multi-location and connected: centralised cross-location administration,
an outbound CloudEvents webhook system, and inbound order ingestion from delivery channels via Deliverect
plus direct Uber Eats / DoorDash adapters. After this phase third-party orders flow into the same
order/KDS pipeline as in-house orders.

### Tasks

#### 8.1 — Outbound webhooks (CloudEvents)

**What**: Let external systems subscribe to platform events.

**Design**:
- `webhook_subscription` table (org-scoped: `url`, `event_types[]`, `secret`, `is_active`).
- BullMQ `webhook-dispatch` consumes the `EventType` stream (same registry as audit, Phase 1.5),
  POSTs CloudEvents 1.0 envelopes with HMAC-SHA256 `Webhook-Signature`; exponential-backoff retries →
  dead-letter after N attempts.

**Testing**:
- `Integration: order.created → subscribed endpoint receives signed CloudEvent`.
- `Integration: endpoint 500s → retried with backoff, then dead-lettered`.
- `Unit: HMAC signature verifies with the subscription secret`.

#### 8.2 — Delivery channel adapters (Deliverect + direct)

**What**: Common adapter interface ingesting external orders and syncing menus/availability.

**Design**:
- `DeliveryAdapter` interface:
  ```ts
  interface DeliveryAdapter {
    channel: 'deliverect'|'uber_eats'|'doordash'|'grubhub';
    syncMenu(locationId: string, menu: Menu): Promise<void>;
    handleInboundOrder(payload: unknown): Promise<RestaurantOrder>; // → normalised order, channel set
    updateOrderStatus(externalOrderId: string, status: DeliveryStatus): Promise<void>;
  }
  ```
- Migrate `online_order_config`, `delivery_fulfillment`. Inbound webhook
  `POST /webhooks/delivery/:channel` verifies signature, maps payload → `restaurant_order`
  (`channel`, `order_type='delivery'`) + `delivery_fulfillment`, then fires to KDS like any order.
- Menu push: on menu publish, enqueue `syncMenu` to all active channels for each location.
- Commission captured in `online_order_config.commission_rate` for reporting.

**Testing**:
- `Integration (mock Deliverect): inbound order webhook → normalised order created, routed to KDS`.
- `Integration: status update propagates to delivery_fulfillment`.
- `Unit: Uber Eats payload maps to canonical order line items + modifiers`.

#### 8.3 — Cross-location administration

**What**: Org-level views and bulk operations across locations (ghost-kitchen friendly).

**Design**:
- Org-scoped endpoints aggregating across `location` (respecting RLS at org level):
  `GET /org/locations/sales-summary`, bulk menu assignment `POST /menus/:id/locations`.
- Ghost kitchen / multi-brand: `location.is_ghost_kitchen`; shared org-level menu items assigned to
  multiple location "brands" via `menu_location` (data-model decision #3).

**Testing**:
- `Integration: ghost-kitchen location shares an org item across two brand menus`.
- `Integration: org sales summary aggregates two locations' daily totals`.

---

## Phase 9: Labour, Reservations, Loyalty & Reporting

### Purpose
Round out operations with workforce timekeeping/scheduling, reservations with guest comms, a loyalty/CRM
layer, food-safety logging, and the reporting/reconciliation dashboards that turn raw transactions into
the analytics buyers expect (Lightspeed-parity). These domains are independent and can be built in parallel.

### Tasks

#### 9.1 — Labour: scheduling & timekeeping

**What**: Weekly schedules, shifts, clock-in/out, and labour-cost tracking.

**Design**:
- Migrate `schedule`, `shift`, `time_entry`.
- `POST /schedules` (week), `POST /schedules/:id/shifts`, `POST /schedules/:id/publish`.
- `POST /time-entries/clock-in` / `clock-out` → computes `total_minutes`, `overtime_minutes` (>40h/wk or
  daily OT rule), `total_pay = rate × hours + OT`. Mandatory-break + early-clock-in rules (SpotOn pattern).
- Labour cost vs `schedule.labour_budget`; variance feeds reporting.

**Testing**:
- `Integration: clock in then out 8.5h later → total_minutes=510, total_pay computed`.
- `Unit: 45h week → 5h overtime at 1.5×`.
- `Integration: clock-in before shift window with early-block rule → 422`.

#### 9.2 — Reservations & guest communication

**What**: Reservations with multi-source attribution and SMS/email notifications.

**Design**:
- Migrate `reservation`, `guest`, `guest_allergen`. `source` tracks direct/OpenTable/Resy/Yelp/phone/walk_in.
- `POST /reservations`; on confirm/reminder/cancel → BullMQ job sends via Twilio (SMS) / SendGrid (email).
- Inbound reservation webhooks `POST /webhooks/reservations/:source` (OpenTable/Resy/Yelp) create/update
  reservations with `external_ref`.
- Seating a reservation links it to a `table_session` (Phase 3).
- GDPR: guest `consent_status`/`consent_date`/`data_retention_until`; comms gated on `opted_in`.

**Testing**:
- `Integration: confirm reservation → SMS job enqueued (Twilio mocked)`.
- `Integration: guest opted_out → no marketing comms sent`.
- `Integration: OpenTable webhook → reservation created with source='opentable'`.

#### 9.3 — Loyalty & CRM

**What**: Points-based loyalty with earn/redeem tied to orders.

**Design**:
- Migrate `loyalty_program`, `loyalty_account`, `loyalty_transaction`.
- On order close: if guest enrolled, earn `floor(net_sales × points_per_dollar)` → `loyalty_transaction`
  (`earn`) + balance update. Redeem at POS: `POST /loyalty/redeem` validates `redemption_threshold`.
- Tier recalculated from `lifetime_points`.

**Testing**:
- `Integration: close $50 order, 1 pt/$ → guest balance +50, transaction recorded`.
- `Integration: redeem below threshold → 422`.

#### 9.4 — Food-safety logs (HACCP / ISO 22000)

**What**: Digital checklists and temperature logs with audit trails.

**Design**:
- Migrate `food_safety_checklist`, `food_safety_log`, `temperature_reading`.
- `POST /temperature-readings` auto-sets `is_in_range` vs thresholds; out-of-range requires
  `corrective_action`. Checklist completion stored as `responses_json`. Exportable audit trail (ISO 22000).

**Testing**:
- `Integration: reading above max threshold without corrective_action → 422`.
- `Integration: completed checklist → food_safety_log with responses_json`.

#### 9.5 — Daily reconciliation & reporting (REST + GraphQL)

**What**: Nightly daily-summary rollups and an analytics GraphQL layer.

**Design**:
- Migrate `daily_summary`. Nightly BullMQ job (per location, per business date, tz-aware) aggregates orders,
  payments, discounts, refunds, labour cost, food cost → upserts `daily_summary` (data-model decision #10).
- GraphQL read schema (standards.md) for flexible multi-outlet reporting: menu profitability (sales mix ×
  margin from snapshotted cost), server performance, table turnover, labour %, food-cost %.
- Back-office dashboards (Next.js server components) consume these.

**Testing**:
- `Integration: run nightly job over fixture day → daily_summary matches hand-computed totals`.
- `Integration (GraphQL): query menu-profitability returns items ranked by margin`.
- `Unit: business-date bucketing respects location timezone (orders near midnight)`.

---

## Phase 10: AI-Native Layer & MCP Server

### Purpose
Deliver the differentiator: an AI layer (Python forecasting service + LLM-powered menu engineering) and an
MCP server exposing operations as tools for AI agents. This is deliberately last — it consumes the data the
prior phases produce (orders, inventory ledger, reservations, labour) and adds intelligence on top without
modifying the core.

### Tasks

#### 10.1 — Demand forecasting service (Python)

**What**: Forecast item/ingredient demand from history + signals to drive inventory and labour.

**Design**:
- `apps/ai-service` (FastAPI). `POST /forecast/demand` `{ locationId, horizonDays, signals?:{reservations,
  weather, events} }` → per-item/ingredient predicted quantities with confidence intervals.
- Model: per-series time-series (Prophet/statsmodels) trained on `inventory_transaction` 'used' + order
  history; regressors for day-of-week, reservations (Phase 9), weather, local events. Target the 95%+
  accuracy benchmark cited in features.md.
- Node `ai` module proxies and caches results; forecasts feed Phase 7 par-level reorder suggestions and
  Phase 9 labour staffing recommendations (covers-by-shift).

**Testing**:
- `pytest: synthetic seasonal series → forecast MAPE within tolerance`.
- `Integration: insufficient history (<N days) → returns low-confidence flag, not an error`.

#### 10.2 — Menu engineering & dynamic pricing (LLM + analytics)

**What**: Surface high-margin underperformers and recommend pricing/menu changes.

**Design**:
- Compute the classic menu-engineering matrix (popularity × margin → star/plough-horse/puzzle/dog) from
  `daily_summary` + line-item sales and snapshotted `cost_price`.
- LLM (Vercel AI SDK, structured output) turns the matrix + trends into ranked, explained recommendations.
- `GET /ai/menu-engineering?locationId=&period=` → `{ items:[{classification, margin, popularity,
  recommendation, rationale}] }`. Dynamic pricing suggestions by daypart/demand (advisory only in v1).
- Prompt template (structured): system = "You are a restaurant menu-engineering analyst…"; user = matrix
  JSON + period; response schema validated with zod.

**Testing**:
- `Unit: classification matrix assigns star/dog correctly for known inputs`.
- `Integration (LLM mocked): returns schema-valid recommendations for each item`.

#### 10.3 — MCP server

**What**: Expose platform operations as MCP tools for AI agents (standards.md differentiator).

**Design**:
- `apps/mcp-server` using `@modelcontextprotocol/sdk`. Tools (auth via scoped service token, RLS-scoped):
  - `get_inventory_levels(locationId)`, `get_sales_summary(locationId, period)`,
    `forecast_demand(locationId, horizonDays)`, `suggest_schedule(locationId, weekStart)`,
    `analyze_menu(locationId, period)`, `get_low_stock(locationId)`.
- Each tool calls the core API with a scoped token; read-only in v1 (no destructive writes via MCP).

**Testing**:
- `Integration: MCP get_inventory_levels returns current stock for the location`.
- `Integration: tool call with token lacking scope → denied`.
- `Unit: tool schemas are valid MCP tool definitions`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (monorepo, DB, tenancy/RLS, auth, audit)   ─── required by everything
    │
Phase 2: Menu Catalogue & Tax            ─── requires 1
    │
Phase 3: Floor Plan & Tables (WS)        ─── requires 1   ┐ (3 can parallel 2)
    │                                                      │
Phase 4: Orders & Payments (POS core)    ─── requires 2,3 │  ← the heart, ships early
    │
Phase 5: POS/KDS/Floor-plan PWA + sync   ─── requires 4   ┐
Phase 6: KDS routing & sequencing        ─── requires 4   ┘ (5 and 6 parallel; 5.3 needs 6)
    │
Phase 7: Inventory, recipes, suppliers   ─── requires 2,4
    │
Phase 8: Multi-location, webhooks, delivery ─ requires 4,7
    │
Phase 9: Labour / Reservations / Loyalty / Food-safety / Reporting ─ requires 4 (sub-tasks parallel)
    │
Phase 10: AI layer & MCP                  ─── requires 7,9
```

**Parallelism opportunities**
- Phase 2 and Phase 3 can be developed concurrently after Phase 1.
- Phase 5 and Phase 6 can be developed concurrently after Phase 4 (5.3 KDS UI depends on Phase 6).
- Within Phase 9, tasks 9.1–9.5 are largely independent and can be split across developers.
- The Python `ai-service` (10.1) can be scaffolded any time after Phase 7 data exists.

---

## Definition of Done (per phase)

Every phase is complete only when:

1. All tasks implemented and merged.
2. All unit and integration tests pass (`pnpm turbo test test:int`; `pytest` for ai-service).
3. ESLint + Prettier pass with zero warnings; `tsc --noEmit` strict passes (ruff + mypy for Python).
4. Testcontainers integration suite passes against real Postgres 16 + Redis 7.
5. New tables added via additive Prisma migrations that apply and roll back cleanly; RLS policies present
   on all new tenant-scoped tables.
6. The phase's feature works end-to-end (Playwright E2E for any user-facing flow).
7. New config / environment variables documented and validated by the zod env schema.
8. New REST endpoints appear in the auto-generated OpenAPI 3.1 spec; new events registered in
   `packages/events`.
9. New mutating endpoints emit CloudEvents audit records.
10. `docker compose up` builds and runs the affected services successfully.
11. No raw PAN / card data stored anywhere (PCI DSS); no PII handled without consent fields (GDPR/CCPA).
```
