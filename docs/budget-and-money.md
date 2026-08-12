# Budget, cost-plus, invoicing, payments, and drawdowns

Events are commercial as well as operational. Many agencies (including destination / luxury work) run **at cost-plus**: track actual (or committed) costs, apply a fee/markup, **invoice** the client, collect payment, and **draw down** against funds received.

This is **in scope** for the platform domain. It is **not** a full accounting/ERP replacement (GL, tax engine, payroll).

## Goals

1. Per-event **budget** (estimated → revised → locked) with line items
2. **Cost tracking** (estimated vs committed vs actual) tied to vendors/categories
3. **Cost-plus commercial terms** on the event (markup %, flat producer fee, hybrid)
4. **Client invoices** issued to the payer (deposit, progress, final / reconciliation)
5. **Client payments** received and **applied** to invoices
6. **Drawdowns** — allocating received client funds against costs / fee
7. Planner visibility: billed vs collected, A/R, remaining client balance, burn vs budget

## Non-goals (still deferred)

- General ledger / double-entry books as a product
- Tax filing engines / e-invoicing legal networks (store tax flags & totals; don’t become a tax authority)
- Multi-currency hedging, payroll
- Replacing the agency’s accountant or Xero/QuickBooks (export/sync later is fine)
- Automated card charging / payment gateway as a must-have for v1 (record payment + apply to invoice first; gateway optional later)
- Fancy PDF brand studio (structured invoice + simple printable/PDF export is enough for pilot)

## Two directions of “invoice”

| Direction | Meaning | Model |
|---|---|---|
| **Vendor → agency** | Supplier bills you | Captured on **Cost** (`invoiced` / `paid`, `confirmationRef`) |
| **Agency → client** | You bill the client | First-class **ClientInvoice** |

Do not overload one table for both.

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

## Costs (spend) — vendor side

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
  actual?: Money             // vendor-invoice / receipted
  status: "estimated" | "committed" | "invoiced" | "paid" | "cancelled"
  confirmationRef?: string   // vendor invoice #
  incurredOn?: Date
}
```

`Cost.status = invoiced` means **the vendor invoiced the agency**, not that the client was billed.

## Client invoices (agency → client)

```ts
type ClientInvoice = {
  id: string
  eventId: string
  billToStakeholderId: string
  number: string                 // human-visible, workspace-unique
  kind: "deposit" | "progress" | "final" | "retainer" | "adjustment" | "other"
  status: "draft" | "issued" | "partially_paid" | "paid" | "void" | "written_off"
  currency: string
  issuedOn?: Date
  dueOn?: Date
  lineItems: InvoiceLine[]
  subtotal: Money
  taxTotal?: Money               // optional; rules tenant-local
  total: Money
  amountApplied: Money           // sum of payment applications
  amountDue: Money               // total − amountApplied (derived or stored)
  commercialSnapshotId?: string  // freeze terms/obligation basis when issued
  notes?: string
  pdfArtifactId?: string         // optional export blob
}

type InvoiceLine = {
  id: string
  label: string
  kind: "deposit" | "cost_pass_through" | "markup" | "fee" | "adjustment" | "other"
  quantity?: number
  unitAmount: Money
  amount: Money
  linkedCostId?: string          // traceability for pass-through / markup lines
  linkedBudgetLineId?: string
}
```

### How invoices get created

| Path | Command / flow |
|---|---|
| Manual | `DraftClientInvoice` — planner builds lines |
| Deposit schedule | From commercial/offering template (“40% deposit”) |
| Progress | Planner selects costs + fee portion → lines |
| Final / reconciliation | `GenerateFinalInvoice` from cost-plus obligation snapshot − prior issued totals |

**Issuing** (`IssueClientInvoice`) freezes totals (and usually attaches `commercialSnapshotId`). Edits after issue require `void` + new invoice or an `adjustment` invoice — don’t silently mutate issued totals.

### Payments apply to invoices

```ts
type PaymentApplication = {
  id: string
  clientPaymentId: string
  clientInvoiceId: string
  amount: Money
}
```

- A `ClientPayment` may apply to one or more invoices (split rare but allowed).
- Unapplied payment amount = payment.amount − sum(applications) (overpayment / retainer float).
- Invoice `amountApplied` / status (`partially_paid` / `paid`) update when applications post.
- Drawdowns still allocate **cash pool** to costs; invoicing is the **A/R / billing** layer. Both coexist.

### Mental model (billing + cash)

```text
Costs (vendor bills you)
   │
   ▼
Cost-plus obligation / fee ──► ClientInvoice (issued to client)
                                   ▲
ClientPayment (in) ──► PaymentApplication ──┘
         │
         ▼
   available funds pool
         │
         ▼
      Drawdown ──► Cost(s) and/or Fee

A/R open     ≈ sum(issued invoice totals) − sum(applications to those invoices)
Cash on hand ≈ cleared payments − posted drawdowns
```

Planners care about both **what we’ve billed** and **what we’ve collected / drawn**.

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

v1: planners **record** payments and **apply** them to invoices. Payment-gateway capture is optional later — same stance as GDS/PMS: track references, don’t require owning the rails.

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
4. Sum(payment applications) for a payment ≤ payment.amount  
5. Sum(applications) to an invoice ≤ invoice.total (unless write-off / adjustment policy)  
6. Reversal: `void` invoice/drawdown; `reversed` payment; never delete audit history  

### Cost-plus billing math

```text
obligation ≈ eligible_costs_basis × (1 + markup%) + flat_fee
previously_billed ≈ sum(issued invoices − void)
to_bill_on_final ≈ obligation − previously_billed
balance_due_ar ≈ sum(invoice.amountDue)
cash_balance ≈ cleared_payments − posted_drawdowns
```

Basis timing (`committed` vs `actual`) is a tenant setting defaulting to `actual` when present else `committed`.

## Snapshots & audit

Changing markup % mid-event is dangerous. Rules:

- `PublishCommercialTerms` / `ReviseCommercialTerms` are commands; revisions are audited
- Prefer `CommercialSnapshot` when **issuing** invoices (especially progress/final) and when locking budget
- All invoice/payment/drawdown/cost status changes go through FSMs and **audit** with actor stamps (including offline)

## Permissions

| Action | Minimum role (pilot) |
|---|---|
| Edit budget lines / terms | `planner` |
| Lock budget | `planner` |
| Draft / issue / void client invoice | `planner` |
| Record client payment / apply to invoice | `planner` |
| Post drawdown | `planner` |
| Void drawdown / reverse payment | `owner` or `planner` |
| View money / invoices | `planner`, `owner`; `host` may see **their** invoices + balance (tenant policy) |
| `viewer` / `coordinator` | No money writes; coordinator **read** optional (default deny for amounts) |

Day-of coordinators often should **not** see full cost-plus margins — tenant policy.

## Relationship to “offering / package”

`Offering` remains the **sold package metadata** (what was quoted). Budget lines and deposit invoice schedules may be generated from a package template but remain editable. Invoices + payments + drawdowns are the **billing and cash** layer; offering is not a ledger.

## Export & delivery

- Structured invoice in DB is source of truth
- PDF/print export for email attachment (delivery channel may be external in v1)
- Optional later: accounting export (CSV / QuickBooks), payment links on issued invoices

## Hypeluxe

Destination weddings: deposit invoice → progress as vendors commit → final cost-plus reconciliation; large vendor costs; staged client payments applied to invoices; arrival-week drawdowns against hotels/transport. Conferences reuse the same invoice/payment/drawdown machinery with different categories.

## Implementation phasing

| Slice | Scope |
|---|---|
| M-money-1 | Schema: Budget, Cost, ClientInvoice, PaymentApplication, ClientPayment, Drawdown + FSMs + audit |
| M-money-2 | UI: budget sheet, draft/issue invoice, record+apply payment, post drawdown, A/R + cash widgets |
| M-money-3 | Generate final/reconciliation invoice from cost-plus snapshot; simple PDF export |
| Later | Payment gateway, accounting export, multi-currency conversion, e-invoicing networks |

Local-first: money rows sync like other structured data; void/reverse/issue must be idempotent under `correlationId`.
