# Platform engineering

How we build and run the system: **policy as data**, **functional core / imperative shell**, **continuous delivery**, and **observability**. Complements [architecture.md](./architecture.md).

## Status vs intent

| Concern | In docs today? | Intent |
|---|---|---|
| Effect Layers / ports | Sketch | Yes — shell adapters behind interfaces |
| Continuous delivery | Thin sketch | Yes — deepen (below) |
| Policy as data | Scattered knobs only | Yes — first-class `Policy` documents |
| Functional core / imperative shell | Implied, not named | Yes — explicit package boundaries |
| Observability | Missing | Yes — traces, metrics, logs, audit bridge |

---

## Functional core, imperative shell

Gary Bernhardt’s split, mapped onto Effect:

```text
┌──────────────────────────────────────────────────────────┐
│ IMPERATIVE SHELLS (effects at the edge)                  │
│  client UI · HTTP/RPC · SQLite/Postgres · sync transport │
│  clock/random · file/PDF · (later) Stripe/QB adapters    │
└───────────────────────────┬──────────────────────────────┘
                            │ Layers provide services
┌───────────────────────────▼──────────────────────────────┐
│ FUNCTIONAL CORE (packages/domain)                        │
│  schemas · FSM transition tables · command handlers      │
│  policy evaluation · money math · gap queries            │
│  pure decisions: given state + command + policy →        │
│       DomainEvent[] | typed error                        │
└──────────────────────────────────────────────────────────┘
```

### Core rules

1. **Core does not import** React, Express, SQL drivers, HTTP clients, or Stripe SDKs.
2. Core command handlers are Effect programs that depend only on **service interfaces** (`EventRepo`, `PolicyRepo`, `Clock`, `IdGen`, `Audit` *port*).
3. **Shells** implement those interfaces (Layers) and perform I/O.
4. Tests for FSMs, cost-plus math, and policy gates run with **in-memory Layers** — no database required.
5. Future automation (payment webhooks, accounting sync) is another **shell** that calls the same core commands — never bypasses them.

### Package boundary

```text
packages/domain/     # functional core (+ ports as interfaces)
packages/client/     # UI shell + local DB Layer + sync fiber
packages/server/     # API shell + Postgres Layer + auth
packages/sync/       # protocol types shared; transport adapters in client/server
packages/policy/     # schemas + evaluators for Policy documents (or live under domain/)
packages/observability/  # tracing/metrics helpers wrapping shells (optional)
packages/tenant-*/   # policy defaults + templates as data, not forked core
```

---

## Policy as data

Hard-coded `if (tenant === "hypeluxe")` is forbidden for product rules. **Policies are versioned documents** evaluated by the core.

### Policy document

```ts
type PolicyDocument = {
  id: string
  scope: "platform" | "workspace" | "eventType" | "event"
  workspaceId?: string
  eventTypeId?: string
  eventId?: string
  version: number
  status: "draft" | "active" | "retired"
  body: PolicyBody          // Effect Schema–validated JSON
  activatedAt?: string
}
```

### What belongs in policy (examples)

| Area | Examples (data, not code branches) |
|---|---|
| Logistics | `allowLogisticsBeforeRsvp`, `requireFlightLinkForAirportPickup` |
| Money | `paymentFundsAvailableAt: "cleared"` (D-006), `drawdownBasis: "actual"` (D-007), `costPlusDefaults`, markup-eligible categories |
| Visibility | `coordinatorCanReadMoney`, `hostSeesMargins`, `hostSeesOwnInvoicesOnly` |
| Invoicing | `autoDraftDepositFromOffering: true` (D-008), `requireCommercialSnapshotOnIssue` |
| Capacity | waitlist enabled, room-block overbook rule |
| Retention | audit retention window |

Tenant packs (Hypeluxe) **ship default policy + templates** as data. Workspaces may override; event-level overrides only where explicitly allowed.

### Evaluation

```text
resolvePolicy(workspace, eventType, event) =
  merge(platformDefaults, tenantPack, workspaceOverrides, eventOverrides)
```

Core commands receive resolved `Policy` (or load via `PolicyRepo`). Changing policy is a data change + activation command (`ActivatePolicy`), audited — not a deploy, unless the **schema** of `PolicyBody` evolves (then migrate + CD).

### Tests

- Table-driven: same command, different policy bodies → allow / deny / different guards
- Hypeluxe pack fixtures validated against `PolicyBody` schema in CI

---

## Continuous delivery

CD is not “we have GitHub Actions.” It means **main is always releasable**.

### Pipeline

```text
PR ──► typecheck · unit (core+FSM+policy) · migration dry-run · lint
     └► optional Playwright smoke against ephemeral stack

main ──► build immutable artifact (container / bundle + schema version)
      └► deploy staging · automated smoke (offline→online, invoice+payment manual flow)
           └► promote same artifact → prod (manual approval OK for pilot; automate later)
```

### Rules

1. **Same artifact** staging and prod (no “prod-only rebuild”).
2. **Feature flags / policy** for risky behavior — prefer policy data or flags over long-lived branches.
3. **Schema migrations** expand/contract compatible; clients tolerate **N−1**.
4. **No deploy** of payment-processor auto-sync until manual command flows are green in staging smoke.
5. Core package versioned; tenant packs versioned independently when possible.

### CD smoke minimum (staging)

- Create event offline → sync
- Multi-day schedule edit
- Guest + travel leg + movement assign
- Draft/issue invoice → record payment → apply → drawdown
- Audit actor preserved across sync
- Illegal FSM transition rejected

---

## Observability

Audit (domain attribution) ≠ observability (runtime health). We need both.

### Three pillars (+ audit)

| Signal | Use |
|---|---|
| **Traces** | Request/command/sync fiber spans (OpenTelemetry). Core commands create spans named after the command (`IssueClientInvoice`). |
| **Metrics** | Command success/failure rates, sync lag, conflict counts, invoice issue latency, authz denials |
| **Logs** | Structured logs in shells only (core returns errors; shell logs with `correlationId`) |
| **Audit** | Business who/what/when — [audit-and-attribution.md](./audit-and-attribution.md) |

### Correlation

Every command carries `correlationId` (already required for sync idempotency). Propagate it to:

- Audit records  
- OTel trace/span attributes  
- Client sync queue entries  
- Future integration webhooks  

### Shell instrumentation

- HTTP/RPC middleware: trace + metrics  
- Sync fiber: lag gauge, conflict counter, apply failure counter  
- DB Layers: optional slow-query spans  
- **Never** log full PII in traces; use ids + redaction policy  

### SLOs (pilot starting point)

| SLO | Target (initial) |
|---|---|
| Sync apply success (non-conflict) | ≥ 99% over 7d |
| API command p95 (staging/prod) | define once baselines exist on machineaid hosting |
| Smoke suite on staging after deploy | pass before prod promote |

Alert on burn rates once metrics exist — not before instrumentation ships.

### Implementation phasing

| Slice | Scope |
|---|---|
| O11y-1 | `correlationId` everywhere; structured logs in server shell |
| O11y-2 | OTel traces on API + sync apply; basic command metrics |
| O11y-3 | Dashboards + staging smoke gate + error budget for sync |

---

## How the pieces fit

```text
UI / webhook / sync apply  (shell)
        │
        ▼
 Authz + load Policy (data) + load aggregate
        │
        ▼
 Core command (FSM + policy gates) → events | error
        │
        ├── Audit.append (business)
        ├── Repo.save (shell)
        └── Span/metric (observability shell)
```

Local-first **device sync** is part of the product data plane. **Payment/accounting auto-sync** is a future shell on the same core — see [budget-and-money.md](./budget-and-money.md#manual-first-automation-later).
