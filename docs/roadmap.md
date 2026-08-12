# Roadmap (documentation → platform)

Effort is described by **subsystem scope**, not calendar duration.

## Phase 0 — Docs & repo home (current)

- [x] Vision, domain, architecture, Hypeluxe brief (this PR)
- [ ] Create private repo under `machineaid` (requires org write access; blocked for this agent)
- [ ] Point Cursor / CI at the new repo; treat these docs as the initial import

## Phase 1 — Domain skeleton

- Effect Schema for Workspace, Event, Schedule (`single_day` | `multi_day`), Vendor, Stakeholder
- In-memory repos + unit tests for create/update event schedule
- Hypeluxe template JSON (event types + sample checklist) with no UI yet

**Risk:** locking schedule shape too early — validate with Hypeluxe single- and multi-day examples.

## Phase 2 — Local client shell

- App shell with local DB
- Auth screens + event list/detail (local-only)
- Schedule editor for single-day and multi-day

**Depends on:** Phase 1 schemas stable enough for migrations.

## Phase 3 — Sync + cloud

- Choose SQL-sync vs hybrid CRDT
- Cloud auth + workspace membership
- Offline mutate → online sync smoke in CI

**Risk:** conflict UX; keep structured fields simple (LWW/row version) before CRDT notes.

## Phase 4 — Hypeluxe vertical pack

- Tenant branding + wedding/gala templates
- Package catalog + vendor assignment flows
- Day-of planner checklist

## Phase 5 — Continuous delivery hardening

- Staging auto-deploy from main
- Playwright: offline event create, multi-day edit, sync assert
- Schema migration discipline (N−1 clients)

## Open decisions

1. Repository name under `machineaid` (e.g. `event-ops`)
2. Sync stack (Electric / PowerSync / CRDT hybrid)
3. Frontend framework (React vs Solid) given Effect interop preference
4. Whether guest/stakeholder portal is in the Hypeluxe pilot
