# Prompt: SSE Audit for Any Change

**Reference:** `specs/sse-flow-spec.md`, `rules.md`

Use this before merging any change that touches: backend SSE emission,
SSEProvider, sse.ts types, channels.json, actions.json, or any page using useSSE().

---

**Paste this to Claude:**

```
Audit the following changed files against:
- specs/sse-flow-spec.md (SSE integrity rules)
- rules.md (Core Rules — source of truth)
- openspec/config.yaml (project rules G-001 through G-008)

Changed files: {{list files with full paths}}

─────────────────────────────────────────────────────────────
SECTION 1: BACKEND AUDIT (SSENotificationService or *Service.java)
─────────────────────────────────────────────────────────────

For each notify*() call or emitEvent() call found:

Q1. Is anything emitted to "public" channel?
    → If yes: What is the entity's current state?
    → DRAFT event or DRAFT-parent ticketType MUST go to "organizer,admin" only (SSE-001, SSE-002)
    → PUBLISHED event MUST go to "public,organizer"
    → Order/ticket events MUST NEVER go to public (SSE-007)

Q2. Does the payload contain a full DTO object?
    → FORBIDDEN on all channels (SSE-003)
    → Payload must contain only: IDs and names (see sse-flow-spec.md §7.1)

Q3. Does the payload contain financial fields?
    → FORBIDDEN except: refundAmount in order:refund on user:{userId} (SSE-003, §7.2)
    → Financial fields: price, currency, totalAmount, unitPrice, paymentId

Q4. Does the payload contain inventory/contract fields?
    → FORBIDDEN (SSE-003): total, sold, available, capacity, benefits, refundPolicy

Q5. Does the payload contain userId or ownerId?
    → FORBIDDEN on public channel (SSE-015)
    → Allowed on organizer, admin, user:{id} channels

Q6. Is the emit call inside or immediately after a @Transactional method?
    → RISK: If inside @Transactional without AFTER_COMMIT guarantee, SSE fires
      before DB commit is visible (SSE-008)
    → Report as: "RISK — emit may occur before transaction commit"

Q7. If the action involves event cancellation and paid orders exist:
    → Is order:refund emitted per affected paid order? (SSE-005)
    → Is the refund committed to DB before order:refund fires?

Q8. Is event:cancel emitted anywhere?
    → Guard must verify source state is PUBLISHED, not DRAFT (SSE-004)

Q9. Is invitation:create emitted?
    → Must emit to BOTH "organizer" AND "user:{inviteeId}" (SSE-006)

─────────────────────────────────────────────────────────────
SECTION 2: SSE SERVICE AUDIT (channels.json, actions.json)
─────────────────────────────────────────────────────────────

Q10. Is the new action defined in actions.json?
Q11. Is the new resource listed in channels.json for the correct channels?
Q12. Is the channel scope (public/organizer/private) correct per rules.md §1?
Q13. Does the config allow the action in channels where the business rules
     forbid it? Config validity ≠ business visibility correctness (SSE-017).

─────────────────────────────────────────────────────────────
SECTION 3: FRONTEND SSEProvider AUDIT
─────────────────────────────────────────────────────────────

Q14. Does ORDER_CONFIRMED handler invalidate Event or TicketType? → FORBIDDEN (SSE-009)
Q15. Does ORDER_CANCELLED or ORDER_EXPIRED invalidate Event or TicketType? → FORBIDDEN (SSE-009)
Q16. Is 'Order' cache invalidated by any non-order event? → FORBIDDEN (SSE-010)
Q17. Does TICKET_TYPE_DEACTIVATED invalidate 'Order'? → FORBIDDEN (SSE-011)
Q18. Does the new case follow the invalidation table in sse-flow-spec.md §8?
     For each new case: state MUST Invalidate and MUST NOT Invalidate columns.
Q19. Does SSEProvider now perform cache invalidation for a domain
     that is also invalidated somewhere else? → Duplicate invalidation (SSE-014)

─────────────────────────────────────────────────────────────
SECTION 4: FRONTEND PAGE/COMPONENT AUDIT
─────────────────────────────────────────────────────────────

Q20. Any console.log in a component or page that consumes lastEvent? → FORBIDDEN (G-006)
Q21. Does the component check affectsThisEntity before calling refetch()?
     → Without this guard, every global broadcast triggers a refetch on every
       mounted instance of the component (broadcast storm risk)
Q22. Does the component perform cache invalidation (invalidateTags)?
     → Cache invalidation MUST only happen in SSEProvider (SSE-014)
     → Component-level refetch() is allowed with affectsThisEntity guard
Q23. Does the component depend on SSE to render authoritative state?
     → FORBIDDEN (P-1): SSE is a hint; REST API is authoritative

─────────────────────────────────────────────────────────────
SECTION 5: SPEC CONSISTENCY AUDIT
─────────────────────────────────────────────────────────────

Q24. Does any rule cited in a code comment or PR description match an
     actual rule ID in sse-flow-spec.md §9? No dangling rule references?
Q25. Does any core-rule reference (e.g., "Core Rule §3.4") point to a
     section that actually exists in rules.md?
Q26. Does this change introduce behavior that contradicts any existing
     rule in sse-flow-spec.md?
Q27. Does this change require adding a new exception to §7.2 (Payload
     exceptions) or a new line to the §8 invalidation table?
     → If yes, the spec MUST be updated in the same PR

─────────────────────────────────────────────────────────────
OUTPUT FORMAT
─────────────────────────────────────────────────────────────

For each finding, output:
  Finding ID:    F-001, F-002, ...
  Severity:      CRITICAL | HIGH | MEDIUM | LOW
  File/Location: exact file + line number
  Violated Rule: SSE-NNN or Core Rule §N.N
  Description:   what the code does
  Why dangerous: specific business or integrity risk
  Fix direction: what must change (code and/or spec)
  Spec update?:  yes/no — does sse-flow-spec.md need updating too?

If no findings: output "AUDIT PASS — no violations found"
```
