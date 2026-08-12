# State machines

Status fields in the domain model are **finite state machines**, not free-form enums. Transitions are the only legal way to change status. Invalid transitions fail with a typed `IllegalTransition` error (Effect).

## Design rules

1. **One machine per aggregate** — Event lifecycle ≠ Transfer day-of ops ≠ Guest RSVP.
2. **Commands, not field writes** — UI/services call `assignTransfer`, not `transfer.status = "assigned"`.
3. **Domain events on success** — e.g. `TransferAssigned`, `GuestRsvpAccepted`; these feed audit (see [audit-and-attribution.md](./audit-and-attribution.md)).
4. **Guards** — some transitions require context (vehicle/vendor present, guest attending, etc.).
5. **Same machine client + server** — optimistic local apply; authoritative re-validation on sync.
6. **Derived gaps are queries** — “missing flight”, “no hotel”, “unassigned pickup” are **not** stored guest statuses; they are computed from related records + RSVP state.

## Notation

```text
STATE --[Command / guard]--> STATE
```

Terminal-ish states may still allow reopen commands where noted.

---

## EventLifecycle

Coarse workspace lifecycle for the event record itself. Does **not** encode day-of logistics.

| From | Command | To | Guard |
|---|---|---|---|
| _(new)_ | `CreateEvent` | `draft` | — |
| `draft` | `PublishEvent` | `active` | schedule has ≥1 unit with date **or** explicit TBA flag |
| `draft` | `ArchiveEvent` | `archived` | — |
| `active` | `CompleteEvent` | `completed` | — |
| `active` | `ArchiveEvent` | `archived` | — |
| `completed` | `ReopenEvent` | `active` | workspace role allows |
| `completed` | `ArchiveEvent` | `archived` | — |
| `archived` | `UnarchiveEvent` | `draft` | returns to draft for rework (not straight to active) |

**Meaning of `active`:** planning/ops in progress (including day-of). Not a substitute for Transfer `en_route`.

---

## GuestRsvp

Invitation / attendance machine for a guest. Separate from travel/stay/transfer machines.

```text
                    ┌──────────────────────────────────────┐
                    ▼                                      │
(new) → invited → attending                                │
              └→ declined                                  │
              └→ waitlist ──[ConfirmFromWaitlist]──→ attending
              └→ attending / declined / waitlist
                     --[MarkNoResponse]--> invited   (rare reset)
```

| From | Command | To | Guard |
|---|---|---|---|
| _(new)_ | `InviteGuest` | `invited` | — |
| `invited` | `AcceptRsvp` | `attending` | — |
| `invited` | `DeclineRsvp` | `declined` | — |
| `invited` | `WaitlistGuest` | `waitlist` | capacity policy on event type (optional) |
| `waitlist` | `ConfirmFromWaitlist` | `attending` | — |
| `waitlist` | `DeclineRsvp` | `declined` | — |
| `attending` | `DeclineRsvp` | `declined` | — |
| `declined` | `AcceptRsvp` | `attending` | — |
| `attending` \| `declined` \| `waitlist` | `ResetRsvp` | `invited` | planner role |

**Removed vs earlier draft:** bare `responded` is not a state — acceptance/decline/waitlist *are* the response.

**Logistics policy:** creating/assigning stays and arrival transfers typically requires `attending` (guard). Planners may still capture draft travel while `invited` if the tenant pack allows (`allowLogisticsBeforeRsvp`).

---

## TravelLeg

Inbound/outbound long-haul (or intercity) legs.

| From | Command | To | Guard |
|---|---|---|---|
| _(new)_ | `PlanTravel` | `planned` | guest exists |
| `planned` | `BookTravel` | `booked` | confirmation optional but recommended |
| `booked` | `CheckInTravel` | `checked_in` | — |
| `booked` \| `checked_in` | `CompleteTravel` | `completed` | — |
| `planned` \| `booked` \| `checked_in` | `CancelTravel` | `cancelled` | — |
| `booked` \| `checked_in` | `MarkDelayed` | `delayed` | — |
| `delayed` | `ClearDelay` | `booked` \| `checked_in` | restore prior via payload |
| `booked` \| `delayed` | `RebookTravel` | `booked` | updates leg details; emits `TravelRebooked` |
| `cancelled` | `PlanTravel` | `planned` | replacement leg (or create new leg id) |

`completed` is terminal for that leg id (corrections → new leg or admin `CorrectTravel` if required later).

---

## Stay

| From | Command | To | Guard |
|---|---|---|---|
| _(new)_ | `RequestStay` | `requested` | guestIds non-empty |
| `requested` | `ConfirmStay` | `confirmed` | property/block set; confirmation optional |
| `confirmed` | `CheckInStay` | `checked_in` | — |
| `checked_in` | `CheckOutStay` | `checked_out` | — |
| `requested` \| `confirmed` | `CancelStay` | `cancelled` | — |
| `confirmed` | `MarkStayNoShow` | `no_show` | — |
| `cancelled` \| `no_show` | `RequestStay` | `requested` | re-issue (prefer new stay id) |

---

## Movement (Transfer + local travel)

**One machine** for airport pickups/dropoffs and in-destination local moves. Discriminator: `scope: "arrival" | "departure" | "local"` (and optional `kind` for UI). Replaces separate Transfer / LocalMove status enums.

| From | Command | To | Guard |
|---|---|---|---|
| _(new)_ | `PlanMovement` | `planned` | guestIds non-empty; from/to set |
| `planned` | `AssignMovement` | `assigned` | vehicle **or** transport vendor set |
| `assigned` | `StartMovement` | `en_route` | actor may be planner or driver role |
| `en_route` | `CompleteMovement` | `completed` | — |
| `planned` \| `assigned` \| `en_route` | `MarkNoShow` | `no_show` | at least one guest expected |
| `planned` \| `assigned` \| `en_route` | `CancelMovement` | `cancelled` | — |
| `assigned` | `UnassignMovement` | `planned` | clears vehicle/vendor |
| `cancelled` \| `no_show` | `PlanMovement` | `planned` | re-open same record or new id |

**Linking:** arrival movements may require `linkedTravelLegId` when tenant policy `requireFlightLinkForAirportPickup` is on.

---

## VendorAssignment (minimal)

| From | Command | To |
|---|---|---|
| _(new)_ | `AssignVendor` | `proposed` |
| `proposed` | `ConfirmVendor` | `confirmed` |
| `proposed` \| `confirmed` | `CancelVendorAssignment` | `cancelled` |
| `confirmed` | `CompleteVendorAssignment` | `completed` |

---

## Task (minimal)

| From | Command | To |
|---|---|---|
| _(new)_ | `CreateTask` | `open` |
| `open` | `StartTask` | `in_progress` |
| `open` \| `in_progress` | `CompleteTask` | `done` |
| `open` \| `in_progress` | `CancelTask` | `cancelled` |
| `done` \| `cancelled` | `ReopenTask` | `open` |

---

## BudgetLifecycle

| From | Command | To | Guard |
|---|---|---|---|
| _(new)_ | `CreateBudget` | `draft` | event exists |
| `draft` | `ActivateBudget` | `active` | ≥1 line |
| `active` | `LockBudget` | `locked` | — |
| `locked` | `UnlockBudget` | `active` | `planner` / `owner` |
| `draft` \| `active` \| `locked` | `ArchiveBudget` | `archived` | — |

Line edits allowed in `draft` / `active`; `locked` rejects line mutations (override command `ForceEditLockedBudget` optional later).

## CostLifecycle

| From | Command | To | Guard |
|---|---|---|---|
| _(new)_ | `EstimateCost` | `estimated` | — |
| `estimated` | `CommitCost` | `committed` | amount set |
| `committed` | `InvoiceCost` | `invoiced` | — |
| `invoiced` | `MarkCostPaid` | `paid` | — |
| `estimated` \| `committed` \| `invoiced` | `CancelCost` | `cancelled` | — |

Amounts may be updated via `ReviseCostAmounts` without changing status when still `estimated` / `committed` (audited).

## ClientPaymentLifecycle

| From | Command | To | Guard |
|---|---|---|---|
| _(new)_ | `RecordClientPayment` | `recorded` | amount &gt; 0 |
| `recorded` | `ClearClientPayment` | `cleared` | — |
| `recorded` \| `cleared` | `ReverseClientPayment` | `reversed` | no posted drawdowns depend on it **or** override |

`cleared` counts toward available funds for drawdowns (tenant may treat `recorded` as available — default: only `cleared`).

## DrawdownLifecycle

| From | Command | To | Guard |
|---|---|---|---|
| _(new)_ | `DraftDrawdown` | `pending` | allocations sum = amount |
| `pending` | `PostDrawdown` | `posted` | funds available (payments − posted drawdowns) ≥ amount; cost caps |
| `pending` \| `posted` | `VoidDrawdown` | `void` | `posted` void restores availability |

## CommercialTerms

Not a long-lived status enum — `PublishCommercialTerms` / `ReviseCommercialTerms` commands write a new versioned terms record + audit (+ optional snapshot). See [budget-and-money.md](./budget-and-money.md).

---

## SyncDelivery (infrastructure, not domain ops)

Per local command / mutation batch:

`pending` → `acked` → `synced`, or `pending` → `rejected` / `conflict`

Conflicts do not silently mutate domain status; they surface for resolution and are audited.

---

## Effect sketch

```ts
const assignMovement = (id: MovementId, vehicleId: VehicleId) =>
  Effect.gen(function* () {
    const mov = yield* MovementRepo.get(id)
    yield* ensureTransition(mov.status, "assigned", "AssignMovement")
    yield* Guard.vehicleOrVendor(vehicleId)
    const next = { ...mov, status: "assigned", vehicleOrVendorId: vehicleId }
    yield* MovementRepo.save(next)
    yield* Audit.append({
      action: "TransferAssigned", // or MovementAssigned
      entityType: "movement",
      entityId: id,
      // actor, clocks, before/after — see audit doc
    })
  })
```

## Open product knobs

- Allow logistics commands while GuestRsvp = `invited`?
- Require travel-leg link for airport arrival pickups?
- Driver role for `StartMovement`, or planner-only in Hypeluxe pilot?
- Client payment: do `recorded` funds count before `cleared`?
- Drawdown basis against cost `committed` vs `actual`?
- Can `host` see full cost-plus margins or only balance due?
