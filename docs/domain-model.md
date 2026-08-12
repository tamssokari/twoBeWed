# Domain model (event-agnostic)

This document defines the target domain for the new platform. It deliberately generalizes the legacy TwoBeWed models (`user`, `client`, `vendor`).

## Core entities

```text
Workspace
  └── Event[]
        ├── Schedule (kind + units)
        ├── Stakeholder[]
        ├── VendorAssignment[]
        ├── Offering / Package (optional)
        ├── Task[] / Checklist
        └── Note[]

Vendor (workspace-scoped catalog)
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
| `eventTypeId` | References a template (wedding, gala, offsite, …) |
| `status` | draft / active / completed / archived |
| `schedule` | See below |
| `location` (optional summary) | Primary city / region; detailed venues live on schedule units |
| `offering` | Selected package + extras |
| `stakeholders` | People/orgs related to the event |
| `vendorAssignments` | Links to catalog vendors with role/notes |
| `tasks`, `notes` | Operational content |

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
  label: string           // "Main day", "Welcome dinner", "Day 2"
  date: string            // calendar date in event timezone
  venue?: VenueRef
  blocks?: TimeBlock[]    // ceremony, cocktail, dinner, session A, …
}
```

| Kind | Shape |
|---|---|
| `single_day` | Typically one `ScheduleUnit` (optionally many time blocks) |
| `multi_day` | Two or more `ScheduleUnit`s (days or major sessions), ordered |

Future kinds (not required for v1): recurring series, open-ended / TBA dates with placeholders.

### Stakeholder

Replaces the assumption that every event has a “couple/client” singleton.

Examples: couple, host, corporate sponsor, festival director, on-site lead. Roles are configurable per event type.

### Vendor

Workspace catalog entry: name, contacts, location, notes, past event links. Assigned to events via `VendorAssignment` (role, status, schedule relevance).

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
| `@twobewed.com` email rule | Tenant auth policy (not global) |

## Invariants

1. Every event belongs to exactly one workspace.
2. Every event has a schedule with `kind` and at least one unit once dates are set (drafts may allow empty/TBA with explicit status).
3. Vertical language (“bride”, “ceremony”) lives in templates and UI copy, not required core fields.
