# v1 slice (complexity budget)

North-star remains an event-agnostic platform. **v1 ships a deep Hypeluxe slice**, not the full taxonomy.

Guiding reviews: compress until Hypeluxe can run a destination weekend + bill/clear/draw + day-of float; generalize after evidence ([design pressure from Muratori/Ousterhout-style review](./design-revisions.md)).

## In v1

| Area | Included |
|---|---|
| Event + schedule | `single_day` / `multi_day` |
| Guests + parties + RSVP | Logistics only when `attending` (D-004) |
| TravelLeg, Stay, Movement | Airport arrival pickup requires flight link (D-005) |
| Vendors + assignments | Minimal FSM |
| **Billing** | ClientInvoice, Payment, PaymentApplication; auto-draft deposit (D-008); issue/record/clear manual |
| **ClientFunds** | Clear payment; **Drawdown** against cost `actual` using **cleared** funds only (D-006, D-007) — see why below |
| Budget + Costs | Plan vs spend; cost-plus terms for obligation/reporting |
| **DayOfCash** | Float, COD/tip payouts, EOD reconcile (D-010); coordinator access |
| Auth | SSO / passwordless; RBAC; PIPEDA+GDPR posture |
| Audit | Via `runCommand` wrapper |
| Local-first | **Ops** aggregates (guests/logistics/day-of cash) |
| Online-authoritative | **Billing + ClientFunds** (invoice/clear/drawdown/budget edits) |
| Policy | Fixed Hypeluxe `ResolvedPolicy` knobs (D-004…D-010), not a multi-scope engine |
| CD smoke | Three journeys below |

### Why drawdowns stay in v1

Planners use drawdowns to **control spend against money actually in hand** and to see whether **payment milestones** support ongoing vendor costs:

- Cannot draw more than **cleared** client payments (milestones acknowledged received)  
- Cannot draw more than cost **actual** (vendor invoice received)  
- Together with budget lines → “are we over plan?” and “have we collected enough to cover what we’re committing to pay?”  

Deposit invoicing alone is not enough for that control loop.

## Explicitly out of v1 (backlog)

| Item | Revisit when |
|---|---|
| `TransferPlan` entity | Filters on Movement by day insufficient |
| Note CRDT / collab notes | Plain notes insufficient |
| Guest portal | Hypeluxe asks for self-serve travel |
| Host margin visibility | After O-005; invoices-only is enough to start if needed |
| Generic policy merge (platform→event) | Second tenant needs different knobs |
| Payment gateway / accounting auto-sync | Manual flows proven |
| Full OTel SLO program | correlationId + structured logs first |
| Rich Task FSM | Checklist on template enough |
| Seating, document vault, email/SMS blasts | Product pull |
| Multi-currency FX | O-016 — default single currency/event |

## Money module facades

Shells/UI call these — not raw repos:

```text
Billing      → Draft/Issue invoice, Record/Apply payment
ClientFunds  → ClearPayment, PostDrawdown (budget/milestone control)
DayOfCash    → IssueFloat, RecordPayout, Reconcile
```

Coordinators: **DayOfCash only** (D-009/D-010). Planners/owners: all three.

## Offline boundary

| Offline OK | Require online (or explicit “pending online” queue — prefer block in v1) |
|---|---|
| Guests, RSVP, travel, stays, movements | Issue/void invoice |
| Day-of cash float/payouts/reconcile | Record/clear payment, apply to invoice |
| Read-only cached money summaries (optional, stale-labeled) | Post/void drawdown, budget lock, commercial terms revise |

## Fixed `ResolvedPolicy` (Hypeluxe pack)

No merge engine in v1 — one flat object:

```ts
type ResolvedPolicy = {
  allowLogisticsBeforeRsvp: false
  requireFlightLinkForAirportPickup: true
  paymentFundsAvailableAt: "cleared"
  drawdownBasis: "actual"
  autoDraftDepositFromOffering: true
  coordinatorCanReadMoney: false
  coordinatorCanManageDayOfCash: true
}
```

## Acceptance journeys (CD smoke)

1. **Arrival logistics** — invite → attending → travel leg → assign airport pickup (flight linked)  
2. **Milestone funds** — attach offering → auto-draft deposit → issue → record payment → **clear** → post drawdown against cost with `actual` → budget remaining visible  
3. **Day-of cash** — planner issues float → coordinator payouts → reconcile  

## Implementation order

1. Pure core: schemas, FSMs, `ResolvedPolicy`, money math, tests  
2. Local shell: guests/logistics/day-of cash  
3. Online shell: Billing + ClientFunds (incl. drawdowns)  
4. Sync for ops; money path online-authoritative  
5. CD smoke = three journeys  
6. Second tenant → consider generalizing policy  

## Complexity rule

New entity / FSM / policy knob requires an entry in [decisions.md](./decisions.md): **“Hypeluxe v1 fails without this?”** Default **no** → backlog.
