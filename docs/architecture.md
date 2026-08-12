# Architecture

## Context

Rebuild (do not retrofit) on top of the TwoBeWed prototype. The legacy Express + Mongoose + AngularJS stack does not match EffectTS, local-first sync, or continuous delivery goals.

## System sketch

```text
┌─────────────────────────────────────────┐
│  Client (React/Solid + Effect services) │
│    UI → Domain services → Local DB      │
│              ↕ sync fibers               │
└──────────────────┬──────────────────────┘
                   │ auth + sync protocol
┌──────────────────▼──────────────────────┐
│  Cloud (Effect Platform / RPC)          │
│    Auth · Authz · Sync · Audit · Mig.   │
│              ↕                           │
│         Postgres (source of truth)      │
└─────────────────────────────────────────┘
```

Domain commands run through **state machines** and append **audit** records (actor stamped on device for offline).

**Also see:** [platform-engineering.md](./platform-engineering.md) — policy as data, functional core / imperative shell, CD, observability.  
**v1 execution cut line:** [v1-slice.md](./v1-slice.md) · [design-revisions.md](./design-revisions.md)

## Functional core / imperative shell (summary)

| Layer | Responsibility |
|---|---|
| **Core** (`packages/domain`) | Schemas, FSM **decision functions**, money math, flat `ResolvedPolicy` — no SQL/HTTP/UI imports |
| **Shells** (`client`, `server`, future integrations) | I/O via Effect Layers; **`runCommand`** = authz → decide → persist → audit → (optional) span |

UI and (later) Stripe/QuickBooks adapters are shells that **only** call core commands / money facades (`Billing`, `ClientFunds`, `DayOfCash`).

## Policy as data (summary)

**v1:** Hypeluxe ships a **flat** `ResolvedPolicy` (D-004…D-010) — not a multi-scope merge engine. Generalize when a second tenant needs different knobs. Details: [v1-slice.md](./v1-slice.md), [platform-engineering.md](./platform-engineering.md#policy-as-data).
## EffectTS spine

Use Effect on client and server for:

| Concern | Approach |
|---|---|
| Domain use cases | Effect programs with typed errors (`Unauthorized`, `IllegalTransition`, `Conflict`, `Validation`, `SyncConflict`) |
| Dependencies | Service interfaces (`EventRepo`, `GuestRepo`, `LogisticsRepo`, `BudgetRepo`, `PaymentRepo`, `VendorRepo`, `Audit`, `Auth`, `Authz`, `Clock`) provided via Layers |
| Schema | Shared Effect Schema for Event, Schedule, Guest, Party, TravelLeg, Stay, Movement, Budget, Cost, ClientInvoice, ClientPayment, Drawdown, Vendor, AuditRecord, etc. |
| State changes | Command handlers enforce FSMs ([state-machines.md](./state-machines.md)); no raw status writes |
| Audit | `Audit.append` on successful commands + sync rejections ([audit-and-attribution.md](./audit-and-attribution.md)) |
| Config | Effect Config for env (no scattered `process.env` reads) |
| HTTP / RPC | `@effect/platform` (or Effect RPC) composing the same domain layers |

UI components call Effect services only — no ad-hoc `fetch` for domain writes.

## Local-first data plane

**Ops (local-first):** guests, logistics, day-of cash — read/write local DB; sync as Effect fibers.

**Money (online-authoritative in v1):** Billing + ClientFunds (invoices, clear payment, **drawdowns**, budget edits) apply on the server when online. Do not pretend offline-safe distributed money in v1. Optional stale read-only caches must be labeled.

**Default sync recommendation for ops:** SQL-oriented sync (e.g. ElectricSQL or PowerSync) with client SQLite/IndexedDB and Postgres as durable source of truth.

**Hybrid for collaboration (backlog):** notes/checklists CRDT — deferred per [v1-slice.md](./v1-slice.md).

Rules:

- Ops primary reads/writes hit the **local** store first  
- Money commands use online apply (block or explicit pending — prefer block in v1)  
- Sync conflicts on ops surface for resolution and are audited  

## Auth & tenancy

- **Authn:** SSO and/or passwordless out of the gate (no homegrown MFA) — [security-and-compliance.md](./security-and-compliance.md)
- Authorization is workspace-scoped RBAC with command-level checks ([identity-and-access.md](./identity-and-access.md))
- Offline: cached membership/capabilities; server re-validates on sync
- Tenant packs (Hypeluxe) supply templates, branding, and default **policy data**
- **Hosting:** machineaid multi-tenant SaaS (Hypeluxe is a customer tenant)

## Continuous delivery

See full pipeline in [platform-engineering.md](./platform-engineering.md#continuous-delivery).

1. **PR:** typecheck, unit tests (Effect + test Layers + FSM + policy fixtures), schema migration dry-run, Playwright smoke
2. **Main → staging:** deploy **immutable** artifact; smoke includes offline→online sync, manual invoice/payment, audit actor preserved
3. **Prod:** promote the **same** artifact; feature flags / policy for risky behavior
4. **Migrations:** versioned; clients tolerate N−1 local schema
5. **Gate:** no payment-processor auto-sync until manual money commands pass staging smoke

## Observability

Audit ≠ ops telemetry. Require OpenTelemetry-style **traces/metrics**, structured **logs** in shells, and `correlationId` linking them to audit. Details and SLO starters in [platform-engineering.md](./platform-engineering.md#observability).

## Proposed monorepo layout (machineaid repo)

```text
packages/
  domain/             # functional core: schemas, FSMs, commands, policy eval ports
  policy/             # PolicyBody schemas + pack defaults (or under domain/)
  client/             # UI + local DB + sync fiber (imperative shell)
  server/             # Auth, RPC, Postgres, audit persistence (imperative shell)
  sync/               # Shared sync protocol types + correlation ids
  observability/      # tracing/metrics helpers (optional package)
  tenant-hypeluxe/    # templates + default policy data (not forked core)
```

## Explicit non-goals for architecture v1

- Sharing business logic by wrapping the existing Mongoose controllers
- Treating REST CRUD as the primary client data path
- Wedding-specific tables in the core schema
- Payments ledger as full GL, document vault, or messaging platform
- Payment-processor or accounting **auto-sync** before manual domain commands are the supported path (future adapters must call the same Effect commands with a system actor)
