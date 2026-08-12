# Customer: Hypeluxe

## Role

**Hypeluxe is the first customer use case**, not the name of the platform. They exercise the event-agnostic core through a tenant pack focused on luxury and destination celebrations — including single-day, multi-day, and **destination** itineraries with full guest logistics.

Hypeluxe is a **customer tenant** on the machineaid-hosted product (not a separately hosted fork).

## What “tenant pack” means

| Deliverable | Examples for Hypeluxe |
|---|---|
| Branding | Logo, color tokens, email/product copy |
| Event type templates | Destination wedding, wedding weekend, private gala; sample **multi-day conference** template for platform validation |
| Default checklists | Vendor booking, room-block lock, arrival manifest, timeline lock, day-of run of show |
| Package catalog | Main offerings + extras Hypeluxe sells |
| Roles / language | Planner, couple, family liaison, VIP guest (labels only) |
| Default **policy** | Logistics-before-RSVP knobs, money visibility, drawdown basis — as data, not forks |

Core storage remains `Event` + `Schedule` + `Guest` + logistics + money + …; Hypeluxe configures how those feel in the product via templates **and policy documents**.

## Use cases to support first

1. **Single-day wedding** — one primary date, time blocks, vendors, on-site notes offline.
2. **Multi-day / destination wedding** — welcome → main day → farewell as schedule units; guests flying in; hotel room block; airport pickups/dropoffs; local shuttles between hotel and venues.
3. **Guest management** — invite/RSVP status, parties/+1s, tags (VIP), dietary/accessibility; planner filters for “missing inbound flight” / “no hotel” / “unassigned pickup”.
4. **Travel & stays** — track inbound/outbound legs and stays against an accommodation block (confirmations, not GDS booking).
5. **Transfers & local travel** — unified **movements** (`arrival` / `departure` / `local`); arrival pickups linked to flights; hotel↔venue shuttles.
6. **Multi-day conference (platform proof)** — multi-day schedule units (sessions), delegate guest list, conference hotels, airport transfers, hotel↔venue shuttles — same schemas as destination wedding logistics.
7. **Budget, invoicing & cost-plus** — event budget, vendor costs, client invoices (deposit/progress/final), payments applied to invoices, drawdowns against costs/fee; A/R and cash balance. Coordinators excluded from this board (D-009) except **day-of cash float** (D-010: issue float, COD/tip payouts, EOD reconcile).
8. **Planner portfolio** — list/filter events by status and date; day-of logistics boards offline-capable; activity trail for attribution.
9. **Vendor roster** — hotels, transport partners, venues reused across events.

## Success for Hypeluxe pilot

- Planner can run arrival day and day-of with poor connectivity and reconcile later
- Destination multi-day itineraries do not require fake “extra events”
- Guest travel / hotel / movement gaps are visible without leaving the event
- Conference-shaped logistics reuse the same guest + movement model
- Cost-plus events show budget vs spend, invoices issued, payments applied, drawdowns, A/R and cash balance
- Status changes follow FSMs; actions are attributable (who changed what)
- Switching UI language/templates does not require engineering changes to core schemas

## Out of scope for pilot

- Public couple-facing marketing site
- Open ticket sales / marketplace
- Automated airline or hotel **booking engines** (confirmation **tracking** is in scope)
- Full accounting/ERP (GL/tax filing networks) — event invoicing + money tracking **is** in scope
- Payment gateway required (manual payment recording + apply to invoice first)
- Fancy branded invoice designer (structured invoice + simple PDF/print yes)
- Automatic payment/accounting sync before manual billing flows are in daily use
- Document vault and email/SMS blast platform
- Full seating-chart product (guest list + parties first)
- Vendor self-serve portal (unless explicitly pulled forward)
