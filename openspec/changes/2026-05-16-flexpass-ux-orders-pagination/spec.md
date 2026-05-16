# Change: FlexPass UX, My Bookings Pagination, Admin Dashboard Fixes

**Date:** 2026-05-16  
**Branch:** develop  
**Status:** Merged to staging

---

## Summary

Batch of UX improvements: FlexPass customer/admin UI, My Bookings page redesign with
server-side pagination, and admin activity log bug fixes. No schema changes.

---

## Changes

### 1. My Orders — server-side pagination with optional status filter

**Problem:** `GET /api/orders/my-orders` previously always fetched 1000 rows; no filtering.  
**Fix:** Added `status` query param (optional). Frontend uses page/size/status for true server-side pagination.

**Backend changes:**
- `OrderRepository` — new method `findByUserIdFiltered(userId, status, pageable)` using
  `(:status IS NULL OR o.status = :status)` JPQL pattern with JOIN FETCH to avoid N+1
- `OrderService.getMyOrders(int page, int size, OrderStatus status)` — delegates to filtered query
- `OrderController` — `@RequestParam(required = false) OrderStatus status` added to `GET /my-orders`

**Frontend changes (`OrderApi.ts`):**
- `getMyOrders` query accepts `{ page?, size?, status? }`
- Status is omitted from params when `undefined` (not sent as empty string)

---

### 2. My Bookings page — paginated redesign (My Orders + My Tickets)

**Problem:** Unbounded list caused infinite scroll on accounts with many orders/tickets.

**My Orders tab (server-side pagination):**
- `ITEMS_PER_PAGE = 9`
- Filter chips: All / CONFIRMED / PENDING / PROCESSING / REFUNDED / CANCELLED / EXPIRED
- "All" chip total loaded via a separate `size=1` count-only request to avoid over-fetching
- Chips with `count === 0` are hidden
- `Showing X–Y of Z` pagination footer

**My Tickets tab (client-side pagination):**
- Filter chips: All / Active / Used / Locked / Cancelled
- Same `ITEMS_PER_PAGE = 9`, same footer pattern
- Filtered on the full ticket list fetched once

---

### 3. FlexPass SellTab — status filter chips + expired listing re-list flow

**Filter type:** `'ALL' | 'AVAILABLE' | 'LISTED' | 'EXPIRED_LISTING'`

| Chip | Meaning |
|---|---|
| All | All sellable tickets |
| Available | Tickets with no active or expired listing |
| Listed | Tickets with PENDING_APPROVAL, APPROVED, PRICE_LOCKED, PAYMENT_PENDING listing |
| Expired Listing | Tickets whose newest listing is EXPIRED and have no currently active listing |

**Expired listing UX:**
- Ticket row shows amber badge "Expired · Re-list" instead of standard status badge
- Selecting such a ticket shows a re-list notice in the right panel before the submit form
- Functionally behaves the same as creating a new listing (backend creates a fresh listing entry)

**Logic:**
```typescript
// Build map: ticketId → has active listing
const activeListingByTicketId = new Map(activeListing.map(...));
// Build map: ticketId → newest EXPIRED listing (only if no active listing)
const expiredListingByTicketId = new Map(
  listings
    .filter(l => l.status === 'EXPIRED' && !activeListingByTicketId.has(l.ticketId))
    .map(l => [l.ticketId, l])
);
```

---

### 4. FlexPass MyListingsTab — status filter chips

**Filter type:** `'ALL' | 'ACTIVE' | 'SOLD' | 'ENDED'`

| Chip | Statuses included |
|---|---|
| All | All listings |
| Active | PENDING_APPROVAL, APPROVED, PRICE_LOCKED, PAYMENT_PENDING |
| Sold | COMPLETED |
| Ended | EXPIRED, CANCELLED, REJECTED, FAILED |

**Visual:**
- Active chip: purple accent
- Sold chip: green accent
- Ended chip: amber accent
- EXPIRED badge: amber (changed from grey)

---

### 5. Admin Activity Log — bug fixes

**5a. Sparkline — no tooltip**  
Added `<ReTooltip>` with `formatter` returning `[string, string]` (required by Recharts
`Formatter<ValueType, NameType>` signature) and `activeDot` for hover target.

```tsx
formatter={(v: unknown) => [`${v ?? 0}`, 'events'] as [string, string]}
```

Note: must cast to `[string, string]` explicitly; returning `[{}, string]` fails `tsc --noEmit`.

**5b. Bar chart click — saved filter chip not reset**  
`onHourClick` now calls `setActiveSavedFilter(-1)` when a bar is clicked, so the
previously highlighted saved-filter chip deactivates.

**5c. Investigate button — hardcoded EVENT_DELETED**  
Was hardcoded to filter by `EVENT_DELETED` regardless of what anomaly was detected.  
Now derives the filter window from the actual spike hours (`isSpike === true`) in `hourlyData`:

```typescript
function onInvestigate() {
  const spikes = hourlyData.filter((h) => h.isSpike);
  if (spikes.length > 0) {
    // Find first and last spike index, compute from/to datetimes
    setFromFilter(from.format('YYYY-MM-DDTHH:mm'));
    setToFilter(to.format('YYYY-MM-DDTHH:mm'));
    setActionFilter('');   // show all actions in spike window
  } else {
    // Fallback: show today's log
    setFromFilter(dayjs().startOf('day').format('YYYY-MM-DDTHH:mm'));
    setToFilter('');
    setActionFilter('');
  }
  setActiveSavedFilter(-1);
  setPage(0);
}
```

---

### 6. Admin Dashboard — new panels

Two new components added to the admin dashboard:

| Component | File | Purpose |
|---|---|---|
| `AnomalySignalsPanel` | `src/components/AdminDashboard/AnomalySignalsPanel.tsx` | Surface real-time anomaly signals from SSE / activity data |
| `SSEHealthPanel` | `src/components/AdminDashboard/SSEHealthPanel.tsx` | Monitor SSE connection health and subscriber counts |

Both exported from `src/components/AdminDashboard/index.ts`.

---

### 7. Organizer Dashboard — remove duplicate CTA

**Problem:** "View All Events →" button appeared twice: once in the "All Events" section header,
once at the bottom of the Event Performance table.

**Fix:** Removed the one in the section header. Only the table-bottom CTA remains.

---

## Affected files

### Frontend
| File | Change |
|---|---|
| `app/dashboard/customer/cart/page.tsx` | My Bookings full redesign (pagination + filters) |
| `app/dashboard/admin/activity-log/page.tsx` | Sparkline tooltip, bar click reset, Investigate logic |
| `app/dashboard/admin/page.tsx` | Add AnomalySignalsPanel + SSEHealthPanel |
| `app/dashboard/organizer/page.tsx` | Remove duplicate View All Events button |
| `app/dashboard/organizer/flexpass/page.tsx` | FlexPass organizer page improvements |
| `src/components/FlexPassCustomer/SellTab.tsx` | Filter chips + expired listing re-list flow |
| `src/components/FlexPassCustomer/MyListingsTab.tsx` | Filter chips |
| `src/components/FlexPassCustomer/MarketplaceTab.tsx` | Enhancements |
| `src/components/FlexPassAdmin/SaleWindowPanel.tsx` | Enhancements |
| `src/components/AdminDashboard/AnomalySignalsPanel.tsx` | New file |
| `src/components/AdminDashboard/SSEHealthPanel.tsx` | New file |
| `src/components/AdminDashboard/index.ts` | Export new panels |
| `src/stores/services/OrderApi.ts` | Add optional status filter to getMyOrders |
| `src/stores/types/event.ts` | Additional fields |
| `src/stores/types/flexpass.ts` | Additional fields |
| `src/providers/SSEProvider.tsx` | Minor improvements |

### Backend
| File | Change |
|---|---|
| `module/order/repository/OrderRepository.java` | Add `findByUserIdFiltered` |
| `module/order/service/OrderService.java` | `getMyOrders` accepts `OrderStatus status` |
| `module/order/controller/OrderController.java` | `status` optional query param |
| `module/flexpass/repository/FlexPassListingRepository.java` | New query methods |
| `module/flexpass/repository/FlexPassSaleWindowRepository.java` | New query methods |
| `module/flexpass/service/FlexPassService.java` | Enhanced filtering |
| `module/event/dto/mapper/EventMapper.java` | Additional fields mapped |
| `module/event/dto/response/EventListResponse.java` | Additional fields |
| `module/flexpass/dto/response/FlexPassMarketplaceListingResponse.java` | Additional field |
