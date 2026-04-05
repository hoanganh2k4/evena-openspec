# Evena — Spec-Driven Development

This directory contains the Spec-Driven Development (SDD) setup for the
Evena monorepo using `@fission-ai/openspec`.

## Directory Layout

```
openspec/
├── config.yaml                     # Project config, rules, global context
├── schemas/
│   └── evena/
│       ├── schema.yaml             # Custom schema — required fields per artifact
│       └── templates/
│           ├── proposal.md         # Tier: full
│           ├── spec.md             # Tier: full + light
│           ├── design.md           # Tier: full
│           ├── tasks.md            # All tiers
│           └── review.md           # All tiers
├── specs/
│   ├── rules.md            # ★ Core business rules (source of truth)
│   ├── data-model.md       # All entities, fields, relationships, DB tables
│   ├── enums.md            # All enum values, state machines, transitions
│   ├── api.md              # All REST endpoints, auth, request/response shapes
│   ├── services.md         # Service layer business logic
│   ├── sse-flow-spec.md    # SSE channels, events, integrity rules
│   └── sse-checklist.md    # SSE review checklist
├── changes/
│   └── YYYY-MM-DD-{slug}/
│       ├── proposal.md  (full only)
│       ├── spec.md      (full + light)
│       ├── design.md    (full only)
│       ├── tasks.md
│       └── review.md
└── prompts/
    ├── system.md           # Paste as Claude system context
    ├── new-change.md       # Start any new change
    ├── audit.md            # Audit files against rules
    ├── validate-spec.md    # Validate spec before implementation
    ├── review.md           # Post-implementation review
    └── blocker-check.md    # Pre-implementation gotcha check
```

## Change Tiers

| Tier | When to use | Artifacts |
|------|------------|-----------|
| **full** | Cross-service, new API contract, DB schema change | proposal → spec → design → tasks → review |
| **light** | Single-service feature, no contract change | spec → tasks → review |
| **patch** | Bug fix, copy, minor refactor | tasks → review |

## Workflow

1. Determine tier
2. Create `openspec/changes/YYYY-MM-DD-{slug}/`
3. Write artifact(s) using templates
4. Use `validate-spec.md` prompt to check for gaps
5. Implement checklist items in `tasks.md`
6. Fill `review.md` — all RC-001–010 checks must pass

## Key Rules (enforced in review)

| ID | Rule |
|----|------|
| G-001 | No hardcoded hex colors |
| G-002 | All user-visible strings use t() + i18n EN+VI |
| G-003 | RTK Query: invalidate item + list tags |
| G-004 | Backend: throw domain exceptions, not JPA violations |
| G-005 | SSE: emit to broadcast + personal channel for owner events |
| G-006 | No console.log in components |
| G-007 | Java: UUID vs Long type safety |
| G-008 | Verified org edit guard in backend + frontend |

## Bootstrap Change

The first example change is at:
`changes/2026-03-10-org-sse-fixes/` — includes spec, tasks (all done), and
a passing review. Use it as a reference for future changes.
