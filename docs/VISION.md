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
5. **Explicit state machines** — Status changes go through commands with legal transitions (see state machines).
6. **Audit & attribution** — Meaningful actions record who/when/what, including offline actors (see audit doc).
7. **Local-first UX** — Reads and writes hit a local store first; sync is background. Venue Wi‑Fi and travel days are unreliable; the product must still work.
8. **EffectTS everywhere it matters** — Typed errors, composable services, and swappable Layers for local DB, sync, and auth.
9. **Continuous delivery** — Main always produces a deployable artifact; staging smoke tests include offline → online sync.
10. **Multi-tenant by design** — Hypeluxe is customer #1, not the product name. Branding, roles, and event-type packs are tenant-scoped.

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
- Document vault (passports, contracts) and built-in email/SMS blasts
- Replacing dedicated CRM systems
- Real-time video or chat as a primary feature

## Success criteria (platform)

- Create and edit an event **entirely offline**, then sync without data loss
- Model **single-day** and **multi-day** schedules without schema hacks
- Run **destination** and **conference** style events with guest travel, accommodations, and movements on the same core model
- Track **budget vs costs**, **client invoices (A/R)**, **payments**, and **drawdowns** under cost-plus (or flat-fee) terms
- Illegal status transitions are rejected; successful commands are **attributable** in an activity trail
- Onboard **Hypeluxe** as a tenant with destination-wedding templates without changing core tables
- Ship to staging via CD on every main merge
