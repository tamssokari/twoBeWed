# Audit and attribution

Every meaningful domain change must be attributable: **who** did **what**, to **which entity**, **when**, from **which device**, under **which workspace/event**.

This is separate from calendar/schedule “events.” Here **audit** records **domain commands / domain events** (state transitions and structured field updates).

## Goals

1. **Attribution** — planner A vs planner B vs future guest self-serve vs system job
2. **Forensics** — why was this guest’s pickup cancelled? who rebooked the flight?
3. **Sync integrity** — offline writes keep the original actor; sync transport user ≠ author
4. **Compliance-ready trail** — PII-aware; retain enough to explain ops decisions

## Non-goals (v1)

- Keystroke-level history for collaborative notes (summarize note revisions separately if needed)
- Full event sourcing as the *only* write path (optional later); **command audit** is enough to start
- Immutable legal hold workflows (can layer on)

## What gets audited

| Audited | Examples |
|---|---|
| State-machine commands | `AcceptRsvp`, `AssignMovement`, `ConfirmStay`, `PublishEvent`, `PostDrawdown`, `RecordClientPayment` |
| Structured field updates on aggregates | guest contact change, travel times, confirmation codes, cost amounts, markup % |
| Sync outcomes | command `rejected` / `conflict` with reason |
| Authz failures (optional, security log) | denied `AssignMovement` / `PostDrawdown` |

| Not audited (v1) | Examples |
|---|---|
| Pure reads | list filters, opening a board |
| Ephemeral UI state | column sort, collapsed panels |
| Every CRDT note character | flush note “revision committed” instead |

## Audit record (canonical)

```ts
type AuditRecord = {
  id: string                 // stable ULID/UUID
  workspaceId: string
  eventId?: string           // null for workspace-level (vendor catalog, members)

  entityType: string         // "event" | "guest" | "travel_leg" | "stay" | "movement" | …
  entityId: string
  action: string             // domain event / command name: "MovementAssigned"

  actor: {
    actorId: string          // user id (or "system:<job>")
    displayNameSnapshot: string
    roleSnapshot?: string    // role at time of action
  }

  atClient: string           // ISO datetime when device committed the command
  atServer?: string          // set when cloud persists / acks
  deviceId: string
  sessionId?: string
  correlationId: string      // command id; used for idempotent sync

  before?: unknown           // sparse snapshot or JSON patch
  after?: unknown
  metadata?: Record<string, unknown>  // e.g. { previousStatus, nextStatus }
}
```

### Attribution rules (local-first)

1. **Stamp actor on the device at write time** — never rewrite `actor` to the user who happened to sync.
2. `correlationId` is unique per command; retries reuse it (idempotent apply).
3. Server sets `atServer` on accept; if transition is illegal on apply, status stays prior, audit gets `action` outcome `rejected` / `conflict` with reason.
4. System jobs use `actorId = "system:<name>"` (e.g. `system:waitlist-promotion`).

## Storage & sync

- Audit records are append-only rows in local DB and Postgres.
- Clients may keep a **recent window** offline; full history is server-authoritative.
- Sync: upload unaudited-acked local audit + commands together (or audit derived server-side from accepted commands — pick one implementation; prefer **client-stamped audit uploaded with command** so offline attribution survives).

## PII

Audit `before`/`after` may contain emails, phone numbers, confirmation codes.

- Minimize payloads (status transitions store statuses, not full guest row, when possible).
- Tenant retention policy (default TBD with Hypeluxe): e.g. retain ops audit N months post-event.
- Export/delete guest flows must define whether audit is redacted vs retained for legitimate ops interest.

## Product surfaces

- Event **Activity** feed (filter by entity, actor, day)
- Per-guest / per-movement “history” drawer
- Conflict resolution UI shows both actors + `correlationId`s

## Architecture hook

`Audit` is an Effect service (`Audit.append`) called from domain command handlers after successful transition (and from sync reject paths). Test Layers use an in-memory audit buffer.

See also: [state-machines.md](./state-machines.md), [identity-and-access.md](./identity-and-access.md).
