# Prompt: Blocker Check (Before Starting Implementation)

Run this before touching any code on a new task. It catches both
implementation gotchas and semantic blockers that make a task undeliverable
without additional groundwork.

---

**Paste this to Claude:**

```
Before I start implementing {{TASK-ID or description}}, check for blockers
in the Evena codebase and specs.

─────────────────────────────────────────────────────────────
PART A: IMPLEMENTATION GOTCHAS
─────────────────────────────────────────────────────────────

A1. RTK Query mutations
    → Does this touch a mutation? Verify invalidatesTags includes
      BOTH item tag {type, id} AND list tag (string form).

A2. SSE channel selection
    → Does this add or modify an SSE emit?
      → What is the entity state at emit time? DRAFT or PUBLISHED?
      → DRAFT events and their TicketTypes → "organizer,admin" only (SSE-001, SSE-002)
      → Orders and tickets → "user:{userId}" only (SSE-007)
      → Is emit-after-commit guaranteed? (SSE-008)
        Confirm: notify*() call is NOT inside @Transactional body.

A3. SSE personal channel
    → Does this target a specific user (org owner, invitee, order owner)?
      → Verify "user:{uuid}" channel used for personal notification
      → Verify sse-service/config/channels.json allows the resource on user:{id} pattern
      → Verify SSEProvider.tsx has a handler for the new normalized type

A4. Java type safety
    → Does this call SSENotificationService with a user ID?
      → Confirm User.id is UUID (not Long)
    → Does this reference Organization/Event/Category/Venue ID?
      → Confirm those are Long (not UUID)

A5. Organization guard
    → Does this modify an organization?
      → Confirm verified-org guard is present in the service layer
        (organization.verified && !isAdmin() → BadRequestException)

A6. i18n
    → Does this add any user-visible string (toast, label, error)?
      → Add to BOTH en and vi translation.json

A7. JPA pre-checks
    → Does this delete or reassign an entity with FK constraints?
      → Add pre-check (count references) and throw BadRequestException
        before the FK constraint violation can occur.

A8. Cache invalidation
    → If adding a new SSEProvider case: does it follow sse-flow-spec.md §8?
      → Specify MUST Invalidate and MUST NOT Invalidate for the new event type.
    → Is the invalidation happening only in SSEProvider (not also in a page component)?

─────────────────────────────────────────────────────────────
PART B: SEMANTIC BLOCKERS
─────────────────────────────────────────────────────────────

B1. Backend endpoint filter
    → If this task depends on customers only seeing PUBLISHED events (SSE-012):
      Has the backend GET /events endpoint been confirmed to enforce
      status=PUBLISHED server-side? If not, fix backend first.

B2. Refund workflow existence
    → If this task adds order:refund SSE (SSE-005):
      Does a refund domain exist in the backend (RefundService or equivalent)?
      Without the domain, order:refund cannot be attached to a real event.
      Block this task until the refund domain is implemented.

B3. Transaction commit ordering
    → If this task adds a new notify*() call:
      Is the call outside the @Transactional boundary?
      If not, does the service use @TransactionalEventListener(AFTER_COMMIT)?
      If neither, the emit may race the commit. Fix the ordering first.

B4. Duplicate invalidation
    → If this task touches SSEProvider AND any hook/page component:
      Is there a risk of adding cache invalidation in both places?
      Confirm SSEProvider is the single invalidation source (SSE-014).

B5. Invitee ID resolution
    → If this task implements invitation:create personal notification (SSE-006):
      Is the inviteeId (UUID) available at the point of SSE emission?
      Currently only inviteeEmail is passed. Resolving UUID from email
      is a prerequisite blocker for this task.

B6. Unresolved spec contradictions
    → Does this task relate to any rule marked as "UNKNOWN" or "NOT IMPLEMENTED"
      in sse-flow-spec.md §16?
      → List the status and confirm whether the unknown must be resolved first.

B7. Payload exception declaration
    → Does this task require sending a field not in sse-flow-spec.md §7.1?
      → If yes, the exception MUST be declared in §7.2 BEFORE implementation.
      → Do not implement and retroactively update spec — update spec first.

B8. Missing actions.json entry
    → Does this task add a new SSE action?
      → The action MUST be added to sse-service/config/actions.json AND
        channels.json BEFORE the backend can emit it.

─────────────────────────────────────────────────────────────
PART C: SPEC HEALTH CHECK
─────────────────────────────────────────────────────────────

C1. Are there any unresolved conflicts between rules.md and sse-flow-spec.md
    that affect this task? If yes, do not implement until the conflict is resolved.

C2. Does this task introduce behavior where SSE becomes the trigger for
    a business operation (refund, cancellation, etc.) rather than the notification?
    → SSE MUST be notification-only (P-1, SSE-013). If the task conflates
      SSE trigger with business logic, block it and redesign.

C3. Does this task assume SSE delivery is guaranteed?
    → SSE delivery is NOT guaranteed (P-4, SSE-013). If the task design fails
      when SSE is missed, block it and add a fallback REST-based recovery path.

─────────────────────────────────────────────────────────────
OUTPUT FORMAT
─────────────────────────────────────────────────────────────

For each check: CLEAR | BLOCKED | RISK (explain)

List any BLOCKED items as hard prerequisites that must be resolved before
implementation can start.

List any RISK items as things to keep in mind and verify during implementation.
```
