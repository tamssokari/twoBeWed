# Event Platform (working docs)

This repository currently contains **TwoBeWed**, an early MEAN-stack prototype for wedding consulting. It is being used as a staging ground for product and architecture documentation toward a broader **event-agnostic** operations platform.

**Intended home:** a new private repository under the [`machineaid`](https://github.com/machineaid) GitHub organization (not creatable from this agent’s GitHub token). Until that repo exists, documentation lives here as PRs.

## Current prototype (legacy)

- **Stack:** MongoDB, Express, AngularJS, Node.js
- **Domain (as built):** users (planners), clients (couples), vendors
- **Entry:** `server.js` → `/api/v1/` routes under `app/apis/`

See [docs/legacy-prototype.md](docs/legacy-prototype.md) for a short map of the existing code.

## Target direction

Build a fully functional, **event-agnostic** platform (single-day, multi-day, and related schedule shapes) using:

- **EffectTS** as the application spine
- **Local-first** sync (offline-capable reads/writes)
- **Continuous delivery** of a single shippable artifact
- **Hypeluxe** as the first customer / tenant use case (weddings and luxury events as a vertical template, not the core domain)

| Doc | Purpose |
|---|---|
| [docs/VISION.md](docs/VISION.md) | Product vision and principles |
| [docs/domain-model.md](docs/domain-model.md) | Event-agnostic domain |
| [docs/architecture.md](docs/architecture.md) | EffectTS, local-first, CD |
| [docs/customers/hypeluxe.md](docs/customers/hypeluxe.md) | First customer use case |
| [docs/roadmap.md](docs/roadmap.md) | Incremental milestones |

## Status

Documentation only. Implementation of the new platform should land in the machineaid repository once it is created.
