# SSE Review Checklist

**Version:** 2.0
**Reference:** `specs/sse-flow-spec.md`

Use this checklist for ANY change that touches:
- `backend/SSENotificationService.java`
- Any `*Service.java` that calls `sseNotificationService.*`
- `frontend/src/providers/SSEProvider.tsx`
- `frontend/src/stores/types/sse.ts`
- `sse-service/config/channels.json` or `actions.json`
- Any frontend page or component that uses `useSSE()` / `lastEvent`

---

## How to complete this checklist

For each item:
- ✅ **PASS** — requirement is satisfied
- ❌ **FAIL** — requirement is violated; must fix before merge
- ➖ **N/A** — rule does not apply to this change (provide reason)

**Any ❌ blocks merge.**

---

## GROUP 1 — Visibility & Access Scope

_These checks prevent DRAFT or private data from reaching unauthorized audiences._

| # | Rule | Check | Result |
|---|------|-------|--------|
| SC-01 | SSE-001 | If change emits `event:create` or `event:update`: is channel `organizer,admin` for DRAFT, `public,organizer` for PUBLISHED? | |
| SC-02 | SSE-002 | If change emits `ticket_type:*`: is parent event status checked before channel selection? DRAFT → `organizer,admin`; PUBLISHED → `public,organizer` | |
| SC-03 | SSE-004 | If change emits `event:cancel`: does it guard that source state is PUBLISHED (not DRAFT)? | |
| SC-04 | SSE-017 | Is the channel choice determined by "who is authorized by Core Rules to see this change?" — not by which action is technically allowed in the config? | |
| SC-05 | SSE-007 | Are all `order:*` and `ticket:*` events going to `user:{userId}` ONLY? No broadcast to `public`, `organizer`, or `admin`? | |
| SC-06 | SSE-012 | Does `getPublicEvents` (or any customer-facing event query) include `status: 'PUBLISHED'` filter? | |

---

## GROUP 2 — Payload Discipline

_These checks prevent sensitive or oversized data from appearing in SSE payloads._

| # | Rule | Check | Result |
|---|------|-------|--------|
| SC-07 | SSE-003 | Does the SSE payload contain ONLY IDs and names? No full DTO objects (EventResponse, OrderResponse, etc.)? | |
| SC-08 | SSE-003 | Does the payload contain any financial field (`price`, `currency`, `totalAmount`, `unitPrice`, `paymentId`)? — FORBIDDEN unless it is `refundAmount` in `order:refund` on `user:{userId}` | |
| SC-09 | SSE-003 | Does the payload contain inventory/contract fields (`total`, `sold`, `available`, `capacity`, `benefits`, `refundPolicy`)? — FORBIDDEN | |
| SC-10 | SSE-015 | Does the payload contain `userId`, `ownerId`, or other user identifiers? — FORBIDDEN on `public` channel; allowed on `organizer`, `admin`, `user:{id}` | |
| SC-11 | SSE-003 | Is payload size within 4KB? Full DTOs violate this limit | |

---

## GROUP 3 — Invalidation Correctness

_These checks prevent cache over-invalidation that could corrupt snapshot-based UI._

| # | Rule | Check | Result |
|---|------|-------|--------|
| SC-12 | SSE-009 | Does `ORDER_CONFIRMED` handler invalidate `Event` or `TicketType` cache? — FORBIDDEN | |
| SC-13 | SSE-009 | Do `ORDER_CANCELLED` or `ORDER_EXPIRED` handlers invalidate `Event` or `TicketType` cache? — FORBIDDEN | |
| SC-14 | SSE-010 | Is `'Order'` cache invalidated by anything other than `order:*` events? — FORBIDDEN | |
| SC-15 | SSE-011 | Does `TICKET_TYPE_DEACTIVATED` handler invalidate `'Order'` cache? — FORBIDDEN | |
| SC-16 | §8 | Does the new SSE event follow the invalidation table in `sse-flow-spec.md §8`? | |
| SC-17 | SSE-014 | Is cache invalidation performed ONLY in `SSEProvider.tsx`? No duplicate invalidation in hooks or page components? | |

---

## GROUP 4 — Transition Semantics

_These checks prevent logical errors in how state transitions are modeled as SSE events._

| # | Rule | Check | Result |
|---|------|-------|--------|
| SC-18 | SSE-004 | Is `event:cancel` used only for PUBLISHED → CANCELLED transition? Not for cancelling a DRAFT? | |
| SC-19 | §6 Tier A note | After `event:cancel` is emitted, do any further public-channel broadcasts occur for that cancelled event entity? — MUST NOT | |
| SC-20 | SSE-005 | If change involves event cancellation and paid orders exist: is `order:refund` SSE emitted per affected order AFTER the refund is committed? | |
| SC-21 | SSE-008 | Is SSE emission happening AFTER the DB transaction commits? Emission inside `@Transactional` without AFTER_COMMIT guarantee is FORBIDDEN | |

---

## GROUP 5 — Delivery Safety

_These checks enforce SSE-as-hint architecture and protect against delivery failure._

| # | Rule | Check | Result |
|---|------|-------|--------|
| SC-22 | SSE-013 | Is the corresponding DB state committed before SSE is emitted? A client refetch after SSE receipt must read the final committed state | |
| SC-23 | SSE-013 | If SSE is never delivered (queue overflow, reconnect), does the client still reach correct state via REST API on next fetch? | |
| SC-24 | SSE-016 | If change modifies SSEProvider reconnect behavior: does it perform targeted invalidation only (`'Order'`, `'Organizer'`) and NOT a full public cache wipe? | |

---

## GROUP 6 — Personal Notification Completeness

_These checks ensure owner/invitee flows have real-time personal visibility._

| # | Rule | Check | Result |
|---|------|-------|--------|
| SC-25 | SSE-005 | If change adds or modifies refund flow: is `order:refund` emitted to `user:{userId}` with at minimum `orderId`, `eventId`, `eventName`? | |
| SC-26 | SSE-006 | If change adds or modifies invitation flow: is `invitation:create` emitted to BOTH `organizer` AND `user:{inviteeId}`? | |
| SC-27 | §6 Tier B | If change adds org approval/verify flow: is `organization:verify` emitted to BOTH `organizer,admin` AND `user:{ownerId}`? | |

---

## GROUP 7 — Spec Consistency

_These checks prevent the spec itself from becoming inconsistent._

| # | Check | Result |
|---|-------|--------|
| SC-28 | Do all rule IDs cited in this change match the rules as defined in `sse-flow-spec.md §9`? No dangling references? | |
| SC-29 | Do all core-rule references (e.g., "Core Rule §1.1") match sections that actually exist in `rules.md`? | |
| SC-30 | Does any new normative statement contradict an existing rule in `sse-flow-spec.md`? | |
| SC-31 | Is any implementation audit status ("NON-COMPLIANT", "MISSING") accidentally embedded in a normative rule definition? (Belongs in §16 Gap Summary only) | |
| SC-32 | Does the payload classification table in §7 remain consistent with any payload changes introduced by this change? | |
| SC-33 | Does the invalidation model table in §8 remain consistent with any SSEProvider changes introduced? | |

---

## GROUP 8 — i18n & Code Quality

| # | Rule | Check | Result |
|---|------|-------|--------|
| SC-34 | G-002 | If change adds SSE-triggered toast messages: are i18n keys added to BOTH `en` and `vi` translation files? | |
| SC-35 | G-006 | Is there any `console.log` in a page or component that consumes `useSSE()` / `lastEvent`? — FORBIDDEN | |
| SC-36 | G-003 | If change adds RTK Query mutations: do they invalidate both item tag AND list tag? | |
