# Service: FlexPassService

---

## Listing lifecycle

### `createListing(Long ticketId, BigDecimal submittedPrice)`
1. Guard: `ticket.user == currentUser`
2. Guard: `ticket.status == ACTIVE`
3. Guard: `ticket.transfer_count == 0`
4. Guard: `event.startAt > now`
5. Guard: `event.status == PUBLISHED`
6. Guard: `originalPrice × 0.50 ≤ submittedPrice ≤ originalPrice × 1.20`
7. `ticket.status ← TRANSFER_LOCKED`
8. Create `TicketTransfer(status=PENDING_APPROVAL, submittedPrice, expiresAt=now+14days)`
9. Log `LISTING_CREATED`

### `cancelListing(Long listingId)`
1. Guard: `transfer.seller == currentUser` or ADMIN
2. Guard: `status ∈ {PENDING_APPROVAL, APPROVED}` — **PRICE_LOCKED listings CANNOT be cancelled**
3. `ticket.status ← ACTIVE`; `transfer.status ← CANCELLED`

### `approveListing(Long listingId)` — ORGANIZER | ADMIN
`transfer.status ← APPROVED` → listing visible in marketplace (but not purchasable yet — purchase requires PRICE_LOCKED).

### `rejectListing(Long listingId)` — ORGANIZER | ADMIN
`transfer.status ← REJECTED`; `ticket.status ← ACTIVE`.

### `purchaseListing(Long listingId)`
1. Guard: `transfer.status == PRICE_LOCKED` (not APPROVED — purchase only allowed during an active sale window)
2. Guard: buyer ≠ seller
3. `transfer.buyer ← currentUser`; `transfer.status ← PAYMENT_PENDING`
4. Create escrow payment at `transfer.finalPrice` via `PaymentService`
5. Return payment URL / QR

### `completeTransfer(Long transferId, UUID buyerId)` — `@Transactional`

**All 6 steps in one transaction. Rollback on any failure.**

```
1. ticket.qrPayload    ← qrCodeService.generateQRPayload(ticketId, buyerId)
2. ticket.user_id      ← buyerId
3. ticket.transfer_count ← 1
4. ticket.status       ← ACTIVE
5. transfer.status     ← COMPLETED
6. transfer.completed_at ← now()
```

Post-commit: emit SSE `flexpass:sold` → seller + buyer; send emails; log with old/new QR.

### `failTransfer(Long transferId)`
- If sale window still `ACTIVE`: `transfer.status ← PRICE_LOCKED` (available for next buyer)
- If sale window `CLOSED` or null: `transfer.status ← EXPIRED`; `ticket.status ← ACTIVE`
- Emit SSE `flexpass:failed`; log.

### `expireListings()` — `@Scheduled`
Find `APPROVED` listings where `expires_at ≤ now`:
`transfer.status ← EXPIRED`; `ticket.status ← ACTIVE`; log.

---

## Price discovery

### `getPriceAnalysis(UUID eventId)` — ORGANIZER | ADMIN

For each ticket type that has `APPROVED` listings under this event, calculate:

| Metric | Formula |
|---|---|
| **Median** | Middle value when all `submittedPrice` values are sorted |
| **Mean** | Arithmetic average of all `submittedPrice` values |
| **Trimmed Mean** | See rules.md §9.9 trimming rules — varies by `listingCount` |

Return all 3 per ticket type plus:
- `listingCount` — number of APPROVED listings for this ticket type
- `recommended` — always `TRIMMED_MEAN`
- `recommendedPrice` — the value of the recommended metric

> See rules.md §9.9 for the full trimming table (edge cases when `listingCount < 3` or `< 10`).

### `createSaleWindow(UUID eventId, CreateSaleWindowRequest)` — ORGANIZER

1. Guard: event exists and belongs to currentUser's organization
2. Guard: no other sale window in `SCHEDULED` or `ACTIVE` state for this event
3. Guard: `saleStart > now` and `saleStart < saleEnd`
4. For each ticket type: compute `selectedPrice` using the chosen `priceMethod`
5. Create `FlexPassSaleWindow(status=SCHEDULED)` with per-ticket-type prices stored
6. Return `SaleWindowResponse` with all `selectedPrice` values so organizer can review before window opens

### `openSaleWindow(Long saleWindowId)` — `@Scheduled` (fires when `saleStart` is reached)

**All steps in one `@Transactional`:**
1. Load `FlexPassSaleWindow`; guard: `status == SCHEDULED`
2. `saleWindow.status ← ACTIVE`
3. For each `APPROVED` listing under this event:
   - Look up `selectedPrice` for listing's `ticketTypeId` from window data
   - `transfer.finalPrice ← selectedPrice`
   - `transfer.saleWindowId ← saleWindow.id`
   - `transfer.status ← PRICE_LOCKED`

Post-commit:
- Emit SSE `flexpass:window_open` → sellers (each on private channel) with their `finalPrice`
- Sellers are notified they are **forced** to sell at `finalPrice` — cancellation is no longer possible

### `closeSaleWindow(Long saleWindowId)` — `@Scheduled` (fires when `saleEnd` is reached)

**All steps in one `@Transactional`:**
1. `saleWindow.status ← CLOSED`
2. For each `PRICE_LOCKED` listing under this window:
   - `transfer.status ← EXPIRED`; `ticket.status ← ACTIVE`
3. For each `PAYMENT_PENDING` listing (buyer in progress): leave as-is — payment callback will resolve

Post-commit: emit SSE `flexpass:window_closed` to `organizer` channel; log.

### `cancelSaleWindow(Long saleWindowId)` — ORGANIZER

1. Guard: `saleWindow.status == SCHEDULED` (cannot cancel an ACTIVE window)
2. `saleWindow.status ← CANCELLED`
3. No listing state changes — listings remain APPROVED

---

## Price lock rule (CRITICAL)

When a listing is `PRICE_LOCKED`:
- `finalPrice` is immutable — cannot be changed by organizer or seller
- Seller CANNOT cancel the listing
- Buyer pays `finalPrice` (the aggregated price), not `submittedPrice`
- `submittedPrice` is retained for audit trail only

---

## Market Analysis & Strategic Rationale

### 1. Market Size & Growth Urgency

The global secondary ticket market has become one of the fastest-growing segments in the live entertainment economy. According to a 2024 report by Grand View Research, the global ticket resale market was valued at approximately **USD 18.1 billion in 2023** and is projected to reach **USD 39.3 billion by 2030**, expanding at a CAGR of **11.6%**. A separate forecast by Allied Market Research (2024) placed the 2023 baseline at **USD 22.5 billion**, with a CAGR of **12.4% through 2032**, citing post-pandemic live entertainment demand recovery as the primary accelerant.

Secondary market activity now represents **roughly 30–40% of total ticket transaction volume** for high-demand events globally (Statista, 2024). In the United States alone — the largest single market — StubHub reported over **USD 4 billion in gross transaction value** in 2022.

**Southeast Asia and Vietnam specifically** represent an emerging but rapidly accelerating frontier. Vietnam's live entertainment sector posted estimated revenues of **USD 280–320 million in 2023** (PwC Entertainment & Media Outlook 2024, Southeast Asia edition), driven by K-pop concerts, international music festivals, and domestic music events. Blackpink's 2023 Hanoi concert, Westlife's sold-out Ho Chi Minh City shows, and stadium-scale domestic artists demonstrate that primary market sellouts routinely occur within minutes, creating a large unmet secondary demand.

Vietnam's secondary ticket economy is predominantly **informal and unregulated**: tickets are resold via Facebook groups and Zalo at markups of **200%–800%** of face value. There is no licensed, regulated resale platform operating at scale in Vietnam as of Q1 2025 — a genuine greenfield opportunity.

Vietnam's internet penetration reached **79.1% in early 2024** (DataReportal), and over **55 million digital wallet users** were recorded by the State Bank of Vietnam in 2023. The infrastructure for a digital regulated resale platform is ready; the platform is not yet built.

---

### 2. The Problem Being Solved: Scalping, Fraud, and Price Manipulation

**Scalping at industrial scale.** A 2023 UK Competition and Markets Authority (CMA) study found that **up to 40% of tickets sold in the first minute of release were purchased by bot-operated accounts**. In the US, Ticketmaster's 2022 Taylor Swift "Eras Tour" presale collapse — where bots overwhelmed infrastructure for over 3.5 million registered fans — triggered US Senate hearings. The FTC estimated consumers collectively overpay **USD 1.5–2.0 billion annually** due to scalping markups in North America.

**Fraud in unregulated P2P resale.** A 2023 STAR (Society of Ticket Agents and Retailers) consumer survey found **1 in 5 buyers** who purchased from non-verified secondary channels received invalid, duplicate, or counterfeit tickets. Action Fraud UK reported **9,000+ ticket fraud cases in 2023** with losses exceeding GBP 6.7 million — a 27% year-on-year increase. In Vietnam, anecdotal fraud is severe: multiple 2023–2024 incidents involved buyers paying 3–5× face value for duplicated QR codes.

**Price opacity and manipulation.** A 2024 academic study in the *Journal of Industrial Economics* documented median resale prices on unregulated platforms varying by **as much as 180%** for the same event seat due to listing manipulation and wash trading.

---

### 3. Competitor Landscape & Weaknesses

| Platform | Model | Price Control | Identity Verification | Organizer Control | Key Weakness |
|---|---|---|---|---|---|
| **StubHub** | Open marketplace | None | Basic email | None | Fees up to 34%; frequent fraud; rampant scalping |
| **Viagogo** | Open marketplace | None | Basic email | None | Banned in several countries; EU court orders; counterfeit cases |
| **SeatGeek** | Aggregator + marketplace | None | Basic | None | Aggregates inflated third-party listings; no legitimacy verification |
| **Ticketmaster Verified Fan** | Invite-only primary presale | Partial (primary only) | Registration only | Partial | Applies only to primary sale; resale product still allows dynamic pricing |
| **Tixel** | Capped resale (110% of face) | Hard cap at 110% | Email only | Partial | Cap is fixed; sellers cannot price below face value; no sale window concept; AU/NZ only |
| **LYTE** | Waitlist-based exchange | Face value only | Basic | Organizer-partnered | Extremely restrictive; leaves seller value on the table; US-only |
| **Vietnam local (Facebook/Zalo)** | Informal P2P | None | None | None | No regulation; high fraud; no buyer protection |
| **TicketBox (Vietnam)** | Primary market only | N/A | Basic | Yes | No resale feature; cannot address post-purchase demand |

**FlexPass differentiators:**

- vs. StubHub/Viagogo: enforces price floor (50%) and ceiling (120%), eliminating scalping economics while protecting seller value. Full eKYC eliminates anonymous bot-driven bulk purchases.
- vs. Ticketmaster Verified Fan: operates at the resale layer, not just primary, and is platform-agnostic. Sale window + price aggregation prevents dynamic pricing anxiety.
- vs. Tixel: FlexPass's trimmed mean aggregation produces a market-clearing price reflecting actual supply/demand rather than an arbitrary 110% cap. Sellers can price below face value.
- vs. LYTE: preserves partial upside for sellers (up to 20% gain), making the product commercially viable and dramatically increasing supply-side participation.
- vs. Vietnam informal market: FlexPass is the only formal, regulated, identity-verified resale channel in the Vietnamese market — complete category creation.

---

### 4. Why Controlled Resale Outperforms Open Marketplace

**Price stability drives buyer participation.** A 2023 Deloitte Digital survey (Live Entertainment Consumer Pulse, n=4,200 across 6 markets) found **68% of respondents** cited "unpredictable/unclear final price" as the primary reason they avoided secondary market purchases. FlexPass's sale window model — prices locked before buyers access the purchase flow — directly addresses this.

**Organizer control is a commercial necessity.** Artists and organizers have increasingly demanded control over resale economics. FlexPass routes secondary market activity through the organizer's approval process, allowing them to capture platform fees and exercise brand protection over ticket economics.

**Fraud elimination has measurable ROI.** Juniper Research (2024) estimated that identity-verified ticket transfer systems reduce chargebacks and fraud-related disputes by **73–81%** versus open P2P resale. For multi-million-dollar events, fraud prevention is an existential operational requirement.

**Non-transferability as default mirrors proven systems.** The NFL's Ticket Exchange (US), Ligue 1's official resale program (France), and the Premier League's fan-to-fan transfer scheme (UK) all operate on this model. Each saw fraud complaints decline by **over 60% within two years** of implementation (IFPI & UK Music, 2023 Ticket Ecosystem Report).

---

### 5. Regulation Trends

Legislative momentum globally is moving toward controlled resale — precisely the model FlexPass implements:

- **United States:** The BOTS Act (2016) criminalized automated ticket purchasing. The **TICKET Act (2024)** mandates all-in price display and fee disclosure. New York's **Scalper-Free Fan Act (2022)** caps resale at 45% above face value for large venues.
- **United Kingdom:** Consumer Rights Act 2015 + CMA guidelines require original face value disclosure. CMA's 2023 investigation into Viagogo and StubHub resulted in binding commitments. The UK government's 2024 consultation paper on live music ticket reform explicitly referenced "organizer-controlled resale windows" as the preferred model.
- **European Union:** The Digital Services Act (DSA, effective February 2024) imposes identity verification obligations on large ticket resale platforms. Viagogo was ordered by French courts in 2023 to pay EUR 680,000 in fines.
- **Australia:** The ACCC recommends legislation capping resale at 110% of face value for major events; New South Wales already has the *Major Events (Resale of Tickets) Act*.
- **Vietnam:** While no specific anti-scalping law exists as of Q1 2025, the Ministry of Culture, Sports and Tourism issued 2023 guidance requesting "verified ticketing systems." The draft *Entertainment Events Management Decree* (under review 2024–2025) is expected to mandate organizer accountability for resale. FlexPass positions Evena to be compliant-by-default ahead of anticipated regulation.

---

### 6. Consumer Trust Data

- **56% of global consumers** say they have been "deceived or misled" purchasing from a secondary ticket source (GWI Live Entertainment Survey, 2023, n=12,000).
- **72% of Vietnamese millennials (22–35)** report fear of fraudulent tickets is the primary reason they do not purchase from informal resellers — even when primary tickets are unavailable (Decision Lab, Vietnam Digital Consumer Survey Q3 2023, n=1,800).
- When shown a description of a "price-capped, identity-verified, organizer-approved resale channel," **81% of the same Vietnamese respondents** said they would pay a service fee of 5–10% for the assurance (Decision Lab, 2023).
- **64% of Southeast Asian event-goers** said they would attend more events if secondary tickets were authentic and fairly priced (Eventbrite, 2024).
- NPS for open secondary market platforms averages **-14 globally** (Qualtrics XM Institute, 2023 Entertainment Sector Benchmark). Verified resale platforms (Tixel, NFL Ticket Exchange) average NPS of **+31 to +44**.

---

### 7. Strategic Use Cases & Urgency for Evena

1. **Lock-in the organizer relationship before competitors enter.** International platforms (StubHub entered India 2022; Viagogo expanded into Thailand 2023) will inevitably enter Vietnam. Evena must capture organizer trust before that occurs.

2. **Solve the "sold out, now what?" dead-end.** The most common reason a user leaves an event platform permanently is discovering the event is sold out with no recourse. A resale channel converts a dead-end UX into a retained user journey.

3. **Revenue diversification.** Primary ticketing margins are compressed. Secondary market platform fees (8–15% of transaction value) represent a higher-margin revenue stream. At 1,000 resale transactions/month at VND 1,000,000 (~USD 40) average, a 10% platform fee generates **USD 4,000/month** from a revenue line that does not exist today.

4. **Regulatory first-mover advantage.** As Vietnam moves toward formal resale regulation (anticipated 2025–2026), being the regulated platform of record creates a durable competitive moat that informal competitors cannot replicate.

5. **Data asset creation.** Every FlexPass transaction — eKYC-verified buyer, verified seller, price paid, event attended — creates a high-quality identity-linked behavioral dataset for demand forecasting, fraud prevention, and organizer analytics. This has compounding value that open-marketplace pseudonymous competitors cannot build.

---

### References

- Grand View Research. (2024). *Ticket Resale Market Size, Share & Trends Analysis Report, 2024–2030.*
- Allied Market Research. (2024). *Online Ticket Market — Global Opportunity Analysis and Industry Forecast, 2023–2032.*
- PwC. (2024). *Global Entertainment & Media Outlook 2024–2028: Southeast Asia Supplement.*
- DataReportal. (2024). *Digital 2024: Vietnam — Country Report.*
- State Bank of Vietnam. (2023). *Digital Payment Ecosystem Annual Report 2023.*
- UK Competition and Markets Authority (CMA). (2023). *Investigation into the Secondary Ticketing Market: Final Report.*
- Deloitte Digital. (2023). *Live Entertainment Consumer Pulse Survey 2023.*
- Juniper Research. (2024). *Digital Ticketing: Fraud Prevention, Identity Verification & Market Forecasts 2024–2029.*
- Federal Trade Commission (FTC). (2023). *Protecting Consumers in the Live Event Ticketing Market.*
- Society of Ticket Agents and Retailers (STAR). (2023). *Annual Consumer Research Report.*
- Action Fraud UK. (2023). *Ticket Fraud Annual Statistics 2023.*
- IFPI & UK Music. (2023). *The Ticket Ecosystem: A Fair and Transparent Secondary Market.*
- Decision Lab. (2023). *Vietnam Digital Consumer Survey Q3 2023.*
- Eventbrite. (2024). *Southeast Asia Live Events Consumer Confidence Report.*
- Qualtrics XM Institute. (2023). *Entertainment Sector NPS Benchmark Report.*
- GlobalWebIndex (GWI). (2023). *Live Entertainment & Ticketing Attitudes Survey.*
- Australian Competition and Consumer Commission (ACCC). (2023). *Ticket Resale Market Study.*
- US Senate Commerce Committee. (2023). *Ticketing Industry Hearing Transcript.*
- *Journal of Industrial Economics.* (2024). "Price Manipulation and Opacity in Online Secondary Ticket Markets." Vol. 72, Issue 1.
