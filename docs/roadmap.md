# Roadmap (documentation → platform)

Effort is described by **subsystem scope**, not calendar duration.

## Phase 0 — Docs & repo home (current)

- [x] Vision, domain, architecture, Hypeluxe brief
- [x] State machines, audit/attribution, identity/RBAC/Party
- [x] Budget, cost-plus, client invoices, payments, drawdowns
- [x] Platform engineering + security decisions
- [x] **v1 slice + design revisions** (drawdowns in; offline split; facades)
- [ ] Create private repo under `machineaid`
- [ ] Point Cursor / CI at the new repo; treat these docs as the initial import

## Phase 1 — Domain skeleton (v1 cut)

- Effect Schema / pure FSMs for v1 entities only ([v1-slice.md](./v1-slice.md))
- Flat Hypeluxe `ResolvedPolicy` (D-004…D-010)
- Money math: Billing + **ClientFunds (incl. drawdown)** + DayOfCash
- `runCommand` tests with in-memory repos + audit buffer
- Journey unit tests for the three acceptance paths

## Phase 2 — Local client shell (ops)

- App shell + local DB
- Guests/logistics/day-of cash (local-first)
- No offline invoice/drawdown apply

## Phase 3 — Online money + sync

- Billing + ClientFunds online-authoritative (issue, clear, **drawdown**, budget)
- Sync ops aggregates; smoke journey 2 (milestones + drawdown) and 1 + 3

## Phase 4 — Hypeluxe pack polish

- Templates, branding, dispatcher views, budget vs drawdown widgets
- Still no payment-processor auto-sync

## Phase 5 — CD + light observability

- Staging smoke = three journeys
- correlationId + structured logs; OTel deep dive later

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
