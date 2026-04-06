# Evena — Data Model Spec

**Version:** 1.0
**Date:** 2026-04-05
**Status:** approved
**Applies to:** backend · frontend · mobile

---

## Overview

All persistent entities use soft-delete / status flags rather than hard-delete where
financial or booking data may be involved (see `rules.md` §7).

Primary keys:
- User, Event: UUID (UUIDv7)
- All other entities: Long (auto-increment)

All entities extend `AuditableEntity` and carry `createdAt` / `updatedAt` timestamps
unless noted otherwise.

---

## 1. Auth Module

### 1.1 User

**Table:** `users`

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | UUID | PK | UUIDv7 generated |
| name | String | NOT NULL | |
| email | String | UNIQUE, NOT NULL | |
| password_hash | String | NOT NULL | bcrypt |
| phone | String | nullable | |
| status | UserStatus | NOT NULL, default=ACTIVE | see Enums §1 |
| avatar_url | String | nullable | S3/MinIO URL |
| email_verified | Boolean | default=false | |
| email_verification_token | String | nullable | cleared on verify |
| password_reset_token | String | nullable | |
| password_reset_expires_at | LocalDateTime | nullable | |
| created_at | LocalDateTime | auto | |
| updated_at | LocalDateTime | auto | |

**Relationships:**
- ManyToMany → `Role` via `user_roles` table (EAGER)
- OneToMany → `Organization` (owner)
- OneToMany → `OrganizationMember`
- OneToMany → `Order`
- OneToMany → `Ticket`
- OneToMany → `ScanLog` (scanned_by)

---

### 1.2 Role

**Table:** `roles`

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-increment |
| name | String | UNIQUE, NOT NULL |
| description | String | nullable |

Seed values: `USER`, `ORGANIZER`, `ADMIN`

---

### 1.3 PendingRegistration

**Table:** `pending_registrations`

Temporary row created during email-based registration flow.
Deleted when email is verified or token expires.

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-increment |
| name | String | NOT NULL |
| email | String | UNIQUE, NOT NULL |
| phone | String | NOT NULL |
| password_hash | String | NOT NULL |
| role_name | String | NOT NULL | `USER` or `ORGANIZER` |
| verification_token | String | UNIQUE, NOT NULL |
| expires_at | LocalDateTime | NOT NULL |
| created_at | LocalDateTime | auto |
| version | Long | optimistic lock |

---

## 2. Event Module

### 2.1 Organization

**Table:** `organizations`

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | Long | PK, auto-increment | |
| owner_user_id | UUID | FK → users.id, NOT NULL | |
| name | String | NOT NULL | Identity field (immutable after APPROVED) |
| description | String | TEXT, nullable | Non-identity |
| logo_url | String | nullable | Non-identity |
| website | String | nullable | Non-identity |
| email | String | nullable | Non-identity |
| phone | String | nullable | Non-identity |
| verified | Boolean | default=false | APPROVED if true |
| created_at | LocalDateTime | auto | |
| updated_at | LocalDateTime | auto | |

> **Note:** The `verified` field maps to `APPROVED` status conceptually.
> Full organization state machine (PENDING / APPROVED / ARCHIVED) is driven by
> `verified` flag + `ARCHIVED` soft-delete. See `rules.md` §2.

**Relationships:**
- ManyToOne → `User` (owner)
- OneToMany → `Event`
- OneToMany → `OrganizationMember` (cascade ALL, orphanRemoval)

---

### 2.2 OrganizationMember

**Table:** `organization_members`

**Unique constraint:** `(organization_id, user_id)`

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-increment |
| organization_id | Long | FK → organizations.id, NOT NULL |
| user_id | UUID | FK → users.id, NOT NULL |
| role | OrganizationRole | NOT NULL | see Enums §6 |
| invitation_accepted | Boolean | NOT NULL, default=false |
| created_at | LocalDateTime | auto |
| updated_at | LocalDateTime | auto |

---

### 2.3 Event

**Table:** `events`

**Indexes:** `organization_id`, `status`, `start_at`

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | UUID | PK | UUIDv7 |
| title | String | NOT NULL | Contractual — immutable after PUBLISHED |
| description | String | TEXT, nullable | Metadata — editable after PUBLISHED |
| start_at | LocalDateTime | NOT NULL | Contractual |
| end_at | LocalDateTime | NOT NULL | Contractual |
| status | EventStatus | NOT NULL, default=DRAFT | see Enums §2 |
| cover_url | String | nullable | Metadata |
| event_version | Integer | NOT NULL, default=1 | incremented on critical updates |
| organization_id | Long | FK → organizations.id, NOT NULL | Contractual |
| category_id | Long | FK → categories.id, NOT NULL | Contractual |
| venue_id | Long | FK → venues.id, NOT NULL | Contractual |
| created_at | LocalDateTime | auto | |
| updated_at | LocalDateTime | auto | |

**Relationships:**
- ManyToOne → `Organization` (LAZY)
- ManyToOne → `Category` (EAGER)
- ManyToOne → `Venue` (EAGER)
- OneToMany → `EventImage` (cascade ALL, orphanRemoval)
- OneToMany → `TicketType` (cascade ALL, orphanRemoval)

**Helper methods:**
- `isPublished()` — true if status ∈ {PUBLISHED, ONGOING, COMPLETED}
- `allowsTicketTypeModification()` — true only for DRAFT
- `incrementVersion()` — call on contractual field changes

**Field classification** (see `rules.md` §3.3):

| Field | Classification |
|---|---|
| title, startAt, endAt, venue, organization, category | Contractual — locked after PUBLISHED |
| description, coverUrl, images | Metadata — editable after PUBLISHED |

---

### 2.4 Category

**Table:** `categories`

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-increment |
| name | String | UNIQUE, NOT NULL |
| description | String | nullable |
| icon_url | String | nullable |

**Managed by:** Admin only

---

### 2.5 Venue

**Table:** `venues`

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-increment |
| name | String | NOT NULL |
| address | String | NOT NULL |
| city | String | NOT NULL |
| lat | Double | nullable |
| lng | Double | nullable |
| capacity | Integer | nullable |
| description | String | TEXT, nullable |

**Managed by:** Admin only

---

### 2.6 TicketType

**Table:** `ticket_types`

**Indexes:** `event_id`, `status`

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | Long | PK, auto-increment | |
| event_id | UUID | FK → events.id, NOT NULL | |
| name | String | NOT NULL | Editable at any time |
| description | String | TEXT, nullable | Editable at any time |
| price | BigDecimal | precision=10, scale=2, NOT NULL | Contractual — locked once ACTIVE |
| currency | String | default='VND' | Contractual — locked once ACTIVE |
| total | Integer | NOT NULL | Capacity — decrease below `sold` forbidden |
| sold | Integer | NOT NULL, default=0 | System-managed |
| per_user_limit | Integer | nullable | Max tickets per user |
| sales_start | LocalDateTime | nullable | Required for activation |
| sales_end | LocalDateTime | nullable | Required for activation |
| status | TicketTypeStatus | NOT NULL, default=DRAFT | see Enums §3 |
| early_bird | Boolean | default=false | |
| early_bird_discount | BigDecimal | nullable | |
| visible | Boolean | default=true | Controls public display |
| created_at | LocalDateTime | auto | |
| updated_at | LocalDateTime | auto | |

**Relationships:**
- ManyToOne → `Event` (LAZY)
- OneToMany → `OrderItem` (cascade ALL, orphanRemoval)

**Helper methods:**
- `getAvailable()` → total - sold
- `getSoldPercentage()` → (sold / total) × 100
- `isAvailable()` — checks availability and sales window
- `isSalesOpen()` — checks if sales are currently active
- `canUserPurchase(quantity)` — validates per-user limit

**Field lock rules** (see `rules.md` §4.3):

| Field | Locked once ACTIVE |
|---|---|
| price, currency, benefits, refundPolicy | Yes |
| total (decrease below sold) | Forbidden always |
| total (increase) | Permitted |
| name, description, visible | Never locked |

---

### 2.7 EventImage

**Table:** `event_images`

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-increment |
| event_id | UUID | FK → events.id, NOT NULL |
| url | String | NOT NULL |
| idx | Integer | NOT NULL |

---

### 2.8 EventFile

**Table:** `event_files`

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-increment |
| event_id | UUID | FK → events.id, NOT NULL |
| file_name | String | nullable |
| url | String | S3/MinIO URL |
| content_type | String | nullable |
| file_size | Long | nullable |
| uploaded_at | LocalDateTime | auto |

---

## 3. Order Module

### 3.1 Order

**Table:** `orders`

**Indexes:** `user_id`, `event_id`, `status`, `created_at`

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | Long | PK, auto-increment | |
| user_id | UUID | FK → users.id, NOT NULL | |
| event_id | UUID | NOT NULL | Denormalized for quick lookups |
| event_version | Integer | NOT NULL | Version at purchase time |
| event_snapshot | TEXT | NOT NULL | JSON blob — full event snapshot |
| status | OrderStatus | NOT NULL, default=PENDING | see Enums §4 |
| total_amount | BigDecimal | precision=10, scale=2, NOT NULL | |
| currency | String | default='VND' | |
| created_at | LocalDateTime | auto | |
| updated_at | LocalDateTime | auto | |

**Relationships:**
- ManyToOne → `User` (LAZY)
- OneToMany → `OrderItem` (cascade ALL, orphanRemoval)
- OneToMany → `Payment` (cascade ALL, orphanRemoval)

**Integrity rule:** Orders MUST store `event_snapshot` and never reference live
event data for display (see `rules.md` §5).

---

### 3.2 OrderItem

**Table:** `order_items`

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | Long | PK, auto-increment | |
| order_id | Long | FK → orders.id, NOT NULL | |
| ticket_type_id | Long | FK → ticket_types.id, NOT NULL | |
| quantity | Integer | NOT NULL | |
| unit_price | BigDecimal | precision=10, scale=2, NOT NULL | Price at purchase time |
| ticket_type_snapshot | TEXT | NOT NULL | JSON blob — full ticketType snapshot |

**Relationships:**
- ManyToOne → `Order` (LAZY)
- ManyToOne → `TicketType` (LAZY)
- OneToMany → `Ticket` (cascade ALL, orphanRemoval)

**Helper:** `getSubtotal()` → unit_price × quantity

---

### 3.3 Ticket

**Table:** `tickets`

**Indexes:** `user_id`, `order_item_id`

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | Long | PK, auto-increment | |
| order_item_id | Long | FK → order_items.id, NOT NULL | |
| user_id | UUID | FK → users.id, NOT NULL | Current owner (changes on transfer) |
| qr_payload | String | UNIQUE, NOT NULL, length=255 | HMAC-signed: `{ticketId}:{userId}:{nonce}:{hmac}` |
| status | TicketStatus | NOT NULL, default=ACTIVE | see Enums §5 |
| transfer_count | Integer | NOT NULL, default=0 | Max 1; enforced at listing creation |
| issued_at | LocalDateTime | NOT NULL | |
| used_at | LocalDateTime | nullable | Set on check-in |
| created_at | LocalDateTime | auto | |
| updated_at | LocalDateTime | auto | |

> **QR payload format:** `{ticketId}:{userId}:{nonce}:{hmac}` where
> `hmac = HMAC-SHA256("{ticketId}:{userId}:{nonce}", QR_SECRET)` (hex-encoded, 64 chars).
> Scanner verifies HMAC then does PK lookup + exact payload match.
> Max length ≈ 145 chars (fits in VARCHAR(255)).
>
> **FlexPass:** `user_id` and `qr_payload` are mutable on transfer (see `rules.md` §11.4).
> `transfer_count` is immutable after reaching 1.

---

### 3.4 ScanLog

**Table:** `scan_logs`

**Indexes:** `event_id`, `scanned_by`, `ticket_id`

**Check constraint:** `result IN ('SUCCESS','VALID','ALREADY_USED','CANCELLED','EXPIRED','INVALID_QR','WRONG_EVENT','TRANSFER_LOCKED')`

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | Long | PK, auto-increment | |
| ticket_id | Long | FK → tickets.id, nullable | Null for INVALID_QR / WRONG_EVENT |
| event_id | UUID | NOT NULL | |
| scanned_by | UUID | FK → users.id, NOT NULL | |
| scanned_at | LocalDateTime | NOT NULL | |
| result | ScanLogResult | NOT NULL | see Enums §8 |
| qr_payload | String | NOT NULL, length=512 | |

> Every scan (validate preview AND confirm check-in) creates a row.

---

### 3.5 TicketTransfer

**Table:** `ticket_transfers`

Records a FlexPass resale listing and its full lifecycle.

**Indexes:** `ticket_id`, `seller_id`, `status`

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | Long | PK, auto-increment | |
| ticket_id | Long | FK → tickets.id, NOT NULL | |
| seller_id | UUID | FK → users.id, NOT NULL | Original owner |
| buyer_id | UUID | FK → users.id, nullable | Set when buyer purchases |
| listing_price | BigDecimal | precision=10, scale=2, NOT NULL | ≤ original × 1.20 |
| original_price | BigDecimal | precision=10, scale=2, NOT NULL | Snapshot from order_items.unit_price |
| status | TicketTransferStatus | NOT NULL, default=PENDING_APPROVAL | see Enums §13 |
| expires_at | LocalDateTime | nullable | Resale window end |
| completed_at | LocalDateTime | nullable | Set on COMPLETED |
| created_at | LocalDateTime | auto | |
| updated_at | LocalDateTime | auto | |

**Relationships:**
- ManyToOne → `Ticket`
- ManyToOne → `User` (seller)
- ManyToOne → `User` (buyer, nullable)

**Business constraint:** Only one `TicketTransfer` may be in state
`PENDING_APPROVAL`, `APPROVED`, or `PAYMENT_PENDING` per ticket at a time.

---

## 4. Payment Module

### 4.1 Payment

**Table:** `payments`

**Indexes:** `order_id`, `idempotency_id` (unique)

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | Long | PK, auto-increment | |
| order_id | Long | FK → orders.id, NOT NULL | |
| provider | PaymentProvider | NOT NULL | see Enums §7 |
| status | PaymentStatus | NOT NULL | see Enums §9 |
| idempotency_id | String | UNIQUE, NOT NULL | `pay_u{userId}_o{orderId}` |
| amount | BigDecimal | precision=10, scale=2, NOT NULL | |
| transaction_id | String | nullable | External provider ID |
| payload | String | TEXT, nullable | Raw provider response |
| error_message | String | nullable | |
| created_at | LocalDateTime | auto | |
| updated_at | LocalDateTime | auto | |

---

## 5. Monitor Module

### 5.1 ActivityLog

**Table:** `activity_logs`

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-increment |
| actor_id | UUID | nullable |
| actor_role | ActorRole | nullable |
| action | LogAction | nullable |
| entity_type | EntityType | nullable |
| entity_id | String | nullable |
| owner_user_id | UUID | nullable |
| event_id | UUID | nullable |
| old_value | JSONB | nullable |
| new_value | JSONB | nullable |
| description | String | nullable |
| ip_address | String | nullable |
| created_at | LocalDateTime | auto |

---

## 6. Entity Relationship Summary

```
User ──< OrganizationMember >── Organization ──< Event ──< TicketType ──< OrderItem
                                                         │                    │
                                                         └──< EventImage      └──< Ticket
                                                         └──< EventFile       │
                                                                               └──< ScanLog
Order (user_id, event_snapshot)
  └──< OrderItem ──< Ticket
  └──< Payment
```

---

## 7. Snapshot Pattern

Booking integrity requires snapshots at purchase time.
The following JSON blobs are stored and MUST NOT be replaced after creation:

| Entity | Snapshot field | Stored in |
|---|---|---|
| Event | `event_snapshot` | `orders.event_snapshot` |
| TicketType | `ticket_type_snapshot` | `order_items.ticket_type_snapshot` |

Both snapshots include at minimum:
- `id`, `title`/`name`, `price`, `currency`, `startAt`/`endAt`, `venueName`

See `rules.md` §5 for the full list of required booking fields.
