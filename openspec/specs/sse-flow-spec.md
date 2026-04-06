# SSE Flow & Realtime Integrity Spec

**Version:** 2.0
**Date:** 2026-03-10
**Status:** approved
**Authors:** team
**Applies to:** frontend · backend · sse-service

---

## 1. Purpose

This spec defines the authoritative rules for all Server-Sent Events (SSE)
behavior in the Evena platform. SSE is not merely a UI convenience layer.
It is a **data-consistency and integrity surface**. Incorrect SSE behavior can:

- Leak unpublished event data to customers (Core Rules §1.1, §3.2)
- Cause UI to render live ticket prices in place of a customer's paid snapshot (Core Rules §4.1, §5)
- Leave customers blind to refunds that are legally required (Core Rules §3.4, §6)
- Broadcast draft event operations to public audiences (Core Rule §1.1)
- Corrupt paid order display through over-invalidation of the RTK Query cache (Core Rule §8)

Every normative rule in this spec traces to one or more sections of `rules.md`.
Where a rule derives from an internal SSE architecture principle rather than a
core business rule, it is marked `[P-n]` (principle).

---

## 2. Scope

This spec covers:

- `sse-service/` — channel routing, permission enforcement, event relay
- `backend/SSENotificationService.java` and all `*Service.java` files that call `notify*()` methods
- `frontend/src/providers/SSEProvider.tsx` — central cache invalidation and personal toast logic
- `frontend/src/stores/types/sse.ts` — SSE action and payload type definitions
- Any frontend page or component that calls `useSSE()` or consumes `lastEvent` directly

---

## 3. Non-Goals

- UI component design or layout
- HTTP REST endpoint design (except where REST and SSE are jointly responsible for a guarantee, this is cross-boundary integrity rule)
- WebSocket migration
- Mobile push notifications
- Email/SMS notification flows

---

## 4. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│ BACKEND (Spring Boot)                                            │
│                                                                  │
│  EventService ──────────────┐                                    │
│  TicketTypeService ─────────┤                                    │
│  OrderService ──────────────┤──► SSENotificationService          │
│  OrganizationService ───────┤         │                          │
│  OrganizationMemberService ─┘         │ POST /api/events/emit    │
└───────────────────────────────────────┼──────────────────────────┘
                                        │
┌───────────────────────────────────────▼──────────────────────────┐
│ SSE SERVICE (Node.js)                                             │
│  ChannelManager                                                   │
│  ├── channels.json  (public, organizer, admin, user:{id})         │
│  ├── actions.json   (all allowed SSE actions per channel)         │
│  └── resources.json (resource → channel permission mapping)      │
│  Validation: is_action_allowed(channel, action)                   │
└──────────────────────────┬────────────────────────────────────────┘
                           │ SSE protocol (text/event-stream)
┌──────────────────────────▼────────────────────────────────────────┐
│ FRONTEND (Next.js)                                                │
│  SSEProvider (React Context)                                      │
│  ├── Connects to channels based on authenticated user role        │
│  ├── Dispatches RTK Query invalidateTags on each event            │
│  └── Renders global Snackbar for personal notifications           │
│                                                                   │
│  RTK Query Cache                                                  │
│  ├── EventAPI        ['Event', {type:'Event', id}]               │
│  ├── OrderAPI        ['Order', 'Ticket']                         │
│  ├── TicketTypeAPI   ['TicketType', {type:'TicketType', id}]     │
│  ├── OrganizerAPI    ['Organizer', {type:'Organizer', id}]       │
│  └── OrganizationMemberAPI  ['Invitation', 'OrganizationMember'] │
└───────────────────────────────────────────────────────────────────┘
```

### Architecture Principles

**P-1 — SSE is a cache-invalidation signal, not a data transport.**
SSE events signal that a cache is stale. The UI MUST fetch fresh data from
the REST API, which enforces all visibility and business rules. SSE payloads
are hints, not authoritative state.

**P-2 — Channel selection is a security and visibility decision.**
Choosing which channel to emit to determines which roles receive the signal.
Emitting to `public` for a DRAFT entity means all customers receive an
invalidation hint. Even if the subsequent REST call filters correctly, the
emit leaks information about internal state changes. Channel selection MUST
respect entity state and role authorization.

**P-3 — Paid state is write-once from the customer's perspective.**
Once an order is CONFIRMED, the customer's view of its total, items, and
ticket snapshots MUST NOT change. SSE MUST NOT trigger cache invalidation
that causes the UI to overwrite snapshot-derived display values with live data.

**P-4 — SSE delivery is not guaranteed.**
Queue overflow, reconnect gaps, and SSE service restarts can cause missed
events. Business rules and state transitions MUST be durable in the database
before SSE is emitted. SSE is additive — the system must be correct without it.

**P-5 — Deactivated entities remain valid for their historical associations.**
When a TicketType is deactivated (Core Rule §4.2), SSE updates the purchase
UI. It MUST NOT affect the display of paid order items that reference the
deactivated TicketType.

---

## 5. Channel Model

| Channel | Who subscribes | Authorized content |
|---------|---------------|-------------------|
| `public` | All authenticated users | PUBLISHED event lifecycle; category/venue admin changes; PUBLISHED ticket type availability |
| `organizer` | Users with ORGANIZER role | Org CRUD; event lifecycle including DRAFT; invitation changes; ticket type changes |
| `admin` | Users with ADMIN role | All organizer content + org approval actions |
| `user:{id}` | Only that specific user | Their own orders, their own tickets, personal org verification, personal invitations |

### Subscription by role (enforced in SSEProvider.tsx)

```
Customer  → public + user:{userId}
Organizer → public + organizer + user:{userId}
Admin     → public + organizer + admin + user:{userId}
```

### Visibility constraint

The `public` channel is for content that customers are authorized to see
under Core Rules. It MUST only carry:

- State transitions of **PUBLISHED** events (`event:publish`, `event:cancel`,
  `event:update` on a published event)
- TicketType changes on **PUBLISHED** events
- Admin-managed global resources (category, venue)

The `public` channel MUST NOT carry:

- Any signal about DRAFT events (their existence, creation, or update)
- Any signal about CANCELLED event internal state after the transition event
- Any order, ticket, or financial data

---

## 6. Event Taxonomy

Events are classified by the audience authorized to receive them.

### Tier A — Public lifecycle events
_Emitted only when the entity state is appropriate for public visibility._

| SSEAction | Trigger | Channel |
|-----------|---------|---------|
| `event:publish` | Event transitions DRAFT → PUBLISHED | `public,organizer` |
| `event:cancel` | Event transitions PUBLISHED → CANCELLED | `public,organizer` |
| `event:update` | Metadata-only update on a PUBLISHED, ONGOING, or COMPLETED event | `public,organizer` |
| `ticket_type:activate` | TicketType transitions DRAFT → ACTIVE on PUBLISHED event | `public,organizer` |
| `ticket_type:update` | Display-only field update on an ACTIVE TicketType (PUBLISHED event) | `public,organizer` |
| `ticket_type:deactivate` | ACTIVE TicketType deactivated on a PUBLISHED event | `public,organizer` |
| `category:create/update/delete` | Admin manages category | `public` |
| `venue:create/update/delete` | Admin manages venue | `public` |

**Note on `event:cancel`:** This is a one-time transition event signaling
the state change from PUBLISHED to CANCELLED. It is the only public-channel
emission related to the CANCELLED lifecycle. After the event reaches steady-state
CANCELLED, no further public-channel broadcasts occur for that event entity.
Refund processing emits separately to private user channels (see Tier C).
The cascade deactivation of TicketTypes triggered by event cancel does NOT
emit individual `ticket_type:deactivate` signals — it is an internal cascade,
not a standalone organizer action.

**Note on `ticket_type:activate`:** This replaces the former `ticket_type:create`
Tier A entry. Creating a TicketType never emits to the public channel. Only the
explicit DRAFT → ACTIVE transition makes a TicketType publicly visible.

### Tier B — Organizer-scoped events
_Emitted for any entity state; never visible to customers._

| SSEAction | Trigger | Channel |
|-----------|---------|---------|
| `event:create` | New DRAFT event created | `organizer,admin` |
| `event:update` | DRAFT or CANCELLED event updated | `organizer,admin` |
| `event:delete` | DRAFT event deleted | `organizer,admin` |
| `ticket_type:create` | New TicketType created (any state, any event state) | `organizer,admin` |
| `ticket_type:update` | TicketType updated while DRAFT | `organizer,admin` |
| `ticket_type:delete` | DRAFT TicketType deleted (no bookings) | `organizer,admin` |
| `ticket_type:deactivate` | DRAFT TicketType deactivated on DRAFT event | `organizer,admin` |
| `organization:create` | New org created | `organizer,admin` |
| `organization:update` | Org details updated | `organizer,admin` |
| `organization:delete` | Org deleted / archived | `organizer,admin` |
| `organization:verify` | Admin approves org (broadcast) | `organizer,admin` |
| `organization:unverify` | Org loses verified status | `organizer,admin` |
| `invitation:accept` | Invitation accepted | `organizer` |
| `invitation:reject` | Invitation rejected | `organizer` |

**Note on `organization:verify`:** The broadcast to `organizer,admin` triggers
list refresh for all organizers. A separate, simultaneous emit goes to
`user:{ownerId}` (Tier C) to deliver the personal toast notification.

**Note on TicketType channel routing:** Channel is determined by **TicketType
state**, not parent event state. A DRAFT TicketType under a PUBLISHED event
emits to `organizer,admin` only (SSE-002 revised). Only ACTIVE TicketTypes
under PUBLISHED events emit to `public,organizer`.

### Tier C — Private user events
_Never broadcast; sent only to the specific user's private channel._

| SSEAction | Trigger | Recipient |
|-----------|---------|-----------|
| `order:create` | Order created (PENDING) | Order owner |
| `order:confirm` | Payment successful | Order owner |
| `order:cancel` | Order cancelled | Order owner |
| `order:expire` | Order expired (15 min timeout) | Order owner |
| `order:refund` | Refund initiated for a paid order | Order owner |
| `ticket:issue` | Ticket generated after payment | Ticket owner |
| `ticket:checkin` | Ticket scanned at venue | Ticket owner |
| `organization:verify` | Personal notification to org owner | Org owner only |
| `invitation:create` | Invitation created — notify invitee | Invitee only |
| `flexpass:approve` | Organizer approves a listing | Seller only (`user:{sellerId}`) |
| `flexpass:reject` | Organizer rejects a listing | Seller only (`user:{sellerId}`) |
| `flexpass:sold` | Transfer completes successfully | Seller + Buyer (`user:{sellerId}`, `user:{buyerId}` — two separate emits) |
| `flexpass:expire` | Listing window expired without a buyer | Seller only (`user:{sellerId}`) |

### Tier B-FlexPass — Organizer-scoped FlexPass events
_Emitted when a listing is created or updated; never visible to customers._

| SSEAction | Trigger | Channel |
|-----------|---------|---------|
| `flexpass:list` | Seller creates a new listing (PENDING) | `organizer,admin` |
| `flexpass:cancel` | Seller withdraws a PENDING listing | `organizer,admin` |

---

## 7. Payload Classification

### 7.1 Minimal Payload Principle

SSE payloads serve one purpose: signal that a cache is stale or that a
personal notification is due. They MUST NOT carry full entity state.

**Allowed fields by event class:**

| Event class | Required | Optional |
|-------------|----------|----------|
| Event events | `eventId`, `eventName` | `organizationId` |
| Organization events | `organizationId`, `organizationName` | `ownerId` (only in organizer/user channels) |
| TicketType events | `ticketTypeId`, `eventId`, `ticketTypeName` | — |
| Order events | `orderId`, `eventId`, `eventName` | `ticketCount` (only for `order:confirm`) |
| Refund events | `orderId`, `eventId`, `eventName` | `refundAmount` (see §7.2) |
| Ticket events | `ticketId`, `eventId`, `eventName` | `ticketTypeName`, `orderId` |
| Invitation events | `invitationId`, `organizationId`, `organizationName` | `inviteeEmail`, `userId`, `userName` |
| Category/Venue | `entityId`, `entityName` | — |
| FlexPass events (organizer-scope) | `listingId`, `ticketId`, `eventId`, `eventName` | `sellerId` (organizer/admin channels only) |
| FlexPass events (personal) | `listingId`, `ticketId`, `eventId`, `eventName` | `refundAmount` (only in `flexpass:expire` on `user:{id}` — see §11.6 rules.md) |

**Forbidden fields in any SSE payload:**

- Full DTO objects (EventResponse, OrderResponse, etc.)
- `price`, `currency`, `unitPrice`, `totalAmount`, `paymentId` — financial fields
- `total`, `sold`, `available`, `capacity` — inventory contract fields
- `benefits`, `refundPolicy`, `perUserLimit` — immutable contract fields
- QR code, barcode, seat assignment data
- `ownerId` on the `public` channel

### 7.2 Declared Exception — `order:refund` refundAmount

The `order:refund` event MAY include `refundAmount` as an additional field.

**Rationale:** Core Rules §3.4 and §6 require refunds to be observable and
explicit. A customer receiving `order:refund` without the refund amount has
no way to verify their refund without navigating to a separate API call.
Since `order:refund` goes ONLY to the private `user:{userId}` channel, the
field is never exposed to shared channels or logs.

**Constraint:** `refundAmount` is the ONLY financial field permitted in any
SSE payload. Its presence is valid only in `order:refund` on `user:{userId}`.
Any other use of financial fields in SSE payloads violates SSE-003.

---

## 8. Invalidation Model

The following table defines which RTK Query cache domains each SSE event
MUST invalidate and MUST NOT invalidate.

| SSE Event | MUST Invalidate | MUST NOT Invalidate |
|-----------|----------------|---------------------|
| `EVENT_CREATED` | `'Event'` | `'Order'`, `'Ticket'`, `'TicketType'` |
| `EVENT_UPDATED` | `'Event'` | `'Order'`, `'Ticket'` |
| `EVENT_PUBLISHED` | `'Event'` | `'Order'`, `'Ticket'` |
| `EVENT_CANCELLED` | `'Event'` | `'Order'` (orders have their own events), `'Ticket'` |
| `EVENT_DELETED` | `'Event'` | `'Order'`, `'Ticket'` |
| `TICKET_TYPE_CREATED` | `'TicketType'` | `'Order'`, `'Ticket'`, `'Event'` (public) |
| `TICKET_TYPE_UPDATED` | `'TicketType'` | `'Order'`, `'Ticket'` |
| `TICKET_TYPE_ACTIVATED` | `'TicketType'`, `'Event'` | `'Order'`, `'Ticket'` |
| `TICKET_TYPE_DEACTIVATED` | `'TicketType'`, `'Event'` | `'Order'`, `'Ticket'` |
| `ORDER_CREATED` | `'Order'` | `'Event'`, `'TicketType'`, `'Ticket'` |
| `ORDER_CONFIRMED` | `'Order'`, `'Ticket'` | `'Event'`, `'TicketType'` |
| `ORDER_CANCELLED` | `'Order'` | `'Ticket'`, `'Event'`, `'TicketType'` |
| `ORDER_EXPIRED` | `'Order'` | `'Ticket'`, `'Event'`, `'TicketType'` |
| `ORDER_REFUNDED` | `'Order'` | `'Ticket'`, `'Event'`, `'TicketType'` |
| `TICKET_ISSUED` | `'Ticket'` | `'Order'`, `'Event'` |
| `TICKET_CHECKED_IN` | `'Ticket'` | all others |
| `ORGANIZATION_*` | `'Organizer'` | all others |
| `INVITATION_*` | `'Invitation'`, `'OrganizationMember'`, `'Organizer'` | `'Event'`, `'Order'` |
| `CATEGORY_*` | `'Category'` | all others |
| `VENUE_*` | `'Venue'` | all others |
| `FLEXPASS_LIST` | `'FlexPassListing'` | `'Order'`, `'Ticket'`, `'Event'`, `'TicketType'` |
| `FLEXPASS_APPROVE` | `'FlexPassListing'` | `'Order'`, `'Ticket'`, `'Event'` |
| `FLEXPASS_REJECT` | `'FlexPassListing'` | `'Order'`, `'Ticket'`, `'Event'` |
| `FLEXPASS_SOLD` | `'FlexPassListing'`, `'Ticket'` | `'Order'`, `'Event'`, `'TicketType'` |
| `FLEXPASS_EXPIRE` | `'FlexPassListing'` | `'Order'`, `'Event'`, `'TicketType'` |
| `FLEXPASS_CANCEL` | `'FlexPassListing'` | `'Order'`, `'Ticket'`, `'Event'` |

**Key principle for this table:**
The frontend cannot determine TicketType state (DRAFT vs ACTIVE) from the
SSE payload. Status-gating is enforced at the backend channel selection level
(SSE-002 revised). A `TICKET_TYPE_CREATED` event arriving at the frontend via
`organizer,admin` channel means the TicketType is DRAFT — no public Event cache
invalidation needed. A `TICKET_TYPE_ACTIVATED` event arriving via
`public,organizer` means the TicketType became visible to customers — both
`'TicketType'` and `'Event'` caches should be invalidated so customers see the
new offering.

---

## 9. Normative Rules

Each rule has an ID, a normative statement, rationale, and examples.
Rules MUST be testable and reviewable.

---

### SSE-001: DRAFT events MUST NOT emit to the public channel

**Normative statement:**
`notifyEventCreated()` and `notifyEventUpdated()` MUST select channels based
on event status. The routing rule is:

- `PUBLISHED`, `ONGOING`, or `COMPLETED` → `"public,organizer"`
- `DRAFT` or `CANCELLED` → `"organizer,admin"`

**Why:** Core Rule §1.1 — Customers may NOT access unpublished data.
Emitting to `public` for a DRAFT event triggers customer RTK Query cache
invalidation. ONGOING and COMPLETED events remain publicly visible and
customers actively viewing them must receive real-time updates. CANCELLED
events are immutable; only the `event:cancel` transition event is public
(see SSE-004), not subsequent updates.

**Applies to:** `EventService.java`, `SSENotificationService.java`

**Compliant:**
```java
boolean isPubliclyVisible = "PUBLISHED".equals(eventStatus)
    || "ONGOING".equals(eventStatus)
    || "COMPLETED".equals(eventStatus);
String channel = isPubliclyVisible ? "public,organizer" : "organizer,admin";
emitEvent("event:update", channel, "Event updated: " + eventName, data);
```

**Forbidden:**
```java
// Omits ONGOING/COMPLETED — attendees watching live events lose real-time updates
String channel = "PUBLISHED".equals(eventStatus) ? "public,organizer" : "organizer,admin";
// Status not checked — always goes to public regardless of DRAFT state
emitEvent("event:create", "public,organizer", "Event created", data);
```

---

### SSE-002: DRAFT TicketTypes MUST NOT emit to the public channel

**Normative statement:**
Channel routing for TicketType SSE events is determined by **TicketType state**,
not by parent event state.

| TicketType state | Channel |
|---|---|
| `DRAFT` (any parent event state) | `"organizer,admin"` |
| `ACTIVE` on PUBLISHED event | `"public,organizer"` |
| `ACTIVE` on DRAFT event | `"organizer,admin"` |
| `DEACTIVATED` on PUBLISHED event | `"public,organizer"` |
| `DEACTIVATED` on DRAFT event | `"organizer,admin"` |

The DRAFT → ACTIVE transition MUST emit `ticket_type:activate` to
`"public,organizer"` (if parent event is PUBLISHED), not `ticket_type:create`.
Creating a TicketType NEVER emits to the public channel.

**Why:** Core Rule §4.2, §1.1. DRAFT TicketTypes are not visible to customers
regardless of parent event state. Broadcasting DRAFT TicketType changes to
`public` would trigger cache invalidation for content customers are not
authorized to see.

**Applies to:** `TicketTypeService.java`, `SSENotificationService.java`

**Compliant:**
```java
// On create: always organizer/admin
emitEvent("ticket_type:create", "organizer,admin", "Ticket type created", data);

// On activate: public if parent event is PUBLISHED
String channel = event.isPublished() ? "public,organizer" : "organizer,admin";
emitEvent("ticket_type:activate", channel, "Ticket type activated", data);

// On update: depends on TicketType state
String channel = ticketType.isActive() && event.isPublished()
    ? "public,organizer" : "organizer,admin";
emitEvent("ticket_type:update", channel, "Ticket type updated", data);
```

**Forbidden:**
```java
// Routing by parent event state only — ignores TicketType state
String channel = event.isPublished() ? "public,organizer" : "organizer,admin";
emitEvent("ticket_type:create", channel, "Ticket type created", data);
// ↑ Wrong: a DRAFT TicketType on a PUBLISHED event would go to public
```

---

### SSE-003: SSE payloads MUST be minimal on all channels

**Normative statement:**
SSE event `data` fields MUST contain only the fields defined in §7.1 for
the event class. Full entity DTOs MUST NOT be passed as SSE payload on any
channel. This applies to organizer, admin, and public channels equally.

The only declared exception is `refundAmount` in `order:refund` (see §7.2).

**Why:** [P-1] SSE is a cache-invalidation signal. The REST API is the
authoritative data source. Sending full DTOs violates separation of concerns,
creates PII risk in logs, and causes payload bloat (>4KB violates HTTP/2 limits).

**Applies to:** `SSENotificationService.java`

**Compliant:**
```java
Map<String, Object> data = new HashMap<>();
data.put("eventId", eventId.toString());
data.put("eventName", eventName);
data.put("organizationId", organizationId);
emitEvent("event:create", channel, "Event created", data);
```

**Forbidden:**
```java
// Passing full EventResponse DTO — includes title, description, dates, venue, etc.
sseNotificationService.notifyEventCreated(orgId, fullEventResponseObject);
```

---

### SSE-004: event:cancel MUST only be emitted on PUBLISHED → CANCELLED transition

**Normative statement:**
The backend MUST emit `event:cancel` only when the event's current state is
PUBLISHED and is transitioning to CANCELLED. The state machine enforces
DRAFT → PUBLISHED → CANCELLED; the SSE layer MUST respect this guard.
A DRAFT event MUST NOT emit `event:cancel`.

**Why:** Core Rule §3.4 defines the CANCELLED lifecycle: "No new bookings
allowed, all paid bookings must be refunded, event becomes immutable."
These consequences apply only to events that were PUBLISHED (which could
have active bookings). Emitting `event:cancel` for a DRAFT event to the
`public` channel would incorrectly suggest to customers that a bookable
event was cancelled.

**Applies to:** `EventService.java`

**Note:** This rule validates the precondition for the `event:cancel` emit.
The emit itself (`public,organizer`) is correct for this transition.

---

### SSE-005: Refund lifecycle MUST be observable via order:refund SSE

**Normative statement:**
When a paid order is refunded (whether triggered by event cancellation or
user request), the backend MUST emit `order:refund` to `user:{userId}` AFTER
the refund is committed to the database.

The payload MUST include: `orderId`, `eventId`, `eventName`.
The payload MAY include: `refundAmount` (see §7.2 — declared exception).

No `order:refund` SSE MUST be emitted for orders that were not in
CONFIRMED status at the time of refund.

**Why:** Core Rule §3.4 — All paid bookings must be refunded when an event
is cancelled. Core Rule §6 — Refunds must be explicit actions. Without a
real-time SSE notification, a customer who has the app open will not know
their refund has been initiated without polling.

**Applies to:** `SSENotificationService.java`, `sse-service/config/actions.json`,
`frontend/src/stores/types/sse.ts`, `SSEProvider.tsx`

**Compliant:**
```java
// After DB commit of refund record:
sseNotificationService.notifyOrderRefunded(
    orderId, userId, eventId, eventName, refundAmount);
// Emits to user:{userId} only
```

**Forbidden:**
- Emitting `order:refund` before the refund record is committed to DB
- Emitting `order:refund` to any channel other than `user:{userId}`
- Triggering refund via SSE (SSE is notification only; refund logic must be backend-driven)

---

### SSE-006: Invitation creation MUST notify the invitee on their personal channel

**Normative statement:**
`notifyInvitationCreated()` MUST emit to TWO channels simultaneously:
1. `organizer` — for badge/list refresh for the inviting org's members
2. `user:{inviteeId}` — so the invitee receives a real-time toast notification

**Why:** The invitee is the primary actor for an invitation. Broadcasting only
to `organizer` means the invitee's pending badge updates via periodic cache
invalidation, but they receive no immediate personal signal. The `user:{inviteeId}`
emit bridges this gap.

**Applies to:** `SSENotificationService.java`, `sse-service/config/channels.json`
(must allow `invitation` resource on `user:{id}` pattern)

**Note:** This requires the inviteeId (UUID) to be resolved at call time in
`OrganizationMemberService`. The current implementation emits to `organizer` only.

---

### SSE-007: Order and ticket SSE events MUST ONLY go to user:{userId}

**Normative statement:**
`order:create`, `order:confirm`, `order:cancel`, `order:expire`, `order:refund`,
`ticket:issue`, and `ticket:checkin` MUST be emitted exclusively to
`user:{userId}`. These events MUST NEVER go to `public`, `organizer`, or `admin`.

**Why:** Core Rules §5, §6, §8. Order and ticket data contains PII and
financial history. Broadcasting to shared channels would expose user purchasing
behavior, order IDs, and attendance patterns to unrelated users.

**Applies to:** `SSENotificationService.java`

---

### SSE-008: SSE emission MUST occur after transaction commit

**Normative statement:**
All `emitEvent()` calls MUST be placed AFTER the enclosing database transaction
has been successfully committed. If an `@Transactional` method emits SSE
inside the transaction body and the transaction later rolls back, the SSE
event has been sent for a state change that was reverted.

**Why:** [P-4] SSE delivery is not guaranteed, but SSE correctness requires
that when an event IS delivered, it reflects true committed state. A client
that refetches on an SSE signal before the DB commit completes will read stale
or pre-commit state, creating a race condition.

**Applies to:** All service methods in `EventService.java`, `OrderService.java`,
`TicketTypeService.java`, `OrganizationService.java`

**Recommended pattern:**
```java
@Transactional
public void createEvent(...) {
    Event saved = eventRepository.save(event);
    // DO NOT emit here — transaction not yet committed
    return mapper.toResponse(saved);
}

// Caller or @TransactionalEventListener:
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void afterCreate(EventCreatedEvent event) {
    sseNotificationService.notifyEventCreated(...);
}
```

**Alternatively:** Call `notifyEventCreated()` immediately after the
`@Transactional` method returns (in the controller layer), ensuring the
transaction has committed before SSE fires.

---

### SSE-009: ORDER_CONFIRMED MUST NOT invalidate Event or TicketType cache

**Normative statement:**
When `ORDER_CONFIRMED` is received in SSEProvider, the frontend MUST invalidate
only `['Order']` and `['Ticket']`. It MUST NOT invalidate `'Event'` or
`'TicketType'` tags.

The same restriction applies to `ORDER_CANCELLED` and `ORDER_EXPIRED`.

**Why:** Core Rule §5 — Bookings MUST NEVER depend on live Event data.
Invalidating Event and TicketType on ORDER_CONFIRMED causes the customer's
order detail page to refetch live event state. If the event was updated since
the order was created, the customer sees inconsistent data next to their
snapshot-based order items. This violates the snapshot integrity guarantee.

**Applies to:** `frontend/src/providers/SSEProvider.tsx`

**Compliant:**
```typescript
case SSENormalizedType.ORDER_CONFIRMED:
  dispatch(OrderAPI.util.invalidateTags(['Order']));
  dispatch(OrderAPI.util.invalidateTags(['Ticket']));
  break;
```

**Forbidden:**
```typescript
case SSENormalizedType.ORDER_CONFIRMED:
  dispatch(OrderAPI.util.invalidateTags(['Order']));
  dispatch(TicketTypeAPI.util.invalidateTags([...])); // FORBIDDEN
  dispatch(EventAPI.util.invalidateTags([...]));      // FORBIDDEN
  break;
```

---

### SSE-010: Order cache MUST only be invalidated by order:* events

**Normative statement:**
RTK Query's `'Order'` cache tag MUST only be invalidated in response to
`ORDER_CREATED`, `ORDER_CONFIRMED`, `ORDER_CANCELLED`, `ORDER_EXPIRED`, or
`ORDER_REFUNDED` events. No other SSE event type MAY trigger Order cache
invalidation.

**Why:** Core Rule §8 — The system MUST NEVER modify paid booking data silently.
Invalidating the Order cache on unrelated events (e.g., event updates, ticket
type changes) creates refetch races that can cause the UI to briefly show stale
or incorrect order state, potentially confusing customers about what they paid.

**Applies to:** `frontend/src/providers/SSEProvider.tsx`

---

### SSE-011: TicketType deactivation MUST NOT affect paid order display

**Normative statement:**
When `TICKET_TYPE_DEACTIVATED` is received:
1. The frontend MUST invalidate `'TicketType'` and `'Event'` caches (to
   update purchase availability UI).
2. The frontend MUST NOT invalidate `'Order'` cache.
3. The order detail page MUST render `item.unitPrice` and `item.subtotal`
   from the order's stored snapshot, not from any re-fetched TicketType data.

**Why:** Core Rules §4.1, §5, [P-5]. A deactivated TicketType remains valid
for existing bookings, check-in, refund, and reporting. The customer's paid
order detail must show what they actually paid, not the current state of the
ticket type.

**Applies to:** `frontend/SSEProvider.tsx`, order detail page components

---

### SSE-012: Public event queries MUST apply a PUBLISHED status filter

**Normative statement:**
Any RTK Query endpoint that fetches events for the public/customer view MUST
include a status filter limiting results to `PUBLISHED` events. The backend
endpoint MUST also enforce this filter server-side, regardless of frontend
parameters.

The comment `// No status filter - show all events to customers` MUST be
removed from `EventApi.ts`. This comment documents a Core Rule violation.

**Why:** Core Rule §1.1 — Customers may NOT access unpublished data. An
unfiltered public event list may return DRAFT events if the backend does not
enforce the filter, allowing customers to view, and potentially attempt to
book, events not ready for public access.

**Applies to:**
- `frontend/src/stores/services/EventApi.ts` (`getPublicEvents` endpoint)
- `backend/EventController.java` `GET /events` endpoint

**Compliant:**
```typescript
getPublicEvents: builder.query({
  query: (params = {}) => ({
    url: '/events',
    params: { ...params, status: 'PUBLISHED' },
  }),
}),
```

---

### SSE-013: SSE MUST NOT be the sole persistence mechanism for state changes

**Normative statement:**
Every state change signaled by SSE MUST already be committed to the database
before the SSE is emitted (see SSE-008). If SSE delivery fails — due to
queue overflow, reconnect, or service restart — the client MUST be able to
recover the correct state by calling the REST API.

The REST API is the single source of truth. SSE is an optimization.

**Why:** [P-4] SSE delivery is not guaranteed. Core Rule §9 — Silent failures
or auto-fixes are not allowed. A customer who misses `ORDER_EXPIRED` must
still see `status: EXPIRED` on the next page load.

**Applies to:** All service methods that call `notify*()` methods.

---

### SSE-014: Cache invalidation MUST have a single authoritative location

**Normative statement:**
`SSEProvider.tsx` is the single location responsible for all RTK Query cache
invalidation driven by SSE events. No other hook, component, or page MAY
call `dispatch(XxxAPI.util.invalidateTags(...))` in response to SSE events.

If a hook or component uses `useSSE()` / `lastEvent`, it MUST NOT also
perform cache invalidation. It MAY trigger a `refetch()` call for a specific
query, subject to all three of the following constraints:

1. **Entity-scoped only.** The `refetch()` call MUST target a query for the
   specific entity affected by the event (e.g., `getOrderById(orderId)`).
   A component-level `refetch()` MUST NOT be used as a substitute for the
   §8 invalidation table. If the required invalidation belongs in §8, it
   belongs in `SSEProvider.tsx` — not in the component.

2. **`affectsThisEntity` guard required.** The call MUST be wrapped in a
   check that the event targets the entity currently rendered by this
   component. Without this guard, every broadcast triggers a refetch on
   every mounted instance of the component (broadcast storm).

3. **Rule citation required.** The code MUST include a comment citing this
   rule ID (`// SSE-014 exception`) and briefly stating why global
   invalidation does not cover this case. A PR description note is also
   required if the exception is non-obvious.

**Why:** [P-1] Duplicate invalidation causes double refetches, race conditions,
and makes SSE behavior unpredictable. A single source of truth for cache
management is essential for debugging and correctness.

**Applies to:** `SSEProvider.tsx`, any file importing `useSSE()`

---

### SSE-015: Private identifiers MUST NOT appear in public channel payloads

**Normative statement:**
`ownerId`, `userId`, and other per-user identifiers MUST NOT appear in
payloads sent to the `public` channel. They MAY appear in `organizer`,
`admin`, or `user:{id}` channels where the audience is known and limited.

**Why:** Broadcasting user UUIDs on the public channel would allow any
subscriber to correlate user identities with organization ownership or
ticket purchases.

**Applies to:** `SSENotificationService.java`

---

### SSE-016: Reconnect MUST NOT corrupt client state

**Normative statement:**
When SSEProvider reconnects after a connection drop (via the `onerror` →
`setupEventSource` retry loop), the client MUST NOT assume missed events
will be replayed. The SSE service does not support event replay or cursor
resumption.

On reconnect, SSEProvider MUST treat user-private state as potentially stale
and SHOULD mark the `'Order'` cache as stale for customers and the `'Organizer'`
cache as stale for organizers. It MUST NOT perform a full wipe of shared
public caches (`'Event'`, `'Category'`, `'Venue'`) on every reconnect, as
this causes unnecessary network traffic for all users.

**Why:** [P-4]. A customer whose order status changed from PENDING to CONFIRMED
while SSE was disconnected must see the correct status when they navigate or
the cache is next used. The REST API is authoritative; a controlled, scoped
invalidation on reconnect brings the client back to known-good state.

**Applies to:** `frontend/src/providers/SSEProvider.tsx`

---

### SSE-017: Channel selection MUST match entity access scope

**Normative statement:**
When choosing which channel to emit to, the deciding factor MUST be "who is
authorized by Core Rules to know about this change?" — not "which channel is
technically configured for this action type."

The SSE service validates that an action is *allowed* in a channel (config
validity). It does NOT validate whether the entity's current state makes it
appropriate to broadcast to that audience (business visibility). The backend
service layer is responsible for this second check.

**Mapping:**

| Authorized audience | Correct channel |
|---------------------|-----------------|
| All users (PUBLISHED events) | `public` |
| Organizers and admin only | `organizer,admin` |
| Admin only | `admin` |
| Specific user | `user:{userId}` |
| Specific owner + all organizers | `organizer,admin` + `user:{ownerId}` |

**Applies to:** All `notify*()` methods in `SSENotificationService.java`

---

### SSE-018: FlexPass listing events MUST go to organizer,admin — never public

**Normative statement:**
`flexpass:list` and `flexpass:cancel` MUST be emitted exclusively to
`"organizer,admin"`. These events signal that a listing is awaiting review.
They MUST NOT be emitted to the `public` channel at any listing state.

The resale market visibility for customers is determined by the REST API
(which returns only `APPROVED` listings). The SSE layer MUST NOT expose
listing existence to the public channel before organizer approval.

**Why:** Core Rule §11.4 — a listing is in `PENDING` state when created and
is not yet buyer-visible. Broadcasting to `public` would leak the existence
of unreviewed resale listings to all customers. Core Rule §11.7 forbids
emitting FlexPass listing events to the public channel.

**Applies to:** `FlexPassService.java`, `SSENotificationService.java`

**Compliant:**
```java
sseNotificationService.notifyFlexPassListing(listingId, ticketId, eventId, eventName, "organizer,admin");
```

**Forbidden:**
```java
// Wrong: listing not yet approved — must not go to public
sseNotificationService.notifyFlexPassListing(listingId, ticketId, eventId, eventName, "public,organizer");
```

---

### SSE-019: FlexPass personal outcome events MUST go only to user:{userId}

**Normative statement:**
`flexpass:approve`, `flexpass:reject`, `flexpass:expire`, and `flexpass:sold`
MUST be emitted exclusively to the specific user's private channel:
- `flexpass:approve` → `user:{sellerId}`
- `flexpass:reject` → `user:{sellerId}`
- `flexpass:expire` → `user:{sellerId}`
- `flexpass:sold` → two emits: `user:{sellerId}` AND `user:{buyerId}` (separate calls)

None of these events MUST go to `public`, `organizer`, or `admin`.

The `flexpass:expire` event MAY include `refundAmount` (same declared-exception
rationale as `order:refund` in §7.2 — private channel, financial observability
required for the affected user).

`flexpass:expire` MUST be followed by a separate `order:refund` emit after the
refund record is committed. They are two distinct events (SSE-005 applies to
the refund emit).

**Why:** Core Rule §11.6 — the refund and transfer outcomes are personal
financial events. Broadcasting them to shared channels would expose a user's
resale activity and financial amounts to unrelated parties. Core Rule §11.7
forbids emitting FlexPass transfer/refund events to any shared channel.

**Applies to:** `SSENotificationService.java`, `FlexPassService.java`

**Compliant:**
```java
// On approval:
emitEvent("flexpass:approve", "user:" + sellerId, "Listing approved", data);

// On sold — two separate emits:
emitEvent("flexpass:sold", "user:" + sellerId, "Your ticket was sold", sellerData);
emitEvent("flexpass:sold", "user:" + buyerId,  "Transfer complete",   buyerData);

// On expire — with refundAmount:
emitEvent("flexpass:expire", "user:" + sellerId, "Listing expired", dataWithRefundAmount);
// Then, after refund record committed:
sseNotificationService.notifyOrderRefunded(orderId, sellerId, eventId, eventName, refundAmount);
```

**Forbidden:**
```java
// Wrong: outcome events must not go to shared channels
emitEvent("flexpass:approve", "organizer", ...);
emitEvent("flexpass:sold",    "public",    ...);
```

---

## 10. Forbidden Behaviors

The following behaviors MUST NOT exist anywhere in the system:

1. **FORBIDDEN:** Emitting `event:create` or `event:update` to `public` when
   event status is DRAFT. (SSE-001)

2. **FORBIDDEN:** Emitting any `ticket_type:*` event to `public` when the
   TicketType state is `DRAFT`, regardless of parent event state. (SSE-002)

3. **FORBIDDEN:** Emitting `ticket_type:create` to `public,organizer` — create
   always goes to `organizer,admin`. Only `ticket_type:activate` goes to
   `public,organizer` (when parent event is PUBLISHED). (SSE-002)

4. **FORBIDDEN:** Passing a full entity DTO as SSE payload on any channel. (SSE-003)

5. **FORBIDDEN:** Including price, currency, capacity, or inventory fields
   in any SSE payload, except `refundAmount` in `order:refund` on `user:{userId}`. (SSE-003, §7.2)

6. **FORBIDDEN:** Emitting `order:*` or `ticket:*` events to any channel
   other than `user:{userId}`. (SSE-007)

7. **FORBIDDEN:** Invalidating `'Order'` cache in response to Event or
   TicketType SSE events. (SSE-010)

8. **FORBIDDEN:** Invalidating `'Event'` or `'TicketType'` cache in response
   to `ORDER_CONFIRMED`, `ORDER_CANCELLED`, or `ORDER_EXPIRED`. (SSE-009)

9. **FORBIDDEN:** Emitting SSE before the enclosing DB transaction commits. (SSE-008)

10. **FORBIDDEN:** Using SSE as the trigger or mechanism for executing refunds;
    refunds MUST be backend-driven business logic. SSE notifies the outcome. (SSE-005)

11. **FORBIDDEN:** Performing cache invalidation in any hook or component other
    than SSEProvider, unless it is a targeted `refetch()` with `affectsThisEntity`
    guard and is documented. (SSE-014)

12. **FORBIDDEN:** Exposing `userId`, `ownerId`, or other user identifiers in
    `public` channel payloads. (SSE-015)

13. **FORBIDDEN:** Emitting `event:cancel` for an event that was never
    PUBLISHED. (SSE-004)

14. **FORBIDDEN:** Emitting individual `ticket_type:deactivate` signals for
    TicketTypes that were cascade-deactivated as a result of event cancellation.
    The cascade is an internal operation; only `event:cancel` is emitted. (SSE-004, §3.5)

15. **FORBIDDEN:** Invalidating `'Event'` public cache in response to
    `TICKET_TYPE_CREATED` — a newly created TicketType is DRAFT and not visible
    to customers. Only `TICKET_TYPE_ACTIVATED` triggers public Event invalidation. (SSE-002, §8)

16. **FORBIDDEN:** Emitting `flexpass:list` or `flexpass:cancel` to the `public`
    channel. Pending listings are not buyer-visible. (SSE-018)

17. **FORBIDDEN:** Emitting `flexpass:approve`, `flexpass:reject`, `flexpass:expire`,
    or `flexpass:sold` to any shared channel (`public`, `organizer`, `admin`).
    These are personal outcome events. (SSE-019)

18. **FORBIDDEN:** Emitting a single `flexpass:sold` to a merged audience — seller
    and buyer MUST receive separate emits on their own `user:{id}` channels. (SSE-019)

19. **FORBIDDEN:** Including `resalePrice`, `originalPrice`, or any financial field
    in `flexpass:list`, `flexpass:approve`, `flexpass:reject`, or `flexpass:sold`
    payloads. Only `flexpass:expire` MAY carry `refundAmount` on `user:{id}`. (SSE-019, §7.2)

20. **FORBIDDEN:** Emitting `flexpass:expire` before the expiry is committed to the
    database, or emitting `order:refund` before the refund record is committed. (SSE-008)

---

## 11. Transaction & Delivery Guarantees

1. **Emit-after-commit (hard requirement, SSE-008):** SSE MUST NOT fire
   inside an uncommitted `@Transactional` block. Use
   `@TransactionalEventListener(phase = AFTER_COMMIT)` or place the emit
   call in the controller layer after the transactional service method returns.

2. **No delivery guarantee:** SSE events may be dropped due to queue overflow
   (SSE service queue maxsize=100 per connection), SSE service restart, or
   client reconnect gap. The system MUST remain correct without SSE delivery.

3. **No replay:** The SSE service does not support event replay from a cursor.
   Clients that reconnect will not receive events missed during the disconnect.

4. **Idempotency:** Receiving the same SSE event twice MUST NOT corrupt UI
   state. RTK Query cache invalidation is naturally idempotent.

5. **No ordering guarantee:** SSE events carry no sequence number. The frontend
   MUST NOT depend on event arrival order. State must always be reconstructed
   from the REST API response, not inferred from SSE event sequence.

---

## 12. Reconnect & Resync Guarantees

1. SSEProvider implements per-channel reconnect with `RECONNECT_DELAY_MS = 3000`.
   This covers transient network drops.

2. On reconnect, SSEProvider SHOULD perform targeted invalidation of the
   user's private state:
   - Invalidate `'Order'` (for all roles — any pending orders may have changed)
   - Invalidate `'Organizer'` (for ORGANIZER role — org list may have changed)

3. SSEProvider MUST NOT wipe shared public caches (`'Event'`, `'Category'`,
   `'Venue'`, `'TicketType'`) on reconnect. These are large datasets; a blanket
   wipe causes visible UI flicker and unnecessary load on the backend.

4. Reconnect does not constitute a state reset. The client must navigate to
   reload critical views if it suspects stale data after a prolonged disconnect.

---

## 13. Security & Privacy Constraints

1. **No auth token in SSE URL.** EventSource does not support HTTP headers.
   The SSE URL does not include a JWT. The security boundary is the channel
   name. The REST API enforces access control on all subsequent data fetches.

2. **Private channel names use UUIDs for entropy, not authorization.**
   `user:{userId}` channel names are non-guessable by construction (UUID v4),
   which reduces the practical risk of accidental channel collision. However,
   UUID entropy is NOT an authorization mechanism. A client that somehow
   learns another user's UUID and subscribes to their channel would receive
   that user's SSE signals — this is itself a security problem, not a
   recoverable situation handled by the REST API.

   The real authorization boundary for the `user:{id}` channel (i.e., the
   mechanism that prevents arbitrary clients from subscribing to any channel)
   depends on the SSE service's subscription enforcement. **This boundary is
   currently an open question:** it is not verified in this spec whether the
   SSE service enforces that only the authenticated user matching `{id}` may
   subscribe to `user:{id}`. Implementors MUST NOT assume the UUID name alone
   provides authorization. If subscription enforcement is absent, it MUST be
   treated as a known security gap and tracked for remediation.

3. **Backend is the authoritative security layer.** SSE is advisory.

4. **No financial data in logs.** `SSENotificationService` logs `type`,
   `channel`, and `message`. The `data` object MUST NOT be logged in production.
   `refundAmount` in `order:refund` MUST NOT appear in log output.

5. **Payload size limit.** SSE payloads MUST NOT exceed 4KB. Full DTOs
   typically exceed this limit and MUST be rejected at the call site.

---

## 14. Testing Requirements

| Test | Layer | Assertion |
|------|-------|-----------|
| DRAFT event → `organizer,admin` only | Backend integration | `EventService.createEvent()` → assert channel is `"organizer,admin"` |
| PUBLISHED event → `public,organizer` | Backend integration | `EventService.publishEvent()` → assert channel is `"public,organizer"` |
| TicketType on DRAFT → `organizer,admin` | Backend integration | `TicketTypeService.createTicketType(draftEvent)` → assert channel is `"organizer,admin"` |
| ORDER_CONFIRMED does not invalidate Event | Frontend unit | `SSEProvider` receives ORDER_CONFIRMED → `EventAPI.invalidateTags` is NOT called |
| ORDER_CONFIRMED does not invalidate TicketType | Frontend unit | Same event → `TicketTypeAPI.invalidateTags` is NOT called |
| Order detail shows snapshot price | Frontend integration | After TicketType price update, order detail still shows original `item.unitPrice` |
| `getPublicEvents` sends status=PUBLISHED | Frontend unit | RTK Query request includes `status=PUBLISHED` parameter |
| Backend GET /events enforces PUBLISHED filter | Backend integration | GET /events returns no DRAFT events |
| No duplicate invalidation | Frontend unit | SSEProvider and no other hook calls `invalidateTags` for same SSE event |
| Emit occurs after commit | Backend integration | SSE service receives event only after DB transaction visible in second connection |

---

## 15. Notes & Open Questions

### N-1: Refund workflow implementation status
`order:refund` SSE requires a refund workflow to exist in the backend.
The current codebase does not have a `RefundService` or `order:refund` action.
This rule is normative but cannot be implemented until the refund domain is built.
See openspec change `2026-03-10-sse-integrity-compliance` task M-008.

### N-2: useSSESync.ts audit required
An audit agent identified `frontend/src/hooks/useSSESync.ts` as a possible
source of duplicate cache invalidation (SSE-014 concern). This file must be
read and audited. If it duplicates SSEProvider invalidation, it must be
deleted or converted to a read-only status hook.

### N-3: Backend GET /events endpoint filter
The backend `/events` endpoint's server-side PUBLISHED filter has not been
confirmed. This must be verified before SSE-012 can be considered fully closed.
See blocker B-1 in openspec.

### N-4: Invitation invitee UUID resolution
SSE-006 requires `notifyInvitationCreated()` to emit to `user:{inviteeId}`.
The current service receives `inviteeEmail` only. The `inviteeId` (UUID) must
be resolved from the email at the service layer. This is a prerequisite for
implementing SSE-006.

### N-5: SSE-008 emit-after-commit implementation approach
The simplest implementation is to move `notify*()` calls from inside
`@Transactional` service methods to the controller layer, after the service
method returns. This guarantees the transaction has committed. Alternatively,
`@TransactionalEventListener(phase = AFTER_COMMIT)` can be used if domain
events are introduced.

---

## 16. [NON-NORMATIVE] Current Implementation Gap Summary

_This section records the compliance status as of 2026-03-12. It is
non-normative and will be updated as gaps are closed. Do NOT use this
section as a substitute for the normative rules above._

| Rule | Status | Notes |
|------|--------|-------|
| SSE-001 | **COMPLIANT** | Fixed: `EventService` + `SSENotificationService.notifyEventCreated/Updated()` now route by event status |
| SSE-002 | **PARTIALLY COMPLIANT** | Fixed: parent-event routing done. Pending: TicketType state routing (DRAFT TT on PUBLISHED event → `organizer,admin`) — requires `ticket_type:activate` action |
| SSE-003 | **COMPLIANT** | Fixed: `notifyEventCreated()` now takes minimal params; no full DTOs |
| SSE-004 | **COMPLIANT** | EventService state machine enforces PUBLISHED precondition |
| SSE-005 | **NOT IMPLEMENTED** | No `order:refund` event, no refund flow, no SSEAction, no SSEProvider handler |
| SSE-006 | **NOT IMPLEMENTED** | `notifyInvitationCreated()` emits to `organizer` only; no `user:{inviteeId}` emit |
| SSE-007 | **COMPLIANT** | All `notifyOrder*` and `notifyTicket*` methods use `user:{userId}` exclusively |
| SSE-008 | **UNKNOWN** | Must verify that all `notify*()` calls are outside `@Transactional` boundaries |
| SSE-009 | **COMPLIANT** | Fixed: `SSEProvider.tsx` no longer invalidates `EventAPI`/`TicketTypeAPI` on `ORDER_CONFIRMED` |
| SSE-010 | **COMPLIANT** | Order cache only invalidated by `ORDER_*` events in SSEProvider |
| SSE-011 | **COMPLIANT** | TicketType deactivation does not invalidate Order cache |
| SSE-012 | **COMPLIANT** | Fixed: `EventApi.ts getPublicEvents` now sends `status=PUBLISHED` filter |
| SSE-013 | **COMPLIANT** | OrderExpiryScheduler updates DB before emitting; REST API returns correct state |
| SSE-014 | **UNKNOWN** | `useSSESync.ts` may duplicate invalidation logic; requires audit (N-2) |
| SSE-015 | **COMPLIANT** | `ownerId` does not appear in public channel payloads |
| SSE-016 | **UNKNOWN** | Reconnect behavior not verified in SSEProvider |
| SSE-017 | **PARTIALLY COMPLIANT** | DRAFT TT on PUBLISHED event routing not yet split; private channels correctly scoped |
| SSE-NEW-01 | **NOT IMPLEMENTED** | `ticket_type:activate` action does not exist yet in actions.json or backend |
| SSE-NEW-02 | **NOT IMPLEMENTED** | `TICKET_TYPE_ACTIVATED` not in `SSENormalizedType` enum or SSEProvider handler |
| SSE-018 | **NOT IMPLEMENTED** | FlexPass backend not built; no `flexpass:list`, `flexpass:cancel` actions or routing |
| SSE-019 | **NOT IMPLEMENTED** | No `flexpass:approve`, `flexpass:reject`, `flexpass:sold`, `flexpass:expire` actions; no SSEProvider handlers; no `FlexPassListing` RTK Query tag |

---

## Appendix: TicketType State × Event State → Channel Decision Table

Channel is determined by **TicketType state** (primary) and parent event state (secondary).

```
TicketType State | Parent Event State | Action              | Channel
─────────────────┼────────────────────┼─────────────────────┼──────────────────────
DRAFT            | DRAFT              | create              | organizer,admin  ✅
DRAFT            | DRAFT              | update              | organizer,admin  ✅
DRAFT            | DRAFT              | delete              | organizer,admin  ✅
DRAFT            | DRAFT              | deactivate(cascade) | organizer,admin  ✅
DRAFT            | PUBLISHED          | create              | organizer,admin  ✅ (new rule)
DRAFT            | PUBLISHED          | update              | organizer,admin  ✅ (new rule)
DRAFT            | PUBLISHED          | delete              | organizer,admin  ✅ (new rule)
DRAFT            | PUBLISHED          | deactivate(cascade) | organizer,admin  ✅ (new rule — no individual emit on event cancel)
DRAFT→ACTIVE     | DRAFT              | activate            | organizer,admin
DRAFT→ACTIVE     | PUBLISHED          | activate            | public,organizer  ← SSE-NEW-01 not implemented
ACTIVE           | PUBLISHED          | update (display)    | public,organizer  ✅
ACTIVE           | PUBLISHED          | deactivate          | public,organizer  ✅
ACTIVE           | DRAFT              | update              | organizer,admin
ACTIVE           | DRAFT              | deactivate          | organizer,admin
DEACTIVATED      | PUBLISHED          | (no further emits)  | —
```

## Appendix: Event State → Channel Decision Table

```
Entity State     | Action                        | Channel
─────────────────┼───────────────────────────────┼──────────────────────────
DRAFT event      | create                        | organizer,admin  ✅
DRAFT event      | update                        | organizer,admin  ✅
DRAFT event      | delete                        | organizer,admin  ✅
DRAFT→PUBLISHED  | publish                       | public,organizer ✅
PUBLISHED event  | update (metadata fields only) | public,organizer ✅
PUBLISHED→CANCEL | cancel (transition)           | public,organizer ✅
CANCELLED event  | refund (per order)            | user:{userId}    ← SSE-005 not implemented
SOLD_OUT event   | (no separate SSE — derived)   | —
Order (any)      | create                        | user:{userId}    ✅
Order (any)      | confirm                       | user:{userId}    ✅
Order (any)      | cancel/expire                 | user:{userId}    ✅
Invitation       | create                        | organizer + user:{inviteeId}  ← SSE-006 partial
```
