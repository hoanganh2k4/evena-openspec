# EVENT BOOKING SYSTEM – CORE RULES

This document defines the **non-negotiable business and architectural rules**
for the Event Booking & Organization platform.

All code, features, and refactors MUST comply with these rules.

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
- Access unpublished data

---

### 1.2 Organizer
Organizers may:
- CRUD organizations they own
- Manage members of their organizations
- CRUD events ONLY if organization is `APPROVED`
- CRUD ticketTypes of their own events
- Publish and cancel their own events

Organizers may NOT:
- Edit admin-managed entities (venues, categories)
- Modify events of other organizations
- Modify published immutable fields

---

### 1.3 Admin
Admins may:
- CRUD venues
- CRUD categories
- CRUD organizations
- Approve / reject organizations
- View all system data

Admins must NOT:
- Modify paid bookings or orders
- Override booking integrity rules

---

## 2. ORGANIZATION RULES

- Organization must be `APPROVED` before:
  - Creating events
  - Publishing events
- Rejected organizations are read-only
- Approval status is controlled by Admin only

---

## 3. EVENT LIFECYCLE RULES

### 3.1 Event States
```text
DRAFT → PUBLISHED → CANCELLED
```
### 3.2 Draft Event
Fully editable

No bookings allowed

### 3.3 Published Event
Once published:

Event becomes fully immutable from the organizer UI

Bookings and payments may exist

Allowed updates: **none** — the Edit action is disabled once an event is PUBLISHED

Forbidden updates:

Time

Date

Location

Format (online/offline)

Description text

Images

UI metadata

To change any content after publish, the organizer must cancel the event and create a new one

Critical updates MUST NOT occur — there is no partial-edit path for published events

### 3.4 Cancelled Event
No new bookings allowed

All paid bookings must be refunded

Event becomes immutable

### 4. TICKET TYPE RULES (VERY IMPORTANT)
### 4.1 TicketType Immutability
Once an Event is published, TicketTypes act as contracts.

The following fields MUST NOT be edited:

Price

Currency

Capacity

Benefits / access rights

Refund policy

### 4.2 TicketType Changes
NOT ALLOWED CHANGE

Deactivate the old TicketType

Create a new TicketType with a new ID

Deactivated TicketTypes:

Cannot be sold

Must remain valid for:

Existing bookings

Check-in

Refund

Reporting

Allowed edits after publish:

Display name

Description wording

UI ordering

### 5. BOOKING RULES
Booking represents a snapshot in time

Booking must store:

event_id

event_version

ticket_type_id

event_snapshot

ticket_snapshot

price_paid

Bookings MUST NEVER:

Depend on live Event data

Depend on live TicketType data

Be silently modified after payment

### 6. ORDER & PAYMENT RULES
Orders are immutable once paid

Paid amount must never change

Refunds must be explicit actions

Partial refunds must be traceable

### 7. DATA INTEGRITY PRINCIPLES
No hard delete for:

Events with bookings

TicketTypes with bookings

Paid orders

Use soft-delete or status flags

All critical changes must be auditable

### 8. FORBIDDEN ACTIONS
The system MUST NEVER:

Modify paid booking data silently

Edit ticket prices after publish

Reduce capacity below sold count

Reassign bookings to a different ticketType

Change historical order amounts

### 9. ERROR HANDLING
Forbidden actions must throw clear validation errors

Silent failures or auto-fixes are not allowed

Business rule violations must be explicit

### 10. FINAL PRINCIPLE
If money has been paid or a user has booked,
the system must protect that data above all else.