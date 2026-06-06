# Evena — Customer Bookings Page (My Orders & My Tickets)

**Version:** 1.0 · **Date:** 2026-05-16

**File:** `app/dashboard/customer/cart/page.tsx`

---

## Overview

The customer bookings page has two tabs: **My Orders** and **My Tickets**.  
Both use paginated display to prevent infinite scroll on accounts with many entries.

```
ITEMS_PER_PAGE = 9
```

---

## My Orders tab

### Pagination strategy: server-side

Uses `useGetMyOrdersQuery({ page, size, status })`.

```typescript
// Primary query (current page data)
useGetMyOrdersQuery({ page: orderPage - 1, size: ITEMS_PER_PAGE, status: orderFilter !== 'ALL' ? orderFilter : undefined })

// Count-only query (for "All" chip total — avoids fetching all rows)
useGetMyOrdersQuery({ page: 0, size: 1 })
```

Total element count comes from `response.data.totalElements`.

### Filter chips

| Chip | Status value | Color accent |
|---|---|---|
| All | (no filter) | default |
| Confirmed | `CONFIRMED` | green |
| Pending | `PENDING` | amber |
| Processing | `PROCESSING` | blue |
| Refunded | `REFUNDED` | purple |
| Cancelled | `CANCELLED` | red |
| Expired | `EXPIRED` | grey |

**Rules:**
- Chips with `count === 0` are hidden (except "All")
- Active chip shows count from `orderTotalElements`
- Switching filter resets `orderPage` to `1`

### Pagination footer

```
Showing {start}–{end} of {total}   [← Prev]  Page X / Y  [Next →]
```

---

## My Tickets tab

### Pagination strategy: client-side

Fetches all tickets via `useGetMyTicketsQuery(undefined, { pollingInterval: 5 * 60 * 1000 })` — polls every 5 minutes to refresh time-window QR tokens.

### Filter chips

| Chip | Ticket statuses |
|---|---|
| All | all |
| Active | `ACTIVE` |
| Used | `USED` |
| Locked | `TRANSFER_LOCKED` |
| Cancelled | `CANCELLED` |

**Rules:**
- Same `ITEMS_PER_PAGE = 9`
- Chips with `count === 0` are hidden (except "All")
- Switching filter resets `ticketPage` to `1`

### Pagination footer

Same format as My Orders tab.

---

## State shape

```typescript
const [orderPage, setOrderPage]     = useState(1);
const [orderFilter, setOrderFilter] = useState<OrderStatus | 'ALL'>('ALL');
const [ticketPage, setTicketPage]   = useState(1);
const [ticketFilter, setTicketFilter] = useState<TicketStatus | 'ALL'>('ALL');
```

---

## QR code display (ticket modal)

When the user opens a ticket's QR modal:
- `QRCodeSVG` renders client-side from `qrPayload` returned by the API
- `qrPayload` is a time-window token (`TW:...`) — rotates every 5 minutes server-side
- Live data is derived from `ticketsData.data.find(t => t.id === selectedTicket.id)` to ensure polling updates are reflected in the open modal

## Design rules

- Both tabs share the same `ITEMS_PER_PAGE` constant
- Infinite scroll is **not allowed** — use explicit pagination controls
- Filter chip count must reflect the filtered total, not the current page size
- "All" chip always shows the global total (from the `size=1` count query for orders, from array length for tickets)
- QR token rotates server-side every 5 minutes; client polling at the same interval ensures the displayed QR is always current
