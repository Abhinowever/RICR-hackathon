# Architecture: SocietyOS

Version 0.2 · Last updated 2026-10-25 · Companion to `prd.md`

This document is the technical source of truth. If code and this file disagree, either fix the code or propose a change here (and log it in `memory.md`). Do not silently diverge.

---

## 1. Principles

1. **Modular monolith.** One deployable app, strict internal module boundaries. No microservices.
2. **Server decides.** Every authorisation check runs on the server, inside the service layer. UI hiding is cosmetic.
3. **Tenant-safe by construction.** Every table has `society_id`; every query runs inside a tenant-scoped transaction with Postgres row-level security as a second lock.
4. **Events over overwrites.** State changes append events (complaint events, ledger entries, audit logs). History is never lost.
5. **Constraints in the database.** Clashes, uniqueness, money invariants, and immutability are enforced by Postgres, not only by application code.
6. **Idempotent by default.** Webhooks, jobs, imports, and invoice generation are safe to run twice.
7. **Boring technology, thin slices.** Prefer well-known libraries; ship vertical slices (schema, service, policy, UI, tests) one feature at a time.

## 2. System context

```
 Visitor / Resident / Committee / Admin  (phone and desktop browsers)
                    |
                    v
        +-----------------------------+
        |  Next.js app (apps/web)     |   Server Components, Server Actions,
        |  public site + app + console|   Route Handlers (webhooks, uploads)
        +-----------------------------+
                    |
                    v
        +-----------------------------+
        |  packages/core (services)   |   one folder per domain module
        |  packages/policy (authz)    |   can(actor, action, resource)
        +-----------------------------+
           |            |          |
           v            v          v
   PostgreSQL +     Object       Job queue
   PostGIS (RLS)    storage      (pg-boss)
                    (S3 API)
           |
           v
   External: Razorpay, email provider (Resend), SMS provider (later), Sentry
```

## 3. Technology decisions

| Concern | Decision | Notes |
|---|---|---|
| Language | TypeScript, `strict: true` | No `any`; `unknown` plus Zod parsing at boundaries |
| Package manager | pnpm workspaces | No Turborepo until build times demand it |
| Framework | Next.js, App Router, latest stable | Pin exact versions at scaffold time |
| Styling | Tailwind CSS (CSS-first config) with tokens from `design.md` | Headless primitives (Radix UI) restyled; no pre-themed kits |
| Database | PostgreSQL with PostGIS | Extensions: `postgis`, `btree_gist`, `pg_trgm`, `citext`, `pgcrypto` |
| ORM and migrations | Drizzle ORM and drizzle-kit | SQL migrations committed; custom SQL for RLS, triggers, constraints |
| Validation | Zod | One schema per input, shared by form and service |
| Auth | Better Auth (email OTP now, phone OTP later) | Sessions in the database; see ADR-003 |
| Authorisation | Custom `policy` package | Typed actions, role matrix as data; see section 7 |
| Map | MapLibre GL JS with GeoJSON from PostGIS; Terra Draw for editing (P1) | No paid tiles required |
| Files | S3-compatible storage (Cloudflare R2 or Supabase Storage), presigned URLs | |
| Jobs | pg-boss (Postgres-backed) | No Redis |
| Email | Resend via an adapter interface | SMS provider adapter later |
| Payments | Razorpay | Orders, checkout, webhooks |
| Search | Postgres full-text (`tsvector`) and `pg_trgm` | Meilisearch only if needed |
| PDF | `@react-pdf/renderer` or server HTML-to-PDF for receipts | Decide at BIL-007 |
| Testing | Vitest (unit, integration against real Postgres), Playwright (e2e), axe (a11y) | |
| Lint and format | ESLint, Prettier, `tsc --noEmit` | |
| Errors and logs | Sentry, `pino` structured logs with request id | |
| Hosting | Vercel and Neon, or one VPS with Docker Compose | Decision pending (Q-002) |

## 4. Repository layout

```
/
├─ CLAUDE.md                     # agent entry point, imports rules/memory/tracker
├─ docs/
│  ├─ prd.md  architecture.md  design.md  rules.md  memory.md  tracker.md
│  └─ specs/<module>.md          # one short spec per module, written before coding
├─ apps/
│  └─ web/                       # Next.js app
│     ├─ src/app/
│     │  ├─ (public)/            # home, about, notices, events, contact, login
│     │  ├─ (app)/               # resident area
│     │  ├─ (console)/           # committee area
│     │  ├─ (admin)/             # admin area
│     │  └─ api/                 # webhooks, upload signing, health
│     ├─ src/components/         # feature components (not primitives)
│     ├─ src/lib/                # actor resolution, action wrappers, utils
│     └─ e2e/                    # Playwright tests
├─ packages/
│  ├─ db/                        # Drizzle schema, migrations, seeds, withTenant()
│  ├─ core/                      # domain services (one folder per module)
│  │  └─ src/{identity,society,complaints,works,booking,notices,billing,map,documents,notifications,search,audit}/
│  ├─ policy/                    # roles, actions, can(), scope filters, matrix tests
│  ├─ ui/                        # design-system primitives and tokens
│  └─ config/                    # shared tsconfig, eslint, env schema
├─ docker-compose.yml            # postgres+postgis, mailpit, minio (local)
└─ .github/workflows/ci.yml
```

Module folder shape inside `packages/core/src/<module>/`:

```
schema.ts      # Zod input/output schemas (shared with forms)
service.ts     # exported functions: (actor, input) => Result<T>
queries.ts     # read models and list queries (scoped by policy)
events.ts      # domain events emitted (for notifications, audit)
errors.ts      # typed errors for the module
index.ts       # the only public entry point of the module
*.test.ts
```

Cross-module rule: a module may import another module's `index.ts` only. Never reach into another module's tables directly; call its service or query.

## 5. Request flow and layering

```
UI (Server Component / Client Component)
  -> Server Action or Route Handler           (apps/web)   parse + auth + call
     -> getActor()                                         resolves user, society, roles
     -> core service: fn(actor, input)         (packages/core)
        -> policy.can(actor, action, resource) (packages/policy)
        -> withTenant(societyId, tx => ...)    (packages/db) RLS-scoped transaction
        -> emit domain events / audit entries
     <- Result<T, AppError>
  <- UI maps Result to state, toast, or redirect
```

Rules of the layers:
- UI and actions contain **no business rules**. They parse input, call a service, and render the result.
- Services contain business rules, call `policy.can` **first**, and are the only code that writes to the database.
- Reads for pages also go through `queries.ts` functions that take the actor and apply scope filters.

### Result and errors

```ts
type Result<T, E extends AppError = AppError> =
  | { ok: true; data: T }
  | { ok: false; error: E };

type AppError = {
  code:
    | 'UNAUTHENTICATED' | 'FORBIDDEN' | 'NOT_FOUND' | 'VALIDATION'
    | 'CONFLICT' | 'INVALID_TRANSITION' | 'RATE_LIMITED'
    | 'PAYMENT_FAILED' | 'INTERNAL';
  message: string;            // safe for end users
  fields?: Record<string, string>; // validation errors by field
  meta?: Record<string, unknown>;
};
```

Services never throw for expected failures; they return `{ ok: false }`. Unexpected errors are thrown, logged to Sentry with the request id, and surfaced as `INTERNAL`. Never leak existence: if the actor may not know a record exists, return `NOT_FOUND` instead of `FORBIDDEN`.

### Pagination and lists
Cursor pagination (`createdAt`, `id`) for all lists; default page size 25, max 100. Filters are validated Zod objects mirrored in the URL query string.

## 6. Multi-tenancy and row-level security

- Every tenant table has `society_id uuid not null` and an index starting with it.
- The app connects as a non-superuser role `app_user` (cannot bypass RLS). Migrations run as `app_owner`.
- Every request runs inside `withTenant`:

```ts
export async function withTenant<T>(societyId: string, fn: (tx: Tx) => Promise<T>): Promise<T> {
  return db.transaction(async (tx) => {
    await tx.execute(sql`select set_config('app.society_id', ${societyId}, true)`);
    return fn(tx);
  });
}
```

- RLS policy template (applied by a migration helper to every tenant table):

```sql
alter table complaints enable row level security;
alter table complaints force row level security;
create policy tenant_isolation on complaints
  using (society_id = current_setting('app.society_id', true)::uuid)
  with check (society_id = current_setting('app.society_id', true)::uuid);
```

- Tables that are not tenant-scoped: `users` and Better Auth tables, `societies` (read restricted by membership), and platform config.
- A test (`packages/db/rls.test.ts`) creates two societies and proves a query in society A can never read or write society B for every tenant table (iterates the schema, so new tables are covered automatically).

## 7. Authentication and authorisation

### Actor

```ts
type Role = 'resident' | 'committee' | 'office_bearer' | 'admin' | 'staff';

type Actor = {
  userId: string;
  societyId: string;
  roles: Role[];                 // active memberships in this society
  departmentIds: string[];       // for committee members
  officeTitle?: 'chairman' | 'secretary' | 'treasurer';
  propertyIds: string[];         // approved occupancies
};
```

`getActor()` (apps/web/src/lib) reads the session, the active society (cookie), and loads memberships and occupancies. No actor means `UNAUTHENTICATED`.

### Policy

All permission rules live in `packages/policy` as **data plus pure functions**, not scattered `if`s.

```ts
type Action =
  | 'complaint.create' | 'complaint.read' | 'complaint.assign' | 'complaint.transition'
  | 'complaint.comment' | 'complaint.internalNote'
  | 'booking.create' | 'booking.decide' | 'resource.manage'
  | 'notice.draft' | 'notice.publish'
  | 'map.public' | 'map.property' | 'map.utilities' | 'map.edit'
  | 'billing.readOwn' | 'billing.readAll' | 'billing.record'
  | 'user.manage' | 'audit.read' | 'society.configure' /* ...extend per module */;

function can(actor: Actor, action: Action, resource?: ResourceRef): boolean;
function scopeFilter(actor: Actor, entity: 'complaint' | 'document' | 'serviceRequest'): SqlFilter;
```

- `can` answers "may this actor perform this action (on this resource)?"
- `scopeFilter` returns the SQL predicate for list queries (self / department / all rules), so lists and counts can never return rows the actor cannot open.
- The role-by-action matrix is a single exported constant, and `matrix.test.ts` iterates **every role x every action** against expected results. Adding an action without matrix entries fails the build.

### Scope rules for complaints (reference implementation)

A complaint is readable by actor A if any of:
1. A is the reporter.
2. visibility = `all` and A is a resident, committee member, office bearer, or admin of the society.
3. visibility = `department` and A is in the complaint's department, or A is office bearer or admin.
4. A is the assignee.
5. A is office bearer or admin (see all).

Internal notes are readable only by committee, office bearer, admin.

### Sessions and security settings
Better Auth sessions stored in Postgres; cookie `httpOnly`, `secure`, `sameSite=lax`. OTP and login endpoints rate limited (per identifier and per IP) using a Postgres-backed limiter table. CSRF: Server Actions with same-site cookies, plus origin checks on route handlers.

## 8. Data model

Conventions: `id uuid primary key default gen_random_uuid()`; `society_id` on all tenant tables; `created_at`, `updated_at timestamptz not null default now()`; money as `bigint` paise (`*_paise`); enums as Postgres enums or `text` with check constraints (be consistent per table); soft delete via `deleted_at` only where listed; append-only tables are marked **(append-only)** and protected by trigger.

### Identity and society

| Table | Key columns |
|---|---|
| `users` | id, name, email (citext, unique), email_verified_at, phone, phone_verified_at, image, created_at (plus Better Auth tables) |
| `societies` | id, name, slug (unique), address, city, state, pincode, timezone, settings jsonb, intro text, logo_key |
| `departments` | id, society_id, key, name; unique (society_id, key) |
| `memberships` | id, society_id, user_id, role, department_id null, office_title null, status (pending/active/suspended); unique (society_id, user_id, role) |
| `zones` | id, society_id, code, name, sort, geom geometry(MultiPolygon,4326) null, layout jsonb null (for SVG fallback); unique (society_id, code) |
| `buildings` | id, society_id, zone_id, name, geom null |
| `properties` | id, society_id, zone_id, building_id null, unit_no, type (house/flat/shop/plot/common), area_sqft null, geom geometry(Geometry,4326) null, layout jsonb null, status; unique (society_id, zone_id, unit_no) |
| `occupancies` | id, society_id, property_id, user_id, kind (owner/tenant/family), starts_on, ends_on null, status (pending/approved/rejected/ended), decided_by null, decided_at null, reason null |
| `amenities` | id, society_id, zone_id null, name, type, description, geom null, resource_id null |

### Infrastructure

| Table | Key columns |
|---|---|
| `assets` | id, society_id, zone_id null, type (valve/pump/transformer/manhole/meter/hydrant/cctv/gate/other), name, geom Point null, installed_on null, status, meta jsonb |
| `utility_lines` | id, society_id, utility (water/drainage/electrical/gas/internet/fire), name, geom LineString, material null, installed_on null, status |

### Complaints

| Table | Key columns |
|---|---|
| `complaint_categories` | id, society_id, key, name, department_id, default_priority, sensitive bool, active |
| `sla_policies` | id, society_id, priority, ack_minutes, resolve_minutes; unique (society_id, priority) |
| `complaint_counters` | society_id, year, last_value (for `CMP-YYYY-NNNNNN`; incremented with `update ... returning` inside the transaction) |
| `complaints` | id, society_id, code (unique per society), reporter_id, category_id, department_id, title, description, priority, visibility (self/department/all), status, property_id null, asset_id null, zone_id null, location geometry(Point,4326) null, assignee_id null, due_at, acknowledged_at null, resolved_at null, closed_at null, reopen_count int default 0, escalated_at null, rating null, search tsvector (generated) |
| `complaint_events` **(append-only)** | id, society_id, complaint_id, actor_id null (null = system), type, from_status null, to_status null, payload jsonb, created_at |
| `complaint_comments` | id, society_id, complaint_id, author_id, body, internal bool, created_at |
| `attachments` | id, society_id, owner_type, owner_id, storage_key, mime, size_bytes, uploaded_by, created_at; index (owner_type, owner_id) |

### Works and requests

| Table | Key columns |
|---|---|
| `vendors` | id, society_id, name, trade, phone, email, status |
| `work_orders` | id, society_id, code, source_type (complaint/project/inspection/manual), source_id null, property_id null, asset_id null, utility_line_id null, vendor_id null, assignee_id null, title, description, status (open/scheduled/in_progress/done/cancelled), scheduled_on null, completed_on null, cost_paise null, notes |
| `projects` | id, society_id, code, title, description, status (planned/approved/in_progress/on_hold/completed/cancelled), budget_paise, vendor_id null, starts_on null, ends_on null, visibility, affected_geom geometry null, created_by |
| `project_updates` | id, society_id, project_id, author_id, body, progress_pct, created_at |
| `service_request_types` | id, society_id, key, name, form_schema jsonb, fee_paise, approver_department_id, active |
| `service_requests` | id, society_id, code, type_id, requester_id, property_id, status (submitted/approved/rejected/in_progress/done/cancelled), form_data jsonb, fee_paise, invoice_id null, decided_by null, decided_at null, reason null |

### Booking

| Table | Key columns |
|---|---|
| `resources` | id, society_id, name, kind, capacity, description, rules jsonb (open_time, close_time, min_minutes, max_minutes, advance_min_hours, advance_max_days, max_active_per_month), fee_paise, deposit_paise, requires_approval bool, approver_department_id null, active |
| `bookings` | id, society_id, resource_id, user_id, property_id, period tstzrange, status (requested/approved/rejected/cancelled/completed/no_show/expired), purpose, guests, fee_paise, invoice_id null, decided_by null, decided_at null, reason null |
| `blackout_dates` | id, society_id, resource_id null (null = all), period tstzrange, reason |

### Communication

| Table | Key columns |
|---|---|
| `notices` | id, society_id, title, body, category, audience (public/residents/committee/zone), zone_id null, pinned, status (draft/published/archived), publish_at, expires_at null, requires_ack, author_id, published_by null |
| `notice_acks` | notice_id, user_id, acked_at; primary key (notice_id, user_id) |
| `events` | id, society_id, title, description, starts_at, ends_at, venue, resource_id null, audience, rsvp_enabled, created_by |
| `event_rsvps` | event_id, user_id, status, guests; primary key (event_id, user_id) |
| `notifications` | id, society_id, user_id, type, title, body, link, read_at null, created_at |
| `notification_preferences` | user_id, society_id, channel, type, enabled; primary key (user_id, society_id, channel, type) |
| `enquiries` | id, society_id, name, email, phone null, message, created_at, handled_at null |

### Finance

| Table | Key columns |
|---|---|
| `charge_heads` | id, society_id, key, name, calc (flat/per_sqft), amount_paise, frequency (monthly/quarterly/yearly/one_time), applies_to_types text[], late_fee jsonb, active |
| `invoices` | id, society_id, number (sequential per society and FY), property_id, billed_user_id null, period_start, period_end, due_date, status (draft/issued/partially_paid/paid/void), total_paise, paid_paise; unique (society_id, property_id, period_start, kind) for generated invoices |
| `invoice_lines` | id, society_id, invoice_id, charge_head_id null, description, amount_paise, source_type null, source_id null |
| `payments` | id, society_id, invoice_id, user_id null, method (razorpay/cash/cheque/bank_transfer), amount_paise, status (created/captured/failed/refunded), gateway_order_id null, gateway_payment_id null, idempotency_key (unique), reference null, received_by null, receipt_no null, created_at |
| `ledger_entries` **(append-only)** | id, society_id, account, debit_paise, credit_paise, ref_type, ref_id, property_id null, memo, reverses_id null, created_at |
| `expenses` (P2) | id, society_id, category, vendor_id null, amount_paise, spent_on, note, attachment_key null |

### Records and system

| Table | Key columns |
|---|---|
| `documents`, `document_versions` (P2) | folder, title, visibility, current_version_id; storage_key, version, uploaded_by |
| `audit_logs` **(append-only)** | id, society_id, actor_id null, action, entity_type, entity_id, diff jsonb, ip, user_agent, created_at |
| `rate_limits` | key, window_start, count |
| `jobs_*` | managed by pg-boss in its own schema |

### Service history is a view, not a table
`property_history_v` unions complaints, work orders, project links, and service requests for a property into (occurred_at, kind, ref_id, title, status, actor, cost_paise). Cost columns are removed by the service layer unless the actor has committee rights. Do not create a separate history table to keep in sync.

## 9. Database constraints and triggers (must exist)

```sql
-- Booking clash prevention (needs btree_gist)
alter table bookings add constraint bookings_no_overlap
  exclude using gist (resource_id with =, period with &&)
  where (status in ('requested','approved'));

-- Append-only tables
create function forbid_mutation() returns trigger language plpgsql as $$
begin raise exception '% is append-only', tg_table_name; end $$;
create trigger complaint_events_immutable before update or delete on complaint_events
  for each row execute function forbid_mutation();
-- repeat for ledger_entries and audit_logs

-- Money sanity
alter table invoices add constraint invoices_paid_le_total check (paid_paise between 0 and total_paise);
alter table payments add constraint payments_positive check (amount_paise > 0);
alter table ledger_entries add constraint ledger_one_side check (
  (debit_paise > 0 and credit_paise = 0) or (credit_paise > 0 and debit_paise = 0));

-- One active primary approval path
create unique index occupancies_one_active_owner on occupancies (property_id)
  where kind = 'owner' and status = 'approved';

-- Search
create index complaints_search_idx on complaints using gin (search);
create index complaints_title_trgm on complaints using gin (title gin_trgm_ops);
```

## 10. State machines

### Complaint (`status`)

States: `submitted`, `acknowledged`, `assigned`, `in_progress`, `on_hold`, `resolved`, `closed`, `rejected`, `reopened`. (`draft` is reserved but unused in MVP; escalation is a flag `escalated_at`, not a status.)

| From | To | Who | Requirements |
|---|---|---|---|
| (new) | submitted | reporter | Created with `due_at`, routed to department; system immediately moves to acknowledged |
| submitted | acknowledged | system or department member | Sets `acknowledged_at` |
| acknowledged | assigned | department member, office bearer, admin | assignee required |
| assigned | in_progress | assignee, department member | |
| in_progress | on_hold | assignee, department member | reason required |
| on_hold | in_progress | assignee, department member | |
| submitted, acknowledged, assigned, in_progress, on_hold | rejected | office bearer, admin, department member | reason required, reporter notified |
| in_progress, assigned | resolved | assignee, department member | resolution note required; sets `resolved_at` |
| resolved | closed | reporter, or system after 5 days | sets `closed_at` |
| resolved | reopened | reporter within 7 days | reason required; increments `reopen_count` |
| reopened | assigned, in_progress | department member | |

Implementation: a single `transitions` table in code (`from`, `to`, `allowedActors`, `requires`) plus one `transition(actor, complaintId, to, input)` function. Every transition runs in one transaction: update status, append event, enqueue notifications, write audit if actor is not the reporter and the change is an override.

### Booking
`requested -> approved | rejected | cancelled | expired`; `approved -> cancelled | completed | no_show`. Auto-approved resources go straight to `approved`.

### Invoice
`draft -> issued -> partially_paid -> paid`; `issued | partially_paid -> void` (office bearer only; creates reversing ledger entries).

### Project
`planned -> approved -> in_progress <-> on_hold -> completed`; `planned | approved | in_progress | on_hold -> cancelled`.

### Occupancy
`pending -> approved | rejected`; `approved -> ended` (manual or by `ends_on`).

## 11. Map architecture

- **Storage:** PostGIS geometry in SRID 4326. For the SVG/image fallback, entities store `layout` jsonb (`{x, y, w, h}` or an SVG path) in a local coordinate system, and the map runs MapLibre in a non-geographic mode, or a lightweight SVG viewer behind the same `MapView` interface.
- **API:** `GET /api/map/features?layer=zones|properties|amenities|utilities|works|complaints&bbox=...` returns GeoJSON **already filtered by policy** (public layer for visitors, own property for residents, everything for committee). Properties payload for residents contains only their own property's attributes; other properties come back as anonymous footprints without personal data.
- **Client:** `MapView` component wraps MapLibre; layers are declared in a config array (id, source, paint, minZoom, requiredAction). The same config drives the legend and the layer toggles.
- **Interaction contract:** every click emits `{ type: 'zone' | 'property' | 'asset' | 'line' | 'project', id }`; the side panel loads details through a normal service call (permission-checked again server-side).
- **Accessibility:** a "List view" toggle renders the same data as a table with keyboard navigation (MAP-012).
- **Editing (P1):** Terra Draw for drawing; writes go through `map.updateGeometry` service, versioned in `audit_logs` (diff holds old and new GeoJSON).
- **Utility line style tokens:** see `design.md` section 10.
- **Performance:** simplify geometry on write for display copies if needed; cache feature responses by (society, layer, role-group) for 60 s.

## 12. Files and uploads

1. Client asks `POST /api/uploads/sign` with `{ ownerType, mime, size }`.
2. Server validates actor, mime allow-list, size limits, and quota; returns a presigned PUT URL and a pending `attachments` row.
3. Client uploads directly to storage.
4. Client calls the service (for example `complaints.create`) with attachment ids; the service verifies each belongs to the actor and is uploaded, then links it.
5. Reads use short-lived signed GET URLs generated after a policy check on the parent record.
6. Nightly job deletes pending attachments older than 24 h and any orphaned objects.

Client compresses images (max 1600 px long edge) before upload. Never trust client mime; sniff the first bytes on the server job or via storage metadata.

## 13. Background jobs (pg-boss)

| Job | Schedule | Purpose |
|---|---|---|
| `complaint.slaCheck` | every 15 min | Set `escalated_at` on overdue complaints; notify secretary |
| `complaint.autoClose` | hourly | Close `resolved` complaints silent for 5 days |
| `booking.expireRequests` | hourly | Expire `requested` bookings older than 24 h undecided |
| `booking.complete` | hourly | Mark finished approved bookings as completed |
| `notice.publishScheduled` | every 5 min | Publish due notices; archive expired |
| `occupancy.expire` | daily 01:00 IST | End occupancies past `ends_on` |
| `invoice.generateMonthly` | 1st of month 02:00 IST | Create invoices per charge head (idempotent) |
| `invoice.lateFee` | daily 03:00 IST | Apply late fee once per overdue period |
| `invoice.reminders` | daily 09:00 IST | 3 days before due, on due date, 3 days after |
| `payment.reconcile` | daily 04:00 IST | Compare gateway vs ledger |
| `notification.dispatch` | queue worker | Send email/SMS for queued notifications with retry and backoff |
| `storage.cleanup` | daily | Remove orphan uploads |

Rules: each job loads its own tenant context per society; jobs are idempotent; failures retry with exponential backoff (max 5), then go to a dead-letter state visible to admin.

## 14. Notifications

Services emit domain events (`complaint.statusChanged`, `booking.decided`, `notice.published`, `invoice.issued`, `payment.captured`). A subscriber maps events to notifications by type, writes `notifications` rows (in-app) in the same transaction, and enqueues email via `notification.dispatch` according to preferences. Templates live in `packages/core/src/notifications/templates/` and use message keys, not hard-coded strings.

## 15. Payments (Razorpay)

1. Resident clicks Pay on an invoice. Server creates a `payments` row (`created`, unique `idempotency_key`) and a Razorpay order for the outstanding amount (paise).
2. Client opens Razorpay Checkout with the order id.
3. On client success, the client only calls `payments.confirm(orderId)`, which **does not mark paid**; it triggers a status fetch.
4. `POST /api/webhooks/razorpay` verifies the HMAC signature on the raw body, deduplicates by event id, and for `payment.captured` marks the payment captured, updates `invoices.paid_paise` and status, posts ledger entries, generates the receipt, and emits `payment.captured`, all in one transaction.
5. `payment.reconcile` job detects anything captured at the gateway but missing locally and alerts.
6. Never store or log card data. Never trust client-reported amounts.

Webhooks respond 200 quickly; heavy work is queued. Unknown or out-of-order events are logged and ignored safely.

## 16. Search

- `complaints.search` is a generated `tsvector` over title, description, and code, plus trigram on title.
- Global search (`search.query(actor, q)`) calls each module's `searchFor(actor, q)` which applies its own `scopeFilter`, merges and ranks results (exact code match first). Residents directory search is committee-only.
- Query length 2-80 chars, debounced on the client, rate limited.

## 17. Observability and operations

- `GET /api/health` (db ping, job queue ping) for uptime checks.
- Every request gets a request id; logs are JSON with `requestId`, `societyId`, `userId`, `action`, `durationMs`.
- Sentry for exceptions in server and client, with PII scrubbing (no OTP, no phone, no email in breadcrumbs).
- Backups: provider-managed daily plus point-in-time recovery if available; monthly restore drill recorded in `memory.md`.

## 18. Testing strategy

| Level | Tool | What |
|---|---|---|
| Unit | Vitest | Pure functions: transitions, policy, SLA calculation, money, rules |
| Integration | Vitest against real Postgres (Docker, template database per run) | Services with real constraints, RLS, triggers, idempotency |
| Policy matrix | Vitest | Every role x every action; every scope filter |
| RLS isolation | Vitest | Two societies, every tenant table |
| End-to-end | Playwright | One happy path per module, each in the correct role; one forbidden-path test per module |
| Accessibility | axe in Playwright | Key pages in light and dark |
| Load smoke | k6 or autocannon | Complaint list with 5,000 rows; booking clash race |

Test data: `packages/db/seed` creates the demo society (3 zones, 40 properties, one user per role, sample complaints in every status, resources, notices). Tests build on this seed, never on production data.

## 19. Environments and configuration

All config is validated at boot by a Zod schema in `packages/config` (app fails fast if missing).

| Variable | Purpose |
|---|---|
| `DATABASE_URL`, `DATABASE_URL_OWNER` | App role and migration role connection strings |
| `BETTER_AUTH_SECRET`, `APP_URL` | Auth secret and canonical URL |
| `STORAGE_ENDPOINT`, `STORAGE_BUCKET`, `STORAGE_ACCESS_KEY`, `STORAGE_SECRET_KEY` | Object storage |
| `EMAIL_PROVIDER_KEY`, `EMAIL_FROM` | Email sending (console/mailpit in dev) |
| `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, `RAZORPAY_WEBHOOK_SECRET` | Payments (test keys outside production) |
| `SENTRY_DSN` | Error tracking |
| `TURNSTILE_SECRET`, `NEXT_PUBLIC_TURNSTILE_SITE_KEY` | Bot protection on public forms |

Environments: `local` (Docker Compose: Postgres+PostGIS, Mailpit, MinIO), `preview` (per pull request, seeded DB branch), `production`. Secrets only in the host's secret store; `.env.example` lists names, never values.

## 20. Security checklist (applies to every module)

- Input validated with Zod at the boundary; output shaped to the actor.
- Policy check before any read or write; `NOT_FOUND` for hidden records.
- Rich text sanitised (allow-list) on write and escaped on render.
- Uploads: allow-listed types, size limits, signed URLs, no public buckets.
- Rate limits on OTP, login, enquiry, uploads, complaint creation.
- Secure headers (CSP, HSTS, frame-ancestors none, referrer policy) set in middleware.
- No secrets in client bundles; only `NEXT_PUBLIC_*` values are public.
- Audit entries for sensitive actions; logs scrub personal data.
- `pnpm audit` and a dependency review in CI; Dependabot or Renovate enabled.

## 21. Deployment

- CI (GitHub Actions): install, lint, typecheck, unit and integration tests (Postgres service container), build, Playwright smoke on preview.
- Migrations: run by CI/CD as `app_owner` before the new version receives traffic; migrations must be backward compatible for one release (expand, then contract).
- Rollback: redeploy previous build; never roll back a migration that dropped data; prefer forward fixes.
- Production checklist lives in `docs/runbook.md` (created in T-611).

## 22. Performance budget

- Public pages: server-rendered, under 150 KB JS on first load, images via `next/image`.
- Map JS loaded only on map routes (dynamic import).
- Every list query has an index matching its default filter and sort; `EXPLAIN` reviewed for any query that touches more than 1,000 rows.
- No N+1 queries: list queries join or batch.

## 23. Architecture decision records (summary)

| ADR | Decision | Reason |
|---|---|---|
| 001 | Modular monolith | Fastest to build and operate for a small team plus agents |
| 002 | PostgreSQL + PostGIS + RLS | One store for relational, geometry, search, constraints, tenancy |
| 003 | Better Auth | Active project, supports email and phone OTP, database sessions, works with Next.js and Drizzle (re-verify plugin support at T-006) |
| 004 | Custom policy package (no CASL) | Needs SQL scope filters, not just boolean checks; matrix as data is easy for agents to extend and test |
| 005 | pg-boss for jobs | No extra infrastructure |
| 006 | Escalation as a flag, not a status | Keeps the state machine small; avoids status explosion |
| 007 | Money as integer paise | Avoids floating-point errors |
| 008 | Service history is a view | No sync problems |
| 009 | Razorpay webhooks as the only source of payment truth | Client callbacks can be forged or lost |

Add new ADRs in `memory.md` (Decisions) and summarise here when they change architecture.
