# Enums: Audit (LogAction · EntityType · ActorRole)

---

## LogAction

Actions recorded in `activity_logs` (rules.md §7.3):

### Event
`EVENT_CREATED` · `EVENT_UPDATED` · `EVENT_DELETED` · `EVENT_APPROVED` · `EVENT_REJECTED`

### TicketType
`TICKET_TYPE_ADDED` · `TICKET_TYPE_UPDATED` · `TICKET_TYPE_DELETED` · `TICKET_TYPE_DEACTIVATED`

### Order
`ORDER_CREATED` · `ORDER_CANCELLED` · `ORDER_EXPIRED` · `ORDER_CONFIRMED`

### Ticket
`TICKET_ISSUED` · `TICKET_USED` · `TICKET_CANCELLED`

### Payment
`PAYMENT_INITIATED` · `PAYMENT_SUCCESS` · `PAYMENT_FAILED` · `PAYMENT_REFUNDED`

### Organization
`ORGANIZATION_ADDED` · `ORGANIZATION_UPDATED` · `ORGANIZATION_DELETED`
`ORGANIZATION_REJECTED` · `ORGANIZATION_APPROVED` · `ORGANIZATION_VERIFIED`
`MEMBER_UPDATED`

> `ORGANIZATION_APPROVED` = admin approves the organization (first-time verification).
> `ORGANIZATION_VERIFIED` = re-verification after an identity-field change (distinct event, same entity).

### FlexPass
`LISTING_CREATED` · `LISTING_APPROVED` · `LISTING_REJECTED` · `LISTING_CANCELLED`
`TRANSFER_COMPLETED` · `TRANSFER_FAILED` · `LISTING_EXPIRED`

---

## EntityType

| Value |
|---|
| `EVENT` |
| `TICKET_TYPE` |
| `ORDER` |
| `TICKET` |
| `PAYMENT` |
| `ORGANIZATION` |
| `TICKET_TRANSFER` |

---

## ActorRole

| Value |
|---|
| `ADMIN` |
| `ORGANIZER` |
| `CUSTOMER` |
