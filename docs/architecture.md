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
## EffectTS spine

Use Effect on client and server for:

| Concern | Approach |
|---|---|
| Domain use cases | Effect programs with typed errors (`Unauthorized`, `IllegalTransition`, `Conflict`, `Validation`, `SyncConflict`) |
| Dependencies | Service interfaces (`EventRepo`, `GuestRepo`, `LogisticsRepo`, `VendorRepo`, `Audit`, `Auth`, `Authz`, `Clock`) provided via Layers |
| Schema | Shared Effect Schema for Event, Schedule, Guest, Party, TravelLeg, Stay, Movement, Vendor, AuditRecord, etc. |
| State changes | Command handlers enforce FSMs ([state-machines.md](./state-machines.md)); no raw status writes |
| Audit | `Audit.append` on successful commands + sync rejections ([audit-and-attribution.md](./audit-and-attribution.md)) |
| Config | Effect Config for env (no scattered `process.env` reads) |
| HTTP / RPC | `@effect/platform` (or Effect RPC) composing the same domain layers |

UI components call Effect services only — no ad-hoc `fetch` for domain writes.

## Local-first data plane

**Default recommendation:** SQL-oriented sync (e.g. ElectricSQL or PowerSync) with client SQLite/IndexedDB and Postgres as durable source of truth — good fit for structured events, guests, parties, travel/stay/movement rows, audit append, vendors, and assignments.

**Hybrid for collaboration:** notes/checklists may use CRDT (Automerge/Loro) or explicit LWW + history if multi-device concurrent editing is required.

Rules:

- All primary reads/writes go to the **local** store first
- Sync runs as background Effect fibers with visible pending/conflict/synced state
- Conflict policy decided per aggregate (structured fields vs notes)

Decision still open: pure SQL-sync vs CRDT vs hybrid — lock before implementation spike.

## Auth & tenancy

- Cloud issues sessions or short-lived tokens + refresh
- Authorization is workspace-scoped RBAC with command-level checks ([identity-and-access.md](./identity-and-access.md))
- Offline: cached membership/capabilities; server re-validates on sync
- Tenant packs (Hypeluxe) supply templates and branding, not separate databases unless scale demands it later

## Continuous delivery

1. **PR:** typecheck, unit tests (Effect + test Layers + FSM transition tables), schema migration dry-run, Playwright smoke
2. **Main → staging:** deploy immutable artifact; smoke includes offline create → online sync + audit actor preserved
3. **Prod:** promote the same artifact; feature-flag sync protocol / schema generation
4. **Migrations:** versioned; clients tolerate N−1 local schema

## Proposed monorepo layout (machineaid repo)

```text
packages/
  domain/          # Effect schemas, FSMs, pure use cases, Audit service interface
  client/          # UI + local DB + sync adapter
  server/          # Auth, authz, sync, audit persistence, admin APIs
  sync/            # Shared sync protocol types + correlation ids
  tenant-hypeluxe/ # Templates, copy, default packages
```

## Explicit non-goals for architecture v1

- Sharing business logic by wrapping the existing Mongoose controllers
- Treating REST CRUD as the primary client data path
- Wedding-specific tables in the core schema
- Payments ledger, document vault, or messaging platform
