# Spec: FlexPass — Flexible Ticket Resale System

**ID:** SPEC-004
**Author:** team
**Date:** 2026-03-24
**Tier:** feature
**Status:** draft

---

## Overview

FlexPass là tính năng cho phép người mua vé đăng ký **vé linh hoạt** — vé có thể được nhượng lại trên sàn nội bộ trong một khoảng thời gian nhất định, với mức giá tối đa +20% giá gốc. Hệ thống gồm hai giao diện:

1. **FlexPass Admin** (organizer): quản lý, phê duyệt/từ chối các listing vé bán lại.
2. **FlexPass Customer** (customer): mua vé kèm FlexPass, niêm yết vé nhượng lại, theo dõi vé.

---

## Scope

| Layer | Thành phần |
|---|---|
| UI organizer | `FlexPassAdmin` page + `ResaleTicketCard`, `FlexPassStatsCard` components |
| UI customer | `FlexPassCustomer` page + `CustomerTicketCard`, `ResaleMarketCard` components |
| Sidebar | Icon mới cho mục **FlexPass Admin** trong sidebar của organizer |
| Data | Mock data (prototype) — không kết nối backend trong phase này |

---

## 1. FlexPass Admin (Organizer View)

### 1.1 Mục đích

Cho phép organizer:
- Xem tổng quan thống kê vé FlexPass của sự kiện.
- Duyệt hoặc từ chối các yêu cầu nhượng lại vé.
- Tìm kiếm và lọc theo trạng thái listing.

### 1.2 Layout

```
[Sidebar] [Header]
          [Body panel: bg-[#f7f7f7] rounded-[20px]]
            ├── Page title: "FlexPass Admin"
            ├── Stats cards (4 cột)
            ├── Search + Status filter bar
            └── Grid vé đang nhượng (2 cột)
```

### 1.3 Stats Cards (4 thẻ)

| Thẻ | Giá trị | Icon | Màu icon bg |
|---|---|---|---|
| Tổng vé FlexPass | `1,245` | `Ticket` (lucide) | `#fef1ff` (tím nhạt) |
| Đang bán lại | `462` | `ShoppingBag` (lucide) | `#eff6ff` (xanh nhạt) |
| Tỷ lệ thành công | `78%` | `TrendingUp` (lucide) | `#ecfdf5` (xanh lá nhạt) |
| Chờ phê duyệt | count động | `AlertCircle` (lucide) | `#fffbeb` (vàng nhạt) |

### 1.4 Filter & Search

- **Search input**: tìm theo tên sự kiện hoặc tên người bán (case-insensitive).
- **Status select**: `Tất cả` / `Chờ duyệt` / `Đã duyệt` / `Từ chối`.
- Filter áp dụng đồng thời (AND logic).

### 1.5 ResaleTicketCard

Mỗi thẻ vé nhượng lại hiển thị:
- Event name + category + số lượng vé.
- Seller info: avatar initials + tên + rating + số giao dịch.
- Giá nhượng + giá gốc (gạch chân) + % tăng giá.
- Status badge: `pending` → amber, `approved` → green, `rejected` → red.
- Actions (chỉ hiện khi `pending`): nút **Từ chối** + **Duyệt**.
- Thời gian niêm yết.

### 1.6 Trạng thái listing

```
pending  → hiện nút Từ chối + Duyệt
approved → chỉ hiện badge "Đã duyệt"
rejected → chỉ hiện badge "Từ chối"
```

### 1.7 Empty state

Khi không tìm thấy vé nào (sau filter/search):
```
bg-white rounded-[12px] border p-[40px] text-center
"Không tìm thấy vé nào"
```

---

## 2. FlexPass Customer (Customer View)

### 2.1 Mục đích

Cho phép customer:
- Xem thông tin sự kiện + trạng thái FlexPass.
- Mua vé với tùy chọn kích hoạt FlexPass.
- Xem và mua vé nhượng lại từ người khác.
- Quản lý vé đang giữ, tải QR, và niêm yết lại.

### 2.2 Layout

```
[Sidebar] [Header]
          [Body panel: bg-[#f7f7f7] rounded-[20px]]
            └── Center container (max-w-[680px] mx-auto)
                  ├── Page header "FlexPass" + badge "concept"
                  ├── Event card
                  ├── Tabs card: [Mua vé | Vé nhượng lại | Vé của tôi]
                  └── Cơ chế hoạt động (step list)
```

### 2.3 Event Card

- Thumbnail gradient + emoji sự kiện (60x60).
- Tên sự kiện + địa điểm + ngày.
- Badges: "Đã xác thực" (green) + "Còn N ngày mở bán lại" (amber).
- Progress bar: % vé đã bán + số lượng FlexPass còn lại.

### 2.4 Tab: Mua vé

- Danh sách `CustomerTicketCard` theo hạng vé.
- Mỗi card: emoji hạng + tên hạng + phân khu + giá + badge "FlexPass".
- Selected state: border xanh + bg xanh nhạt.
- Toggle **Kích hoạt FlexPass**: nhượng lại trong 14 ngày, tối đa +20%.
- CTA button: "Tiếp tục đặt vé".

### 2.5 Tab: Vé nhượng lại

- Danh sách `ResaleMarketCard`: vé đang được niêm yết bởi người khác.
- Mỗi card: emoji + tên hạng + seller info + giá nhượng + % so giá gốc.
- Footer note: "Vé được khoá trong escrow · thanh toán khi xác nhận danh tính".

### 2.6 Tab: Vé của tôi

- Vé đang giữ: tên sự kiện + số lượng + ngày + FlexPass còn N ngày.
- Chi tiết giá mua + giá nhượng tối đa (highlighted green).
- Actions: **Tải QR vé** + **Nhượng lại**.
- Lịch sử giao dịch: text summary.

### 2.7 Cơ chế hoạt động (step list)

4 bước mô tả luồng FlexPass:
1. Người mua chọn FlexPass → vé gắn eKYC.
2. Nếu không đi → niêm yết lại, giá tối đa +20%.
3. Người mua mới xác thực CCCD → không sang tay thêm lần nữa.
4. Hết hạn niêm yết mà chưa bán → hoàn tiền 70–80% (tùy tier).

---

## 3. Sidebar Icon — FlexPass Admin

### 3.1 Yêu cầu

File icon mới thay thế icon hiện tại (copy/paste icon) ở mục **FlexPass Admin** trong sidebar organizer.

**Path**: `frontend/assets/icon/flexpass-admin/`

### 3.2 Thiết kế icon

**Concept**: Ticket ngang có stub bên phải, dòng chấm chia đôi, mũi tên lên/xuống trong stub thể hiện tính linh hoạt (resale).

**Kích thước**: 20×20px viewBox

**Cấu trúc path (evenodd)**:
- Outer ticket: hình chữ nhật bo góc (2–18, 5–15) với notch lõm tại x=2 và x=18, tâm y=10.
- Divider dots: 3 ô vuông nhỏ tại x=12 (y=7.2–7.8, 9.7–10.3, 12.2–12.8) → tạo ra lỗ trắng (đường chấm ngăn cách).
- Stub arrows: tam giác lên (14,9.3)–(16,9.3)–(15,7.7) và tam giác xuống (14,10.7)–(16,10.7)–(15,12.3) → lỗ trắng hình mũi tên trong stub.

**Hai variant màu**:
- `flexpass-admin-dark-blue.svg`: fill `#36437C`
- `flexpass-admin-pink.svg`: fill `#F36BF9`

---

## 4. Routing

| Route | Component | activePage |
|---|---|---|
| `/flexpass-admin` | `FlexPassAdmin` | `"flexpass-admin"` |
| `/flexpass` | `FlexPassCustomer` | `"flexpass"` |

---

## 5. Business Rules

| Rule | Giá trị |
|---|---|
| FlexPass resale window | 14 ngày sau khi mua |
| Giá nhượng tối đa | +20% giá gốc |
| Refund nếu không bán được | 70–80% tùy tier |
| Sang tay tối đa | 1 lần (người mua mới không thể nhượng lại) |
| Xác thực danh tính | eKYC (CCCD) bắt buộc trước khi nhận vé nhượng |

---

## 6. Spec References

- `new-feature/app/pages/FlexPassAdmin.tsx` — prototype organizer view
- `new-feature/app/pages/FlexPassCustomer.tsx` — prototype customer view
- `new-feature/app/components/flexpass/` — prototype components
- `new-feature/imports/flexpass_concept.html` — original concept HTML
