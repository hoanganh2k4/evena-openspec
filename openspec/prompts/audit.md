# Prompt: Codebase Audit

Use this to audit a file or subsystem against project rules.

---

**Paste this to Claude:**

```
Audit the following file(s) against the Evena project rules
defined in openspec/config.yaml:

{{file paths or "the entire frontend/src/stores/services/ directory"}}

For each violation found, report:
- Rule ID (G-001 through G-008)
- File path and line number
- Current code
- What it should be
- Severity (warning / error)

Do NOT fix anything yet — just report. I'll decide which to fix.
```
