# Tasks: {{title}}

**ID:** TASK-XXX
**Spec:** SPEC-XXX <!-- or DSGN-XXX for full tier -->
**Author:** {{author}}
**Date:** {{YYYY-MM-DD}}
**Status:** open <!-- open | in-progress | done | blocked -->

---

## Checklist

### backend

- [ ] **T1** — `CategoryService.java`: …
- [ ] **T2** — `SSENotificationService.java`: …
- [ ] **T3** — `XxxController.java`: …
- [ ] **T4** — Add/modify `@Query` in `XxxRepository.java`

### sse-service

- [ ] **T5** — `config/channels.json`: add `xxx` to `allowed_resources`

### frontend

- [ ] **T6** — `src/stores/services/XxxApi.ts`: fix `invalidatesTags`
- [ ] **T7** — `src/providers/SSEProvider.tsx`: add case for `SSENormalizedType.XXX_CREATED`
- [ ] **T8** — `src/stores/types/sse.ts`: add `SSEAction.XXX` + `SSENormalizedType.XXX`
- [ ] **T9** — `src/components/Xxx/Xxx.tsx`: implement component
- [ ] **T10** — `src/i18n/locales/en/translation.json`: add i18n keys
- [ ] **T11** — `src/i18n/locales/vi/translation.json`: add i18n keys

---

## Notes

<!-- Anything that might trip up implementation -->
- Remember: RTK Query `invalidatesTags` must include BOTH item tag AND list tag
- Remember: User.id is UUID, not Long — verify types before calling SSENotificationService
- Remember: Add i18n keys to BOTH en and vi files

---

> Next step: After all items done, create `REV-XXX` in `changes/{{slug}}/review.md`
