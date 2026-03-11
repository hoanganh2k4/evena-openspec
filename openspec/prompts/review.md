# Prompt: Post-Implementation Review

Use this after completing all tasks in tasks.md.

---

**Paste this to Claude:**

```
I have finished implementing all tasks in
openspec/changes/{{slug}}/tasks.md.

Please perform a review:

1. Read the spec at openspec/changes/{{slug}}/spec.md
2. Read each changed file listed in tasks.md
3. Fill in openspec/changes/{{slug}}/review.md
   — go through RC-001 to RC-010 one by one
   — set status to pass / fail / needs-changes
   — list any blocking issues

If status is needs-changes, list the exact files and lines
to fix. Do NOT auto-fix — show me first.
```
