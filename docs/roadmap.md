# Roadmap (documentation → platform)

Effort is described by **subsystem scope**, not calendar duration.

## Phase 0 — Docs & repo home (current)

- [x] Vision, domain, architecture, Hypeluxe brief
- [x] State machines, audit/attribution, identity/RBAC/Party
- [ ] Create private repo under `machineaid` (requires org write access; blocked for this agent)
- [ ] Point Cursor / CI at the new repo; treat these docs as the initial import

## Phase 1 — Domain skeleton

- Effect Schema for Workspace, Member, Event, Schedule, Stakeholder, Party, Guest, TravelLeg, Stay, AccommodationBlock, Movement, Vendor, VendorAssignment, Task, AuditRecord
- FSM modules + unit tests for illegal vs legal transitions
- In-memory repos; `Audit` buffer in tests
- Hypeluxe template JSON (destination wedding + sample multi-day conference)

**Risk:** locking schedule + logistics shapes too early — validate with destination wedding *and* conference examples.

## Phase 2 — Local client shell

- App shell with local DB
- Auth screens + event list/detail (local-only)
- Schedule editor for single-day and multi-day
- Guest list + party + RSVP commands (local-only)
- Activity feed reading local audit

**Depends on:** Phase 1 schemas + FSMs stable enough for migrations.

## Phase 3 — Sync + cloud

- Choose SQL-sync vs hybrid CRDT
- Cloud auth + workspace membership/RBAC re-check on sync
- Offline mutate → online sync smoke (guest + travel leg + audit actor preserved)
- Reject illegal/authz commands with audited outcomes

**Risk:** conflict UX; keep structured logistics fields simple (LWW/row version) before CRDT notes.

## Phase 4 — Logistics boards + Hypeluxe pack

- Accommodation blocks/allotment, stays, movement plans (arrival/departure/local)
- Arrival-day dispatcher views (offline-capable)
- Tenant branding + destination wedding / gala templates + conference proof template
- Vendor assignment flows (hotels, transport)

## Phase 5 — Continuous delivery hardening

- Staging auto-deploy from main
- Playwright: offline event create, multi-day edit, guest travel + movement assign, sync + audit assert
- Schema migration discipline (N−1 clients)

## Explicitly deferred (unless pulled forward)

- Payments / deposits / invoicing
- Document vault (passports, contracts)
- Email/SMS communications platform
- Guest self-serve portal (model allows; pilot may stay planner-only)
- Vendor login, SSO/SAML

## Open decisions

1. Repository name under `machineaid` (e.g. `event-ops`)
2. Sync stack (Electric / PowerSync / CRDT hybrid)
3. Frontend framework (React vs Solid) given Effect interop preference
4. Guest-facing itinerary portal in Hypeluxe pilot vs planner-only
5. Allow logistics before RSVP `attending`?
6. Require travel-leg link for airport arrival pickups?
7. Driver role for `StartMovement` vs planner/coordinator only
