# Decisions log

Working decisions for the event platform. Prefer short entries here; promote to full ADRs in-repo when implementation starts.

## Decided

| ID | Decision | Date | Notes |
|---|---|---|---|
| D-001 | **Compliance: PIPEDA + GDPR** | 2026-08-12 | Design to the stricter overlapping control; see [security-and-compliance.md](./security-and-compliance.md) |
| D-002 | **Authn: SSO / passwordless out of the gate** | 2026-08-12 | Avoid rolling our own MFA; use IdP / passwordless provider factors |
| D-003 | **Hosting: machineaid product** | 2026-08-12 | machineaid operates staging/prod; Hypeluxe is a tenant/customer |
| D-004 | **Policy: `allowLogisticsBeforeRsvp` = false** | 2026-08-12 | Hypeluxe default: logistics only when GuestRsvp is `attending` |
| D-005 | **Policy: `requireFlightLinkForAirportPickup` = true** | 2026-08-12 | Arrival airport pickups must link a TravelLeg before AssignMovement |
| D-006 | **Policy: drawdown funds = `cleared` only** | 2026-08-12 | `recorded` = client claim; `cleared` = planner/host acknowledged receipt. Only cleared counts toward available funds |
| D-007 | **Policy: drawdown cost basis = `actual`** | 2026-08-12 | Cap drawdowns on cost `actual` (amount from vendor invoice/receipt **received**). If `actual` unset, no draw against that cost until receipted (or override). Prefer not to use bare `committed` as spendable basis |
| D-008 | **Policy: `autoDraftDepositFromOffering` = true** | 2026-08-12 | Attaching an offering with a deposit schedule auto-creates a **draft** ClientInvoice; planner still issues manually |
| D-009 | **Policy: `coordinatorCanReadMoney` = false** | 2026-08-12 | Coordinators do not see budgets, client invoices, cost-plus margins, or drawdown/A/R. **Exception:** day-of cash (D-010) |
| D-010 | **Policy: `coordinatorCanManageDayOfCash` = true** | 2026-08-12 | Coordinators manage day-of float: (1) petty cash/float issued, (2) COD/tip payouts from float, (4) end-of-day reconciliation. Not guest cash intake. Scoped to event day(s), not full ledger |
| D-011 | **v1 includes drawdowns** | 2026-08-12 | Planner control loop: cleared milestones + actual costs + budget; not deferred |
| D-012 | **v1 offline split** | 2026-08-12 | Ops local-first; Billing + ClientFunds online-authoritative |
| D-013 | **v1 policy = flat ResolvedPolicy** | 2026-08-12 | No multi-scope policy merge engine until second tenant |

## Open (need input)

| ID | Question | Options |
|---|---|---|
| O-001 | Repo name under `machineaid` | e.g. `event-ops`, `gather`, other |
| O-002 | Sync stack | Electric / PowerSync / CRDT / hybrid |
| O-003 | Frontend | React / Solid / defer |
| O-004 | Guest portal in pilot | planner-only / guest view+submit travel |
| O-005 | Host money visibility | invoices+balance only / also margins |
| O-006 | Seating / floor plans | out / thin wedge |
| O-007 | Documents | out / upload+tag only |
| O-008 | Notifications | out / in-app only |
| O-015 | `StartMovement` actor | planner/coordinator / add driver role |
| O-016 | Multi-currency | single currency per event (pilot) / FX phase-2 |

## Doc follow-ups queued after opens resolve

- Hypeluxe fixture pack outline (policy D-004…D-010 + journey fixtures)
- Host money visibility (O-005) once chosen
- Security retention defaults once Hypeluxe agrees
