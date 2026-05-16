# Evena — FlexPass Customer UI

**Version:** 1.0 · **Date:** 2026-05-16

**Files:**
- `src/components/FlexPassCustomer/SellTab.tsx`
- `src/components/FlexPassCustomer/MyListingsTab.tsx`
- `src/components/FlexPassCustomer/MarketplaceTab.tsx`

---

## SellTab — "Sell a Ticket"

### Filter chips

```typescript
type SellFilter = 'ALL' | 'AVAILABLE' | 'LISTED' | 'EXPIRED_LISTING';
```

| Chip | Meaning |
|---|---|
| All | All tickets the user purchased for this event |
| Available | Tickets with no active or expired listing (can be listed) |
| Listed | Tickets with listing status ∈ {PENDING_APPROVAL, APPROVED, PRICE_LOCKED, PAYMENT_PENDING} |
| Expired Listing | Tickets whose newest listing is EXPIRED **and** have no currently active listing |

### Expired Listing detection

```typescript
// 1. Build active-listing set
const activeListingByTicketId = new Map(
  listings
    .filter(l => ['PENDING_APPROVAL','APPROVED','PRICE_LOCKED','PAYMENT_PENDING'].includes(l.status))
    .map(l => [l.ticketId, l])
);

// 2. Build expired-listing map (only for tickets with NO active listing)
const expiredListingByTicketId = new Map(
  listings
    .filter(l => l.status === 'EXPIRED' && !activeListingByTicketId.has(l.ticketId))
    .map(l => [l.ticketId, l])
);
```

A ticket counted as "Expired Listing" is one that was listed, the listing expired, and no
new listing has been created since. If the seller has already re-listed (new APPROVED listing),
the ticket counts as "Listed", NOT "Expired Listing".

### Expired listing UX

- Ticket row in the left panel shows an **amber** badge: `Expired · Re-list`
- Selecting that ticket shows a re-list notice in the right panel above the submit form
- Functionally identical to creating a brand-new listing — backend creates a fresh `TicketTransfer` row
- No special backend endpoint; uses the same `POST /api/flexpass/listings`

---

## MyListingsTab — "My Listings"

### Filter chips

```typescript
type ListingFilter = 'ALL' | 'ACTIVE' | 'SOLD' | 'ENDED';
```

| Chip | Statuses included | Accent color |
|---|---|---|
| All | all listings | default |
| Active | PENDING_APPROVAL, APPROVED, PRICE_LOCKED, PAYMENT_PENDING | purple |
| Sold | COMPLETED | green |
| Ended | EXPIRED, CANCELLED, REJECTED, FAILED | amber |

### EXPIRED badge color

EXPIRED status badge must use **amber** styling, not grey.  
Rationale: EXPIRED means the sale window closed without a buyer — it is an actionable state
(user may want to re-list), so amber communicates "attention needed" better than grey.

---

## Design rules (both tabs)

- Filter chips with `count === 0` are hidden (except "All")
- Switching filter resets to first page / first item if pagination is present
- Counts in chips must reflect filtered totals, not page-level counts
- Re-listing (expired → new listing) requires no special backend route — same `POST /api/flexpass/listings`
