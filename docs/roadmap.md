# Roadmap (documentation → platform)

Effort is described by **subsystem scope**, not calendar duration.

## Phase 0 — Docs & repo home (current)

- [x] Vision, domain, architecture, Hypeluxe brief (this PR)
- [ ] Create private repo under `machineaid` (requires org write access; blocked for this agent)
- [ ] Point Cursor / CI at the new repo; treat these docs as the initial import

## Phase 1 — Domain skeleton

- Effect Schema for Workspace, Event, Schedule (`single_day` | `multi_day`), Stakeholder, Guest, TravelLeg, Stay, AccommodationBlock, Transfer / LocalMove, Vendor
- In-memory repos + unit tests for schedule edits and guest logistics attachments
- Hypeluxe template JSON (destination wedding + sample multi-day conference) with no UI yet

**Risk:** locking schedule + logistics shapes too early — validate with destination wedding *and* conference examples.

## Phase 2 — Local client shell

- App shell with local DB
- Auth screens + event list/detail (local-only)
- Schedule editor for single-day and multi-day
- Guest list CRUD + RSVP status (local-only)

**Depends on:** Phase 1 schemas stable enough for migrations.

## Phase 3 — Sync + cloud

- Choose SQL-sync vs hybrid CRDT
- Cloud auth + workspace membership
- Offline mutate → online sync smoke in CI (include a guest + travel leg)

**Risk:** conflict UX; keep structured logistics fields simple (LWW/row version) before CRDT notes.

## Phase 4 — Logistics boards + Hypeluxe pack

- Accommodation blocks, stays, pickup/dropoff plans, local shuttles
- Arrival-day and transfer dispatcher-style views (offline-capable)
- Tenant branding + destination wedding / gala templates + conference proof template
- Package catalog + vendor assignment flows (hotels, transport)

## Phase 5 — Continuous delivery hardening

- Staging auto-deploy from main
- Playwright: offline event create, multi-day edit, guest travel + pickup assign, sync assert
- Schema migration discipline (N−1 clients)

## Open decisions

1. Repository name under `machineaid` (e.g. `event-ops`)
2. Sync stack (Electric / PowerSync / CRDT hybrid)
3. Frontend framework (React vs Solid) given Effect interop preference
4. Whether a guest-facing itinerary portal is in the Hypeluxe pilot (planner-only logistics first is viable)
5. Shared `Movement` type vs separate Transfer / LocalMove models
