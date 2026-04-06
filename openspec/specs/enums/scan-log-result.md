# Enum: ScanLogResult

Every scan attempt — both `/tickets/validate` (preview) and `/tickets/scan` (check-in) — writes a `ScanLog` row.

| Value | Meaning | Written by |
|---|---|---|
| `SUCCESS` | Check-in confirmed; ticket marked USED | `/tickets/scan` |
| `VALID` | Ticket valid preview; **not** checked in | `/tickets/validate` |
| `ALREADY_USED` | Ticket already checked in | Both |
| `CANCELLED` | Ticket is cancelled | Both |
| `EXPIRED` | Ticket is expired | Both |
| `INVALID_QR` | HMAC invalid or payload not found | Both |
| `WRONG_EVENT` | Ticket belongs to a different event | Both |
| `TRANSFER_LOCKED` | Ticket locked for FlexPass transfer | Both |

## Notes

- `ticket_id` is `null` when `result = INVALID_QR`
- DB check constraint must list **all 8 values** — migration required if enum changes
- `INVALID_QR` covers both tampered/forged QR (HMAC fail) and rotated QR (FlexPass transfer)
