# Roadmap (documentation → platform)

Effort is described by **subsystem scope**, not calendar duration.

## Phase 0 — Docs & repo home (current)

- [x] Vision, domain, architecture, Hypeluxe brief
- [x] State machines, audit/attribution, identity/RBAC/Party
- [x] Budget, cost-plus, client invoices, payments, drawdowns
- [x] Platform engineering: policy as data, functional core/shell, CD, observability
- [x] Security/compliance decisions: PIPEDA+GDPR, SSO/passwordless, machineaid hosting
- [ ] Create private repo under `machineaid` (requires org write access; blocked for this agent)
- [ ] Point Cursor / CI at the new repo; treat these docs as the initial import

## Phase 1 — Domain skeleton

- Effect Schema for Workspace, Member, Event, Schedule, Stakeholder, Party, Guest, TravelLeg, Stay, AccommodationBlock, Movement, Budget, Cost, ClientInvoice, PaymentApplication, ClientPayment, Drawdown, CommercialTerms, Vendor, VendorAssignment, Task, AuditRecord, **PolicyDocument**
- FSM modules + unit tests for illegal vs legal transitions (incl. money machines)
- Policy evaluator + Hypeluxe default policy fixtures
- In-memory repos; `Audit` buffer in tests (functional core only)
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

## Phase 4 — Logistics boards + money + Hypeluxe pack

- Accommodation blocks/allotment, stays, movement plans (arrival/departure/local)
- Arrival-day dispatcher views (offline-capable)
- Budget sheet, client invoices (draft/issue), costs, **manual** payments + apply to invoice, drawdowns, A/R + cash widgets
- Tenant branding + destination wedding / gala templates + conference proof template
- Vendor assignment flows (hotels, transport)

**Note:** Do not start payment-processor or accounting auto-sync in this phase.

## Phase 5 — Continuous delivery + observability hardening

- Staging auto-deploy from main; promote same artifact to prod
- Playwright smoke gate (offline event, multi-day, logistics, invoice+payment+drawdown, audit)
- Schema migration discipline (N−1 clients)
- OTel traces/metrics on command + sync path; `correlationId` end-to-end
- Policy pack activation tested in CI (schema validate Hypeluxe defaults)

## Explicitly deferred (unless pulled forward)

- Full GL / tax engine / e-invoicing networks (simple PDF invoice export in scope)
- Payment processing and automatic sync to Stripe/banks/QuickBooks (manual flows first; automation as command adapters later)
- Document vault (passports, contracts) — pending O-007
- Email/SMS communications platform — pending O-008
- Guest self-serve portal — pending O-004
- Vendor login
- Homegrown password+MFA (SSO/passwordless is **in**, not deferred)

## Decided (see [decisions.md](./decisions.md))

- D-001 PIPEDA + GDPR
- D-002 SSO / passwordless out of the gate
- D-003 machineaid hosts/operates the product

## Open decisions

See [decisions.md](./decisions.md) O-001 … O-016 (repo name, sync, UI, pilot scope, policy defaults, day-of, multi-currency).
