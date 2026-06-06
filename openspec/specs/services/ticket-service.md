# Service: TicketService & QRCodeService

---

## QRCodeService

### `generateQRPayload(Long ticketId, UUID userId)`
Generates static HMAC-signed payload (stored in DB): `{ticketId}:{userId}:{nonce}:{hmac}`
- `nonce` = 32-char hex (UUID without hyphens)
- `hmac` = `HMAC-SHA256("ticketId:userId:nonce", QR_SECRET)` — hex-encoded
- Called once at ticket issuance; result stored in `ticket.qr_payload`

### `generateTimeWindowToken(Long ticketId, UUID userId)`
Generates rotating time-window token (returned in API responses): `TW:{ticketId}:{userId}:{timeSlot}:{hmac}`
- `timeSlot` = `floor(System.currentTimeMillis() / 1000 / 300)` — changes every 5 minutes
- `hmac` = `HMAC-SHA256("TW:ticketId:userId:timeSlot", QR_SECRET)`
- Called in `OrderMapper.toTicketResponse()` on every API response — not stored in DB

### `verifyTimeWindowToken(String payload) → Optional<TimeWindowResult>`
1. Split by `:` → expect 5 parts; `parts[0]` must equal `"TW"`
2. Parse `timeSlot` from `parts[3]`; compare to `floor(now / 300)` — strict equality (no grace period)
3. Constant-time HMAC verify over `"TW:parts[1]:parts[2]:parts[3]"`
4. Return `Optional.of(TimeWindowResult(ticketId, userId))` if valid; `empty()` if expired or tampered

### `verifyAndExtractTicketId(String payload) → Optional<Long>`
Static QR verification (fallback):
1. Split by `:` → expect 4 parts
2. Recompute expected HMAC from parts[0..2]
3. Constant-time compare with parts[3]
4. Return `Optional.of(ticketId)` if valid; `empty()` if tampered/malformed

### `generateQRCodeBase64(String payload)`
Renders any payload as PNG QR code → returns Base64 string.

---

## TicketService

### Shared QR lookup (used by scan + validate)
```
1. verifyTimeWindowToken(payload)
   → present → findByIdWithDetails(ticketId).filter(userId matches) → ticketOpt

2. (if empty) verifyAndExtractTicketId(payload)
   → empty → INVALID_QR (signature invalid)
   → present → findByIdWithDetails(ticketId).filter(qrPayload equals) → ticketOpt

3. ticketOpt empty → INVALID_QR
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
