# Prompt: Validate Implementation Against Core Rules + SSE Spec

**Reference:** `specs/sse-flow-spec.md`, `rules.md`

Use this after completing any SSE-related implementation task to verify
the result satisfies Core Rules, SSE rules, and spec consistency.

---

**Paste this to Claude:**

```
Validate the following implementation against three layers:
  Layer 1: rules.md (Core Rules — source of truth, sections 1–10)
  Layer 2: specs/sse-flow-spec.md §9 (Normative Rules SSE-001 through SSE-017)
  Layer 3: spec internal consistency

Changed files: {{list files with full paths and line ranges}}

══════════════════════════════════════════════════════════════
LAYER 1: CORE RULE VALIDATION
══════════════════════════════════════════════════════════════

Check each core rule section against the changed files:

CR §1.1 (Customers may NOT access unpublished data):
  a. Does any SSE emit to "public" channel for DRAFT events? → VIOLATION
  b. Does any SSE emit to "public" for DRAFT-event TicketTypes? → VIOLATION
  c. Does getPublicEvents fetch without status=PUBLISHED filter? → VIOLATION
  d. Does any SSE payload contain full DRAFT entity details? → VIOLATION

CR §1.2 (Organizer rights):
  a. When an organizer is invited, does the invitee receive personal notification?
  b. Are DRAFT event updates correctly scoped to organizer channel?

CR §3.2 (DRAFT events are fully editable, no bookings):
  a. Do TicketType SSE operations on DRAFT events only reach organizer/admin?

CR §3.3 (PUBLISHED events are partially immutable):
  a. When event:update SSE fires for a PUBLISHED event, does the payload
     contain only safe metadata (not time, date, location, format)?
  b. Confirm: SSE does not carry the fields it's forbidden to update.

CR §3.4 (CANCELLED event — refund all paid bookings):
  a. When an event is cancelled, is order:refund SSE emitted per paid order?
  b. Is the refund committed to DB before order:refund fires?

CR §4.1 (Published TicketType fields are immutable contracts):
  a. Does ticket_type:update SSE payload contain price, capacity, or contract fields?
     → FORBIDDEN (SSE-003)
  b. Can ORDER_CONFIRMED SSE cause customer to see a different price than they paid?
     → Check: does it invalidate TicketType cache? → FORBIDDEN (SSE-009)

CR §5 (Bookings must never depend on live Event data):
  a. Does ORDER_CONFIRMED invalidate Event or TicketType cache? → VIOLATION
  b. Does the order detail page show order.items[].unitPrice from the stored
     snapshot, not from a re-fetched TicketType?

CR §6 (Refunds are explicit actions):
  a. Is there an order:refund SSE event when a refund is initiated?
  b. Does the refund SSE go to user:{userId} only?
  c. Is the refundAmount the only financial field in the payload?

CR §8 (Forbidden actions):
  a. Can any SSE-triggered cache invalidation cause the UI to display
     modified financial data next to a paid order? → VIOLATION
  b. Can any SSE flow reduce displayed capacity below the value the customer
     purchased at? (TicketType capacity fields in payload) → VIOLATION

CR §9 (Error handling — no silent failures):
  a. Is the DB state committed before SSE fires? (SSE-008)
  b. If SSE is never delivered, does the client reach correct state on next REST call?

══════════════════════════════════════════════════════════════
LAYER 2: SSE SPEC VALIDATION
══════════════════════════════════════════════════════════════

For each rule, state PASS / FAIL / N/A with evidence:

SSE-001: DRAFT events MUST NOT emit to public channel
SSE-002: DRAFT-event TicketTypes MUST NOT emit to public channel
SSE-003: Payloads must be minimal (no DTOs, no financial fields except declared exception)
SSE-004: event:cancel only valid on PUBLISHED → CANCELLED transition
SSE-005: Refund lifecycle observable via order:refund SSE
SSE-006: invitation:create emits to organizer + user:{inviteeId}
SSE-007: Order and ticket events ONLY to user:{userId}
SSE-008: SSE emission after transaction commit
SSE-009: ORDER_CONFIRMED must not invalidate Event or TicketType
SSE-010: Order cache only invalidated by order:* events
SSE-011: TicketType deactivation must not affect Order cache
SSE-012: Public event queries apply PUBLISHED status filter
SSE-013: SSE is not the sole persistence mechanism
SSE-014: Cache invalidation only in SSEProvider
SSE-015: Private identifiers must not appear in public channel payloads
SSE-016: Reconnect must not corrupt client state
SSE-017: Channel selection matches entity access scope

══════════════════════════════════════════════════════════════
LAYER 3: SPEC CONSISTENCY VALIDATION
══════════════════════════════════════════════════════════════

Check for internal inconsistencies in specs and code comments:

SC-A: Rule numbering consistency
  → Do all rule IDs cited in code comments, PR descriptions, or spec
    cross-references match the actual rule IDs in sse-flow-spec.md §9?

SC-B: Core-rule reference accuracy
  → Do all "Core Rule §N.N" references point to sections that exist in rules.md?
  → rules.md has sections 1–10 only. Any reference to §11+ is invalid.

SC-C: Payload contradiction check
  → Does any code send a field that sse-flow-spec.md §10 Forbidden list prohibits?
  → Does any code send refundAmount in a context OTHER than order:refund on user:{userId}?

SC-D: Transition vs steady-state semantics
  → Is event:cancel used for the PUBLISHED→CANCELLED transition (correct)?
  → Is event:cancel used for any other purpose (incorrect)?
  → After CANCELLED steady state, does any code emit further public broadcasts
    for that event entity?

SC-E: Normative vs audit language separation
  → Are any "NON-COMPLIANT", "MISSING", or "UNKNOWN" labels present in the
    normative rule definitions (§9)? These belong only in §16.

SC-F: Invalidation table consistency
  → Does the invalidation behavior in SSEProvider match the table in §8?
  → Does any new SSEProvider case contradict the table?

SC-G: Payload table consistency
  → Does any changed SSE payload include fields not in §7.1 allowed fields?
  → Is any new exception properly declared in §7.2?

══════════════════════════════════════════════════════════════
OUTPUT FORMAT
══════════════════════════════════════════════════════════════

For each violation:
  Layer:         1 (Core Rule) | 2 (SSE Rule) | 3 (Spec Consistency)
  Finding:       short description
  Evidence:      exact file, line, code
  Conflict:      which rule/section is violated
  Fix:           what must change in code and/or spec

Final verdict:
  COMPLIANT     — all three layers pass; ready to merge
  NEEDS CHANGES — list of violations with fix instructions
  BLOCKED       — cannot validate; specify what context is missing
```
