### Account
- Admin
    - Email: admin@eventtickets.com
    - Passwword: Admin@123
- Organizer:
    - Email: admin@eventtickets.com
    - Passwword: Admin@123
- Customer:
    - Email: nguyenhuyhoanganh900@gmail.com
    - Passwword: Admin@123
### Always follow rules of:
/home/haa900/DACN/evena-openspec/openspec/specs/rules.md
/home/haa900/DACN/evena-openspec/openspec/specs/sse-flow-spec.md
### Admin

---

## Venue

**Suite file:** `sse-test/testcases/admin/venue.json`
**Browser group:** `admin_4_venue_list` — 4 admin browsers on `/dashboard/admin` → Venues tab, all idle
**SSE channel:** `public` | **Max delivery:** 3 000 ms

### Payload templates (`_registry/payloads.json`)
| Key | Ref path | Use when |
|-----|----------|----------|
| `venue.create_test` | — | creating a fresh venue |
| `venue.update_from_existing` | `{{venues.data.content.0.*}}` | precondition = `fetch_first_venue` (paginated response) |
| `venue.update_from_created` | `{{venue.*}}` | precondition = inline create (`store_result_as: venue`, single-resource unwrap) |

### Precondition strategies
| Strategy | `store_result_as` | Ref path prefix | Teardown required |
|----------|-------------------|-----------------|-------------------|
| `$ref: fetch_first_venue` | `venues` | `{{venues.data.content.0.` | No |
| inline `POST /api/venues` | `venue` | `{{venue.` | **Yes** — DELETE in teardown |

---

### Case 1 · Create venue `[BROADCAST_REFETCH]`
- **Actor:** Browser-A — `POST /api/venues` body: `venue.create_test` → `store_result_as: venue`
- **Precondition:** none
- **Steps:** A posts new venue
- **SSE** (all): `venue:create` on `public`
- **UI** (all): `item_in_list` for `{{venue.id}}`
- **API** (observers): `GET /api/venues?page=0&size=100` → contains `{{venue.id}}`
- **Teardown:** `DELETE /api/venues/{{venue.id}}`

---

### Case 2.2 · Update — no concurrent editors `[BROADCAST_REFETCH]`
- **Actor:** Browser-A — `PUT /api/venues/{{venues.data.content.0.id}}` body: `venue.update_from_existing` + `$patch.description`
- **Precondition:** `$ref: fetch_first_venue` → `store_result_as: venues` (paginated)
- **Ref path:** `{{venues.data.content.0.id}}`, `{{venues.data.content.0.version}}`, etc.
- **Steps:** A puts update
- **SSE** (all): `venue:update` on `public`
- **UI** (all): `item_in_list` for `{{venues.data.content.0.id}}`
- **API** (observers): `GET /api/venues?page=0&size=100` → contains target id
- **Teardown:** none (uses existing data)

---

### Case 2.1 · Concurrent update — stale lock `[STALE_LOCK]`
- **Actor:** Browser-A — `PUT /api/venues/{{venue.id}}` body: `venue.update_from_created` + `$patch.description`
- **Observers (modal open):** B, C, D — `pre_state: update_modal_open` (via `$override`)
- **Precondition:** inline `POST /api/venues` with `$patch.name` = unique name → `store_result_as: venue`
- **Ref path:** `{{venue.id}}`, `{{venue.version}}`, etc.
- **Steps:**
  1. `$flow: open_modal_for_browsers` — browser_ids: [B, C, D], resource_id: `{{venue.id}}`
  2. A → `PUT /api/venues/{{venue.id}}`
- **SSE** (all): `venue:update` on `public`
- **UI** (`state:update_modal_open` = B, C, D):
  - `stale_warning_visible` — message_contains: `"been modified by another user"`
  - `form_submit_disabled`
- **Teardown:** `DELETE /api/venues/{{venue.id}}`

---

### Case 3.1 · Delete — while editor has modal open `[DELETE_LOCK]`
- **Actor:** Browser-A — `DELETE /api/venues/{{venue.id}}`
- **Observer (modal open):** Browser-B — `pre_state: update_modal_open` (via `$override`)
- **Precondition:** inline `POST /api/venues` with `$patch.name` = unique name → `store_result_as: venue`
- **Ref path:** `{{venue.id}}`
- **Steps:**
  1. B — `ui: open_modal` for `{{venue.id}}`
  2. A — `DELETE /api/venues/{{venue.id}}`
- **SSE** (all): `venue:delete` on `public`
- **UI:**
  - B (`state:update_modal_open`): `$preset: ui_delete_lock` → `form_submit_disabled` + `stale_warning_visible`
  - C, D (`state:idle`): `$preset: ui_item_not_in_list` for `{{venue.id}}`
- **API** (all): `GET /api/venues?page=0&size=100` → excludes `{{venue.id}}`
- **Teardown:** none (resource already deleted in step)

---

### Case 3.2 · Delete — no concurrent editors `[BROADCAST_REFETCH]`
- **Actor:** Browser-A — `DELETE /api/venues/{{venue.id}}`
- **Precondition:** inline `POST /api/venues` with `$patch.name` = unique name → `store_result_as: venue`
- **Ref path:** `{{venue.id}}`
- **Steps:** A deletes venue
- **SSE** (all): `venue:delete` on `public`
- **UI** (all): `item_not_in_list` for `{{venue.id}}`
- **API** (observers): `GET /api/venues?page=0&size=100` → excludes `{{venue.id}}`
- **Teardown:** none (resource already deleted in step)

---

## Category

**Suite file:** `sse-test/testcases/admin/category.json`
**Browser group:** `admin_4_category_list` — 4 admin browsers on `/dashboard/admin` → Categories tab, all idle
**SSE channel:** `public` | **Max delivery:** 3 000 ms

### Payload templates (`_registry/payloads.json`)
| Key | Ref path | Use when |
|-----|----------|----------|
| `category.create_test` | — | creating a fresh category |
| `category.update_from_existing` | `{{categories.data.0.*}}` | precondition = `fetch_first_category` (array response, **not** paginated) |
| `category.update_from_created` | `{{category.*}}` | precondition = inline create (`store_result_as: category`, single-resource unwrap) |

> **Note:** `GET /api/categories` returns a plain array, not a paginated object.
> Correct ref path: `{{categories.data.0.id}}` (NOT `.content.0.`)

### Precondition strategies
| Strategy | `store_result_as` | Ref path prefix | Teardown required |
|----------|-------------------|-----------------|-------------------|
| `$ref: fetch_first_category` | `categories` | `{{categories.data.0.` | No |
| inline `POST /api/categories` | `category` | `{{category.` | **Yes** — DELETE in teardown |

---

### Case 1 · Create category `[BROADCAST_REFETCH]`
- **Actor:** Browser-A — `POST /api/categories` body: `category.create_test` → `store_result_as: category`
- **Precondition:** none
- **Steps:** A posts new category
- **SSE** (all): `category:create` on `public`
- **API** (observers): `GET /api/categories` → contains `{{category.id}}`
- **UI** (all): `item_in_list` for `{{category.id}}`
- **Teardown:** `DELETE /api/categories/{{category.id}}`

---

### Case 2.2 · Update — no concurrent editors `[BROADCAST_REFETCH]`
- **Actor:** Browser-A — `PUT /api/categories/{{categories.data.0.id}}` body: `category.update_from_existing` + `$patch.description`
- **Precondition:** `$ref: fetch_first_category` → `store_result_as: categories` (array)
- **Ref path:** `{{categories.data.0.id}}`, `{{categories.data.0.version}}`, etc.
- **Steps:** A puts update
- **SSE** (all): `category:update` on `public`
- **UI** (all): `item_in_list` for `{{categories.data.0.id}}`
- **API** (observers): `GET /api/categories` → contains target id
- **Teardown:** none (uses existing data)

---

### Case 2.1 · Concurrent update — stale lock `[STALE_LOCK]`
- **Actor:** Browser-A — `PUT /api/categories/{{category.id}}` body: `category.update_from_created` + `$patch.description`
- **Observers (modal open):** B, C, D — `pre_state: update_modal_open` (via `$override`)
- **Precondition:** inline `POST /api/categories` with `$patch.name` = unique name → `store_result_as: category`
- **Ref path:** `{{category.id}}`, `{{category.version}}`, etc.
- **Steps:**
  1. `$flow: open_modal_for_browsers` — browser_ids: [B, C, D], resource_id: `{{category.id}}`
  2. A → `PUT /api/categories/{{category.id}}`
- **SSE** (all): `category:update` on `public`
- **UI** (`state:update_modal_open` = B, C, D):
  - `stale_warning_visible` — message_contains: `"been modified by another user"`
  - `form_submit_disabled`
- **Teardown:** `DELETE /api/categories/{{category.id}}`

---

### Case 3.1 · Delete — while editor has modal open `[DELETE_LOCK]`
- **Actor:** Browser-A — `DELETE /api/categories/{{category.id}}`
- **Observer (modal open):** Browser-B — `pre_state: update_modal_open` (via `$override`)
- **Precondition:** inline `POST /api/categories` with `$patch.name` = unique name → `store_result_as: category`
- **Ref path:** `{{category.id}}`
- **Steps:**
  1. B — `ui: open_modal` for `{{category.id}}`
  2. A — `DELETE /api/categories/{{category.id}}`
- **SSE** (all): `category:delete` on `public`
- **UI:**
  - B (`state:update_modal_open`): `$preset: ui_delete_lock` → `form_submit_disabled` + `stale_warning_visible`
  - C, D (`state:idle`): `$preset: ui_item_not_in_list` for `{{category.id}}`
- **API** (all): `GET /api/categories` → excludes `{{category.id}}`
- **Teardown:** none (resource already deleted in step)

---

### Case 3.2 · Delete — no concurrent editors `[BROADCAST_REFETCH]`
- **Actor:** Browser-A — `DELETE /api/categories/{{category.id}}`
- **Precondition:** inline `POST /api/categories` with `$patch.name` = unique name → `store_result_as: category`
- **Ref path:** `{{category.id}}`
- **Steps:** A deletes category
- **SSE** (all): `category:delete` on `public`
- **UI** (all): `item_not_in_list` for `{{category.id}}`
- **API** (observers): `GET /api/categories` → excludes `{{category.id}}`
- **Teardown:** none (resource already deleted in step)

## Organization

**Suite file:** `sse-test/testcases/admin/organization.json`
**Browser group:** `admin_4_org_mix` — A,B (admin, admin/Organizations tab), C,D (organizer — separate account — organizer org list; D navigated to detail in verify step)
**SSE channels:** `admin`, `organizer` | **Max delivery:** 3 000 ms

> **Account separation:** A,B use `admin@eventtickets.com` (subscribes `public + admin + organizer + user:{id}`). C,D use the separate organizer account (`anh.nguyenhaa2652004@hcmut.edu.vn`, subscribes `public + organizer + user:{id}`). This validates true channel isolation: `admin` channel events must NOT reach C,D.

> **Creation strategy:** Org is created by **organizer browser C** as a *step* (not precondition), so the organizer is the real owner and owner's `user:{id}` personal notifications go to C/D's user ID. Admin (A) runs the teardown DELETE.

### Payload templates (`_registry/payloads.json`)
| Key | Ref path | Use when |
|-----|----------|----------|
| `organization.create_test` | — | creating a fresh org (base template) |

### Step strategies
| Who creates | How | `store_result_as` | Ref path prefix |
|-------------|-----|-------------------|-----------------|
| C (organizer, step) | `POST /api/organizations` with `browser_id: C` | `org` | `{{org.` |

---

### Case 1 · Verify organization `[BROADCAST_REFETCH]` — two-phase
> **Flow:** Organizer creates → admin sees via SSE → admin verifies → organizer notified via SSE on list page (C) and detail page (D).

- **Precondition:** none
- **Steps:**
  1. C (organizer) — `POST /api/organizations` body: `organization.create_test` + `$patch.name` → `store_result_as: org`
  2. D — `ui: navigate` to `/dashboard/organizer/organizations/{{org.id}}` (detail page)
  3. A (admin) — `PATCH /api/organizations/{{org.id}}/verify`
- **SSE (A, B):** `organization:create` on `admin`
- **SSE (A, B):** `organization:verify` on `admin`
- **SSE (C, D):** `organization:verify` on `organizer`
- **API** (B, C): `GET /api/organizations?page=0&size=100` → contains `{{org.id}}`
- **UI (A, B, C):** `item_in_list` for `{{org.id}}` (D on detail page — excluded from list assertion)
- **UI (C):** `has_text` on `[data-id={{org.id}}]` → `"Verified"` (verified chip on org card)
- **UI (D):** `has_text` on detail page → `"Verified"` (verified icon in detail header)
- **Teardown:** `DELETE /api/organizations/{{org.id}}` (admin A)

---

### Case 2 · Delete organization `[BROADCAST_REFETCH]`
> **Flow:** Organizer creates org → admin deletes → both admin and organizer sides see org removed.

- **Precondition:** none
- **Steps:**
  1. C (organizer) — `POST /api/organizations` body: `organization.create_test` + `$patch.name` → `store_result_as: org`
  2. A (admin) — `DELETE /api/organizations/{{org.id}}`
- **SSE (A, B):** `organization:delete` on `admin`
- **SSE (C, D):** `organization:delete` on `organizer`
- **API (A, B, C):** `GET /api/organizations?page=0&size=100` → excludes `{{org.id}}`
- **UI (A, B, C):** `item_not_in_list` for `{{org.id}}`
- **Teardown:** none (deleted in step)

---

### Organizer

---

## Organization

**Suite file:** `sse-test/testcases/organizer/organization.json`
**SSE channels:** `organizer` (A,B,C), `admin` (D) | **Max delivery:** 3 000 ms

> **API endpoint separation:**
> - Organizer browsers list endpoint: `GET /api/organizations/my-organizations` (plain array, no pagination)
> - Admin browser list endpoint: `GET /api/organizations?page=0&size=100` (paginated, `data.content`)

### Browser layouts per case
| Case | Group | Layout |
|------|-------|--------|
| Create (Case 1) | `organizer_3_org_create` | A,B (organizer, org list), C (admin, Organizations tab) |
| Update Case 2.1 | `organizer_4_org_list` | A,B,C (organizer, org list), D (admin, Organizations tab) |
| Update Case 2.2 | `organizer_3_org_create` | A (organizer, org list), B (navigated to org detail in step), C (admin) |
| Update Case 2.3 | `organizer_4_org_list` + `$override B,C: update_modal_open` | B,C have edit modal open; A,D idle |
| Delete Case 3.1 | `organizer_4_org_list` + `$override B: update_modal_open` | B has edit modal open; A,C,D idle |
| Delete Case 3.2 | `organizer_4_org_list` | A,B,C (organizer, org list), D (admin, Organizations tab) |

### Payload templates
| Key | Ref path | Use when |
|-----|----------|----------|
| `organization.create_test` | — | creating org |
| `organization.update_from_created` | `{{org.*}}` | inline create precondition |

### Precondition strategies
| Strategy | `store_result_as` | Ref path prefix | Teardown `actor_bid` |
|----------|-------------------|-----------------|----------------------|
| inline `POST /api/organizations` (actor A) | `org` | `{{org.` | C or D (admin) |

> **Teardown note:** `actor_bid` on precondition/teardown items is now respected by the runner. Admin browser (C or D) must be used for teardown when the org may be verified.
>
> **Backend constraint:** `PUT /api/organizations/{id}` returns 400 if non-admin changes `name` of a **verified** org. Non-identity fields (description, logo, website, email, phone) are always editable.

---

### Case 1 · Create organization `[BROADCAST_REFETCH]`
- **Actor:** Browser-A (organizer on org list) — `POST /api/organizations` body: `organization.create_test` → `store_result_as: org`
- **Precondition:** none
- **Steps:** A posts new org
- **SSE (A, B):** `organization:create` on `organizer`
- **SSE (C):** `organization:create` on `admin`
- **UI (A, B, C):** `item_in_list` for `{{org.id}}`
- **API (B):** `GET /api/organizations/my-organizations` → contains `{{org.id}}`
- **API (C):** `GET /api/organizations?page=0&size=100` → contains `{{org.id}}`
- **Teardown:** `DELETE /api/organizations/{{org.id}}` (`actor_bid: C` — admin)

---

### Case 2.1 · Update unverified org — no concurrent editors `[BROADCAST_REFETCH]`
- **Actor:** Browser-A (organizer on org list) — `PUT /api/organizations/{{org.id}}`
- **Precondition:** inline create unverified org → `store_result_as: org`
- **Steps:** A puts update (`$patch.description`)
- **SSE (A, B, C):** `organization:update` on `organizer`
- **SSE (D):** `organization:update` on `admin`
- **UI (A, B, C, D):** `item_in_list` for `{{org.id}}`
- **API (B, C):** `GET /api/organizations/my-organizations` → contains `{{org.id}}`
- **API (D):** `GET /api/organizations?page=0&size=100` → contains `{{org.id}}`
- **Teardown:** `DELETE /api/organizations/{{org.id}}` (`actor_bid: D` — admin)

---

### Case 2.2 · Verified org — Edit button disabled on list and detail page `[UI_ONLY]`
- **No SSE step** — tests frontend guard only (button state, not click behavior)
- **Precondition:** inline create org (actor A, organizer)
- **Steps:**
  1. C (admin) — `PATCH /api/organizations/{{org.id}}/verify` (fires `organization:verify` SSE → UI refetches)
  2. B — `ui: navigate` to `/dashboard/organizer/organizations/{{org.id}}` (detail page)
- **UI (A — list page):** `edit_button_disabled` with `resource_ref: {{org.id}}` → `[data-id="{id}"] span > button[disabled]`
- **UI (B — detail page):** `edit_button_disabled` (no resource_ref) → `button[disabled].MuiIconButton-root` in detail header
- **Teardown:** `DELETE /api/organizations/{{org.id}}` (`actor_bid: C` — admin; organizer cannot delete verified org)

---

### Case 2.3 · Concurrent update — stale lock `[STALE_LOCK]`
- **Actor:** Browser-A (organizer on org list) — `PUT /api/organizations/{{org.id}}`
- **Observers (modal open):** B, C — `pre_state: update_modal_open` (via `$override`)
- **Precondition:** inline create unverified org → `store_result_as: org`
- **Steps:**
  1. `$flow: open_modal_for_browsers` — browser_ids: [B, C], resource_id: `{{org.id}}`
  2. A → `PUT /api/organizations/{{org.id}}`
- **SSE (A, B, C):** `organization:update` on `organizer`
- **UI (state:update_modal_open = B, C):**
  - `stale_warning_visible` — message_contains: `"been modified by another user"`
  - `form_submit_disabled`
- **Teardown:** `DELETE /api/organizations/{{org.id}}` (`actor_bid: D` — admin)

---

### Case 3.1 · Delete — while editor has modal open `[DELETE_LOCK]`
- **Actor:** Browser-A (organizer on org list) — `DELETE /api/organizations/{{org.id}}`
- **Observer (modal open):** Browser-B — `pre_state: update_modal_open` (via `$override`)
- **Precondition:** inline create org → `store_result_as: org`
- **Steps:**
  1. B opens update modal for `{{org.id}}`
  2. A → `DELETE /api/organizations/{{org.id}}`
- **SSE (A, B, C):** `organization:delete` on `organizer`
- **SSE (D):** `organization:delete` on `admin`
- **UI:**
  - B (`state:update_modal_open`): `stale_warning_visible` + `form_submit_disabled`
  - A, C (organizer, idle): `item_not_in_list` for `{{org.id}}`
  - D (admin): `item_not_in_list` for `{{org.name}}` (use name to avoid GenericDataTable numeric false positives)
- **Teardown:** none (deleted in step)

---

### Case 3.2 · Delete — no concurrent editors `[BROADCAST_REFETCH]`
- **Actor:** Browser-A (organizer on org list) — `DELETE /api/organizations/{{org.id}}`
- **Precondition:** inline create org → `store_result_as: org`
- **Steps:** A deletes org
- **SSE (A, B, C):** `organization:delete` on `organizer`
- **SSE (D):** `organization:delete` on `admin`
- **UI (A, B, C):** `item_not_in_list` for `{{org.id}}`
- **UI (D — admin):** `item_not_in_list` for `{{org.name}}` (use name to avoid GenericDataTable numeric false positives)
- **API (B, C):** `GET /api/organizations/my-organizations` → excludes `{{org.id}}`
- **API (D):** `GET /api/organizations?page=0&size=100` → excludes `{{org.id}}`
- **Teardown:** none (deleted in step)

---

## Draft Event

**Suite file:** `sse-test/testcases/organizer/draft-event.json`
**SSE channels:** `organizer`, `admin` | **Max delivery:** 3 000 ms

### Browser layout
| ID | Account | Page | Role |
|----|---------|------|------|
| A | organizer (actor) | `/dashboard/organizer` (dashboard) | actor |
| B | organizer | `/dashboard/organizer/events` (event list) | observer |
| C | organizer | `/dashboard/organizer/events/{{event.id}}` (event detail) | observer |
| D | admin | `/dashboard/admin` | observer |

> Case 1 (Create): only A, B, D (event detail doesn't exist yet).
> Cases 2 & 3: all 4 browsers. C has `pre_state: update_modal_open` for STALE_LOCK/DELETE_LOCK cases.

### Payload templates
| Key | Ref path | Use when |
|-----|----------|----------|
| `event.create_draft_test` | — | creating a draft event |
| `event.update_from_created` | `{{event.*}}` | inline create precondition |

### Precondition strategies
| Strategy | `store_result_as` | Ref path prefix | Teardown required |
|----------|-------------------|-----------------|-------------------|
| inline `POST /api/events` | `event` | `{{event.` | **Yes** — DELETE in teardown |

---

### Case 1 · Create draft event `[BROADCAST_REFETCH]`
- **Actor:** Browser-A — `POST /api/events` body: `event.create_draft_test` → `store_result_as: event`
- **Browsers:** A, B, D (3 browsers; no event detail page yet)
- **Precondition:** none
- **Steps:** A posts new event
- **SSE:** `event:create` on `organizer,admin`
- **UI (A, B):** `item_in_list` for `{{event.id}}` (event list + dashboard)
- **API (B, D observers):** `GET /api/events/my-events` → contains `{{event.id}}`
- **Teardown:** `DELETE /api/events/{{event.id}}`

---

### Case 2.1.1 · Concurrent update from event list — stale lock `[STALE_LOCK]`
- **Actor:** Browser-A (dashboard) — `PUT /api/events/{{event.id}}`
- **Observers (modal open):** B (event list), C (event detail) — `pre_state: update_modal_open`
- **Precondition:** inline create draft event → `store_result_as: event`
- **Steps:**
  1. `$flow: open_modal_for_browsers` — browser_ids: [B], resource_id: `{{event.id}}` (list page modal)
  2. `$flow: open_modal_for_browsers` — browser_ids: [C], resource_id: `{{event.id}}` (detail page form)
  3. A → `PUT /api/events/{{event.id}}`
- **SSE:** `event:update` on `organizer,admin`
- **UI (state:update_modal_open = B, C):**
  - `stale_warning_visible` — message_contains: `"been modified by another user"`
  - `form_submit_disabled`
- **Teardown:** `DELETE /api/events/{{event.id}}`

---

### Case 2.1.2 · Concurrent update from event detail — stale lock `[STALE_LOCK]`
- **Actor:** Browser-C (event detail page) — `PUT /api/events/{{event.id}}`
- **Observers (modal open):** B (event list modal) — `pre_state: update_modal_open`
- **Precondition:** inline create draft event → `store_result_as: event`
- **Steps:**
  1. `$flow: open_modal_for_browsers` — browser_ids: [B], resource_id: `{{event.id}}`
  2. C → `PUT /api/events/{{event.id}}` (edit form on detail page, not modal)
- **SSE:** `event:update` on `organizer,admin`
- **UI (state:update_modal_open = B):**
  - `stale_warning_visible`
  - `form_submit_disabled`
- **Teardown:** `DELETE /api/events/{{event.id}}`

---

### Case 2.2.1 · Update from event list — no concurrent editors `[BROADCAST_REFETCH]`
- **Actor:** Browser-B (event list) — `PUT /api/events/{{event.id}}`
- **Precondition:** inline create draft event → `store_result_as: event`
- **Steps:** B puts update
- **SSE:** `event:update` on `organizer,admin`
- **UI (all):** event row visible with updated data on dashboard (A), event list (B)
- **UI (C):** event detail page shows updated fields
- **API (observers):** `GET /api/events/my-events` → contains `{{event.id}}`
- **Teardown:** `DELETE /api/events/{{event.id}}`

---

### Case 2.2.2 · Update from event detail — no concurrent editors `[BROADCAST_REFETCH]`
- **Actor:** Browser-C (event detail) — `PUT /api/events/{{event.id}}`
- **Precondition:** inline create draft event → `store_result_as: event`
- **Steps:** C puts update via detail page form
- **SSE:** `event:update` on `organizer,admin`
- **UI (all):** event data updated on A (dashboard), B (event list), C (detail page)
- **Teardown:** `DELETE /api/events/{{event.id}}`

---

### Case 3.1 · Delete — while editor has modal open `[DELETE_LOCK]`
- **Actor:** Browser-A (dashboard) — `DELETE /api/events/{{event.id}}`
- **Observer (modal open):** Browser-B (event list modal) — `pre_state: update_modal_open`
- **Precondition:** inline create draft event → `store_result_as: event`
- **Steps:**
  1. B opens event update modal for `{{event.id}}`
  2. A → `DELETE /api/events/{{event.id}}`
- **SSE:** `event:delete` on `organizer,admin`
- **UI:**
  - B (`state:update_modal_open`): `$preset: ui_delete_lock`
  - A, C, D (`state:idle`): `$preset: ui_item_not_in_list` for `{{event.id}}`
- **API (all):** `GET /api/events/my-events` → excludes `{{event.id}}`
- **Teardown:** none (deleted in step)

---

### Case 3.2 · Delete — no concurrent editors `[BROADCAST_REFETCH]`
- **Actor:** Browser-A — `DELETE /api/events/{{event.id}}`
- **Precondition:** inline create draft event → `store_result_as: event`
- **Steps:** A deletes event
- **SSE:** `event:delete` on `organizer,admin`
- **UI (all):** `item_not_in_list` for `{{event.id}}`
- **API (observers):** `GET /api/events/my-events` → excludes `{{event.id}}`
- **Teardown:** none (deleted in step)

---

## Draft Tickettype

**Suite file:** `sse-test/testcases/organizer/draft-ticket-type.json`
**Pattern:** `SELECTIVE_REFETCH` — event fires on `organizer,admin` but only event detail pages (subscribed to `TicketType` RTK tags) refetch ticket list
**SSE channel:** `organizer`, `admin` | **Max delivery:** 3 000 ms

### Browser layout (5 browsers, all organizer account)
| ID | Page | Expected behavior |
|----|------|-------------------|
| A | `/dashboard/organizer/events/{{event.id}}` | actor (also refetches) |
| B2 | `/dashboard/organizer/events/{{event.id}}` | refetches ticket list |
| B3 | `/dashboard/organizer/events/{{event.id}}` | refetches ticket list |
| B1 | `/dashboard/organizer/events` | receives SSE on `organizer` but **no ticket list change** (doesn't use `useGetTicketTypesQuery`) |
| B4 | `/dashboard/organizer` | receives SSE on `organizer` but **unchanged** |
| B5 | `/dashboard/organizer/organizations/{{org.id}}` | receives SSE on `organizer` but **unchanged** |

> All 5 browsers receive `ticket_type:*` on `organizer` channel.
> Selectivity is determined by RTK Query tag subscriptions, not SSE channel membership.
> SSEProvider invalidates `{ type: 'TicketType', id: eventId }` → only pages using `useGetTicketTypesQuery(eventId)` refetch.

### Payload templates
| Key | Ref path | Use when |
|-----|----------|----------|
| `ticket_type.create_draft_test` | — | creating draft ticket type (price > 0, qty > 0) |
| `ticket_type.update_from_created` | `{{ticket_type.*}}` | inline create precondition |

### Precondition strategies
- Always use a **pre-existing DRAFT event** (fetch or create) → `store_result_as: event`
- Ticket type is inline-created within preconditions for Update/Delete cases → `store_result_as: ticket_type`

---

### Case 1 · Create draft ticket type `[SELECTIVE_REFETCH]`
- **Actor:** Browser-A — `POST /api/events/{{event.id}}/ticket-types` → `store_result_as: ticket_type`
- **Precondition:** fetch (or create) draft event → `store_result_as: event`
- **Steps:** A posts new ticket type
- **SSE:** `ticket_type:create` on `organizer,admin`
- **SSE assertion (received):** A, B2, B3 on `organizer` channel — `ticket_type:create`
- **SSE assertion (not received):** B1 on `public` channel — `ticket_type:create` (confirms not on public)
- **UI (A, B2, B3):** `item_in_list` for `{{ticket_type.id}}` (ticket type appears in event detail)
- **UI (B1, B4, B5):** no change to their visible data
- **Teardown:** `DELETE /api/events/{{event.id}}/ticket-types/{{ticket_type.id}}`

---

### Case 2 · Update draft ticket type `[SELECTIVE_REFETCH]`
- **Actor:** Browser-A — `PUT /api/events/{{event.id}}/ticket-types/{{ticket_type.id}}`
- **Precondition:** create draft event + create draft ticket type → `store_result_as: ticket_type`
- **Steps:** A puts update (description patch)
- **SSE:** `ticket_type:update` on `organizer,admin`
- **SSE assertion (received):** A, B2, B3 — `ticket_type:update` on `organizer`
- **UI (A, B2, B3):** ticket type row shows updated data
- **UI (B1, B4, B5):** unchanged
- **Teardown:** `DELETE /api/events/{{event.id}}/ticket-types/{{ticket_type.id}}`

---

### Case 3 · Delete draft ticket type `[SELECTIVE_REFETCH]`
- **Actor:** Browser-A — `DELETE /api/events/{{event.id}}/ticket-types/{{ticket_type.id}}`
- **Precondition:** create draft event + create draft ticket type → `store_result_as: ticket_type`
- **Constraint:** ticket type must be DRAFT and have zero sales (backend enforces this)
- **Steps:** A deletes ticket type
- **SSE:** `ticket_type:delete` on `organizer,admin`
- **SSE assertion (received):** A, B2, B3 — `ticket_type:delete` on `organizer`
- **UI (A, B2, B3):** `item_not_in_list` for `{{ticket_type.id}}`
- **UI (B1, B4, B5):** unchanged
- **Teardown:** none (deleted in step)

---

## Active Tickettype (on Published Event)

**Suite file:** `sse-test/testcases/organizer/active-ticket-type.json`
**SSE channel:** `public`, `organizer` (when parent event is PUBLISHED) | **Max delivery:** 3 000 ms

> **Channel routing rule (backend):**
> `ticket_type:activate` → `public,organizer` only when parent event is PUBLISHED.
> `ticket_type:deactivate` → `public,organizer` only when TT was ACTIVE/SOLD_OUT on a PUBLISHED event.

> **Cache invalidation chain (SSEProvider):**
> All `ticket_type:*` events invalidate BOTH `TicketType` AND `Event` RTK tags.
> This means (1)(4)(5) also refetch their event queries → min price updates automatically.

### Browser layout (5 browsers, all organizer account)
| ID | Page | Expected behavior on activate |
|----|------|-------------------------------|
| A | `/dashboard/organizer/events/{{event.id}}` | actor; TT status ACTIVE in detail |
| B2 | `/dashboard/organizer/events/{{event.id}}` | TT status ACTIVE, all detail fields refresh |
| B3 | `/dashboard/organizer/events/{{event.id}}` | same as B2 |
| B1 | `/dashboard/organizer/events` | event card min price updates |
| B4 | `/dashboard/organizer` | event card min price updates |
| B5 | `/dashboard/organizer/organizations/{{org.id}}` | event card min price updates |

### Precondition flow
1. Fetch or create a **PUBLISHED event** → `store_result_as: event`
2. Create **DRAFT ticket type** (with price > 0, salesStart/salesEnd set) → `store_result_as: ticket_type`
3. For deactivate case: also activate the TT first (PATCH activate) before the test step

---

### Case 1 · Activate ticket type (DRAFT → ACTIVE) on published event
- **Actor:** Browser-A — `PATCH /api/events/{{event.id}}/ticket-types/{{ticket_type.id}}/activate`
- **Precondition:** published event + draft TT with valid salesStart/salesEnd/price
- **Steps:** A patches activate
- **SSE:** `ticket_type:activate` on `public,organizer`
- **SSE assertion (received):** all 5 browsers on `organizer` — `ticket_type:activate`
- **UI (A, B2, B3):** TT row shows status = ACTIVE; activate button hidden; deactivate button shown
- **UI (B1, B4, B5):** event card min price updated (via Event tag invalidation → `useGetMyEventsQuery` refetch)
- **API (observers):** `GET /api/events/{{event.id}}/ticket-types` → TT has `status: ACTIVE`
- **Teardown:** `PATCH /api/events/{{event.id}}/ticket-types/{{ticket_type.id}}/deactivate`

---

### Case 2 · Deactivate ticket type (ACTIVE → DEACTIVATED) on published event
- **Actor:** Browser-A — `PATCH /api/events/{{event.id}}/ticket-types/{{ticket_type.id}}/deactivate`
- **Precondition:** published event + ACTIVE ticket type (create draft TT, then activate it)
- **Steps:** A patches deactivate
- **SSE:** `ticket_type:deactivate` on `public,organizer`
- **SSE assertion (received):** all 5 browsers on `organizer` — `ticket_type:deactivate`
- **UI (A, B2, B3):** TT row shows status = DEACTIVATED; deactivate button hidden
- **UI (B1, B4, B5):** event card min price updated (may decrease/change based on remaining active TTs)
- **Teardown:** none (TT is deactivated, not deleted; teardown only if test created the event)

---

## Publish Event

**Suite file:** `sse-test/testcases/organizer/publish-event.json`
**SSE channel:** `public`, `organizer` | **Max delivery:** 3 000 ms

> **Precondition:** event must be DRAFT with a VERIFIED organization. Ticket types are optional for publish (depending on backend rules).

### Browser layout
| ID | Account | Page | Expected behavior |
|----|---------|------|-------------------|
| A | organizer (actor) | `/dashboard/organizer/events/{{event.id}}` | publishes event |
| B | organizer | `/dashboard/organizer/events` | event status changes to PUBLISHED in list |
| C | organizer | `/dashboard/organizer` | event card appears/updates on dashboard |
| D | customer | `/dashboard/customer` | published event appears in public listing |

### Cache invalidation chain
- SSEProvider handles `event:publish` → invalidates `{ type: 'Event', id: eventId }` + global `'Event'`
- All pages using `useGetMyEventsQuery` or `useGetEventsQuery` auto-refetch
- Admin dashboard does **not** query events — no update there

---

### Case 1 · Publish draft event `[BROADCAST_REFETCH]`
- **Actor:** Browser-A — `PATCH /api/events/{{event.id}}/publish`
- **Precondition:** create draft event under verified org → `store_result_as: event`
- **Steps:** A patches publish endpoint
- **SSE:** `event:publish` on `public,organizer`
- **SSE assertion (received):** all 4 browsers — `event:publish` (A, B, C on `organizer`; D on `public`)
- **UI (A):** event status badge changes to PUBLISHED; publish button hidden
- **UI (B):** event list shows PUBLISHED status for `{{event.id}}`
- **UI (C):** event card on dashboard shows PUBLISHED
- **UI (D - customer):** event appears in public event listing → `item_in_list` for `{{event.id}}`
- **API (B observer):** `GET /api/events/my-events` → event has `status: PUBLISHED`
- **API (D observer):** `GET /api/events` (public) → contains `{{event.id}}`
- **Teardown:** `PATCH /api/events/{{event.id}}/cancel` (or DELETE if still allowed)

---

## Cancelled Event

**Suite file:** `sse-test/testcases/organizer/cancel-event.json`
**SSE channels:** `public`, `organizer` | **Max delivery:** 3 000 ms | **Pattern:** `BROADCAST_REFETCH`

> **Cascade behavior:** When `event:cancel` fires, backend silently deactivates ALL non-DEACTIVATED ticket types. No individual `ticket_type:deactivate` signals are emitted (spec §10 #14).
>
> **Frontend cache gap:** `event:cancel` SSE causes SSEProvider to invalidate only `Event` tags, NOT `TicketType` tags. Ticket type statuses on the event detail page may remain stale until the page refetches the TT list. This is a **known gap** — document it, do not paper over it in tests.
>
> **Redirect guard:** Customer event detail page (`/dashboard/customer/events/[id]`) uses a `useEffect` status guard that redirects to `/dashboard/customer` when event status is not `PUBLISHED`. The redirect is triggered by SSE-invalidated refetch — not SSE directly.

### Browser layout
| ID | Account | Page | Role | Expected behavior |
|----|---------|------|------|-------------------|
| A | organizer (actor) | `/dashboard/organizer/events/{{event.id}}` | actor | navigates to event detail, calls cancel API |
| B | organizer | `/dashboard/organizer/events` | observer | event row status changes to CANCELLED (EventRowCard status chip) |
| C | organizer | `/dashboard/organizer` | observer | event card on dashboard shows CANCELLED |
| D | customer | `/dashboard/customer` | observer | cancelled event disappears from public listing |
| E | admin | `/dashboard/admin` | observer | receives `event:cancel` on organizer channel |
| F | organizer | `/dashboard/organizer/events/{{event.id}}` | observer | event detail status chip updates to CANCELLED |
| G | customer | `/dashboard/customer/events/{{event.id}}` | observer | status guard redirects to `/dashboard/customer` |

---

### Case 1 · Cancel published event `[BROADCAST_REFETCH]`

**Preconditions:**
1. `$ref: fetch_first_venue`, `$ref: fetch_first_category`, `$ref: fetch_first_org`
2. Admin (E) approves org via `PATCH /api/organizations/{{orgs.data.0.id}}/verify`
3. Organizer creates DRAFT event → `store_result_as: event`
4. Organizer creates + activates a ticket type for the event
5. Organizer publishes event via `PATCH /api/events/{{event.id}}/publish`

**Steps:**
1. Browser-A navigates to `/dashboard/organizer/events/{{event.id}}`
2. Browser-A calls `PATCH /api/events/{{event.id}}/cancel` (expect 200)
3. *(inline assert)* Browser-G: `current_url` contains `/dashboard/customer` within 5 000 ms — captures live redirect transition triggered by SSE refetch

**SSE assertions:**
| ID | Browsers | Channel | Event type | Expect |
|----|----------|---------|------------|--------|
| `sse-cancel-on-organizer` | A, B, C, F | `organizer` | `event:cancel` | received (12 000 ms) |
| `sse-cancel-on-public` | D, G | `public` | `event:cancel` | received (12 000 ms) |
| `sse-cancel-on-admin` | E | `organizer` | `event:cancel` | received (12 000 ms) |
| `sse-no-cascade-tt-deactivate` | D | `public` | `ticket_type:deactivate` | **not_received** (wait 3 000 ms) |

**API assertions:**
| ID | Browser | Endpoint | Expect |
|----|---------|----------|--------|
| `api-event-removed-from-public` | D | `GET /api/events?page=0&size=100` | `not_contains_resource: {{event.id}}` |

**UI assertions:**
| ID | Browsers | Type | Condition |
|----|----------|------|-----------|
| *(preset)* `ui_item_in_list` | B | `item_in_list` | event row still present in organizer list (not removed, status changed) |
| `ui-cancel-status-event-list` | B | `has_text` "CANCELLED" | EventRowCard status chip shows CANCELLED |
| `ui-cancel-status-organizer-dashboard` | C | `has_text` "CANCELLED" | EventCard on organizer dashboard shows CANCELLED |
| `ui-cancel-status-chip-event-detail` | F | `has_text` "CANCELLED" | Event detail page status chip updated to CANCELLED |
| `ui-cancel-customer-event-gone` | D | `item_not_in_list` | Cancelled event absent from customer public listing |

**Teardown:** Admin (E) calls `DELETE /api/events/{{event.id}}` (organizers cannot delete non-DRAFT events)