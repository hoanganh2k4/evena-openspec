# Service: TicketService & QRCodeService

---

## QRCodeService

### `generateQRPayload(Long ticketId, UUID userId)`
Generates HMAC-signed payload: `{ticketId}:{userId}:{nonce}:{hmac}`
- `nonce` = 32-char hex (UUID without hyphens)
- `hmac` = `HMAC-SHA256("ticketId:userId:nonce", QR_SECRET)` — hex-encoded

### `verifyAndExtractTicketId(String payload) → Optional<Long>`
1. Split by `:` → expect 4 parts
2. Recompute expected HMAC from parts[0..2]
3. Constant-time compare with parts[3]
4. Return `Optional.of(ticketId)` if valid; `empty()` if tampered/malformed

### `generateQRCodeBase64(String payload)`
Renders payload as PNG QR code → returns Base64 string.

---

## TicketService

### Shared QR lookup (used by scan + validate)
```
1. qrCodeService.verifyAndExtractTicketId(payload)
   → empty → INVALID_QR (signature invalid)

2. ticketRepository.findById(ticketId)
     .filter(t -> t.qrPayload.equals(payload))
   → empty → INVALID_QR (QR rotated by FlexPass transfer)
```

### `scanTicket(ScanRequest)` — confirms check-in
1. QR verification (above)
2. Guard: `ticket.event == request.eventId` → else `WRONG_EVENT`
3. Switch on `ticket.status`:
   - `USED` → `ALREADY_USED`
   - `CANCELLED` → `CANCELLED`
   - `EXPIRED` → `EXPIRED`
   - `TRANSFER_LOCKED` → `TRANSFER_LOCKED`
   - `ACTIVE` → set `status=USED`, `usedAt=now`; write `ScanLog(SUCCESS)`; emit SSE

### `validateTicket(ScanRequest)` — preview, no state change
Same lookup + event check. Writes `ScanLog` with appropriate result.
Does **not** modify `ticket.status`.

### `checkInTicket(Long ticketId)` — check-in by ID (no QR)
Guard: `status == ACTIVE`. Set `status=USED`, `usedAt=now`. Write `ScanLog(SUCCESS)`.

### `getScanLogs(UUID eventId)`
Returns all `ScanLog` rows ordered by `scannedAt DESC`.
Guard: caller must be org member or ADMIN.
