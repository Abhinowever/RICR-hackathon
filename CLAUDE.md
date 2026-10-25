# SocietyOS: agent entry point

Multi-tenant web app for housing societies (complaints, works, booking, notices, billing, digital map). You are the engineering agent working with one human lead. Work in small, verified, vertical slices.

## Always loaded (read before doing anything)

@docs/rules.md
@docs/memory.md
@docs/tracker.md

## Read on demand (only the sections the task needs)

- `docs/prd.md`: what to build; requirement IDs and acceptance criteria
- `docs/architecture.md`: stack, layering, data model, state machines, jobs, payments, map
- `docs/design.md`: tokens, components, wireframes, accessibility
- `docs/specs/<module>.md`: module spec for the current task (write it first for non-trivial work)

## Start of session

1. From `tracker.md`, find the current focus and the next `TODO` task whose dependencies are `DONE`.
2. Say which task and requirement IDs you will do, and your plan in 2-3 lines.
3. Ask before starting if the task touches authorisation, money, RLS, or migrations that change existing data.

## End of session

Run `pnpm check`, update `docs/tracker.md` and `docs/memory.md`, summarise what changed, what was verified, and what is next.

## Non-negotiables (full list in rules.md)

- Server-side policy check in every service; lists use `scopeFilter`.
- Every tenant table has `society_id` plus RLS; every request runs in `withTenant`.
- Money is integer paise; ledger, complaint events, and audit logs are append-only.
- Design tokens and `packages/ui` only; zero radius, 2px borders, status as text plus glyph.
- No feature without tests, including a forbidden-path test.
- When unsure or blocked, stop and ask with options and a recommendation.
