# Identity and access

## People vs accounts vs roles

The product has several “kinds of person.” They must not be collapsed into one table.

| Concept | Has login? | Purpose |
|---|---|---|
| **User** | Yes | Authenticated human (planner, host with portal access, later guest portal) |
| **WorkspaceMember** | User linked to workspace | Membership + **role** inside a tenant |
| **Stakeholder** | Optional link to User | Host/couple/sponsor on an event (commercial / ownership) |
| **Guest** | Optional link to User (later) | Attendee ops + logistics record |
| **Vendor contact** | Optional later | Person at a vendor org; not the same as Guest |

```text
User
  └── WorkspaceMember (workspaceId, role[])
        └── acts on Events in that workspace

Event
  ├── Stakeholder (may reference userId?)
  └── Guest (may reference userId? later)
        └── Party membership
```

## Party (household / group)

Guests often travel, room, and transfer as a group.

```ts
type Party = {
  id: string
  eventId: string
  label?: string              // "Smith family", "Speaker party A"
  primaryGuestId: string      // invite authority / default contact
  guestIds: string[]          // includes primary
}
```

Rules:

- A guest belongs to **at most one** party per event (v1).
- Stays and movements may reference `guestIds` **or** `partyId` (expand to members at assign time).
- +1s are guests in the same party, not a boolean on the primary alone.

## Roles (RBAC)

Roles are **workspace-scoped**; event-level overrides can come later.

### Pilot role set

| Role | Capabilities (summary) |
|---|---|
| `owner` | Full workspace admin, billing (when exists), delete |
| `planner` | Full event ops: guests, logistics commands, vendors, publish event |
| `coordinator` | Day-of ops: movements, stays check-in, guest field edits; no workspace admin |
| `viewer` | Read events/guests/logistics; no state-changing commands |
| `host` | Stakeholder portal: read own event; limited guest visibility per policy |
| `guest` | (Later) own itinerary read/submit travel; no other guests |

Hypeluxe pilot can start with `owner` / `planner` / `viewer` only; keep `coordinator` / `host` / `guest` in the model so we don’t paint into a corner.

### Command-level authz

State-machine commands declare required roles (Effect `Authz.require("planner")` before transition). Examples:

| Command | Minimum role |
|---|---|
| `PublishEvent` | `planner` |
| `AcceptRsvp` (planner-entered) | `coordinator` |
| `AssignMovement` | `coordinator` |
| `StartMovement` | `coordinator` (driver role TBD) |
| `InviteGuest` | `planner` |
| `RecordClientPayment` / `ApplyPaymentToInvoice` / `PostDrawdown` | `planner` |
| `DraftClientInvoice` / `IssueClientInvoice` / `VoidClientInvoice` | `planner` |
| `LockBudget` / `ReviseCommercialTerms` | `planner` |
| View cost-plus margins | `planner` / `owner` (host: own invoices + balance optional) |

Denied attempts may be security-logged; they do not create domain audit success rows.

## Authn (mechanism)

- Cloud sessions or short-lived access token + refresh.
- Workspace membership resolved on each command (cached in local DB for offline).
- **Offline:** device may execute commands only if a member session was established and cached capabilities allow; on sync, server re-checks role at `atServer` and may reject with authz error (audited as rejected).

## Guest self-serve (later)

When enabled: Guest record links `userId`; that user gets `guest` role **scoped to their guestId** (row-level), not workspace-wide `viewer`.

## Explicitly deferred

| Area | Stance |
|---|---|
| **Full accounting/ERP** | Event budget/cost-plus/**client invoicing**/payments/drawdowns are in scope; GL/tax filing/payroll are not |
| **Payment gateway / auto accounting sync** | Future only — after manual issue/record/apply/drawdown flows are proven; integrations must call the same domain commands |
| **Documents** | No passport/contract vault in v1 (note as Hypeluxe follow-on) |
| **Communications** | No built-in email/SMS blast in v1; RSVP may be planner-entered; intake channel TBD |
| **Vendor login** | Not in pilot |
| **SSO / SAML** | Not required for pilot |

Deferred does not mean “never”; it means out of early phases unless Hypeluxe pulls it forward.
