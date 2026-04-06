# Object: ScanLog

**Table:** `scan_logs` | **PK:** Long | **Module:** order
**Indexes:** `event_id`, `scanned_by`, `ticket_id`

## Schema

**DB check constraint:**
`result IN ('SUCCESS','VALID','ALREADY_USED','CANCELLED','EXPIRED','INVALID_QR','WRONG_EVENT','TRANSFER_LOCKED')`

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | Long | PK, auto-increment | |
| ticket_id | Long | FK → tickets.id, nullable | null for `INVALID_QR` |
| event_id | UUID | NOT NULL | |
| scanned_by | UUID | FK → users.id, NOT NULL | |
| scanned_at | LocalDateTime | NOT NULL | |
| result | ScanLogResult | NOT NULL | see [enums/scan-log-result.md](../../enums/scan-log-result.md) |
| qr_payload | String | NOT NULL, length=512 | Raw payload as scanned |

## Rules

- Every scan attempt writes a row — both `/tickets/validate` (preview) and `/tickets/scan` (check-in)
- `ticket_id` is `null` when `result = INVALID_QR` (no ticket found)
- Rows are append-only — never updated or deleted
- DB constraint must be updated whenever `ScanLogResult` enum changes (migration required)
