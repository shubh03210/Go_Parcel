# GoParcel — Architecture Conflict Matrix

Audit date 22 November 2026. Conflicts between the architecture (`docs/architecture/GOPARCEL_ARCHITECTURE.md` v0.1), the master PRD v2.0, the decision register, and inside the architecture itself. Section headings are quoted exactly; no line numbers are used.

Type key: **ARCH↔PRD** architecture contradicts or cannot meet the PRD · **ARCH↔ARCH** internal contradiction · **ARCH↔DEC** architecture vs decision register status · **PRD↔PRD** contradiction inside the PRD set that the architecture inherits.

## 1. Architecture vs PRD

| # | Side A (document · section) | Side B (document · section) | Conflict | Severity | Finding | Resolution owner |
|---|---|---|---|---|---|---|
| C-01 | Architecture · §10.2 Encryption and secrets ("handover codes are stored as HMACs ... Plain values exist only in memory and in the outgoing message") | PRD · HAND-004 step 6 ("Sender's screen shows the 6-digit pickup code"); HAND-006 step 2 (link shows "the delivery code") | Hash-only storage cannot redisplay codes. | HIGH | AUD-010 | Architect |
| C-02 | Architecture · §8.4 Payments (port has refund/split/transfer only; "funds never sit in GoParcel's bank account") | PRD · DISP-002 (GoParcel pays sender up to ₹5,000 from reserve); PAY-005 (cash refunds "refunded by GoParcel"); PAY-006 (dues-funded credits paid out) | Required movements need GoParcel-own funds and a pay-to-sender operation not in the port or the chart. | CRITICAL | AUD-001 | PO + counsel + architect |
| C-03 | Architecture · §8.5 Ledger (accounts listed; one cash example) | PRD · LEDGER-001 entry table ("Online earning released after dispute window: Credit"); BOOK-005 compensation "from the retained amount" | No posting rules for the online lifecycle, cancellations, payouts. | HIGH | AUD-002 | Architect + CA |
| C-04 | Architecture · §8.4 ("after PAYMENT_EXPIRED → automatic refund" only) | PRD · BOOK state machine (ACCEPTED_AWAITING_PAYMENT → CANCELLED_BY_SENDER); PAY-001 (retry within window) | Success after cancellation or on a second attempt is unhandled. | HIGH | AUD-003 | Architect |
| C-05 | Architecture · §7.3 The accept transaction (re-checks traveler, DEC-051 online, shipment, capacity) | PRD · BR-03 ("Traveler eligibility is re-checked at acceptance"); MATCH-001 conditions; TRIP-003 (all fields editable before acceptance) | Trip status, stops, window, cash cutoff, sender status not re-checked. | HIGH | AUD-007 | Architect (+PO for edit rule) |
| C-06 | Architecture · §7 / §8 (no cancellation design) | PRD · BOOK-005, BOOK-006, TRIP-004, HAND-004 (decline), HAND-009, SHIP-004 | Required workflows missing. | HIGH | AUD-008 | Architect |
| C-07 | Architecture · §8.4 ("posts ledger entries, then calls the provider") | PRD · §6.3 Payout state machine (INITIATED → PAID / FAILED); LEDGER-001 ("Payout sent: Debit") | Ledger records payout before it is sent; no reversal on FAILED. | HIGH | AUD-009 | Architect |
| C-08 | Architecture · §8.1 Identity and sessions; §8.9 KYC (stores face score, not photo) | PRD · AUTH-004 ("Fresh selfie liveness, matched to the KYC photo on file"; recycled-number rule) | Recovery flow not designed; nothing to match against. | HIGH | AUD-012 | Architect + counsel |
| C-09 | Architecture · §8.1; §8.10 Fraud (`restrictAccount` only) | PRD · AUTH-005 (withdraw trips/parcels, cancel requests, route confirmed bookings to ops) | Cascade missing. | MEDIUM | AUD-017 | Architect |
| C-10 | Architecture · §11.3 (`CHECK (weight_used_g BETWEEN 0 AND weight_cap_g)` per size-class row) | PRD · TRIP-001 ("Number of S, M, L slots and total weight cap") | Weight cap per class vs total. | MEDIUM | AUD-022 | Architect |
| C-11 | Architecture · §8.3 Pricing (AD-14 city-pair table, straight-line × factor fallback) | PRD · PRICE-001 ("PostGIS distance between master-list coordinates, mapped to CFG-DISTANCE-BANDS") | Different input. **Already flagged by the architecture** as needing PO approval. | — (correctly flagged) | — | PO |
| C-12 | Architecture · §6.1 Module map / AD-10 (Privacy module, Support inside Operations) | Decision register · ARCH-10 (18 domains) | Adds a 19th module. **Already flagged** as needing approval. | — (correctly flagged) | — | PO |
| C-13 | Architecture · §8.6 Courier evidence (checksum verified at upload) | PRD · HAND-007 ("Checksums verified on read"; export with integrity manifest) | Read-time verification and export not designed (T-9). | MEDIUM | AUD-028 | Architect |
| C-14 | Architecture · §7.7 (`payments.dispute-window-close`: "PENDING_WINDOW → ELIGIBLE if no hold") | PRD · §6.3 Payout state machine ("window closed, beneficiary verified") | Beneficiary condition omitted. | LOW | AUD-009 | Architect |
| C-15 | Architecture · §8.2 ("Station matching uses master-list station IDs") | PRD · SHIP-001 (origin from "station / city master list"); TRIP-001 (car stops are meeting points) | Exact-ID matching may not connect cities, stations, bus stands and meeting points. | MEDIUM | AUD-023 | PO |
| C-16 | Architecture · §8.9 ("blind index over (doc_type, normalised doc number)") | PRD · KYC-001 ("full Aadhaar number is never stored"); §10.2 (Aadhaar use for counsel) | Whether an HMAC of Aadhaar is permissible and whether the full number is even available. | MEDIUM | AUD-037 | Counsel |
| C-17 | Architecture · §10.7 Threats (maps to PRD, defers to pen test) | PRD · §10.3 ("Threat model per release") | No threat model. | MEDIUM | AUD-032 | Architect / security |
| C-18 | Architecture · §7.4 ("Cash confirmation ... run before code entry ... posts commission + GST") | PRD · HAND-004 step 4 (decline: "no penalty to either party at that moment") | A decline after the cash tap leaves the traveler with a commission debit. | MEDIUM | AUD-020 | Architect |

## 2. Inside the architecture document

| # | Statement A (section) | Statement B (section) | Conflict | Severity | Finding |
|---|---|---|---|---|---|
| I-01 | §7.2 Transactions and lock order: "shipment → bookings → trip_capacity → delivery → payout/ledger ... acquires locks only in this order and is used by every multi-row command" | §7.4 Delivery and custody transitions: Courier moves the delivery, then a transactional event updates Booking, which then calls Shipments | Pickup locks delivery → booking → shipment (reverse). | HIGH | AUD-006 |
| I-02 | §1 Assumptions (OQ-02): "For cash bookings only the booking cutoff applies" | §7.3 The accept transaction: DEC-051 guard "(online)"; no cutoff check for cash | Cash has no time guard at accept. | HIGH | AUD-007 |
| I-03 | §7.7 Scheduled work: jobs "created in the same transaction as the state that needs them" | §14.2 Runtime topology: API on transaction-mode pooler; pg-boss is a separate client | pg-boss does not join Prisma's transaction by default (unverified adapter path). | HIGH | AUD-013 |
| I-04 | §10.6 Supabase hardening: least-privilege API role; migrations only with owner role in CI/CD | AD-03 pg-boss (self-migrates its schema at start unless disabled) | Runtime DDL vs least privilege. | HIGH (part of) | AUD-013 |
| I-05 | §6.3 Module dependencies: "a module may call modules below it, never above" | §6.3 table: Booking → Payments (same layer), Matching → Routes/Shipments/Pricing (same layer); §8.4 Payments places holds (Disputes, higher); §7.6 Notifications/Tracking (L0) handlers "load what they need through contracts" | Rule broken by its own examples. | MEDIUM | AUD-014 |
| I-06 | §7.1: "Every successful transition writes a StateEvent ... in the same transaction" (StateEvent owned by Tracking, §11.2) | §6.3: "Everyone → Tracking, Notifications: Only through integration events (outbox), never direct calls"; §6.2 rule 2 (own tables only) | Same-transaction write into another module's table vs outbox only. | MEDIUM | AUD-016 |
| I-07 | §6.2 rule 2: "A module reads and writes only its own tables" | §11.5 Retention: `privacy.retention` "anonymises personal fields" across entity types; §12: `platform/config` reads `admin.config_value` | Cross-module writes/reads. | MEDIUM | AUD-014 |
| I-08 | §2 G-4 / ARCH-08 "ready to extract modules later" | §7.6 transactional events run other modules' handlers inside one DB transaction | Extraction would need distributed transactions. | MEDIUM | AUD-015 |
| I-09 | §7.4: IN_TRANSIT → EXCEPTION includes "overdue" | §7.4 / §7.7: "Overdue jobs only raise alerts and ops queue items" | Is overdue a state change or an alert? | MEDIUM | AUD-024 |
| I-10 | §6.1: "18 confirmed domains plus two the PRD uses" | §6.1 table and AD-10: one new module (Privacy), "19th module" | Count mismatch. | LOW | AUD-043 |
| I-11 | §11.1 schema list (no `travelers`) | §6.1 Travelers owns Vehicle, car documents, capability state | Missing schema. | LOW | AUD-043 |
| I-12 | §8.1 "access JWT ... in memory or a cookie" | §9.4 cookies are first-party, "no CORS needed" | Token storage and CSRF posture undecided. | MEDIUM | AUD-030 |
| I-13 | §0.2 "Approving this document approves all AD-xx" | §1 assumed answers to open P1 questions; §8.5 "BR-38 ... is enforced" | Approval could be read as approving open product decisions. | MEDIUM | AUD-036 |

## 3. Architecture vs decision register statuses

| # | Decision (register status) | Architecture treatment (section) | Issue | Finding |
|---|---|---|---|---|
| R-01 | D-15 missed-handover fallback (Proposed) | §1 "D-15 approved with a 24 h window"; §7.4 resume/return exits | Treated as approved for design. Acceptable as an assumption if not read as approval. | AUD-036 |
| R-02 | RV-01..RV-04 (Proposed) | §1 "RV-01..RV-04 approved"; §8.4 payouts per RV-04; §8.5 BR-38 enforced | Same. | AUD-036 |
| R-03 | DEC-051 (Confirmed; parameter and cash scope TBD, OQ-02) | §7.3 guard for online; §1 cutoff for cash | Cash part not implemented in §7.3. | AUD-007 |
| R-04 | DEC-052 / DEC-057 (Provisional) | §7.4 resume exit "(DEC-052, provisional)"; T-10 | Correctly kept provisional. | — |
| R-05 | ARCH-10 (Confirmed, 18 domains) | AD-10 adds Privacy | Correctly flagged. | — |
| R-06 | D-11 / BR-31 (Confirmed: funds held by provider) | §8.4 port and §8.5 chart | Compatible for online escrow only; incompatible with compensation, cash refunds and dues-funded payouts (needs counsel). | AUD-001 |
| R-07 | D-18 (Confirmed: payout per delivery) | §8.4 payout run aggregates credits net of dues | Per-delivery vs aggregated payout needs confirmation. | AUD-009 |
| R-08 | D-16 (Confirmed: in-page camera only) | §8.6 camera-only enforced in the browser | Not enforceable server-side; treat as UX, not evidence guarantee. | AUD-027 |

## 4. Inside the PRD set (inherited by the architecture)

| # | Statement A | Statement B | Conflict | Finding |
|---|---|---|---|---|
| P-01 | PRD · PRICE-001 Quote valid until `CFG-QUOTE-TTL` (30 min) | PRD · BOOK-001 request TTL 6 h; PRICE-003 quote on request honoured | Which governs at accept? | AUD-041 |
| P-02 | PRD · HAND-012 "then LOST or DELIVERED-with-claim" | PRD · Delivery state machine (no EXCEPTION → DELIVERED) | Missing edge. | AUD-024 |
| P-03 | PRD · Booking state machine (no CANCELLED_BY_TRAVELER from ACCEPTED_AWAITING_PAYMENT) | PRD · BOOK-006 (traveler cancels or no-shows) | Traveler cancel before payment has no state. | AUD-008 |
| P-04 | PRD · D-18 "Payout per delivery" / §6.3 per-delivery payout machine | PRD · PAY-006 netting across all credits and dues | Granularity. | AUD-009 |
| P-05 | PRD · `CFG-RECYCLED-NUMBER-DAYS` 180 | Real-world operator recycling period (believed shorter, ~90 days; unverified) | Takeover window. | AUD-012 |
| P-06 | PRD header: "There are no companion files" | Companion files present and self-described as companions | Authority ambiguity. | AUD-044 |
| P-07 | PRD · §5.7 Offline (photos only) | PRD · BR-29 custody only on server code verification; short halts | No offline handover. | AUD-011 |
| P-08 | PRD · TRIP-003 "Before any booking request is accepted: all fields editable" | PRD · BOOK-001 / PRICE-003 request carries locked quote and stops | Edits while requests are pending. | AUD-007 |
