# Domain model (event-agnostic)

This document defines the target domain for the new platform. It deliberately generalizes the legacy TwoBeWed models (`user`, `client`, `vendor`).

## Core entities

```text
Workspace
  └── Event[]
        ├── Schedule (kind + units)
        ├── Stakeholder[]
        ├── Guest[]
        │     └── LogisticsProfile (travel, stays, transfers, local moves)
        ├── AccommodationBlock[]     (room blocks / hotels)
        ├── TransferPlan[]           (pickup / dropoff waves)
        ├── VendorAssignment[]
        ├── Offering / Package (optional)
        ├── Task[] / Checklist
        └── Note[]

Vendor (workspace-scoped catalog)  // includes hotels, transport, DMCs, …
OfferingTemplate / EventTypeTemplate (tenant packs)
```

### Workspace

Tenant boundary for an agency, producer, or brand (e.g. Hypeluxe). Holds members, roles, vendor catalog, and enabled event-type templates.

### Event

A planned occasion owned by a workspace. Not inherently a wedding.

| Field (conceptual) | Notes |
|---|---|
| `id`, `workspaceId` | Identity |
| `title` | Display name |
| `eventTypeId` | References a template (destination wedding, multi-day conference, gala, …) |
| `status` | draft / active / completed / archived |
| `schedule` | See below |
| `destination` (optional) | City/region/country + primary timezone; critical for destination events |
| `location` (optional summary) | Legacy-friendly summary; detailed venues live on schedule units |
| `offering` | Selected package + extras |
| `stakeholders` | Decision-makers / hosts (not the full guest list) |
| `guests` | Attendees managed for ops + logistics |
| `accommodationBlocks` | Hotels / room blocks for the event |
| `transferPlans` | Coordinated pickup/dropoff waves |
| `vendorAssignments` | Links to catalog vendors with role/notes |
| `tasks`, `notes` | Operational content |

**Representative event types (templates):** destination wedding, local wedding weekend, multi-day conference, corporate offsite, private gala.

### Schedule

**Schedule is not a single date field.**

```ts
type ScheduleKind = "single_day" | "multi_day"

type Schedule = {
  kind: ScheduleKind
  timezone: string
  units: ScheduleUnit[] // ordered
}

type ScheduleUnit = {
  id: string
  label: string           // "Main day", "Welcome dinner", "Day 2", "Keynotes"
  date: string            // calendar date in event timezone
  venue?: VenueRef
  blocks?: TimeBlock[]    // ceremony, session track, breakout, dinner, …
}
```

| Kind | Shape | Examples |
|---|---|---|
| `single_day` | Typically one `ScheduleUnit` (optionally many time blocks) | Local ceremony + reception |
| `multi_day` | Two or more ordered `ScheduleUnit`s | Destination wedding weekend; 3-day conference |

Future kinds (not required for v1): recurring series, open-ended / TBA dates with placeholders.

### Stakeholder vs Guest

| | Stakeholder | Guest |
|---|---|---|
| Who | Hosts, couple, sponsor, conference chair, on-site lead | Invitees / attendees / +1s |
| Purpose | Ownership, approvals, commercial relationship | Attendance, hospitality, logistics |
| Cardinality | Few | Often dozens to hundreds+ |

Roles are configurable per event type (e.g. “bride/groom” vs “delegate/speaker”).

### Guest management

Guests are first-class event records—not a bolt-on spreadsheet.

| Field (conceptual) | Notes |
|---|---|
| Identity | Name, contact, household/party grouping, dietary, accessibility |
| Invitation / RSVP | invited → responded → attending / declined / waitlist (ops RSVP, not public ticket sales) |
| Party | Link +1s / family / shared roommates |
| Tags | VIP, speaker, vendor-as-guest, staff |
| LogisticsProfile | Nested or related travel/stay/transfer/local legs (below) |

Planners need list/filter views: arrivals by day, missing flights, unassigned pickups, hotel unassigned, etc.

### Logistics (travel, stays, transfers, local)

Logistics are **structured ops records** attached to guests (and sometimes to event-level plans). The platform **tracks and coordinates** bookings; it does not replace airline GDSs or hotel PMS inventory.

#### Long-haul / arrival travel (`TravelLeg`)

Flights, trains, coaches into the destination (and departures home).

```ts
type TravelLeg = {
  guestId: string
  direction: "inbound" | "outbound"
  mode: "flight" | "train" | "bus" | "other"
  carrier?: string
  flightOrNumber?: string
  from: LocationRef
  to: LocationRef
  departAt?: DateTime
  arriveAt?: DateTime
  confirmationCode?: string
  status: "planned" | "booked" | "checked_in" | "completed" | "cancelled"
}
```

#### Accommodations (`Stay` + `AccommodationBlock`)

- **AccommodationBlock** — event-level hotel/villa room block (vendor link, dates, allotment).
- **Stay** — guest (or party) room assignment within a block or ad-hoc property.

```ts
type Stay = {
  guestIds: string[]      // room sharers
  propertyId: string      // vendor or block
  checkIn: Date
  checkOut: Date
  roomType?: string
  confirmationCode?: string
  status: "requested" | "confirmed" | "checked_in" | "checked_out" | "cancelled"
}
```

#### Pickups / dropoffs (`Transfer`)

Airport ↔ hotel ↔ venue waves; may be individual or shared vehicles.

```ts
type Transfer = {
  guestIds: string[]
  kind: "pickup" | "dropoff" | "point_to_point"
  from: LocationRef
  to: LocationRef
  windowStart: DateTime
  windowEnd?: DateTime
  vehicleOrVendorId?: string
  linkedTravelLegId?: string  // e.g. meet flight XY123
  status: "planned" | "assigned" | "en_route" | "completed" | "no_show" | "cancelled"
}
```

**TransferPlan** groups transfers into waves (e.g. “Thursday AM arrivals”) for dispatcher-style views.

#### Local travel (`LocalMove`)

In-destination movement that is not a primary airport transfer: hotel → venue shuttles, island boats, group coaches between conference hotels, dinner transport.

Same shape as `Transfer` with `kind` / tags distinguishing **local** loops from arrival/departure transfers—or a shared `Movement` type with `scope: "arrival" | "departure" | "local"`.

### Vendor

Workspace catalog entry: name, contacts, location, notes, past event links. Categories include venues, hotels, transport, DMC, AV, catering, etc. Assigned to events via `VendorAssignment`.

### Offering / package

Commercial shape attached to an event (main package + extras). Templates come from the tenant pack (Hypeluxe packages) but storage remains generic.

### Task & note

Operational work items and freeform notes. Strong candidates for CRDT or careful merge policies under local-first sync.

## Mapping from legacy TwoBeWed

| Legacy | Target |
|---|---|
| `user` | Workspace member (planner/producer) |
| `client` (+ `weddingDate`) | `Event` + `Schedule` + stakeholders |
| `vendor` | `Vendor` + assignments |
| _(none)_ | `Guest` + logistics (travel, stay, transfer, local) |
| `@twobewed.com` email rule | Tenant auth policy (not global) |

## Invariants

1. Every event belongs to exactly one workspace.
2. Every event has a schedule with `kind` and at least one unit once dates are set (drafts may allow empty/TBA with explicit status).
3. Guests belong to exactly one event; logistics records reference guests (or event-level blocks/plans).
4. Vertical language (“bride”, “delegate”, “ceremony”) lives in templates and UI copy, not required core fields.
5. Destination is optional metadata; logistics may still exist for non-destination events (e.g. local shuttles).
