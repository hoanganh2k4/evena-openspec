# Prompt: Start a New Change

Use this prompt when beginning any new feature, fix, or refactor.

---

**Paste this to Claude:**

```
I want to implement: {{brief description}}

1. Determine the tier (full / light / patch) based on:
   - Does it touch multiple services? → full
   - Single service, no new API contract? → light
   - Bug fix / copy / minor refactor? → patch

2. Create the appropriate artifacts in
   openspec/changes/{{YYYY-MM-DD}}-{{slug}}/
   using the templates in openspec/schemas/evena/templates/

3. Before writing any code, show me the spec (or tasks for a patch)
   and wait for my approval.

4. After I approve, implement the checklist items one by one,
   marking each [x] as you go.

5. When all tasks are done, fill in review.md against RC-001–RC-010.
```
