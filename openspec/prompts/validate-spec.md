# Prompt: Validate a Spec Before Implementation

Use this before starting work on a spec to catch gaps early.

---

**Paste this to Claude:**

```
Validate openspec/changes/{{slug}}/spec.md against the schema at
openspec/schemas/evena/schema.yaml.

Check:
1. All required fields are present and non-empty
2. API changes table is complete (method, path, auth)
3. SSE events section covers both broadcast AND personal channels
   if the feature has owner-targeted events
4. Error cases table covers all business rule violations
5. i18n keys listed for every user-visible string introduced
6. No scope creep vs the Out of Scope section

Report any gaps. Do not start implementation until gaps are addressed.
```
