# Memory: SocietyOS

Persistent project memory shared by the human and the agent across sessions. Keep it **short, factual, and current**. This file is auto-loaded every session, so every line costs context: prefer one line per entry, link to details instead of pasting them.

Last updated: 2026-10-25

## How to use this file

- **Read** it at the start of every session.
- **Append** (never rewrite history) when you: make or learn a decision, hit a gotcha, confirm a convention, or have a question the human must answer.
- **Never delete** an entry. If something is superseded, mark it `SUPERSEDED by D-0xx` and add the new entry.
- Entry IDs: `D-` decisions, `G-` gotchas, `C-` conventions, `Q-` open questions, `I-` ideas. Numbers only increase.
- Keep each entry to 1-3 lines. Date every entry (`YYYY-MM-DD`).
- If a section grows past about 40 entries, propose an archive to `docs/memory-archive.md` and keep a one-line pointer here.

---

## 1. Project snapshot

| | |
|---|---|
| What | Multi-tenant web app for housing societies: complaints, works, booking, notices, billing and payments, digital map, history, search |
| Codename | SocietyOS (final name open, Q-001) |
| Stage | Planning complete; build starting at task T-001 |
| Pilot | One society (details to be added: name, size, zones, committee contacts) |
| Team | One human lead developer plus Claude Code agent |
| Timeline | 6 weeks (week 0 foundations, weeks 1-4 MVP, weeks 5-6 billing and hardening) |
| Timezone / locale / currency | Asia/Kolkata, `en-IN`, INR (stored as integer paise) |
| Where to look | Scope `docs/prd.md` · Tech `docs/architecture.md` · Look `docs/design.md` · Rules `docs/rules.md` · Status `docs/tracker.md` |

## 2. Decisions

| ID | Date | Decision | Why | Status |
|---|---|---|---|---|
| D-001 | 2026-10-25 | Modular monolith in a pnpm workspace (`apps/web`, `packages/{db,core,policy,ui,config}`) | Fastest to ship and operate with a tiny team plus agents | Active |
| D-002 | 2026-10-25 | Next.js App Router, TypeScript strict, Tailwind with tokens, Radix primitives | One codebase for public site and app; strong agent support | Active |
| D-003 | 2026-10-25 | PostgreSQL + PostGIS + RLS by `society_id`; Drizzle ORM | Relational integrity, geometry, constraints, tenancy in one store | Active |
| D-004 | 2026-10-25 | Better Auth with email OTP first, phone OTP in P1 | Passwordless suits residents; Auth.js is now maintained under Better Auth. Re-verify plugin support at T-006 | Active |
| D-005 | 2026-10-25 | Custom `policy` package: role x action matrix as data plus `scopeFilter` SQL predicates | Lists must be filtered in SQL, not only checked per row; matrix is testable | Active |
| D-006 | 2026-10-25 | Escalation is a flag (`escalated_at`), not a complaint status | Keeps the state machine small; deviates from the early plan doc, which listed Escalated as a state | Active |
| D-007 | 2026-10-25 | Complaints, service requests, and projects are separate entities | A complaint is a resident issue; a request asks for something; a project is society-wide work. Mixing them breaks reporting | Active |
| D-008 | 2026-10-25 | Service history is a database view over complaints, work orders, projects, requests | No second source of truth to keep in sync | Active |
| D-009 | 2026-10-25 | Money is integer paise; ledger is append-only; Razorpay webhooks are the only source of payment truth | Correctness and auditability | Active |
| D-010 | 2026-10-25 | Map: MapLibre with GeoJSON from PostGIS; SVG/image layout fallback behind the same interface | Pilot may not have survey data | Active |
| D-011 | 2026-10-25 | Jobs with pg-boss; search with Postgres FTS plus pg_trgm; no Redis or Meilisearch in MVP | Fewer moving parts | Active |
| D-012 | 2026-10-25 | Brutalist design: zero radius, 2px borders, hard shadows, green on neutral greys, status as text plus glyph | Founder brief; see `design.md` | Active |
| D-013 | 2026-10-25 | Visibility scope model self / department / all on complaints, requests, documents | From the founder sketch (self / dept / all) | Active |
| D-014 | 2026-10-25 | Resident onboarding requires committee approval of the property claim | Prevents strangers from claiming a house | Active |

## 3. Conventions (confirmed)

| ID | Convention |
|---|---|
| C-001 | Requirement IDs (`CMP-004`) and task IDs (`T-203`) appear in branch names, commits, PRs, and tests |
| C-002 | Complaint code format `CMP-YYYY-NNNNNN`; other codes: work orders `WO-`, projects `PRJ-`, service requests `SRQ-`, invoices `INV-FY-NNNN`, receipts `RCT-FY-NNNN` |
| C-003 | Roles in code: `resident`, `committee`, `office_bearer` (chairman, secretary, treasurer by `office_title`), `admin`, `staff` |
| C-004 | Default departments: Maintenance, Water and Electrical, Security, Finance, Community and Events, Gardens and Amenities |
| C-005 | UI vocabulary: complaint, resident, committee, property, booking, notice (never ticket, user, unit in UI text) |
| C-006 | Financial year runs April to March for invoice and receipt numbering |
| C-007 | Branch format `t-<id>-<slug>`; conventional commits |

## 4. Glossary

| Term | Meaning |
|---|---|
| Society / RWA | Housing society or resident welfare association; one tenant in the system |
| Zone | Section of the society (D1, D2, ...): block, sector, or lane |
| Property | A unit: house, flat, shop, plot, or common area |
| Occupancy | A user's approved link to a property (owner, tenant, family) |
| Committee | Elected body running the society; split into departments |
| Office bearer | Chairman, Secretary, or Treasurer |
| Maintenance | Recurring charges paid by residents to run the society |
| Sinking fund | Reserve for major repairs, collected with maintenance |
| NOC | No Objection Certificate issued by the society (for example for renovation) |
| AGM | Annual General Meeting |
| SLA | Time limits to acknowledge and resolve a complaint, by priority |
| Scope (visibility) | self / department / all: who may see an item |
| Utility layer | Map layer for water, drainage, electrical, gas, internet, or fire lines |
| Asset | A point object on the map: valve, pump, transformer, manhole, meter, hydrant |
| Work order | A unit of work done at a property or asset, with vendor and cost |
| Project | Society-wide work (road development, gas pipeline, painting) with budget and public updates |
| RLS | PostgreSQL row-level security |
| Paise | 1/100 of a rupee; the unit used to store all money |

## 5. Gotchas and learnings

*(Empty at start. Add entries as `G-001` ... when something surprising is discovered: a library quirk, a failed approach, a data issue.)*

| ID | Date | Gotcha | Fix or workaround |
|---|---|---|---|
| | | | |

## 6. Open questions

Make the stated default, keep moving, and record the answer when the human decides.

| ID | Question | Default until answered | Owner |
|---|---|---|---|
| Q-001 | Final product name? Shortlist: Nivaas, Parisar, CivicGrid; keep SocietyOS as codename | Use SocietyOS in code, a `brand.ts` constant for display name | Human |
| Q-002 | Hosting: Vercel + Neon, or a VPS with Docker Compose? | Build host-agnostic; deploy preview on Vercel + Neon | Human |
| Q-003 | Does the pilot society have a layout plan or survey (CAD, PDF, image)? | Assume image only; use the SVG/layout fallback first | Human |
| Q-004 | SMS provider choice (MSG91 or Twilio) and need for WhatsApp? | Email only in MVP; SMS adapter stubbed | Human |
| Q-005 | Maintenance charge rules: flat per property type or per sq ft? Late fee rule? | Support both; configure at billing setup | Human |
| Q-006 | Should tenants pay maintenance directly or via owner? | Invoice goes to the approved owner; tenant may pay on behalf | Human |
| Q-007 | Pilot size (properties, zones, committee members)? | 3 zones, 40 properties in seed; import real data via CSV | Human |
| Q-008 | Booking fees: collect online in MVP or only record offline? | Offline record in MVP; online in v1.0 with billing | Human |
| Q-009 | Data residency or hosting-region requirements? | Mumbai/Singapore region if available | Human |
| Q-010 | Are there society bylaws that limit who may publish notices or approve bookings? | Office bearers publish; committee approves bookings for assigned resources | Human |

## 7. Ideas parking lot

Ideas are **not** scope. Do not build them without a tracker item.

| ID | Idea |
|---|---|
| I-001 | Complaint heatmap on the map by category and month |
| I-002 | Preventive-maintenance calendar for pumps, lifts, and transformers |
| I-003 | Vendor rating based on resolution time and resident feedback |
| I-004 | QR codes on assets that open their service history |
| I-005 | Resident polls and e-voting for AGM resolutions |
| I-006 | WhatsApp notifications and a complaint bot |
| I-007 | Visitor and delivery pre-approval with gate guard app |
| I-008 | Utility consumption (water and electricity) per property with anomaly alerts |

## 8. Founder sketch notes (raw intent, kept for reference)

- Map shows zones D1-D6; clicking a zone opens a detailed view; houses clickable with total complaints; coloured lines for each utility parameter.
- Member sees: my status, events, public notices. Committee: process complaints, access complaints, complete history. Access tiers: members, committee (members and secretary), admin.
- Items have a visibility box: self / dept / all.
- Early names "Family Sync" and "Civic Community" were rejected.
- Build order from the sketch: research, data, role target, tools, then charts and map.

## 9. Session notes (append-only, newest at the bottom)

| Date | Session summary |
|---|---|
| 2026-10-25 | Planning docs written: PRD, architecture, design, rules, tracker, memory. No code yet. Next: T-001 repo scaffold. |
