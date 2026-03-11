# EVENT BOOKING SYSTEM – CORE RULES

This document defines the **non-negotiable business and architectural rules**
for the Event Booking & Organization platform.

All code, features, and refactors MUST comply with these rules.

**Version:** 2.0
**Date:** 2026-03-12
**Status:** approved

---

## 1. USER ROLES & PERMISSIONS

### 1.1 Customer
Customers may:
- View only `PUBLISHED` events
- Create / update / cancel their own bookings
- Create / pay / view their own orders

Customers may NOT:
- Modify events
- Modify ticketTypes
- Access unpublished data (DRAFT events, DRAFT ticketTypes, PENDING/REJECTED orgs)

---

### 1.2 Organizer
Organizers may:
- Create organizations
- Edit **non-identity fields** of their own `APPROVED` organizations (see §2.3)
- Manage members of their own organizations
- CRUD events ONLY if organization is `APPROVED`
- Edit event **contractual fields** only while event is `DRAFT`
- Edit event **metadata fields** while event is `DRAFT` or `PUBLISHED`
- CRUD ticketTypes of their own events (subject to TicketType state rules in §4)
- Publish and cancel their own events

Organizers may NOT:
- Edit admin-managed entities (venues, categories)
- Modify events of other organizations
- Edit **identity fields** of their organization once it is `APPROVED`
- Delete an `APPROVED` organization
- Edit contractual event fields once event is `PUBLISHED`
- Edit contractual TicketType fields once TicketType is `ACTIVE`

---

### 1.3 Admin
Admins may:
- CRUD venues
- CRUD categories
- CRUD organizations (including archiving)
- Approve / reject / archive organizations
- View all system data
- Edit events and ticketTypes (same field restrictions as organizer apply)

Admins must NOT:
- Modify paid bookings or orders
- Override booking integrity rules
- Hard-delete entities with bookings or paid orders

---

## 2. ORGANIZATION RULES

### 2.1 Organization States

```text
PENDING → APPROVED → ARCHIVED
PENDING → REJECTED
REJECTED → PENDING  (re-review, Admin only)
```

### 2.2 Pre-conditions for organization actions

- Organization must be `APPROVED` before:
  - Creating events
  - Publishing events
- `REJECTED` organizations are read-only for organizer
- `ARCHIVED` organizations are read-only; existing events are unaffected but
  no new events may be created
- Approval status is controlled by Admin only
- Backend MUST validate `organization.status == APPROVED` on every event
  create and event publish operation (frontend filtering is insufficient)

### 2.3 Organization field classification

Once an organization becomes `APPROVED`, **identity fields** become immutable
for all actors. Changes to identity fields require an admin-managed
re-verification workflow.

| Field | Classification | Editable after APPROVED? |
|---|---|---|
| Legal name | Identity | **No** |
| Registration / incorporation number | Identity | **No** |
| Tax identification number | Identity | **No** |
| Bank account details | Identity | **No** |
| Logo / avatar | Non-identity | Yes (by organizer or admin) |
| Description / bio | Non-identity | Yes (by organizer or admin) |
| Contact email and phone | Non-identity | Yes (by organizer or admin) |
| Social media links / website URL | Non-identity | Yes (by organizer or admin) |
| Members | Operational | Yes (by organizer or admin) |

### 2.4 Organization deletion rules

- `PENDING` organizations with no events: may be soft-deleted by owner or Admin
- `APPROVED` organizations: MUST NOT be deleted. Admin may archive.
- `ARCHIVED` organizations: read-only soft-deleted state; data retained for audit.
- Admin MUST NOT archive an organization that has `PUBLISHED` events with
  active paid bookings. Those events must be cancelled and refunded first.

---

## 3. EVENT LIFECYCLE RULES

### 3.1 Event States

```text
DRAFT → PUBLISHED → CANCELLED
DRAFT → PUBLISHED → COMPLETED
COMPLETED → PUBLISHED      (auto: capacity freed by booking cancellations)
COMPLETED → CANCELLED
```

### 3.2 Draft Event

- Fully editable (all fields)
- No bookings allowed
- Not visible to customers
- May be soft-deleted (preferred) by organizer or admin

### 3.3 Published Event

**Contractual fields — MUST NOT be edited after publish:**

| Field | Reason |
|---|---|
| Event start time / end time | Core booking contract |
| Location / venue | Core booking contract |
| Event format (online / offline) | Core booking contract |
| Associated organization | Immutable identity |
| Category | Structural |
| Event title | Referenced in booking snapshots |

**Metadata fields — MAY be edited after publish:**

| Field | Constraint |
|---|---|
| Description text | Display only |
| Banner / cover image | Display only |
| Contact information | Operational |
| FAQ content | See restriction below |
| Notes | See restriction below |

**Restriction on FAQ and Notes:** These fields MUST NOT contain refund policy,
entry requirements, age restrictions, or any information that materially
defines the purchase terms. Such terms belong in `TicketType.refundPolicy`
and `TicketType.benefits`, which are immutable once ACTIVE.

**Bookings:** Allowed while event is `PUBLISHED` and at least one `ACTIVE`
TicketType has remaining capacity.

**Deletion:** PROHIBITED. Organizer must cancel the event instead.

### 3.4 Sold-Out Event

- Auto-transition: system sets `COMPLETED` when no `ACTIVE` TicketType has
  remaining capacity (`sold >= capacity` across all ACTIVE types)
- No new bookings accepted
- Event remains publicly visible with "Sold Out" status
- Auto-transition back to `PUBLISHED` when capacity is freed (booking cancellation)
- Organizer may cancel a `COMPLETED` event

### 3.5 Cancelled Event

- No new bookings allowed
- All paid bookings MUST be refunded (explicit backend action, not silent)
- Event becomes fully immutable
- CANNOT be deleted; data retained for audit and refund resolution
- SSE emits `event:cancel` to `public,organizer` (one-time transition signal)
- On cancellation: all `ACTIVE` TicketTypes → `DEACTIVATED` (cascade)
- On cancellation: all `DRAFT` TicketTypes → `DEACTIVATED` (cascade, see §4.7)

### 3.6 Event deletion rules

| Event state | Has bookings | Allowed action |
|---|---|---|
| DRAFT | No | Soft-delete by organizer or admin |
| DRAFT | Yes | Soft-delete only (bookings must be cancelled first) |
| PUBLISHED | Any | PROHIBITED — cancel instead |
| COMPLETED | Any | PROHIBITED — cancel instead |
| CANCELLED | Any | PROHIBITED — data retained for refund resolution |

---

## 4. TICKET TYPE RULES

### 4.1 TicketType States

```text
DRAFT → ACTIVE → SOLD_OUT (auto)
ACTIVE → DEACTIVATED
DRAFT  → DEACTIVATED     (cascade on event cancel — see §3.5)
```

### 4.2 TicketType DRAFT State

TicketTypes created under a **PUBLISHED** event start in `DRAFT` state.
TicketTypes on `DRAFT` events are also in `DRAFT` state by default.

Rules for DRAFT TicketTypes:
- **NOT visible to customers**, regardless of parent event state
- Fully editable (all fields: price, capacity, currency, name, description)
- May be hard-deleted if no bookings exist
- Do NOT count toward event capacity totals
- Do NOT block event publish or cancel operations

Activation requirements (DRAFT → ACTIVE):
- `price > 0`
- `capacity > 0`
- `salesStart` and `salesEnd` must be set and valid
- Transition is triggered explicitly by organizer (not automatic)

### 4.3 TicketType ACTIVE State and Immutability

Once a TicketType transitions to `ACTIVE`, it becomes a public offering and
**all contractual fields lock immediately**.

The following fields MUST NOT be edited once `ACTIVE`:

| Field | Reason |
|---|---|
| Price | Core purchase contract |
| Currency | Core purchase contract |
| Capacity (decrease only — see §4.3a) | Prevents oversell |
| Benefits / access rights | Core purchase contract |
| Refund policy | Core purchase contract |

**§4.3a — Capacity modification rule:**
- Capacity MUST NOT be **decreased** below current `sold` count at any time
- Capacity **increases** are permitted after `ACTIVE` (more seats available)
- Any capacity change after `ACTIVE` MUST be logged in the audit trail

Fields that MAY be edited at any time regardless of state:
- Display name
- Description wording
- UI ordering / sort index

### 4.4 TicketType SOLD_OUT State

- Auto-transition: system sets `SOLD_OUT` when `sold >= capacity`
- TicketType remains publicly visible with "Sold Out" indicator
- Auto-transition back to `ACTIVE` if capacity is freed (booking cancellation)
- Cannot accept new bookings while `SOLD_OUT`

### 4.5 TicketType DEACTIVATED State

Deactivated TicketTypes:
- Cannot be sold or accept new bookings
- MUST remain valid for: existing bookings, check-in, refund, reporting
- MUST NOT be deleted once they have been `ACTIVE` or have bookings
- Retain all historical data for audit and financial reconciliation

### 4.6 TicketType deactivate-and-replace pattern

If a contractual field must change after a TicketType is `ACTIVE`:

1. Deactivate the existing TicketType
2. Create a new TicketType (starts as `DRAFT`)
3. Set required fields (price, capacity, etc.)
4. Activate the new TicketType explicitly

### 4.7 TicketType cascade on event cancellation

When an event transitions to `CANCELLED`:

| TicketType state at cancel | Action |
|---|---|
| `ACTIVE` | Auto-deactivated (cascade) |
| `SOLD_OUT` | Auto-deactivated (cascade) |
| `DRAFT` | Auto-deactivated (cascade) — not deleted, retained for audit |
| `DEACTIVATED` | No change |

### 4.8 TicketType deletion rules

| State | Has bookings | Allowed action |
|---|---|---|
| DRAFT | No | May be deleted (hard or soft) |
| DRAFT | Yes | Soft-delete only |
| ACTIVE | No bookings | Deactivate (deletion discouraged) |
| ACTIVE | Has bookings | Deactivate only — MUST NOT delete |
| DEACTIVATED | Any | MUST NOT delete |
| SOLD_OUT | Any | MUST NOT delete |

---

## 5. BOOKING RULES

Booking represents a snapshot in time.

Booking must store:

- `event_id`
- `event_version`
- `ticket_type_id`
- `event_snapshot`
- `ticket_snapshot`
- `price_paid`

Bookings MUST NEVER:

- Depend on live Event data
- Depend on live TicketType data
- Be silently modified after payment

Bookings are only valid for `ACTIVE` TicketTypes on `PUBLISHED` events at
the time of booking creation. Subsequent state changes (deactivation,
cancellation) do not retroactively invalidate a booking — they trigger
the refund workflow instead.

---

## 6. ORDER & PAYMENT RULES

- Orders are immutable once paid
- Paid amount must never change
- Refunds must be explicit backend actions
- Partial refunds must be traceable
- `order:refund` SSE notifies the customer; it does not trigger the refund

---

## 7. DATA INTEGRITY PRINCIPLES

### 7.1 No hard delete for critical entities

MUST use soft-delete or status flags for:

- Events with any bookings (regardless of payment state)
- TicketTypes that have ever been `ACTIVE` or have bookings
- Paid orders
- Cancelled events

### 7.2 Soft-delete states

| Entity | Soft-delete state |
|---|---|
| Organization | `ARCHIVED` |
| Event | `CANCELLED` (for post-publish) or soft-deleted status for DRAFT |
| TicketType | `DEACTIVATED` |
| Order | `CANCELLED` or `EXPIRED` |

### 7.3 Auditability

All critical changes MUST be logged with: `actor_id`, `action`, `entity_type`,
`entity_id`, `old_value` (JSON), `new_value` (JSON), `timestamp`.

Audit-required actions include:

- Organization verification / rejection / archiving
- Event publish / cancel
- TicketType activation / deactivation
- Booking creation
- Order confirmation
- Refund initiation
- Any post-publish metadata edit on an event
- Any capacity change on an `ACTIVE` TicketType

---

## 8. FORBIDDEN ACTIONS

The system MUST NEVER:

- Modify paid booking data silently
- Edit contractual TicketType fields (price, currency, capacity) once `ACTIVE`
- Decrease capacity below the current sold count
- Reassign bookings to a different ticketType
- Change historical order amounts
- Allow customers to view DRAFT events or DRAFT ticketTypes
- Allow bookings on DRAFT or CANCELLED events
- Allow bookings on DRAFT or DEACTIVATED ticketTypes
- Hard-delete entities with bookings or paid orders
- Allow organizer to edit event contractual fields after publish
- Archive an organization with PUBLISHED events having active paid bookings
  without first cancelling those events

---

## 9. ERROR HANDLING

- Forbidden actions must throw clear validation errors
- Silent failures or auto-fixes are not allowed
- Business rule violations must be explicit
- Error messages must identify the violated rule (e.g., "TicketType is ACTIVE — price cannot be edited")

---

## 10. FINAL PRINCIPLE

If money has been paid or a user has booked,
the system must protect that data above all else.
