# Budget, cost-plus, payments, and drawdowns

Events are commercial as well as operational. Many agencies (including destination / luxury work) run **at cost-plus**: track actual (or committed) costs, apply a fee/markup, bill or collect from the client, and **draw down** against funds received.

This is **in scope** for the platform domain. It is **not** a full accounting/ERP replacement (GL, tax engine, payroll).

## Goals

1. Per-event **budget** (estimated → revised → locked) with line items
2. **Cost tracking** (estimated vs committed vs actual) tied to vendors/categories
3. **Cost-plus commercial terms** on the event (markup %, flat producer fee, hybrid)
4. **Client payments** received (deposits, progress, final) with attribution
5. **Drawdowns** — allocating received client funds against costs / fee
6. Planner visibility: remaining client balance, burn vs budget, unallocated payments

## Non-goals (still deferred)

- General ledger / double-entry books as a product
- Tax filing, multi-currency hedging, payroll
- Replacing the agency’s accountant or Xero/QuickBooks (export/sync later is fine)
- Automated card charging / payment gateway as a must-have for v1 (manual “payment recorded” is enough to start; gateway optional later)

## Commercial model on Event

```ts
type CostPlusTerms = {
  mode: "cost_plus_percent" | "flat_fee" | "hybrid"
  markupPercent?: number     // e.g. 15 on eligible costs
  flatFee?: Money            // producer/planner fee
  feeTaxable?: boolean       // tenant policy; store flag only
  currency: string           // ISO currency for the event
  notes?: string
}

type EventCommercial = {
  terms: CostPlusTerms
  budgetId: string
}
```

**Cost-plus percent:** client obligation ≈ eligible actual/committed costs × (1 + markup%) + other fixed fees.  
**Flat fee:** costs passed through (or absorbed per policy) + agreed fee.  
**Hybrid:** markup on some categories + flat fee.

Eligibility of which cost categories receive markup is tenant/event config (`markupEligible: boolean` on category or line).

## Budget

```ts
type Budget = {
  id: string
  eventId: string
  status: "draft" | "active" | "locked" | "archived"  // FSM
  currency: string
  version: number            // revisions bump version; lock freezes for ops
}

type BudgetLine = {
  id: string
  budgetId: string
  categoryId: string         // venue, lodging, transport, F&B, fee, contingency, …
  label: string
  estimatedAmount: Money
  linkedVendorId?: string
  linkedOfferingId?: string
  markupEligible: boolean
  notes?: string
}
```

Budget is the **plan**. Costs are the **spend**. They relate by category (and optional line link) but costs can exist without a perfect 1:1 budget line (with a warning in UI).

## Costs (spend)

```ts
type Cost = {
  id: string
  eventId: string
  categoryId: string
  budgetLineId?: string
  vendorId?: string
  label: string
  estimated?: Money
  committed?: Money          // PO / contract
  actual?: Money             // invoice / receipted
  status: "estimated" | "committed" | "invoiced" | "paid" | "cancelled"
  confirmationRef?: string   // vendor invoice #, etc.
  incurredOn?: Date
}
```

Rollups:

- **Budget remaining** ≈ sum(budget lines) − sum(relevant costs)
- **Cost-plus client obligation (estimate)** ≈ f(terms, costs)
- Prefer explicit snapshot fields when presenting “amount due” so markup rule changes don’t silently rewrite history (see snapshots below)

## Client payments

Money **received from the client** (stakeholder payer), tracked on the event — not guest incidental payments (those can be a later type).

```ts
type ClientPayment = {
  id: string
  eventId: string
  payerStakeholderId?: string
  amount: Money
  method?: "wire" | "cheque" | "card" | "cash" | "other"
  receivedOn: Date
  status: "recorded" | "cleared" | "reversed"
  externalRef?: string       // bank ref / Stripe id later
  notes?: string
}
```

v1: planners **record** payments (confirmation-style tracking). Payment-gateway capture is optional later — same stance as GDS/PMS: track references, don’t require owning the rails.

## Drawdowns

A **drawdown** allocates client funds against costs and/or fee — the ops meaning of “we used $X of the client’s money for Y.”

```ts
type Drawdown = {
  id: string
  eventId: string
  amount: Money
  status: "pending" | "posted" | "void"
  postedOn?: Date
  allocations: DrawdownAllocation[]
  notes?: string
}

type DrawdownAllocation = {
  kind: "cost" | "fee" | "retainer_hold"
  costId?: string            // when kind = cost
  amount: Money
  label?: string             // when kind = fee / hold
}
```

**Invariants (v1):**

1. Sum(allocations.amount) = drawdown.amount  
2. Posted drawdowns against costs cannot exceed that cost’s `actual`/`committed` (policy pick) without an override role  
3. Sum(posted drawdowns) cannot exceed sum(cleared client payments) without override (`owner`/`planner`)  
4. Reversal: `void` a drawdown (and optionally `reversed` a payment); never delete audit history  

### Mental model

```text
ClientPayment (in) ──► pool of available funds
                         │
                         ▼
                      Drawdown ──► Cost(s) and/or Fee
                         │
                         ▼
              Remaining client balance =
                cleared payments − posted drawdowns
```

**Cost-plus billing view** (reporting, not necessarily auto-invoice):

```text
obligation ≈ eligible_costs_basis × (1 + markup%) + flat_fee
balance_due ≈ obligation − cleared_payments
```

Basis timing (`committed` vs `actual`) is a tenant setting defaulting to `actual` when present else `committed`.

## Snapshots & audit

Changing markup % mid-event is dangerous. Rules:

- `PublishCommercialTerms` / `ReviseCommercialTerms` are commands; revisions are audited
- Optional `CommercialSnapshot` when locking budget or posting a large drawdown wave (freeze the terms + obligation used for that period)
- All payment/drawdown/cost status changes go through FSMs and **audit** with actor stamps (including offline)

## Permissions

| Action | Minimum role (pilot) |
|---|---|
| Edit budget lines / terms | `planner` |
| Lock budget | `planner` |
| Record client payment | `planner` |
| Post drawdown | `planner` |
| Void drawdown / reverse payment | `owner` or `planner` |
| View money | `planner`, `owner`; `host` optional per tenant (often summary only) |
| `viewer` / `coordinator` | No money writes; coordinator **read** optional (default deny for amounts) |

Day-of coordinators often should **not** see full cost-plus margins — tenant policy.

## Relationship to “offering / package”

`Offering` remains the **sold package metadata** (what was quoted). Budget lines may be generated from a package template but remain editable. Payments/drawdowns are the **cash movement** layer; offering is not a ledger.

## Hypeluxe

Destination weddings are classic cost-plus + large vendor costs + staged client deposits + arrival-week drawdowns against hotels/transport. Multi-day conferences may use the same machinery with different categories.

## Implementation phasing

| Slice | Scope |
|---|---|
| M-money-1 | Schema: Budget, BudgetLine, Cost, ClientPayment, Drawdown + FSMs + audit |
| M-money-2 | UI: budget sheet, record payment, post drawdown, balance widgets |
| M-money-3 | Cost-plus obligation report + commercial snapshots |
| Later | Payment gateway, accounting export, multi-currency conversion |

Local-first: money rows sync like other structured data; void/reverse must be idempotent under `correlationId`.
