# Domain model (event-agnostic)

This document defines the target domain for the new platform. It deliberately generalizes the legacy TwoBeWed models (`user`, `client`, `vendor`).

**Related:** [state-machines.md](./state-machines.md) (legal status transitions), [identity-and-access.md](./identity-and-access.md) (users, parties, RBAC), [audit-and-attribution.md](./audit-and-attribution.md), [budget-and-money.md](./budget-and-money.md) (cost-plus, invoicing, payments, drawdowns).

## Core entities

```text
Workspace
  ├── Member[]  (User + role)
  ├── Vendor[]  (catalog)
  └── Event[]
        ├── Schedule (kind + units)
        ├── Stakeholder[]
        ├── Party[]
        ├── Guest[]
        ├── TravelLeg[]              // first-class rows (not only nested JSON)
        ├── Stay[]
        ├── AccommodationBlock[]
        ├── Movement[]               // arrival | departure | local
        ├── TransferPlan[]           // optional grouping of movements
        ├── Budget + BudgetLine[]
        ├── Cost[]                   // vendor-side spend
        ├── ClientInvoice[]          // agency → client billing
        ├── ClientPayment[]
        ├── PaymentApplication[]     // payment → invoice
        ├── Drawdown[]
        ├── CommercialTerms (versioned)
        ├── VendorAssignment[]
        ├── Offering / Package (optional)
        ├── Task[] / Checklist
        └── Note[]
```

`LogisticsProfile` in earlier drafts meant “the set of logistics rows for a guest,” not a nested document blob. **Source of truth is relational:** `TravelLeg`, `Stay`, `Movement` reference `guestId` / `guestIds` / `partyId`.

### Workspace

Tenant boundary for an agency, producer, or brand (e.g. Hypeluxe). Holds members, roles, vendor catalog, enabled event-type templates, and **policy documents** (see [platform-engineering.md](./platform-engineering.md#policy-as-data)).

### Event

A planned occasion owned by a workspace. Not inherently a wedding.

| Field (conceptual) | Notes |
|---|---|
| `id`, `workspaceId` | Identity |
| `title` | Display name |
| `eventTypeId` | References a template (destination wedding, multi-day conference, gala, …) |
| `status` | FSM: `draft` → `active` → `completed` / `archived` — see state machines |
| `schedule` | See below |
| `destination` | Optional **trip destination** (city/region/country + default timezone). Used for destination weddings/conferences |
| `offering` | Selected package + extras (quoted shape; **not** the ledger) |
| `commercial` | Cost-plus / fee terms — see [budget-and-money.md](./budget-and-money.md) |
| `budget`, `costs` | Plan vs vendor spend |
| `clientInvoices`, `clientPayments`, `paymentApplications` | Billing (A/R) + cash in |
| `drawdowns` | Allocate collected funds to costs/fee |
| `stakeholders` | Decision-makers / hosts (not the full guest list); payers often stakeholders |
| `parties`, `guests` | Attendance + hospitality |
| `accommodationBlocks`, `stays`, `travelLegs`, `movements` | Logistics |
| `transferPlans` | Optional waves grouping movements |
| `vendorAssignments` | Links to catalog vendors |
| `tasks`, `notes` | Operational content |

**Location model (single story):**

| Concept | Role |
|---|---|
| `Event.destination` | Where the *trip* is (macro). Optional for local single-venue events |
| `ScheduleUnit.venue` | Where a *day/session* happens |
| `TimeBlock.venue` (optional override) | Where a *block* happens if different from the unit |
| Movement `from` / `to` | Where people *move* (airport, hotel, venue refs) |

There is **no** separate legacy `location` summary field — derive display summaries from `destination` + primary venue.

**Representative event types (templates):** destination wedding, local wedding weekend, multi-day conference, corporate offsite, private gala.

### Schedule

**Schedule is not a single date field.**

```ts
type ScheduleKind = "single_day" | "multi_day"

type Schedule = {
  kind: ScheduleKind
  timezone: string          // usually aligns with destination timezone when set
  units: ScheduleUnit[]     // ordered
}

type ScheduleUnit = {
  id: string
  label: string             // "Main day", "Welcome dinner", "Day 2", "Keynotes"
  date: string              // calendar date in schedule timezone
  venue?: VenueRef
  blocks?: TimeBlock[]      // ceremony, session track, breakout, dinner, …
}
```

| Kind | Shape | Examples |
|---|---|---|
| `single_day` | Typically one `ScheduleUnit` (optionally many time blocks) | Local ceremony + reception |
| `multi_day` | Two or more ordered `ScheduleUnit`s | Destination wedding weekend; 3-day conference |

Future kinds (not required for v1): recurring series, open-ended / TBA dates with placeholders.

### Stakeholder vs Guest

| | Stakeholder | Guest |
|---|---|---|
| Who | Hosts, couple, sponsor, conference chair, on-site lead | Invitees / attendees / +1s |
| Purpose | Ownership, approvals, commercial relationship | Attendance, hospitality, logistics |
| Cardinality | Few | Often dozens to hundreds+ |

Roles/labels are configurable per event type (e.g. “bride/groom” vs “delegate/speaker”).

### Party

Household or travelling group. See [identity-and-access.md](./identity-and-access.md).

- `primaryGuestId` + `guestIds`
- Stays/movements may target a party (expanded to members)

### Guest management

Guests are first-class event records—not a bolt-on spreadsheet.

| Field (conceptual) | Notes |
|---|---|
| Identity | Name, contact, dietary, accessibility |
| `partyId?` | Optional party membership |
| Invitation / RSVP | FSM: `invited` \| `attending` \| `declined` \| `waitlist` |
| Tags | VIP, speaker, vendor-as-guest, staff |
| `userId?` | Optional link when guest portal exists |

**Derived gap flags (queries, not stored statuses):** missing inbound travel, no stay, unassigned arrival pickup, etc. — typically only for `attending` guests.

### Logistics

The platform **tracks and coordinates** bookings and confirmations; it does **not** replace airline GDSs or hotel PMS inventory systems.

#### TravelLeg

Inbound/outbound long-haul (or intercity) legs. Status FSM includes `planned`, `booked`, `checked_in`, `delayed`, `completed`, `cancelled`.

#### AccommodationBlock + Stay

- **AccommodationBlock** — event-level hotel/villa room block: vendor link, date range, **allotment** (rooms reserved), optional overbook policy.
- **Stay** — guest/party room assignment within a block or ad-hoc property. Status FSM includes `no_show`.

Allotment math: confirmed stays consume block capacity; waitlist/overbook behavior is tenant policy.

#### Movement (unified transfer + local)

Replaces separate Transfer / LocalMove types.

```ts
type Movement = {
  id: string
  eventId: string
  scope: "arrival" | "departure" | "local"
  kind?: "pickup" | "dropoff" | "point_to_point" | "shuttle"  // UI hint
  guestIds: string[]
  partyId?: string
  from: LocationRef
  to: LocationRef
  windowStart: DateTime
  windowEnd?: DateTime
  vehicleOrVendorId?: string
  linkedTravelLegId?: string
  transferPlanId?: string
  status: "planned" | "assigned" | "en_route" | "completed" | "no_show" | "cancelled"
}
```

**TransferPlan** groups movements into waves (e.g. “Thursday AM arrivals”) for dispatcher views.

### Vendor + VendorAssignment

Workspace catalog (venues, hotels, transport, DMC, AV, catering, …). Assignments have their own FSM (`proposed` → `confirmed` → `completed` / `cancelled`).

### Offering / package

Commercial **quote metadata** attached to an event (what was sold). Cash movement is **ClientPayment** + **Drawdown**; spend is **Cost**; plan is **Budget**. See [budget-and-money.md](./budget-and-money.md).

### Budget, costs, invoices, payments, drawdowns

First-class for cost-plus and retainer-style events:

- **Budget / BudgetLine** — planned amounts by category  
- **Cost** — vendor spend (`invoiced` = vendor billed *us*)  
- **ClientInvoice** — we bill the client (deposit / progress / final); lines may pass through costs + markup + fee  
- **ClientPayment** + **PaymentApplication** — funds in, applied to invoices  
- **Drawdown** — allocate cleared funds against costs and/or fee  
- **CommercialTerms** — cost-plus %, flat fee, hybrid  

Not a full accounting suite (no GL). Simple PDF/export for invoices; payment gateways optional later.

### Task & note

- **Task** — FSM: `open` → `in_progress` → `done` / `cancelled`
- **Note** — freeform; collaborative editing may use CRDT; audit as revision commits, not keystrokes

### Documents & communications (deferred)

Passport scans, contracts, email/SMS invite blasts — **out of v1** unless Hypeluxe pulls them forward. RSVP may be planner-entered. (Money/budget is **not** deferred — see above.)

## Mapping from legacy TwoBeWed

| Legacy | Target |
|---|---|
| `user` | User + WorkspaceMember |
| `client` (+ `weddingDate`) | `Event` + `Schedule` + stakeholders |
| `vendor` | `Vendor` + assignments |
| _(none)_ | Party, Guest, TravelLeg, Stay, Movement, Budget, Cost, ClientInvoice, ClientPayment, Drawdown, AuditRecord |
| `@twobewed.com` email rule | Tenant auth policy (not global) |

## Invariants

1. Every event belongs to exactly one workspace.
2. Every event has a schedule with `kind`; once dates are set, ≥1 unit (drafts may allow TBA with explicit flag).
3. Guests belong to exactly one event; a guest is in at most one party (v1).
4. Logistics rows reference the event and guest(s)/party; confirmation codes are stored on legs/stays when known.
5. Status changes only via documented state-machine commands.
6. Vertical language lives in templates/UI copy, not required core fields.
7. Meaningful commands append audit records with offline-safe actor stamps.
8. Posted drawdowns cannot exceed available client funds (cleared payments − posted drawdowns) without an explicit override; allocation sums must equal drawdown amount.
9. Payment applications to an invoice cannot exceed invoice total (except write-off policy); applications for a payment cannot exceed payment amount.
10. Issued client invoice totals are immutable; corrections via void + replacement or adjustment invoice.
