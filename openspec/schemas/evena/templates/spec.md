# Spec: {{title}}

**ID:** SPEC-XXX
**Proposal:** PROP-XXX <!-- omit for light/patch tier -->
**Author:** {{author}}
**Date:** {{YYYY-MM-DD}}
**Tier:** <!-- full | light | patch -->
**Status:** draft

---

## Overview

<!-- 2–5 sentences describing what this spec covers. -->

## User Stories

<!-- Format: "As a <role>, I want <action>, so that <outcome>." -->
- As a …, I want …, so that …

## Out of Scope

<!-- Explicit exclusions — prevents scope creep -->
-

---

## API Changes

<!-- List every new or modified endpoint.
     Use: METHOD /path — description
     Write "none" if no changes. -->

| Method | Path | Change | Auth |
|--------|------|--------|------|
| POST   | /api/... | New | ORGANIZER |

## Database Changes

<!-- List schema changes (new tables, columns, indexes, constraints).
     Write "none" if no changes. -->
-

## SSE Events

<!-- List new or modified SSE events.
     Format: action → channel(s) | payload fields
     Write "none" if no changes. -->
-

## Frontend Changes

<!-- List components/pages/hooks to create or modify. -->
-

---

## Error Cases

<!-- Document expected error responses -->
| Scenario | HTTP Status | Error message |
|----------|-------------|---------------|
| … | 400 | … |

## i18n Keys Required

<!-- New translation keys — add to BOTH en and vi translation.json -->
```
translation.json path: messages.xxx.yyy
  EN: "..."
  VI: "..."
```

## Security Notes

<!-- Any auth checks, ownership guards, or role restrictions -->

---

> Next step (full tier): Create `DSGN-XXX` in `changes/{{slug}}/design.md`
> Next step (light/patch tier): Create `TASK-XXX` in `changes/{{slug}}/tasks.md`
