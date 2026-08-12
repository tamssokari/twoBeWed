# Decisions log

Working decisions for the event platform. Prefer short entries here; promote to full ADRs in-repo when implementation starts.

## Decided

| ID | Decision | Date | Notes |
|---|---|---|---|
| D-001 | **Compliance: PIPEDA + GDPR** | 2026-08-12 | Design to the stricter overlapping control; see [security-and-compliance.md](./security-and-compliance.md) |
| D-002 | **Authn: SSO / passwordless out of the gate** | 2026-08-12 | Avoid rolling our own MFA; use IdP / passwordless provider factors |
| D-003 | **Hosting: machineaid product** | 2026-08-12 | machineaid operates staging/prod; Hypeluxe is a tenant/customer |

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
| O-009 | `allowLogisticsBeforeRsvp` | yes / no |
| O-010 | `requireFlightLinkForAirportPickup` | yes / no |
| O-011 | Drawdown funds | `recorded` / `cleared` |
| O-012 | Drawdown cost basis | `committed` / `actual` |
| O-013 | Auto-draft deposit invoice | yes / no |
| O-014 | Coordinator sees money | yes / no |
| O-015 | `StartMovement` actor | planner/coordinator / add driver role |
| O-016 | Multi-currency | single currency per event (pilot) / FX phase-2 |

## Doc follow-ups queued after opens resolve

- Acceptance journeys + smoke list
- Hypeluxe fixture pack outline
- Security one-pager expanded with retention defaults once Hypeluxe agrees
