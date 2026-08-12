# Product vision

## One-liner

An **event-agnostic operations platform** for planners, producers, and agencies: plan and run events offline-first, sync across devices, and specialize by customer vertical (starting with Hypeluxe) without forking the core.

## Why not “wedding software”

Weddings are a common and valuable use case, but the operating loop is the same for many event types:

- A **workspace** (agency / producer / planner) owns work
- An **event** has stakeholders, vendors, tasks, and notes
- The **schedule** may be a single day or span multiple days/sessions
- **Vendors** and **packages/offerings** are reusable across events

Hard-coding “wedding date” and “couple” as the center of the model blocks festivals, corporate offsities, multi-day celebrations, galas, and pop-ups. Those should be **templates and tenant config**, not separate products.

## Principles

1. **Event-agnostic core** — Domain nouns are workspace, event, schedule, vendor, task, note, stakeholder. Verticals add templates and language, not parallel schemas.
2. **Schedule is first-class** — Single-day and multi-day are equal citizens (see domain model).
3. **Local-first UX** — Reads and writes hit a local store first; sync is background. Venue Wi‑Fi is unreliable; the product must still work.
4. **EffectTS everywhere it matters** — Typed errors, composable services, and swappable Layers for local DB, sync, and auth.
5. **Continuous delivery** — Main always produces a deployable artifact; staging smoke tests include offline → online sync.
6. **Multi-tenant by design** — Hypeluxe is customer #1, not the product name. Branding, roles, and event-type packs are tenant-scoped.

## Who it’s for

| Actor | Needs |
|---|---|
| Agency / planner | Portfolio of events, vendor network, checklists, notes |
| Event stakeholders (couple, host, client org) | Shared view of their event, limited edit rights |
| Vendor (later) | Assignments, contacts, schedules for their jobs |

## Non-goals (near term)

- Full public guest RSVP / ticketing marketplace
- Replacing dedicated accounting or CRM systems
- Real-time video or chat as a primary feature

## Success criteria (platform)

- Create and edit an event **entirely offline**, then sync without data loss
- Model **single-day** and **multi-day** schedules without schema hacks
- Onboard **Hypeluxe** as a tenant with wedding/luxury templates without changing core tables
- Ship to staging via CD on every main merge
