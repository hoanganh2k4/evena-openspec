# Tasks: FlexPass — Flexible Ticket Resale System

**Change:** SPEC-004
**Date:** 2026-03-24

---

## Phase 1 — Assets & Icons

- [ ] Thay thế icon FlexPass Admin trong `frontend/assets/icon/flexpass-admin/`
      bằng icon ticket+stub+mũi tên (thiết kế theo SPEC-004 §3.2)
      - `flexpass-admin-dark-blue.svg` (fill `#36437C`)
      - `flexpass-admin-pink.svg` (fill `#F36BF9`)

- [ ] Cập nhật `FlexPassIcon` trong `Sidebar.tsx` để dùng icon mới từ assets
      (thay vì dùng chung `p3a8e4f70` với Orders icon)

---

## Phase 2 — Shared Components

- [ ] Copy `FlexPassStatsCard.tsx` vào `frontend/src/components/flexpass/`
      (từ `new-feature/app/components/flexpass/FlexPassStatsCard.tsx`)

- [ ] Copy `ResaleTicketCard.tsx` vào `frontend/src/components/flexpass/`
      (từ `new-feature/app/components/flexpass/ResaleTicketCard.tsx`)

- [ ] Copy `CustomerTicketCard.tsx` vào `frontend/src/components/flexpass/`
      (từ `new-feature/app/components/flexpass/CustomerTicketCard.tsx`)

- [ ] Copy `ResaleMarketCard.tsx` vào `frontend/src/components/flexpass/`
      (từ `new-feature/app/components/flexpass/ResaleMarketCard.tsx`)

---

## Phase 3 — Pages

- [ ] Tạo page `FlexPassAdmin` tại `frontend/src/pages/organizer/FlexPassAdmin.tsx`
      theo spec §1 (stats cards, search/filter, resale ticket grid)

- [ ] Tạo page `FlexPassCustomer` tại `frontend/src/pages/customer/FlexPassCustomer.tsx`
      theo spec §2 (event card, 3 tabs, how-it-works)

---

## Phase 4 — Routing & Sidebar

- [ ] Thêm route `/flexpass-admin` → `FlexPassAdmin` vào router organizer

- [ ] Thêm route `/flexpass` → `FlexPassCustomer` vào router customer

- [ ] Thêm nav item "FlexPass Admin" vào sidebar organizer với icon mới
      `activePage="flexpass-admin"` → active color `#F36BF9`

---

## Phase 5 — Styles

- [ ] Kiểm tra `index.css` / `tailwind.config` đã có các utility classes dùng trong components
      (không cần thêm nếu đã dùng Tailwind standard classes)
