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

Xem đầy đủ business rules tại `rules.md` §11.

---

## 6. Transfer Security Architecture

### 6.1 Attack vectors & mitigations

| Attack | Mitigation |
|---|---|
| Screenshot QR trước khi transfer | QR rotation — `qrPayload` đổi UUID mới ngay khi transfer hoàn thành |
| Double-spend (scan & transfer đồng thời) | `TRANSFER_LOCKED` status — scanner từ chối check-in khi ticket đang locked |
| Re-transfer (Customer 2 bán lại) | `transfer_count` field — chặn listing nếu ≥ 1 |
| TOCTOU race (check-in trong lúc payment đang xử lý) | Ticket bị lock từ lúc listing được tạo (PENDING_APPROVAL) |
| Escrow failure (tiền chuyển nhưng vé không) | Single `@Transactional` — rollback toàn bộ nếu bất kỳ bước nào fail |

### 6.2 Transfer state machine

```
Ticket:
  ACTIVE → TRANSFER_LOCKED   (listing created)
  TRANSFER_LOCKED → ACTIVE   (cancelled / failed / expired → seller gets ticket back)
  TRANSFER_LOCKED → ACTIVE   (completed → buyer gets ticket, new QR)

TicketTransfer:
  PENDING_APPROVAL → APPROVED       (organizer)
  PENDING_APPROVAL → REJECTED       (organizer → ticket unlocked)
  APPROVED → PAYMENT_PENDING        (buyer pays)
  PAYMENT_PENDING → COMPLETED       (payment SUCCESS → QR rotated atomically)
  PAYMENT_PENDING → FAILED          (payment FAILED → ticket unlocked)
  APPROVED → CANCELLED              (seller cancels)
  APPROVED → EXPIRED                (resale window elapsed)
```

### 6.3 Atomic QR rotation (critical path)

Khi `PAYMENT_PENDING → COMPLETED`, các bước sau PHẢI xảy ra trong **1 transaction**:

```
1. ticket.qrPayload  ← UUID mới   // QR cũ trở thành INVALID_QR ngay lập tức
2. ticket.user_id    ← buyer.id   // đổi owner
3. ticket.transfer_count ← 1      // chặn re-transfer
4. ticket.status     ← ACTIVE     // mở khoá cho buyer
5. transfer.status   ← COMPLETED
6. transfer.completed_at ← now()
```

Nếu bất kỳ bước nào fail → rollback hoàn toàn. QR của seller vẫn valid cho đến khi transaction commit thành công.

### 6.4 Database schema

**Thêm cột vào `tickets`:**
```sql
ALTER TABLE tickets ADD COLUMN transfer_count INT NOT NULL DEFAULT 0;
```

**Bảng mới `ticket_transfers`:**
```sql
CREATE TABLE ticket_transfers (
    id             BIGSERIAL PRIMARY KEY,
    ticket_id      BIGINT        NOT NULL REFERENCES tickets(id),
    seller_id      UUID          NOT NULL REFERENCES users(id),
    buyer_id       UUID          REFERENCES users(id),        -- null cho đến khi có buyer
    listing_price  DECIMAL(10,2) NOT NULL,
    original_price DECIMAL(10,2) NOT NULL,                    -- snapshot từ order_items.unit_price
    status         VARCHAR(30)   NOT NULL DEFAULT 'PENDING_APPROVAL',
    expires_at     TIMESTAMP,
    completed_at   TIMESTAMP,
    created_at     TIMESTAMP     NOT NULL DEFAULT now(),
    updated_at     TIMESTAMP     NOT NULL DEFAULT now()
);
CREATE INDEX idx_tt_ticket  ON ticket_transfers(ticket_id);
CREATE INDEX idx_tt_seller  ON ticket_transfers(seller_id);
CREATE INDEX idx_tt_status  ON ticket_transfers(status);
```

### 6.5 API endpoints

| Method | Path | Auth | Mô tả |
|---|---|---|---|
| POST | `/api/flexpass/listings` | [Auth] | Tạo listing — lock ticket |
| DELETE | `/api/flexpass/listings/{id}` | [Auth] | Huỷ listing — unlock ticket |
| GET | `/api/flexpass/listings?eventId=` | [Public] | Marketplace |
| GET | `/api/flexpass/listings/my` | [Auth] | Listing của tôi |
| PATCH | `/api/flexpass/listings/{id}/approve` | [ORGANIZER\|ADMIN] | Duyệt |
| PATCH | `/api/flexpass/listings/{id}/reject` | [ORGANIZER\|ADMIN] | Từ chối |
| POST | `/api/flexpass/listings/{id}/purchase` | [Auth] | Mua — tạo escrow |

Payment callback tái sử dụng `/api/payment/callback` hiện có, nhận diện
transfer order qua `order.type = FLEXPASS`.

### 6.6 New enum values

**TicketStatus** — thêm:
- `TRANSFER_LOCKED` — ticket đang trong quá trình chuyển nhượng

**ScanLogResult** — thêm:
- `TRANSFER_LOCKED` — scanner từ chối vì ticket đang bị lock

**TicketTransferStatus** (enum mới):
`PENDING_APPROVAL`, `APPROVED`, `REJECTED`, `PAYMENT_PENDING`,
`COMPLETED`, `FAILED`, `CANCELLED`, `EXPIRED`

---

## 7. Spec References

- `new-feature/app/pages/FlexPassAdmin.tsx` — prototype organizer view
- `new-feature/app/pages/FlexPassCustomer.tsx` — prototype customer view
- `new-feature/app/components/flexpass/` — prototype components
- `new-feature/imports/flexpass_concept.html` — original concept HTML
