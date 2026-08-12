# Event Platform (working docs)

This repository currently contains **TwoBeWed**, an early MEAN-stack prototype for wedding consulting. It is being used as a staging ground for product and architecture documentation toward a broader **event-agnostic** operations platform.

**Intended home:** a new private repository under the [`machineaid`](https://github.com/machineaid) GitHub organization (not creatable from this agent’s GitHub token). Until that repo exists, documentation lives here as PRs.

## Start here

| If you want… | Read |
|---|---|
| What we ship first | [docs/v1-slice.md](docs/v1-slice.md) |
| What’s decided vs open | [docs/decisions.md](docs/decisions.md) |
| Why the approach was compressed | [docs/design-revisions.md](docs/design-revisions.md) |
| Domain / money / FSMs (reference) | domain-model, budget-and-money, state-machines |
| Next build phases | [docs/roadmap.md](docs/roadmap.md) |

**Next concrete step outside docs:** create the `machineaid` repo, import these docs, start Phase 1 schemas against the v1 slice. Still need stack calls (O-001…O-003) before Phase 2–3 spike.

## Current prototype (legacy)

- **Stack:** MongoDB, Express, AngularJS, Node.js
- **Domain (as built):** users (planners), clients (couples), vendors
- **Entry:** `server.js` → `/api/v1/` routes under `app/apis/`

See [docs/legacy-prototype.md](docs/legacy-prototype.md) for a short map of the existing code.

## Target direction

Build a fully functional, **event-agnostic** platform (single-day, multi-day, destination, conference) using:

- **EffectTS** as the application spine
- **Local-first** sync (offline-capable reads/writes)
- **Continuous delivery** of a single shippable artifact
- **Guest management + logistics** (travel, accommodations, pickups/dropoffs, local travel)
- **Budgets, client invoicing, cost-plus, payments, and drawdowns**
- **Hypeluxe** as the first customer / tenant use case (destination weddings as a vertical template, not the core domain)

| Doc | Purpose |
|---|---|
| [docs/VISION.md](docs/VISION.md) | Product vision and principles |
| [docs/domain-model.md](docs/domain-model.md) | Event-agnostic domain |
| [docs/budget-and-money.md](docs/budget-and-money.md) | Cost-plus, invoicing, payments, drawdowns |
| [docs/state-machines.md](docs/state-machines.md) | Status FSMs and commands |
| [docs/audit-and-attribution.md](docs/audit-and-attribution.md) | Who/when/what audit trail |
| [docs/identity-and-access.md](docs/identity-and-access.md) | Users, parties, RBAC |
| [docs/architecture.md](docs/architecture.md) | EffectTS, local-first, CD summary |
| [docs/platform-engineering.md](docs/platform-engineering.md) | Policy as data, core/shell, CD, observability |
| [docs/security-and-compliance.md](docs/security-and-compliance.md) | PIPEDA/GDPR, SSO/passwordless, machineaid hosting |
| [docs/decisions.md](docs/decisions.md) | Decided vs open calls |
| [docs/v1-slice.md](docs/v1-slice.md) | What ships in v1 (cut line) |
| [docs/design-revisions.md](docs/design-revisions.md) | Muratori/Ousterhout-driven approach changes |
| [docs/customers/hypeluxe.md](docs/customers/hypeluxe.md) | First customer use case |
| [docs/roadmap.md](docs/roadmap.md) | Incremental milestones |

## Status

Documentation only. Implementation of the new platform should land in the machineaid repository once it is created.
