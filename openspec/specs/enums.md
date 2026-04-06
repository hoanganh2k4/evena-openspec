# Evena — Enums & State Machines

**Version:** 1.0
**Date:** 2026-04-05
**Status:** approved
**Applies to:** backend · frontend · mobile

All state transitions listed here are normative. Transitions not shown are
**forbidden** — attempting them must throw a clear validation error (`rules.md` §9).

---

## 1. UserStatus

| Value | Meaning |
|---|---|
| `ACTIVE` | Normal user; can log in |
| `INACTIVE` | Deactivated by admin; cannot log in |
| `SUSPENDED` | Temporarily suspended; cannot log in |
| `PENDING_VERIFICATION` | Registered but email not yet verified |

**Transitions:**

```
PENDING_VERIFICATION → ACTIVE   (on email verification)
ACTIVE → INACTIVE               (admin action)
ACTIVE → SUSPENDED              (admin action)
INACTIVE → ACTIVE               (admin action)
SUSPENDED → ACTIVE              (admin action)
```

---

## 2. EventStatus

| Value | Meaning |
|---|---|
| `DRAFT` | Not published; invisible to customers |
| `PUBLISHED` | Public; bookings accepted |
| `ONGOING` | Event has started; no new bookings |
| `COMPLETED` | Event ended or sold out |
| `CANCELLED` | Cancelled; all paid bookings refunded |

**Transitions:**

```
DRAFT     → PUBLISHED   (organizer/admin publish action)
PUBLISHED → ONGOING     (auto: scheduler when startAt reached)
PUBLISHED → CANCELLED   (organizer/admin)
ONGOING   → COMPLETED   (auto: scheduler when endAt reached)
ONGOING   → CANCELLED   (organizer/admin)
COMPLETED → PUBLISHED   (auto: when booking cancellation frees capacity)
COMPLETED → CANCELLED   (organizer/admin)
```

**Customer-visible statuses:** `PUBLISHED`, `ONGOING`
(see `rules.md` §1.1, §3.3b)

**Auto-transitions driven by:** `EventScheduler` (runs every 5 minutes)

---

## 3. TicketTypeStatus

| Value | Meaning |
|---|---|
| `DRAFT` | Not visible to customers; fully editable |
| `INACTIVE` | Reserved status (not yet used in business logic) |
| `ACTIVE` | Public; can be purchased; contractual fields locked |
| `SOLD_OUT` | Auto-set when sold ≥ total; reverts when capacity freed |
| `DEACTIVATED` | Permanently removed from sale; historical data retained |

**Transitions:**

```
DRAFT       → ACTIVE       (organizer/admin explicit activation)
ACTIVE      → SOLD_OUT     (auto: when sold >= total)
SOLD_OUT    → ACTIVE       (auto: when booking cancelled, sold < total)
ACTIVE      → DEACTIVATED  (organizer/admin)
DRAFT       → DEACTIVATED  (cascade on event cancel)
ACTIVE      → DEACTIVATED  (cascade on event cancel)
SOLD_OUT    → DEACTIVATED  (cascade on event cancel)
```

**Activation requirements (DRAFT → ACTIVE):**
- `price > 0`
- `total > 0`
- `salesStart` and `salesEnd` are set and valid

**Contractual fields locked once ACTIVE:**
`price`, `currency`, `total` (decrease only), benefits, refundPolicy

---

## 4. OrderStatus

| Value | Meaning |
|---|---|
| `PENDING` | Created but payment not yet initiated |
| `PROCESSING` | Payment in progress (gateway redirect) |
| `CONFIRMED` | Payment succeeded; tickets issued |
| `CANCELLED` | Cancelled by user or cascade (event cancel) |
| `EXPIRED` | Payment window elapsed; sold count released |
| `REFUNDED` | Refund completed |

**Transitions:**

```
PENDING    → PROCESSING   (payment initiated)
PROCESSING → CONFIRMED    (payment callback SUCCESS)
PROCESSING → CANCELLED    (payment callback FAILED)
PENDING    → CANCELLED    (user cancel OR event cancel cascade)
PENDING    → EXPIRED      (TTL elapsed)
CONFIRMED  → REFUNDED     (admin/organizer explicit refund)
```

**Rules:**
- Once `CONFIRMED`, amount is immutable (`rules.md` §6)
- Cascade cancel from event: only `PENDING` orders are cancelled; `CONFIRMED` orders trigger refund workflow

---

## 5. TicketStatus

| Value | Meaning |
|---|---|
| `ACTIVE` | Valid; not yet used |
| `USED` | Checked in (check-in confirmed) |
| `CANCELLED` | Cancelled (linked order cancelled) |
| `EXPIRED` | Event ended; ticket no longer valid |
| `TRANSFER_LOCKED` | Ticket is listed for FlexPass transfer; cannot be scanned |

**Transitions:**

```
ACTIVE → USED             (check-in via /tickets/scan or /tickets/{id}/check-in)
ACTIVE → CANCELLED        (order cancelled)
ACTIVE → EXPIRED          (event completed/cancelled)
ACTIVE → TRANSFER_LOCKED  (FlexPass: seller creates listing)
TRANSFER_LOCKED → ACTIVE  (FlexPass: listing cancelled / transfer failed / expired)
TRANSFER_LOCKED → ACTIVE  (FlexPass: transfer COMPLETED — new owner, new QR)
```

> A `TRANSFER_LOCKED` ticket MUST NOT be checked in. See `rules.md` §11.2.

---

## 6. OrganizationRole

Roles within a single organization. A user may hold different roles in different organizations.

| Value | Description |
|---|---|
| `OWNER` | Organization creator; full control |
| `MANAGER` | Can manage events and members |
| `COORDINATOR` | Can manage event logistics |
| `SCANNER` | Can scan tickets at events |
| `VIEWER` | Read-only access |

---

## 7. PaymentProvider

| Value | Description |
|---|---|
| `CASH` | Cash payment recorded manually |
| `CARD` | Card payment (Stripe or similar) |
| `MOMO` | MoMo e-wallet (Vietnam) |

---

## 8. ScanLogResult

Every scan attempt (preview `validate` AND confirmed `scan`) writes a row.

| Value | Meaning | Written by |
|---|---|---|
| `SUCCESS` | Ticket valid and check-in confirmed | `/tickets/scan` |
| `VALID` | Ticket valid preview — not yet checked in | `/tickets/validate` |
| `ALREADY_USED` | Ticket already checked in | Both endpoints |
| `CANCELLED` | Ticket is cancelled | Both endpoints |
| `EXPIRED` | Ticket is expired | Both endpoints |
| `INVALID_QR` | QR payload not found in system | Both endpoints |
| `WRONG_EVENT` | Ticket belongs to a different event | Both endpoints |
| `TRANSFER_LOCKED` | Ticket is locked for FlexPass transfer | Both endpoints |

> `ticket_id` is `null` for `INVALID_QR` results.

**DB check constraint must include all 8 values** (migration required if enum changes).

---

## 9. PaymentStatus

| Value | Meaning |
|---|---|
| `PENDING` | Payment initiated, awaiting gateway |
| `SUCCESS` | Gateway confirmed payment |
| `FAILED` | Gateway rejected payment |
| `REFUNDED` | Refund processed |

---

## 10. LogAction

Actions recorded in the audit trail (`rules.md` §7.3):

### Event actions
`EVENT_CREATED`, `EVENT_UPDATED`, `EVENT_DELETED`, `EVENT_APPROVED`, `EVENT_REJECTED`

### TicketType actions
`TICKET_TYPE_ADDED`, `TICKET_TYPE_UPDATED`, `TICKET_TYPE_DELETED`, `TICKET_TYPE_DEACTIVATED`

### Order actions
`ORDER_CREATED`, `ORDER_CANCELLED`, `ORDER_EXPIRED`, `ORDER_SUCCESSFULL`

### Ticket actions
`TICKET_ISSUED`, `TICKET_USED`, `TICKET_CANCELLED`

### Payment actions
`PAYMENT_INITIATED`, `PAYMENT_SUCCESS`, `PAYMENT_FAILED`, `PAYMENT_REFUNDED`

### Organization actions
`ORGANIZATION_ADDED`, `ORGANIZATION_UPDATED`, `ORGANIZATION_DELETED`,
`ORGANIZATION_REJECTED`, `ORGANIZATION_APPROVED`, `ORGANIZATION_VERIFIED`,
`MEMBER_UPDATED`

---

## 11. EntityType

Used in `ActivityLog` to identify which entity type was affected.

| Value |
|---|
| `EVENT` |
| `TICKET_TYPE` |
| `ORDER` |
| `TICKET` |
| `PAYMENT` |
| `ORGANIZATION` |

---

## 12. ActorRole

Actor classification for audit logs.

| Value |
|---|
| `ADMIN` |
| `ORGANIZER` |
| `CUSTOMER` |

---

## 13. TicketTransferStatus

Status of a FlexPass resale listing / transfer record.

| Value | Meaning |
|---|---|
| `PENDING_APPROVAL` | Listing created; awaiting organizer review |
| `APPROVED` | Organizer approved; visible in marketplace |
| `REJECTED` | Organizer rejected; ticket unlocked back to seller |
| `PAYMENT_PENDING` | Buyer initiated payment; escrow in progress |
| `COMPLETED` | Payment confirmed; QR rotated; ownership transferred |
| `FAILED` | Payment failed; ticket unlocked back to seller |
| `CANCELLED` | Seller cancelled listing before a buyer appeared |
| `EXPIRED` | Resale window elapsed without a buyer; ticket unlocked |

**Transitions:**

```
PENDING_APPROVAL → APPROVED        (organizer approves)
PENDING_APPROVAL → REJECTED        (organizer rejects → ticket ACTIVE)
APPROVED → PAYMENT_PENDING         (buyer purchases)
APPROVED → CANCELLED               (seller cancels → ticket ACTIVE)
APPROVED → EXPIRED                 (resale window ends → ticket ACTIVE)
PAYMENT_PENDING → COMPLETED        (payment SUCCESS → QR rotation, owner change)
PAYMENT_PENDING → FAILED           (payment FAILED → ticket ACTIVE)
```

See `rules.md` §11 for enforcement rules.

---

## 14. CurrencyMinimum

Minimum price for paid ticket types (price = 0 is allowed for free tickets).

| Currency | Minimum |
|---|---|
| VND | 1 000 |
| USD | 1 |
| EUR | 1 |
| GBP | 1 |
| SGD | 1 |
