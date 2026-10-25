# PRD: SocietyOS (Society Management System)

| | |
|---|---|
| Codename | SocietyOS (final product name still open, see `memory.md` Q-001) |
| Version | 0.2 |
| Status | Approved for build |
| Last updated | 2026-10-25 |
| Companion docs | `architecture.md` (how), `design.md` (look), `rules.md` (agent rules), `tracker.md` (status), `memory.md` (decisions and context) |

Requirement IDs (for example `CMP-004`) are referenced by `tracker.md`, commits, and tests. Never renumber an ID; add new ones at the end of a module.

Priority key: **P0** = MVP (must ship in weeks 0-4), **P1** = v1.0 (weeks 5-6), **P2** = later.

---

## 1. Summary

SocietyOS is an end-to-end web application for housing societies (apartment complexes, gated communities, RWAs). It replaces WhatsApp groups, paper registers, and spreadsheets with one permission-aware system covering complaints, service requests, society works, resource booking, notices, maintenance billing and payments, a digital map of the society and its utilities, and a searchable history of everything.

It is a **multi-tenant** product (many societies, strict data isolation), launched first with one pilot society.

## 2. Problem

- Complaints get lost in chat groups; there is no owner, status, or SLA.
- Nobody can answer "what work was done at this property or pipeline, when, by whom, at what cost?"
- Hall and ground bookings clash and cause disputes.
- Maintenance dues live in registers; collection is slow and opaque.
- Knowledge of drainage, water, electrical, and gas networks lives in one person's head.
- Residents have no transparency; the committee has no accountability trail.

## 3. Goals and non-goals

### Goals
1. Resident can raise a complaint in under 60 seconds and see live status.
2. Every complaint has an owner, status, SLA, and complete audit trail.
3. A property's full service history is reachable in two clicks from the map.
4. Resources can be booked without clashes, with approval and fees where configured.
5. Maintenance dues can be billed and paid online and reconcile automatically.
6. Committee gets one dashboard answering: what is open, overdue, and costing money.
7. Access is strictly role-, scope-, and ownership-based, enforced on the server.

### Non-goals (MVP)
- Native mobile apps (web is mobile-first; PWA later).
- Visitor/gate management, polls and voting, accounting export, IoT meters (all later).
- Multi-language UI (structure text for it, ship English).
- Marketplace, classifieds, or social feed features.

## 4. Users and roles

| Role | Description | Key rights |
|---|---|---|
| Visitor | Not logged in | Public site, public notices and events, contact form |
| Resident | Owner or tenant linked to an approved property occupancy | Own property, own complaints, bookings, dues, public and resident notices |
| Committee member | Elected member in a department | Process complaints in own department, see society-wide data |
| Office bearer | Chairman, Secretary, Treasurer | All committee rights plus approvals, finance, publishing |
| Admin | Society or platform super-admin | Users, roles, setup, geometry, config, audit |
| Staff / vendor | Plumber, electrician, guard (P2) | Only assigned work orders |

A user may hold several roles in a society (a committee member is also a resident). Roles are per society.

### Visibility scope (applies to complaints, requests, documents)
- **self**: creator plus the people handling it.
- **department**: creator plus the owning committee department.
- **all**: every resident of the society (and public where explicitly marked).

Defaults: society-wide issues default to `all`; categories flagged sensitive (security, disputes, finance) force `department`.

## 5. Success metrics

| Metric | Target at pilot day 60 |
|---|---|
| Complaints raised through the app vs. other channels | 80% or more |
| Median time to acknowledge | under 2 hours |
| Complaints resolved within SLA | 85% or more |
| Residents onboarded and approved | 70% of occupied properties |
| Maintenance dues paid online (once billing is live) | 50% or more |
| Booking clashes | 0 |
| Cross-tenant or cross-role data leaks | 0 |

## 6. Scope and phases

| Phase | Weeks | Contents |
|---|---|---|
| MVP | 0-4 | Public site, auth and roles, society and property setup, onboarding and approval, complaints, notices, events, resource booking, work orders, service history, basic map, dashboards, audit log, in-app and email notifications |
| v1.0 | 5-6 | Billing, Razorpay payments, receipts, ledger, reports, utility layers on map, projects, service requests, global search, hardening |
| v1.1 | later | Vendor and staff accounts with portal, SLA escalation rules engine, document vault, polls, visitor management |
| v2 | later | PWA/mobile, WhatsApp, IoT meters, multi-society self-onboarding, accounting export |

Cut list if time runs short, in this order: complaint heatmap, document vault, event RSVP, PDF reports, utility layers beyond water and electrical. **Never cut:** roles and policy layer, complaint lifecycle, audit log, backups.

---

## 7. Functional requirements

### 7.1 Authentication and onboarding (AUTH)

| ID | Pri | Requirement | Acceptance criteria |
|---|---|---|---|
| AUTH-001 | P0 | Passwordless login with email OTP | 6-digit code, expires in 10 min, single use; 5 wrong attempts lock that email for 15 min; rate limited per email and per IP |
| AUTH-002 | P0 | Sessions | Secure httpOnly cookie, 30-day rolling expiry; sign out; "sign out everywhere" |
| AUTH-003 | P0 | Active society and role context | A user with several roles can switch; every request resolves one actor (user, society, roles, department) |
| AUTH-004 | P1 | Phone OTP verification and login | Same rules as AUTH-001; SMS provider behind an adapter; console output in development |
| AUTH-005 | P0 | Resident onboarding | After first login user picks zone, property, relation (owner/tenant) and submits a claim; status `pending`; no resident features until approved |
| AUTH-006 | P0 | Approval queue | Office bearer or admin sees pending claims, approves or rejects with reason; requester notified; action audited |
| AUTH-007 | P1 | Household members | Resident adds family members under the property with limited rights (own complaints and bookings only) |
| AUTH-008 | P1 | Tenancy end | Occupancy with an end date auto-ends; access to that property is removed; history retained |

### 7.2 Society setup (SOC)

| ID | Pri | Requirement | Acceptance criteria |
|---|---|---|---|
| SOC-001 | P0 | Society profile | Name, address, contact, timezone (default Asia/Kolkata), logo, intro text used on public site |
| SOC-002 | P0 | Zones CRUD | Unique code per society (for example D1); name; sort order; cannot delete a zone with properties |
| SOC-003 | P0 | Properties CRUD and CSV import | Unique (zone, unit number); types house/flat/shop/plot/common; import has dry-run, row-level errors, and is idempotent |
| SOC-004 | P0 | Departments and committee assignment | Default departments seeded; membership has department and optional office title |
| SOC-005 | P0 | Complaint categories | Each category maps to a department, default priority, sensitive flag |
| SOC-006 | P0 | SLA policy | Acknowledge and resolve hours per priority; defaults in section 8 |
| SOC-007 | P1 | Amenities list | Public list with photo, description, optional link to a bookable resource |

### 7.3 Complaints (CMP)

| ID | Pri | Requirement | Acceptance criteria |
|---|---|---|---|
| CMP-001 | P0 | Raise complaint | Fields: category, title (5-120 chars), description, priority suggestion, visibility, location (property, common area, or map point), up to 5 images; works on a phone in under 60 s |
| CMP-002 | P0 | Human-readable code | `CMP-YYYY-NNNNNN`, sequential per society, never reused |
| CMP-003 | P0 | Routing and SLA | Category sets department; `due_at` computed from SLA policy and priority at creation |
| CMP-004 | P0 | Lifecycle | Only transitions in `architecture.md` section 10 are allowed; invalid ones return a typed error; each transition requires the right role and, where stated, a reason |
| CMP-005 | P0 | Event log | Every change appends a `complaint_events` row; events are never updated or deleted |
| CMP-006 | P0 | Visibility scoping | Lists, detail, search, and map respect self/department/all; verified by role-matrix tests |
| CMP-007 | P0 | Comments | Public thread between resident and handlers; internal notes visible to committee only |
| CMP-008 | P0 | Attachments | Residents: jpeg/png/webp, 5 files, 8 MB each; committee may also attach PDF; stored via signed URLs |
| CMP-009 | P0 | Committee board and table | Kanban by status plus table; filters: department, zone, priority, status, age, overdue, assignee; saved filter in URL |
| CMP-010 | P0 | Assignment | Assign to a committee member or vendor (P1); workload count per person |
| CMP-011 | P0 | Resident confirmation | Resident can confirm (closes) or reopen within 7 days of resolution with a reason; auto-close after 5 days silence |
| CMP-012 | P1 | Duplicate hint | While raising, show open complaints in same zone/category from the last 7 days |
| CMP-013 | P0 | Overdue and escalation flag | Job flags overdue complaints; secretary is notified; overdue shown prominently in committee views |
| CMP-014 | P1 | Heatmap data | Count per zone/property by period and category for the map |
| CMP-015 | P0 | Abuse limit | Max 10 new complaints per resident per day |
| CMP-016 | P1 | Satisfaction rating | 1-5 rating and optional note on close |

### 7.4 Notices and events (NTC, EVT)

| ID | Pri | Requirement | Acceptance criteria |
|---|---|---|---|
| NTC-001 | P0 | Notice CRUD | Title, body (rich text, sanitised), category, attachments |
| NTC-002 | P0 | Audience | public, residents, committee, or a specific zone |
| NTC-003 | P0 | Schedule, pin, expire | Publish-at and expire-at; pinned notices first; status draft/published/archived |
| NTC-004 | P1 | Read acknowledgement | Notice can require acknowledgement; committee sees who has not acknowledged |
| NTC-005 | P0 | Publishing rights | Committee member can draft; office bearer or admin publishes |
| NTC-006 | P0 | Public notices on site | Visitors see only audience=public and published |
| EVT-001 | P0 | Events | Title, description, start/end, venue (free text or resource), audience, shown on dashboards and calendar |
| EVT-002 | P2 | RSVP | Residents RSVP with guest count; organiser sees list |

### 7.5 Resource booking (BKG)

| ID | Pri | Requirement | Acceptance criteria |
|---|---|---|---|
| BKG-001 | P0 | Resource CRUD with rules | Kind, capacity, open/close time, min and max duration, advance window, fee, deposit, requires-approval flag |
| BKG-002 | P0 | Availability calendar | Day, week, and month views showing booked and blocked slots in society timezone |
| BKG-003 | P0 | Request a booking | Server validates hours, duration, advance window, blackouts, per-resident limits; clear error per rule |
| BKG-004 | P0 | Clash prevention | Enforced by a database exclusion constraint; concurrent requests for the same slot cannot both succeed |
| BKG-005 | P0 | Approval | Auto-approve when flag is off; otherwise assigned committee member or office bearer approves or rejects with reason |
| BKG-006 | P1 | Fee and deposit | Booking creates an invoice; confirmed only after payment or committee override |
| BKG-007 | P1 | Cancellation | Free cancellation up to N hours before start (configurable); later cancellation follows resource rule |
| BKG-008 | P0 | Blackout dates | Per resource or society-wide, with reason; existing bookings in the window are flagged |
| BKG-009 | P0 | My bookings and committee view | Residents see own; committee sees all with filters |
| BKG-010 | P1 | Per-resident limits | For example max 2 active bookings per resource per month |

### 7.6 Digital map (MAP)

| ID | Pri | Requirement | Acceptance criteria |
|---|---|---|---|
| MAP-001 | P0 | Render society | Zones, properties, and amenities from stored geometry; visitors see only the public layer (zones, amenities) |
| MAP-002 | P0 | Zone interaction | Click a zone: zoom in, side panel with totals (complaints, open complaints, active works; dues for committee) scoped by permission |
| MAP-003 | P0 | Property interaction | Click a property: drawer with details, complaint count, service history, dues; residents only for own property, committee for all |
| MAP-004 | P0 | Geometry import | Admin uploads GeoJSON for zones and properties with validation and preview; draw/edit tools are P1 |
| MAP-005 | P0 | Layout fallback | If no survey data, admin uploads an image/SVG layout and places entities by simple coordinates; same UI and data model |
| MAP-006 | P1 | Utility layers | Water, drainage, electrical, gas, internet, fire; toggle with legend; line colour plus dash style; committee and admin only |
| MAP-007 | P1 | Asset cards | Click valve, pump, transformer, manhole: type, installed date, last service, linked work orders |
| MAP-008 | P1 | Active works overlay | Hatched fill for affected areas of projects; visible to all residents |
| MAP-009 | P1 | Complaint layer | Filter by category and date; heat intensity |
| MAP-010 | P0 | Location picker | Complaint form can pick a property, zone, or point on the map (falls back to dropdowns) |
| MAP-011 | P1 | Search to map | Selecting a property in search flies the map to it |
| MAP-012 | P0 | Non-map equivalent | Every map interaction has a keyboard-accessible list or table equivalent |

### 7.7 Service history and works (HIS, WRK, SRQ)

| ID | Pri | Requirement | Acceptance criteria |
|---|---|---|---|
| HIS-001 | P0 | Property history timeline | Complaints, work orders, projects, and requests touching a property, newest first, filterable |
| HIS-002 | P1 | Asset history | Same for an asset or utility line |
| HIS-003 | P0 | Cost privacy | Costs and vendor details hidden from residents; visible to committee |
| HIS-004 | P1 | Export | CSV export of a property's history for committee |
| WRK-001 | P0 | Vendors | Name, trade, phone, email, status |
| WRK-002 | P0 | Work orders | Created from a complaint, project, or manually; status, assignee or vendor, scheduled and completed dates, cost, notes; completion can resolve the source complaint |
| WRK-003 | P1 | Projects | Title, description, status, budget, spent, vendor, dates, visibility, public progress updates with optional photos |
| WRK-004 | P1 | Project geometry | Affected area highlighted on the map |
| WRK-005 | P1 | Budget vs actual | Spent is the sum of linked work-order costs and expenses |
| WRK-006 | P2 | Preventive maintenance | Recurring inspections for assets that generate work orders |
| SRQ-001 | P1 | Request types | Admin defines types (gas connection, NOC, parking sticker, address proof) with a form schema, fee, approver department |
| SRQ-002 | P1 | Submit and decide | Resident submits; approver approves or rejects with reason; status visible to the resident |
| SRQ-003 | P1 | Fees | Fee invoiced and collected through the payments engine |

### 7.8 Billing and payments (BIL, PAY)

| ID | Pri | Requirement | Acceptance criteria |
|---|---|---|---|
| BIL-001 | P1 | Charge heads | Maintenance, sinking fund, parking, water, penalty; flat or per-sq-ft; frequency; applies to property types |
| BIL-002 | P1 | Monthly invoice generation | Scheduled job, idempotent per (property, period); preview before issue |
| BIL-003 | P1 | Dues view | Resident sees own invoices, arrears, receipts; treasurer sees all |
| BIL-004 | P1 | Late fee | Configurable rule applied by job once per overdue period |
| BIL-005 | P1 | Ledger | Append-only double-entry style entries; corrections are reversing entries |
| BIL-006 | P1 | Defaulters report | Filter by zone and age of dues; CSV export |
| BIL-007 | P1 | Receipts | PDF receipt per payment, numbered sequentially |
| BIL-008 | P1 | Offline payments | Treasurer records cash/cheque/bank transfer with proof and reference |
| BIL-009 | P2 | Expenses | Record society expenses with vendor and attachment |
| PAY-001 | P1 | Online payment | Razorpay order and checkout (UPI, cards, netbanking) |
| PAY-002 | P1 | Webhooks | Signature verified, idempotent, safe to receive twice or out of order |
| PAY-003 | P1 | Reconciliation | Daily job compares gateway and ledger and reports mismatches |
| PAY-004 | P2 | Refunds | Admin-initiated refund with ledger reversal |
| PAY-005 | P1 | Card data | Never stored or logged |

### 7.9 Cross-cutting (SRCH, DOC, NTF, AUD, RPT, PUB)

| ID | Pri | Requirement | Acceptance criteria |
|---|---|---|---|
| SRCH-001 | P1 | Global search | One bar; searches complaints, notices, properties, bookings, documents, and (committee only) residents; results already permission-filtered |
| SRCH-002 | P1 | Typo tolerance | Trigram matching on titles and names |
| DOC-001 | P2 | Document vault | Folders, visibility scope, versions |
| NTF-001 | P0 | In-app notifications | Bell with unread count; mark read; deep link to the item |
| NTF-002 | P0 | Email notifications | Complaint status change, approval decisions, new public-to-residents notice |
| NTF-003 | P1 | Preferences | Per user, per channel, per event type |
| NTF-004 | P2 | SMS / WhatsApp | Provider adapter |
| AUD-001 | P0 | Audit log | Append-only record of role changes, approvals, status overrides, finance edits, geometry edits, exports: actor, action, entity, diff, IP, time |
| AUD-002 | P0 | Audit viewer | Admin and office bearer (read-only) can filter by actor, entity, date |
| RPT-001 | P1 | Committee dashboard | Open complaints by zone and age, SLA breaches, collection vs dues, upcoming bookings, projects in progress |
| RPT-002 | P1 | CSV export | Every committee table can export what the user is allowed to see |
| RPT-003 | P2 | PDF reports | Monthly summary |
| PUB-001 | P0 | Public site | Home, About, Amenities, Notices, Events, Contact, Login |
| PUB-002 | P0 | Enquiry form | Rate limited and bot-protected; delivered to committee inbox and stored |
| PUB-003 | P0 | SEO and performance | Server-rendered, metadata, sitemap, Lighthouse performance 90 or more on mobile |

---

## 8. Business rules

### Complaint SLA defaults (configurable per society)

| Priority | Examples | Acknowledge | Resolve |
|---|---|---|---|
| Critical | Water or power outage, fire, lift stuck, safety | 30 min | 4 h |
| High | Leakage, blocked drain, security lapse | 4 h | 24 h |
| Normal | Repairs, housekeeping | 12 h | 72 h |
| Low | Cosmetic, suggestions | 48 h | 7 days |

Rules: resident suggests priority; committee may change it (audited). Reopen allowed 7 days after resolution. Auto-close 5 days after resolution without response.

### Booking rules
- Slots are half-open ranges `[start, end)`; back-to-back bookings are allowed.
- A `requested` booking holds the slot until decided or expired (24 h default).
- Society events may create blackout windows that override existing requests (requests are flagged for the committee).

### Billing rules
- All money is stored as integer paise in INR; no floating point anywhere.
- An invoice is immutable once issued; corrections use credit notes (negative lines on a new invoice) or a void plus reissue.
- Ledger entries are never edited or deleted.
- Receipt numbers are sequential per society and financial year (April-March).

## 9. Non-functional requirements

| Area | Requirement |
|---|---|
| Performance | Mobile 4G: LCP under 2.5 s on public pages; app route transitions under 1 s; p95 API under 500 ms for list endpoints (up to 5,000 rows paginated) |
| Scale target | One society: 1,000 properties, 5,000 users. Architecture ready for 100 societies without redesign |
| Availability | 99.5% monthly for MVP; planned maintenance announced |
| Security | OWASP ASVS Level 1 as a baseline; server-side authorisation on every call; RLS by society; rate limiting on OTP, login, forms; secure headers; dependency audit in CI |
| Privacy | Follow India's DPDP Act expectations: consent at onboarding, purpose limitation, data export and deletion on request, breach response process; role-based visibility of personal data |
| Accessibility | WCAG 2.2 AA; full keyboard operation; map has a non-visual equivalent |
| Browsers | Last 2 versions of Chrome, Edge, Safari, Firefox; Android Chrome and iOS Safari |
| Responsive | Mobile-first from 360 px; committee tables scroll inside their container |
| Backups | Daily automated database backup, point-in-time recovery if the host supports it, RPO 24 h, RTO 4 h, restore tested monthly |
| Observability | Error tracking, structured logs with request id, job success/failure metrics |
| i18n readiness | No hard-coded user-facing strings outside message files; dates and currency via Intl |
| Data retention | Complaints, ledger, audit: retained at least 8 years; uploaded files follow the parent record |

## 10. Assumptions and dependencies

- Pilot society provides a layout plan and a list of zones and properties (needed by week 1).
- Razorpay merchant onboarding started in week 1 (takes several days).
- An email provider account (Resend or similar) is available by week 2.
- Hosting account (Vercel and Neon, or a VPS) is available by week 0.
- Committee mandates the app as the official complaint and booking channel.

## 11. Open questions

Tracked in `memory.md` under "Open questions". Do not block on them; make the stated default and record the choice.

## 12. Release criteria (MVP)

1. All P0 requirements pass their acceptance criteria.
2. Role-matrix tests pass: each role attempts each action; forbidden ones fail.
3. Seeded demo society works end to end on a phone.
4. No open critical or high bugs; Lighthouse and axe checks pass on key pages.
5. Backups verified by a test restore.
6. Pilot committee has completed a guided walkthrough and signed off.
