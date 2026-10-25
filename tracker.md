# Tracker: SocietyOS

Single source of truth for **what is done, in progress, and next**. The agent updates this file at the end of every session (see `rules.md` section 1). Humans may reorder priorities.

Last updated: 2026-10-25

## Status legend

`TODO` not started · `DOING` in progress (max 1-2 at a time) · `REVIEW` implemented, waiting for human review/merge · `DONE` merged and meets the Definition of Done · `BLOCKED` waiting on something (say what in Notes) · `CUT` removed from scope (say why)

Task IDs: `T-0xx` foundations, `T-1xx` week 1, `T-2xx` week 2, and so on. Requirement IDs refer to `prd.md`. Never renumber a task; add new ones at the end of a phase.

---

## Current focus

| | |
|---|---|
| Phase | Phase 0: Foundations |
| Active task | T-001 (not started) |
| Next up | T-002, T-003, T-005 |
| Blockers | None |
| Human decisions needed | Q-002 (hosting), Q-003 (layout plan). See `memory.md` section 6 |

## Progress summary

| Phase | Tasks | Done | Target |
|---|---|---|---|
| 0 Foundations | 11 | 0 | Days 1-3 |
| 1 Identity and society setup | 8 | 0 | Week 1 |
| 2 Complaints | 12 | 0 | Week 2 |
| 3 Notices, public site, dashboards | 7 | 0 | Week 3 |
| 4 Booking, history, map v1 | 9 | 0 | Week 4 |
| 5 Billing and payments | 8 | 0 | Week 5 |
| 6 Utilities, projects, search, hardening | 12 | 0 | Week 6 |

Milestone demos: end of week 0 (themed shell and login), week 1 (society setup and resident claim), week 2 (full complaint journey), week 3 (public site and dashboards), week 4 (booking and map; MVP sign-off), week 5 (sandbox payment), week 6 (release candidate and pilot).

---

## Phase 0: Foundations (days 1-3)

| ID | Task and done-when | Requirements | Depends | Status |
|---|---|---|---|---|
| T-001 | **Repo scaffold.** pnpm workspace with `apps/web`, `packages/{db,core,policy,ui,config}`; TypeScript strict; ESLint, Prettier; Vitest; Playwright; husky and lint-staged; scripts from `rules.md` section 12. *Done when `pnpm check` and `pnpm build` pass on an empty app* | n/a | none | TODO |
| T-002 | **Local infrastructure.** `docker-compose.yml` with Postgres+PostGIS, Mailpit, MinIO; extensions (`postgis`, `btree_gist`, `pg_trgm`, `citext`, `pgcrypto`); `app_owner` and `app_user` roles; env schema with Zod and `.env.example`. *Done when `pnpm db:up && pnpm db:migrate` works from a clean machine* | n/a | T-001 | TODO |
| T-003 | **Design tokens and primitives.** `tokens.css` (light/dark), fonts, theme provider with cookie, Button, Input, Select, Textarea, Checkbox, Card, Chip, Table shell, Dialog, Toast. *Done when a style page renders every primitive in both themes and passes axe* | design.md 2-7 | T-001 | TODO |
| T-004 | **App shells.** Public, resident, console, admin layouts; top bar, rails, mobile bottom tabs; role switcher stub; error and not-found pages. *Done when each layout renders at 360 and 1440 px with placeholder pages* | design.md 5 | T-003 | TODO |
| T-005 | **Schema v1 and tenancy.** Identity and society structure tables (`societies`, `departments`, `memberships`, `zones`, `buildings`, `properties`, `occupancies`, `amenities`); `withTenant`; RLS helper; RLS isolation test that iterates all tenant tables. *Done when isolation test passes* | SOC-002, SOC-003, AUTH-003 | T-002 | TODO |
| T-006 | **Authentication.** Better Auth with email OTP (Mailpit in dev), rate limit, lockout, sessions, sign out everywhere; `getActor()`. Re-verify plugin support first (memory D-004). *Done when a user can log in with an emailed code and the actor resolves* | AUTH-001, AUTH-002, AUTH-003 | T-002, T-005 | TODO |
| T-007 | **Policy package.** Roles, actions, matrix as data, `can()`, `scopeFilter()` skeleton, `matrix.test.ts` iterating role x action. *Done when adding an action without matrix entries fails the build* | architecture 7 | T-005 | TODO |
| T-008 | **Audit log.** `audit_logs` table with append-only trigger; `audit.record()` service; same-transaction helper. *Done when update/delete on the table raises an error and a test proves it* | AUD-001 | T-005 | TODO |
| T-009 | **Seed script.** Deterministic demo society: 3 zones (D1-D3), 40 properties, one user per role, departments, categories, SLA policies, resources, sample notices. *Done when `pnpm db:seed` is idempotent and all roles can log in locally* | n/a | T-005, T-006 | TODO |
| T-010 | **CI.** GitHub Actions: install, lint, typecheck, test with Postgres service container, build; preview deploy; Dependabot/Renovate. *Done when a PR runs all checks green* | n/a | T-001, T-002 | TODO |
| T-011 | **Docs in repo.** Place `docs/*.md` and root `CLAUDE.md`; create `docs/specs/` and `docs/runbook.md` stubs. *Done when the agent starts a session and reports the current focus from the tracker* | n/a | T-001 | TODO |

## Phase 1: Identity and society setup (week 1)

| ID | Task and done-when | Requirements | Depends | Status |
|---|---|---|---|---|
| T-101 | **Society settings (admin).** Profile, timezone, logo, intro text, committee contacts. *Done when saved values appear on the public home placeholder* | SOC-001 | T-004, T-007 | TODO |
| T-102 | **Zones and properties.** CRUD plus CSV import with dry-run, row errors, idempotency; delete protection. *Done when the 40-property CSV imports twice with identical result* | SOC-002, SOC-003 | T-101 | TODO |
| T-103 | **Departments and memberships.** Assign roles, departments, office titles; user directory for admin and office bearers. *Done when admin can make a user committee in Water and Electrical* | SOC-004, AUTH-003 | T-007 | TODO |
| T-104 | **Resident onboarding.** First-login flow: pick zone, property, relation; creates `pending` occupancy; consent capture; "waiting for approval" screen. *Done when a new user is blocked from resident features until approved* | AUTH-005 | T-006, T-102 | TODO |
| T-105 | **Approval queue.** Office bearer/admin approves or rejects with reason; notification to requester; audit entry; one active owner constraint. *Done when full claim-to-approval journey passes e2e* | AUTH-006 | T-104, T-008 | TODO |
| T-106 | **Profile and household.** Edit profile, add household members with limited rights. *Done when household member can log in and see only own items* | AUTH-007 | T-105 | TODO |
| T-107 | **Complaint categories and SLA setup.** Admin screens for categories (department, priority, sensitive) and SLA policies. *Done when defaults from PRD section 8 are seeded and editable* | SOC-005, SOC-006 | T-103 | TODO |
| T-108 | **Role switcher and context.** Active society and role context, multi-role users. *Done when a committee-resident user switches views and policy respects the active context* | AUTH-003 | T-103 | TODO |

## Phase 2: Complaints (week 2)

| ID | Task and done-when | Requirements | Depends | Status |
|---|---|---|---|---|
| T-201 | **Complaint schema.** `complaints`, `complaint_events` (append-only), `complaint_comments`, `attachments`, `complaint_counters`; indexes; RLS; search column. *Done when migration, RLS test, and trigger test pass* | CMP-002, CMP-005 | T-107 | TODO |
| T-202 | **Upload service.** Presigned upload, mime and size validation, pending attachments, cleanup job. *Done when a 5-image upload works against MinIO with invalid files rejected* | CMP-008 | T-201 | TODO |
| T-203 | **Create complaint service.** Validation, code generation, routing to department, `due_at` from SLA, daily limit, event row, audit where needed. *Done when integration tests cover rules and policy* | CMP-001, CMP-002, CMP-003, CMP-015 | T-201, T-007 | TODO |
| T-204 | **Transition engine.** Transition table, `transition()` with required reasons, events in one transaction, `INVALID_TRANSITION`. *Done when every row of architecture 10 has a passing allowed and forbidden test* | CMP-004, CMP-005 | T-203 | TODO |
| T-205 | **Visibility and scope queries.** `scopeFilter` for complaints; list and detail queries; role-matrix tests for self/department/all. *Done when no role can read a complaint outside its scope in tests* | CMP-006 | T-203, T-007 | TODO |
| T-206 | **Resident complaint UI.** List, raise (mobile first, photos, location dropdowns), detail with timeline and comments. *Done when a resident raises and tracks a complaint on a 360 px screen in under 60 s* | CMP-001, CMP-007 | T-202, T-205 | TODO |
| T-207 | **Committee console.** Board and table with filters, saved in URL; assign; internal notes; workload counts. *Done when committee processes a complaint from submitted to resolved* | CMP-009, CMP-010 | T-204, T-205 | TODO |
| T-208 | **Duplicate hint.** Similar open complaints in same zone and category (7 days) shown while raising. *Done when hint appears and can be dismissed* | CMP-012 | T-206 | TODO |
| T-209 | **Resolution loop.** Resolve with note, resident confirm or reopen (7 days), auto-close job (5 days). *Done when time-travel tests prove the windows* | CMP-011 | T-204 | TODO |
| T-210 | **Notifications v1.** Domain events to in-app notifications and email adapter; templates via message keys; dispatch job with retry. *Done when status changes appear in the bell and Mailpit* | NTF-001, NTF-002 | T-204 | TODO |
| T-211 | **SLA monitor.** `slaCheck` job sets `escalated_at`, notifies secretary; overdue chip in lists. *Done when overdue complaints are flagged within 15 min in tests* | CMP-013 | T-204, T-210 | TODO |
| T-212 | **Complaint e2e and policy suite.** Playwright journey per role; forbidden path tests; axe on key pages. *Done when suite is green in CI* | CMP-001..CMP-015 | T-206..T-211 | TODO |

## Phase 3: Notices, public site, dashboards (week 3)

| ID | Task and done-when | Requirements | Depends | Status |
|---|---|---|---|---|
| T-301 | **Notices.** CRUD, audiences, pin, schedule and expiry job, draft by committee and publish by office bearer, attachments, acknowledgement (P1). *Done when each audience sees only what it should* | NTC-001..NTC-006 | T-210 | TODO |
| T-302 | **Events.** CRUD, calendar list, dashboard widgets (RSVP is P2). *Done when events show on public site and dashboards by audience* | EVT-001 | T-301 | TODO |
| T-303 | **Public site.** Home, About, Amenities, Notices, Events, Contact, enquiry form with rate limit and Turnstile, SEO metadata, sitemap. *Done when Lighthouse mobile performance is 90 or more* | PUB-001..PUB-003, SOC-007 | T-301, T-302 | TODO |
| T-304 | **Resident dashboard.** My status, open complaints, bookings placeholder, dues placeholder, notices, events. *Done when it matches design.md 8.2 at mobile and desktop* | CMP-009 | T-206, T-301 | TODO |
| T-305 | **Committee dashboard v1.** Open complaints by zone and age, SLA breaches, workload, notices. *Done when counts match seeded data and link to filtered lists* | RPT-001 | T-207, T-211 | TODO |
| T-306 | **Notification centre.** Full list, mark read, deep links; preferences page (P1). *Done when unread counts update without reload on navigation* | NTF-001, NTF-003 | T-210 | TODO |
| T-307 | **Polish and audit of weeks 1-3.** Responsive and theme pass, axe, empty/error states, copy review. *Done when review checklist in design.md 13-14 passes* | n/a | T-301..T-306 | TODO |

## Phase 4: Booking, history, map v1 (week 4)

| ID | Task and done-when | Requirements | Depends | Status |
|---|---|---|---|---|
| T-401 | **Booking schema.** `resources`, `bookings` with exclusion constraint, `blackout_dates`; RLS. *Done when a concurrent double-booking test fails the second insert* | BKG-004 | T-307 | TODO |
| T-402 | **Resource management.** Admin CRUD with rules, fees, approval flag, approver department. *Done when seeded resources are editable* | BKG-001, BKG-008 | T-401 | TODO |
| T-403 | **Booking service.** Rule validation (hours, duration, advance window, limits), auto-approve or approval flow, cancel, expiry job. *Done when each rule has a failing and passing test* | BKG-003, BKG-005, BKG-007 | T-401 | TODO |
| T-404 | **Booking UI.** Calendar (day, week, month), request flow, my bookings, committee approvals. *Done when a resident books the hall and a committee member approves it* | BKG-002, BKG-009 | T-403 | TODO |
| T-405 | **Vendors and work orders.** CRUD, create from complaint, completion resolves complaint, cost fields committee-only. *Done when a complaint becomes a work order and back* | WRK-001, WRK-002 | T-204 | TODO |
| T-406 | **Service history.** `property_history_v` view, timeline UI for property, cost hiding. *Done when resident sees own history and committee sees any, without costs for residents* | HIS-001, HIS-003 | T-405 | TODO |
| T-407 | **Map v1 data and import.** GeoJSON import for zones and properties with preview, layout fallback, public features API filtered by policy. *Done when the seed society renders from imported data* | MAP-001, MAP-004, MAP-005 | T-102, T-007 | TODO |
| T-408 | **Map v1 UI.** MapView, zone click panel, property drawer, list view equivalent, mobile bottom sheet. *Done when design.md 8.6 flows work by mouse, touch, and keyboard* | MAP-002, MAP-003, MAP-012 | T-407, T-406 | TODO |
| T-409 | **Complaint location picker.** Map picker with dropdown fallback in the complaint form. *Done when a complaint can be raised from a map click and shows on the property drawer* | MAP-010 | T-408, T-206 | TODO |

**MVP sign-off gate (end of week 4):** all P0 acceptance criteria pass; role-matrix and RLS tests green; demo on a phone with seeded data; backups configured.

## Phase 5: Billing and payments (week 5)

| ID | Task and done-when | Requirements | Depends | Status |
|---|---|---|---|---|
| T-501 | **Billing schema.** `charge_heads`, `invoices`, `invoice_lines`, `payments`, `ledger_entries` (append-only), constraints. *Done when constraints and triggers have tests* | BIL-001, BIL-005 | MVP gate | TODO |
| T-502 | **Charge setup and invoice generation.** Admin charge heads, monthly job, preview, idempotency. *Done when running twice creates no duplicates* | BIL-001, BIL-002 | T-501 | TODO |
| T-503 | **Dues UI and receipts.** Resident dues and history, treasurer invoice list, receipt PDF. *Done when a receipt PDF is generated with sequential number* | BIL-003, BIL-007 | T-502 | TODO |
| T-504 | **Razorpay payment.** Order creation, checkout, signed webhook, idempotent capture, ledger and receipt in one transaction (sandbox). *Done when a sandbox payment marks the invoice paid and duplicate webhooks change nothing* | PAY-001, PAY-002, PAY-005 | T-501 | TODO |
| T-505 | **Offline payments.** Treasurer records cash, cheque, transfer with proof and reference. *Done when ledger and invoice status update correctly with audit entry* | BIL-008 | T-501 | TODO |
| T-506 | **Defaulters and ledger reports.** Filters, CSV export. *Done when totals reconcile with ledger in tests* | BIL-006, RPT-002 | T-503 | TODO |
| T-507 | **Booking and request fees.** Booking creates invoice; confirmation after payment or override. *Done when a paid booking confirms automatically* | BKG-006, SRQ-003 | T-504, T-403 | TODO |
| T-508 | **Late fees and reminders; reconcile job.** Late-fee rule, reminder schedule, gateway reconciliation report. *Done when time-travel tests prove one late fee per period* | BIL-004, PAY-003 | T-504 | TODO |

## Phase 6: Utilities, projects, search, hardening (week 6)

| ID | Task and done-when | Requirements | Depends | Status |
|---|---|---|---|---|
| T-601 | **Infrastructure data.** `assets`, `utility_lines` schema; admin import and basic edit. *Done when seeded water and electrical lines load* | MAP-006, MAP-007 | T-407 | TODO |
| T-602 | **Utility layers on map.** Layer toggles, legend, colour plus dash styles, permission gating, asset cards. *Done when residents cannot fetch utility features by API* | MAP-006, MAP-007 | T-601 | TODO |
| T-603 | **Projects.** CRUD, public progress updates, affected area on the map, budget vs spent. *Done when a project shows hatched on the map to all residents* | WRK-003, WRK-004, WRK-005, MAP-008 | T-405, T-407 | TODO |
| T-604 | **Service requests.** Types with form schema, submit and decide, status tracking, fees. *Done when an NOC request completes end to end* | SRQ-001, SRQ-002 | T-507 | TODO |
| T-605 | **Global search.** Per-module `searchFor`, trigram, permission filtered, UI bar and results page, fly to map. *Done when search never returns an item the actor cannot open (tested)* | SRCH-001, SRCH-002, MAP-011 | T-205, T-406 | TODO |
| T-606 | **Reports and exports.** Committee reports, CSV exports for every committee table. *Done when exports respect scope and are audited* | RPT-001, RPT-002 | T-506 | TODO |
| T-607 | **Complaint heatmap.** Counts by zone and property on map. *(cut-list item 1)* | MAP-009, CMP-014 | T-602 | TODO |
| T-608 | **Document vault.** Folders, scopes, versions. *(cut-list item 2)* | DOC-001 | T-605 | TODO |
| T-609 | **Security hardening.** Rate limits review, CSP and headers, RLS and policy audit, dependency audit, upload checks, secrets scan, pen-test checklist. *Done when checklist in architecture 20 is fully ticked* | all | T-606 | TODO |
| T-610 | **Accessibility pass.** Keyboard, screen reader spot checks, axe on all pages, both themes. *Done when no serious axe violations remain* | design 13 | T-606 | TODO |
| T-611 | **Deploy and operations.** Production environment, backups and restore drill, monitoring, health checks, `docs/runbook.md`. *Done when a test restore is recorded in memory.md* | NFR | T-609 | TODO |
| T-612 | **Pilot onboarding and UAT.** Real society data import, committee training, feedback log, bug triage. *Done when the committee signs off* | PRD 12 | T-611 | TODO |

---

## Blockers

| Date | Task | Blocker | Needs |
|---|---|---|---|
| | | | |

## Decisions pending (summary; details in `memory.md`)

Q-001 name · Q-002 hosting · Q-003 layout plan format · Q-004 SMS provider · Q-005 charge rules · Q-008 booking fees online or offline

## Session log (append-only, newest at the bottom)

| Date | Agent/human | Tasks touched | Summary | Next |
|---|---|---|---|---|
| 2026-10-25 | Human + Claude | planning | PRD, architecture, design, rules, memory, tracker written | T-001 scaffold |
