# Product vision

## One-liner

An **event-agnostic operations platform** for planners, producers, and agencies: plan and run events offline-first, sync across devices, and specialize by customer vertical (starting with Hypeluxe) without forking the core.

## Why not “wedding software”

Weddings are a common and valuable use case, but the operating loop is the same for many event types:

- A **workspace** (agency / producer / planner) owns work
- An **event** has stakeholders, **guests**, vendors, tasks, and notes
- The **schedule** may be a single day or span multiple days/sessions
- **Guest logistics** (travel, stays, transfers) matter whenever people must arrive and move
- **Vendors** and **packages/offerings** are reusable across events

Hard-coding “wedding date” and “couple” as the center of the model blocks destination weekends, multi-day conferences, corporate offsities, festivals, galas, and pop-ups. Those should be **templates and tenant config**, not separate products.

**Example templates (not separate products):** destination weddings, multi-day conferences, local single-day ceremonies, executive offsities.

## Principles

1. **Event-agnostic core** — Domain nouns are workspace, event, schedule, guest, party, logistics (incl. movements), vendor, task, note, stakeholder. Verticals add templates and language, not parallel schemas.
2. **Schedule is first-class** — Single-day and multi-day are equal citizens (see domain model).
3. **Guests + logistics are first-class** — Attendance is not enough; arrival, lodging, pickups/dropoffs, and local travel are core ops data (see domain model).
4. **Budgets, invoicing & cost-plus are first-class** — Events carry budgets, costs, client invoices, payments, and drawdowns (see budget & money doc). Not a full accounting suite.
5. **Manual ops before automation** — Establish complete human-driven flows (record payment, apply to invoice, post drawdown, issue invoice) before payment gateways or accounting auto-sync. Automation must map onto those same commands — it does not invent a parallel path.
6. **Explicit state machines** — Status changes go through commands with legal transitions (see state machines).
7. **Audit & attribution** — Meaningful actions record who/when/what, including offline actors (see audit doc).
8. **Local-first UX** — Reads and writes hit a local store first; sync is background. Venue Wi‑Fi and travel days are unreliable; the product must still work.
9. **EffectTS everywhere it matters** — Typed errors, composable services, and swappable Layers for local DB, sync, and auth.
10. **Functional core / imperative shell** — Domain decisions in a pure core; I/O and frameworks only in shells ([platform-engineering.md](./platform-engineering.md)).
11. **Policy as data** — Tenant/workspace rules are versioned documents evaluated by the core, not hard-coded tenant branches.
12. **Continuous delivery** — Main always produces a deployable artifact; staging smoke must pass before prod promote.
13. **Observability** — Traces, metrics, structured logs, correlated with audit via `correlationId`.
14. **Multi-tenant by design** — Hypeluxe is customer #1, not the product name. Branding, roles, policy packs, and event-type templates are tenant-scoped. **machineaid** owns and hosts the product.
15. **SSO / passwordless auth** — Prefer IdP SSO and passwordless; do not roll custom MFA ([security-and-compliance.md](./security-and-compliance.md)).
16. **PIPEDA + GDPR** — Privacy and retention designed for both.

## Who it’s for

| Actor | Needs |
|---|---|
| Agency / planner / coordinator | Portfolio of events, guest lists, logistics boards, **budget/invoices/payments/drawdowns** (planner), vendors, checklists, activity trail |
| Event stakeholders (couple, host, client org) | Shared view of their event, guest progress, limited edit rights |
| Guests (later portal) | Their itinerary: flights/trains, hotel, pickup windows, local moves |
| Vendor (later) | Assignments, contacts, schedules for their jobs (incl. transport partners) |

## Non-goals (near term)

- Public ticketing marketplace / open ticket sales
- Replacing dedicated GDS airline booking or full PMS hotel inventory systems (we **track** bookings and **confirmation codes**; we do not book inventory as a carrier/hotel)
- Full accounting/ERP (GL, tax engine) — we **do** track event budgets, cost-plus terms, **client invoices**, payments, and drawdowns
- Payment gateway as a hard dependency (manual payment recording + apply to invoice first; gateway optional later)
- E-invoicing legal networks / tax authority integrations (store tax totals; don’t file)
- Auto-sync to accounting/payment systems before the manual command flows are proven
- Document vault (passports, contracts) and built-in email/SMS blasts
- Replacing dedicated CRM systems
- Real-time video or chat as a primary feature

## Success criteria (platform)

- Create and edit an event **entirely offline**, then sync without data loss
- Model **single-day** and **multi-day** schedules without schema hacks
- Run **destination** and **conference** style events with guest travel, accommodations, and movements on the same core model
- Track **budget vs costs**, **client invoices (A/R)**, **payments**, and **drawdowns** under cost-plus (or flat-fee) terms
- Illegal status transitions are rejected; successful commands are **attributable** in an activity trail
- Product rules are expressed as **policy data**; Hypeluxe is a pack, not a fork
- Core logic is testable without UI/DB; shells are replaceable Layers
- Ship to staging via CD on every main merge; smoke includes offline→online + manual billing path; basic observability on the command path
