# GoParcel — Architecture Re-audit Report (Round 2)

| Item | Value |
|---|---|
| Document under audit | `docs/architecture/GOPARCEL_ARCHITECTURE.md` v0.2 (Draft for product-owner review, 22 Nov 2026) |
| Baseline | Round-1 audit of v0.1 (`docs/architecture/audit/`, findings AUD-001..AUD-051); v0.1 archived at `docs/architecture/archive/GOPARCEL_ARCHITECTURE_v0.1.md` |
| Product source of truth | `docs/product/PRD/GOPARCEL_MASTER_PRD.md` v2.1, incl. Appendix A (config), C.7 (PD-01..PD-20, which override Section 4 text where they conflict), D (open questions), Section 4, §6 BR-01..38, §7 NFRs |
| Audit date | 22 November 2026 |
| Auditor role | Independent principal software / security / database / payments architect. Did not write v0.1 or v0.2. |
| Method | Document review and reasoning only. **No code exists, no test was run, and no vendor or library behaviour was checked online in this round.** Every library or vendor claim is labelled unverified. |
| Companion files | `GOPARCEL_PRODUCTION_READINESS_REPORT_V2.md`, `GOPARCEL_REAUDIT_DECISION_LOG.md` (same folder) |

No existing file was changed by this audit.

---

## 1. Summary

v0.2 is a substantial, honest revision. It applies the twenty product-owner decisions, adds a funds-flow model, a chart of accounts and posting rules, a complete canonical lock order with a per-flow lock table, an explicit allowed-dependency list, provider-as-source-of-truth payment handling, payable items with payout batches, two-way encrypted handover codes, recipient OTP, recovery flows, a reconciliation design, a threat-model starter and a foundation spike. The round-1 CRITICAL finding (AUD-001) is structurally answered, and no round-1 finding is left untouched.

What is still wrong is mostly **internal consistency between the new interlocking rules**. Four problems are serious enough to rate HIGH:

1. **The custody-value cap (PD-10) can be bypassed.** The cap counts CONFIRMED and IN_FULFILMENT bookings but not ACCEPTED_AWAITING_PAYMENT, and payment confirmation never re-checks it (AUD2-001).
2. **One payout is executed as two or three provider operations under one reference and one state.** A partial failure reverses the whole payout and the next run can pay the escrow-funded part a second time (AUD2-002).
3. **Offline-queued handovers (PD-11) are not reconciled with cancellation and no-show.** A booking can be cancelled and refunded while the traveler physically holds the parcel; the queue lives only in browser memory; there is no server-side "verification pending" state and no delivery-stage sync deadline (AUD2-003).
4. **The exhaustive allowed-dependency list (§6.3) is contradicted by at least ten flows the document itself describes** (Identity calling KYC for selfie re-match, Ledger calling Identity to restrict, lower-module sweepers running multi-aggregate flows, Booking needing dispute outcomes, Fraud needing claims data, and others). This was a P0 item in round 1 and still blocks the CI lint rule (AUD2-004).

No new CRITICAL finding was found. No two flows listed in the §7.2 lock table can deadlock with each other on the rows they list; the remaining lock-order risk is in flows the table omits and in the sweeper claim pattern.

### 1.1 Counts

**Round-1 findings AUD-001..AUD-049 (defects):**

| Status | Count |
|---|---|
| Fixed | 33 |
| Partially fixed | 15 |
| Not fixed | 0 |
| No longer applicable | 1 |
| **Total** | **49** |

AUD-050 and AUD-051 were informational and are not scored.

**New findings (AUD2-xxx):**

| Severity | Count |
|---|---|
| CRITICAL | 0 |
| HIGH | 4 |
| MEDIUM | 11 |
| LOW | 8 |
| INFO | 2 |
| **Total** | **25** |

Defects are AUD2-001..AUD2-023. AUD2-024 and AUD2-025 are informational. Optional suggestions that are not defects are listed separately in §3.4.

---

## 2. Round-1 finding status

"Fixed" means the original problem is closed by the v0.2 design. Where the fix introduced a different defect, that defect is a new AUD2 finding and is cross-referenced; it does not reopen the original finding unless the original problem itself remains. "Partially fixed" means part of the original problem remains.

| ID | Round-1 title (short) | R1 severity | Status | v0.2 section | Justification |
|---|---|---|---|---|---|
| AUD-001 | Funds flow not defined end-to-end | CRITICAL | Partially fixed | 8.4.1 Where money sits; 8.4.2 Every money movement | M1–M11, merchant balance, reserve, `payoutToBeneficiary`, sender beneficiary and vendor criteria close the structural gap (PD-07). Still open: composite payouts (AUD2-002), no movement that releases retained cancellation revenue or withheld tax from escrow and no trigger for the commission release (AUD2-008), online RETURNED/LOST money outcome TBD (T-15, AUD2-015), counsel and CA sign-off (T-1, T-6). No longer CRITICAL. |
| AUD-002 | Posting rules missing | HIGH | Partially fixed | 8.4.3 Chart of accounts; 8.4.4 Posting rules | 24 balanced rules and the missing accounts were added; every rule E1–E24 balances (§4.3). Gaps: COD-refusal dues, overpaid-dues refund, zero-net payout, E23 funding account, sender-dues write-off, escrow release of K−S and W (AUD2-007, 008, 009). CA review pending. |
| AUD-003 | Out-of-order / late payment events | HIGH | Fixed | 8.5 Payments | Provider fetched on every webhook/poll; FAILED/EXPIRED → SUCCEEDED allowed; one refund rule in the winning transaction; partial unique index on automatic refunds; order expiry = `payment_due_at`. Residual expiry race in AUD2-010. |
| AUD-004 | Refund/provider retries not idempotent | HIGH | Partially fixed | 8.5 Payments ("Refund states") | Stable references and an UNKNOWN state close the double-refund path. A refund or payout whose fetch returns "not found" has no defined resolution, and `createOrder` idempotency is not a vendor criterion (AUD2-014). |
| AUD-005 | Single worker; expiry overtakes confirmation | HIGH | Partially fixed | 7.7 Deadlines, sweepers and the worker; 8.5 ("Check before expiring") | Two workers, check-before-expire, queue-age alerts and an outage drill are in place. The 15-minute max-defer expires bookings without a provider answer, which PD-08 does not allow, and the expiry transaction does not say it checks the local attempt state (AUD2-010). |
| AUD-006 | Lock order contradicted by own flows | HIGH | Partially fixed | 7.2 Transactions, lock order and the orchestrated flows | Canonical order now covers user, traveler profile, trip, payment attempt, payable item, hold, payout and party balance; every listed flow follows it, and no listed pair can deadlock. Still missing: delivery postings, payout completion/failure, dispute decision, contingent credits (AUD2-017), and the sweeper claim pattern contradicts the order (AUD2-011). |
| AUD-007 | Accept re-checks too little | HIGH | Fixed | 7.3 The accept transaction | Trip `FOR SHARE`, `assertStillMatches` (rules 1–8), PD-01 time guard for online and cash, sender eligibility, PD-15 quote rule, and PD-16 cancellation of pending requests on material trip edits (§8.7). |
| AUD-008 | Cancellation workflows absent | HIGH | Fixed | 8.7 Cancellations, missed handovers and outcomes | Table covers every trigger with allowed states, resulting states, money and marks; CancellationReview added. Residual inconsistencies: AUD2-006 (cascade state), AUD2-019 (review lifecycle), AUD2-003 (offline). |
| AUD-009 | Payout can double-pay / pay held money | HIGH | Partially fixed | 8.6 Payouts | Payable items, clearing account, singleton key, one-non-failed-payout index and re-check under lock are correct. But one payout is executed as several provider operations under one reference and one state, so a partial failure followed by E7 can re-pay the escrow-funded part (AUD2-002). Zero-net payouts not handled (AUD2-007). |
| AUD-010 | Codes hash-only but must be displayed | HIGH | Fixed | 8.8 Handover and custody ("Codes") | AES-GCM for display plus HMAC for verification; logged decryption for the authorised viewer only. |
| AUD-011 | No offline/degraded handover | HIGH | Partially fixed | 8.8 ("Offline queue") | PD-11 two-way codes and server-verified queue applied. Queue is memory-only, there is no server-side pending state, no delivery-stage sync deadline, and no rule for cancellation/no-show while a verification is queued (AUD2-003). |
| AUD-012 | Recovery / recycled numbers not designed | HIGH | Fixed | 8.1 Identity, sessions and recovery; 8.11 KYC | `last_seen_at`, 90-day threshold (PD-13), selfie re-match against a stored face reference, payout cooling periods read by Payments. The selfie path makes Identity depend on KYC, which §6.3 forbids (AUD2-004); accounts with no KYC face reference are not covered (AUD2-021). |
| AUD-013 | pg-boss transactional claim; pooler; privileges | HIGH | Fixed | 7.6, 7.7, 10.6, 14.1 | Deadlines on aggregates; outbox the only in-transaction write; pg-boss worker-only, migrations in CD; connectivity spike. Pending spike [VERIFY]. |
| AUD-014 | Layering violated by own flows | MEDIUM | Partially fixed | 6.3 Layers and allowed dependencies | An explicit list, a consumer layer, holds in Payments, config in the kernel and a retention contract were added. At least ten described flows still break the list (AUD2-004). |
| AUD-015 | Transactional events hide coupling | MEDIUM | Fixed | 6.3 ("Booking is the orchestrator"); 7.6 | In-transaction handlers removed; Booking orchestrates; payment confirmation is an integration event. |
| AUD-016 | StateEvent ownership conflict | MEDIUM | Fixed | 6.1 ("Changes since v0.1"); 7.1 | Per-module `state_event` in the transaction; Tracking builds the timeline from outbox events. |
| AUD-017 | Suspension cascades not designed | MEDIUM | Partially fixed | 8.1 ("Restriction cascade") | Events, handlers and the user `FOR SHARE` lock were added. The state table has no ACCEPTED_AWAITING_PAYMENT → CANCELLED_BY_SYSTEM edge, the Routes/Shipments handlers bypass Booking orchestration, and "withdraw trip" has no defined trip state (AUD2-006). |
| AUD-018 | Idempotency record semantics | MEDIUM | Partially fixed | 7.5 Idempotency | Record inserted in the business transaction (no stuck keys). Storing deterministic 4xx contradicts "any failure rolls back the whole transaction" (AUD2-016). |
| AUD-019 | Invariants without DB/lock protection | MEDIUM | Fixed | 11.3 Database-enforced invariants; 8.4.5 | Request limit under the shipment lock; unique cash confirmation, automatic refund, pending attempt, payout item; links capped under lock with auto-refund. Overpayment paths and posting remain (AUD2-009). |
| AUD-020 | Cash debit before pickup | MEDIUM | Fixed | 8.5 ("Cash confirmation"); 7.2 | E8/E9 posted in the pickup/delivery transaction; reversal on an upheld cash dispute. |
| AUD-021 | Overlap constraint blocks delays | MEDIUM | Fixed | 11.3 | Scheduled period only; estimates in other columns. |
| AUD-022 | Weight cap per size class | MEDIUM | Fixed | 11.3; 7.2 | Trip-level `trip_weight` row locked before per-class rows; whole-trip capacity (PD-18). |
| AUD-023 | Exact station-ID matching | MEDIUM | Fixed | 8.2 Matching and search | City-level matching with concrete stops (PD-14). AD-14 distance source still needs product-owner approval (T-5). |
| AUD-024 | Terminal transitions have no owner | MEDIUM | Partially fixed | 7.1 State machines; 8.7 | Shipment CLOSED, delivery CANCELLED and booking CLOSED_* owners defined. Booking COMPLETED has no consistent owner, cash bookings never complete, and trip COMPLETED needs data Routes cannot read (AUD2-005, AUD2-004). |
| AUD-025 | Payment window extension | MEDIUM | Fixed | 8.5 ("Payment-window extension") | `payment_due_at` stored; extension keeps departure − new due ≥ cutoff. Provider order-expiry update is unverified (AUD2-023). |
| AUD-026 | Recipient link = right to collect | MEDIUM | Fixed | 9.3 Recipient access | Recipient OTP (PD-12), tap-to-reveal, no-store, no-referrer, crawler page. Token handling for link delivery is inconsistent (AUD2-012). |
| AUD-027 | False delivery / code phishing | MEDIUM | Fixed | 8.8 ("Plausibility"); 8.9; 10.7 | Plausibility flag and hold, server times, location not evidence, user-supplied evidence stated, "never share your code". Residuals: no hold for cash/COD, no correction path after a false DELIVERED (AUD2-013). |
| AUD-028 | Evidence backup / integrity | MEDIUM | Fixed | 8.9 Evidence | Replica store, integrity sweep, signed export (T-9 open). |
| AUD-029 | OTP abuse / rate limits | MEDIUM | Fixed | 8.1; 10.4 | Per-number/IP/device/prefix/global budgets, circuit breaker, edge limits primary, token bucket. |
| AUD-030 | Admin same origin; CSRF | MEDIUM | Fixed | 9.4 Web ↔ API and the admin host | Separate host, own cookies, strict CSP, access token in memory, `SameSite=Strict`. Staff token transport not stated (AUD2-023). |
| AUD-031 | Reconciliation incomplete | MEDIUM | Fixed | 8.5 ("Reconciliation") | Item types, provider-reference keys, tolerance, holds, post-restore replay. Systematic escrow residuals will raise items (AUD2-008). |
| AUD-032 | No threat model | MEDIUM | Partially fixed | 10.7 Threat model | Trust boundaries and a starter table; full STRIDE deferred to before Stage P (T-13), which matches the round-1 P2 priority. |
| AUD-033 | No API inventory | MEDIUM | Partially fixed | 9.6 Core endpoint inventory | Core pilot inventory added; per-module tables deferred to P1 (T-14). Missing even from the core list: ratings (RATE-001), account deletion (AUTH-006), data export / privacy requests (PRIV-002), CancellationReview decision. |
| AUD-034 | No physical data model | MEDIUM | Partially fixed | 11.5 Key fields | Key fields for money and custody tables fixed; full schema per module deferred to P1 (T-14). |
| AUD-035 | Prisma limits | MEDIUM | Fixed | 11.4 Migrations and Prisma limits; 14.1 | Raw-SQL registry with CI existence test, typed raw-SQL locks, drift behaviour in the spike [VERIFY]. |
| AUD-036 | Approval adopts open decisions | MEDIUM | Fixed | 0.2 What approval means | Approval approves only AD-xx; PD-20 confirms. |
| AUD-037 | Aadhaar blind index uncertain | MEDIUM | Fixed | 8.11 KYC | Provider-agnostic dedup signals; counsel on hashing (T-12). |
| AUD-038 | Key management undefined | MEDIUM | Fixed | 10.2 Encryption and key management | KEK per environment, wrapped data keys, separate HMAC/pepper secrets, KMS access limited to runtime identities, rotation, break-glass. Vendor with AD-09. |
| AUD-039 | No cost estimate | MEDIUM | Partially fixed | 14.6 Cost model | Structure only; numbers wait for vendor quotes (OQ-19). Correctly refuses to guess. |
| AUD-040 | Notification data access / retention | MEDIUM | Fixed | 8.12; 8.8; 11.6 | Logged `getContactForNotification`, no codes in messages, location never stored, recipient retention added (period TBD counsel). |
| AUD-041 | Quote validity vs request lifetime | MEDIUM | Fixed | 8.3 Pricing | PD-15 applied; accept never re-prices. |
| AUD-042 | No lockout recovery | MEDIUM | Fixed | 8.8 ("Lockout recovery") | Counterparty regeneration after OTP, ops command, risk flag. |
| AUD-043 | Module/schema list inconsistencies | LOW | Fixed | 6.1 Module map; 11.1 | Module count correct (18 + Privacy); `travelers` schema present; cash ownership per PD-19. |
| AUD-044 | PRD companion files claim | LOW | No longer applicable | (PRD, PD-20) | Product-side fix: companion files carry a SUPERSEDED banner. Nothing for the architecture to do. |
| AUD-045 | Matching query details | LOW | Fixed | 8.2; 8.4.5; 11.1 | Cached dues balance, `geography(Point,4326)` + GiST, latest estimate used for arrival. |
| AUD-046 | Outbox + pg-boss duplication | LOW | Fixed | 7.6 | Outbox kept; one retry/dead-letter policy through pg-boss. |
| AUD-047 | Webhook ingestion edge cases | LOW | Fixed | 8.5 ("Webhooks") | Event + outbox in one transaction; 503 when DB down; direct routing. |
| AUD-048 | Failure-path tests missing | LOW | Fixed | 15 Testing strategy | Failure-path, concurrency and ledger property suites listed. |
| AUD-049 | No withholding hook | LOW | Fixed | 8.4.4 (E5); 8.6 | `W` and `tds_payable` in the payout formula. Escrow release of `W` missing (AUD2-008). |

---

## 3. New findings

### 3.1 HIGH

#### AUD2-001 — Custody-value cap ignores bookings awaiting payment, so PD-10 can be bypassed

| Field | Value |
|---|---|
| Severity | **HIGH** |
| Category | Fraud / Domain design |
| Section | 7.3 The accept transaction (BOOK-002, WF-A); 8.2 Matching and search |
| Related | PD-10, AUD-032, AUD2-004, §8.10 |
| Approval needed | Product owner (confirm which states count; see Q2-02) |

**Problem.** §7.3: "the sum of declared values on the traveler's CONFIRMED and IN_FULFILMENT bookings, plus this booking, must be ≤ the traveler's cap tier". ACCEPTED_AWAITING_PAYMENT bookings are excluded. The payment-confirmed flow (§7.2, §8.5) moves a booking to CONFIRMED without locking `traveler_profile` or re-checking the cap. The `travelers.v_traveler_eligibility` "custody headroom" used by matching has the same basis.

**Impact.** A new traveler with a ₹5,000 cap (example value from Appendix A) can accept several online requests in a row. Each accept sees only CONFIRMED and IN_FULFILMENT bookings, so each passes. All of them later become CONFIRMED on payment. The traveler then holds several times the cap in declared value. PD-10 exists specifically to limit collusive or fraudulent compensation claims against the reserve, so the main control on reserve exposure from new accounts can be bypassed by an ordinary sequence of accepts.

**Recommended correction.** Count ACCEPTED_AWAITING_PAYMENT, CONFIRMED and IN_FULFILMENT bookings in the cap (accept already serialises on `traveler_profile`). Use the same definition in the matching view. Optionally re-check in the payment-confirmed flow. If it does, that flow must lock `traveler_profile` first (rank 2), before the shipment. Add a concurrency test: N sequential online accepts by one traveler whose total exceeds the cap.

---

#### AUD2-002 — One payout runs as several provider operations under one reference and one state; a partial failure can pay twice

| Field | Value |
|---|---|
| Severity | **HIGH** |
| Category | Payments |
| Section | 8.6 Payouts (PAY-006, PD-09; AUD-009); 8.4.4 Posting rules (E5, E6a, E6b, E7) |
| Related | AUD-009, PD-09, D-18, BR-17, AUD2-014 |
| Approval needed | Product owner (PD-09 says "one transfer"; see Q2-04); none for the design fix |

**Problem.** The payout run creates one `payout` row with reference `payout:<id>` and then, "using `releaseSplit` for escrow-funded items and `payoutToBeneficiary` for merchant-funded items", calls the provider. E6b also needs the dues portion of escrow-funded items (`D_e`) released from escrow to the merchant balance. That is up to three provider operations: escrow → traveler, merchant → traveler, escrow → merchant. Yet:
- there is one reference for all of them, so provider-side idempotency cannot tell them apart;
- payout states (`INITIATED → PAID | FAILED | UNKNOWN`) cannot represent "escrow part paid, merchant part failed";
- E7 on FAILED is an "exact reversal of E5" and puts **all** items back to ELIGIBLE;
- the next daily run creates a **new** payout with a new reference, so the provider will not deduplicate the escrow release that already succeeded;
- how `D` is split between escrow-funded and merchant-funded items is not stated.

PD-09 also says "one transfer per traveler". If the vendor cannot combine an escrow release and a merchant payout into one transfer, the PD-09 assumption does not hold and nothing in v0.2 says so.

**Impact.** A traveler with an escrow earning plus a dues-funded or compensation credit (common under BR-38 and RV-04) can receive the escrow-funded part twice after a partial failure. The ledger would show one payout. Reconciliation catches the mismatch, but only after the money has gone. Recovery depends on future dues netting from a traveler who may stop using the platform.

**Recommended correction.** Model one **provider instruction per funding leg** (for example `payout_leg` with its own reference `payout:<id>:escrow`, `payout:<id>:merchant`, `payout:<id>:dues-release`, its own state and its own UNKNOWN handling). The payout is PAID when all legs are PAID. On a partial failure, reverse and re-queue only the items of the failed leg. Define how `D` is allocated across legs (for example escrow items first). Post E6a and E6b per leg. Make "can one transfer combine escrow and merchant funds?" a vendor question under T-1, and record the PD-09 consequence if not.

---

#### AUD2-003 — Offline-queued handovers are not reconciled with cancellation, no-show or expiry; the queue is memory-only and has no server-side state

| Field | Value |
|---|---|
| Severity | **HIGH** |
| Category | Domain design / Reliability |
| Section | 8.8 Handover and custody ("Offline queue"); 7.1 State machines; 8.7 Cancellations, missed handovers and outcomes; 13.2 Handover screens |
| Related | PD-11, AUD-011, BR-09, BR-12, BR-29, HAND-004, HAND-009, BOOK-005/006 |
| Approval needed | Product owner (see Q2-01) |

**Problem.**
1. **Cancellation race.** While a pickup verification sits in a device queue, the server still sees the delivery as PENDING_PICKUP and the booking as CONFIRMED. §8.7 allows "Sender cancels late", "Traveler cancels or no-show" (BOOK-006, reported by the sender) and trip cancellation from CONFIRMED until pickup. If either is applied before the queued entry syncs, the booking is cancelled and refunded. The queued entry is then rejected because the booking is no longer CONFIRMED, while the traveler physically holds the parcel. This is an honest-confusion case, not only fraud. If the traveler entered the sender's code, the sender's phone shows nothing, so the sender may report a traveler no-show.
2. **No server-side pending state.** PD-11 says "the delivery shows 'verification pending'". In v0.2 that is only a client-side screen. The server cannot know a verification is pending until it syncs, so ops, the counterparty and the cancellation rules cannot act on it.
3. **Memory-only queue.** §8.8 and §13.2 keep the entry "in memory". Mid-range Android phones routinely discard background tabs. A reload, tab kill or screen lock loses the entry after the parties have separated. Re-entering it then needs the other party's code, which can only be obtained by phone. That is exactly what the "never share your code by phone" messaging forbids.
4. **Delivery stage not covered.** `handover_sync_due_at` is defined only from "departure from the pickup stop". There is no sync deadline for a queued delivery or return verification; only the 2 h overdue alert applies.
5. Concurrency of "both sides submit" is listed as a test, but the expected outcome (the second submission returns success, not CODE_INVALID) is not stated.

**Impact.** Custody records and physical custody diverge. The core invariant of the product is that custody changes only on a verified code (BR-29). The outcome is refunds paid on parcels that are in transit, unpaid travelers, reliability marks against the wrong party, and disputes with no state path to resolve them.

**Recommended correction.**
- Between the scheduled stop time and `handover_sync_due_at`, hold sender-cancel and traveler-no-show reports at that stop as "pending sync". Apply them only after the deadline with no verification. Alternatively, accept a queued verification whose `claimed_at` precedes the cancellation and route the conflict to ops. Either way, write the rule into §8.7.
- Persist the queued entry on the device (IndexedDB) until the server acknowledges it.
- Add a server-visible "handover claimed offline" signal when either phone regains signal first. For example, the counterparty's "I handed over" tap records a non-custody claim.
- Define `handover_sync_due_at` for delivery and return stages (from the stop's scheduled or estimated departure).
- State that a duplicate correct submission after verification returns the original success.

---

#### AUD2-004 — The "exhaustive" allowed-dependency list is contradicted by flows the document describes

| Field | Value |
|---|---|
| Severity | **HIGH** (P0-class: it is the source of the CI lint rule for build step 1) |
| Category | Architecture conflict / Domain design |
| Section | 6.3 Layers and allowed dependencies [PROPOSED AD-11]; 6.2 Inside a module (rule 2) |
| Related | AUD-014 (round-1 P0-6), AUD2-005, AUD2-006, ARCH-06, NFR-MAINT-01 |
| Approval needed | None (AD-11 is still Proposed) |

**Problem.** §6.3 says the list is exhaustive and that a module may learn of a higher module's actions "only by subscribing to that module's integration events, and only when the publisher is on its allowed list". The document itself requires all of the following, and each one breaks that rule:

| # | Described in | Needed dependency | Why it breaks §6.3 |
|---|---|---|---|
| 1 | 8.1 recovery, phone change | Identity → KYC (selfie re-match against the KYC face reference) | Identity may call only Platform; KYC is above it |
| 2 | 8.5 Dues; 7.7 `ledger.dues-age` | Ledger → Identity (`Identity.restrict`) | Ledger may call only Platform |
| 3 | 7.7 `courier.sweep-missed` ("return when passed") | Courier → Booking/Payments (return booking per HAND-010, sender dues E23 / COD) | Multi-aggregate flow owned by a lower module; Courier may call only Routes, Shipments |
| 4 | 7.7 `shipments.sweep-expiry` (SHIP-005 cancels pending requests) | Shipments → Booking | Upward |
| 5 | 7.7 `routes.sweep-progress` (TRIP-006: COMPLETED when every delivery terminal; EXPIRED when no bookings) | Routes → Courier, Booking | Upward reads/subscriptions not on Routes' list |
| 6 | 7.1 booking COMPLETED "or dispute resolved"; WF-F | Booking learns dispute outcome | Disputes (L6) not on Booking's list, so Booking may not subscribe |
| 7 | 8.10 PD-10 pair-claim review; FRAUD-001 rules (ticket name mismatch, recipient number = traveler number) | Fraud → Disputes, Routes, Shipments | Not on Fraud's list |
| 8 | 6.1 TravelerProfile "completed count"; 8.2 `travelers.v_traveler_eligibility` "custody headroom" | Travelers needs booking/delivery/declared-value data | Travelers (L2) cannot read or subscribe to Booking/Courier/Shipments; a view joining those tables also breaks §6.2 rule 2 |
| 9 | RATE-001 ratings (Users owns Rating) | Users must know a delivery happened between the two parties | Users may call only Identity |
| 10 | AUTH-006 deletion blockers (active bookings, disputes, dues, legal hold) | Privacy/Identity → Booking, Disputes, Ledger | Privacy may call only the retention contract |
| 11 | 8.8 EARLY_DELIVERY "holds the payable item" | If Courier places it: Courier → Payments | Not allowed; only valid if Booking places it (not stated) |
| 12 | 8.1 "Booking routes … to Operations"; Disputes owns CustodyResolutionCase | Booking → Operations/Disputes | Upward unless it is an event (not stated) |

The list itself is acyclic (checked; §4.2). The problem is completeness: code that follows the text will fail the lint rule on day one, or the rule will be relaxed.

**Impact.** Same as round-1 AUD-014. The rule meant to keep the monolith extractable will be broken or switched off in build step 1, and each developer will invent a workaround. Several of these flows are money- or state-critical (dues hard block, returns, booking completion).

**Recommended correction.** Fix each row explicitly:
- (1) Move the face reference and re-match to a KYC contract and let the recovery use case live in a module allowed to call both Identity and KYC (for example KYC orchestrates "re-match then `Identity.changePhone`").
- (2) Move `dues-age` to Payments, which may call Identity and Ledger.
- (3)(4)(5) Make Booking the owner of the multi-aggregate sweeps (missed-collection return, shipment expiry with request cancellation, trip progress/completion). Alternatively, let the lower module emit an event that Booking handles.
- (6)(7)(9)(10) Extend the allowed **subscription** list (events, not calls), for example Booking subscribes to `disputes.*`, Fraud to `disputes.*`, `routes.*`, `shipments.*`, Users to `courier.delivery.*`, Privacy calls a `deletionBlockers` contract on each module.
- (8) Let Booking own custody exposure (it already computes it at accept) and publish it as a Booking view. Feed the completed count to Travelers by event, with the subscription listed.

Re-run the acyclicity check after the change.

---

### 3.2 MEDIUM

#### AUD2-005 — Booking completion and the dispute window have no consistent owner; cash bookings never reach COMPLETED

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Domain design |
| Section | 7.1 State machines; 7.2 (row "Dispute window close"); 7.7 Deadlines, sweepers and the worker; 11.5 Key fields |
| Related | AUD-024, AUD2-004, BR-11, DISP-001, WF-F |
| Approval needed | None |

**Problem.** Three columns describe the same window: `courier.delivery.dispute_window_ends_at`, `payments.payable_item.window_ends_at` and the booking's COMPLETED condition. The only sweeper is `payments.sweep-window` (Payments, L4). The lock table lists "Dispute window close" as a Booking use case (`booking → payable_item → hold`). Payments cannot call Booking. Payable items exist only for online earnings and credits, so **cash-at-pickup and COD bookings have no payable item and nothing ever moves them IN_FULFILMENT → COMPLETED**. Booking also cannot learn that a dispute was resolved (AUD2-004 row 6).

**Impact.** Cash bookings stay IN_FULFILMENT forever. Reports, KPIs, ratings eligibility and any rule keyed on COMPLETED are wrong. Two sweepers may race on the same window with different notions of "closed".

**Recommended correction.** Use one source of truth: `delivery.dispute_window_ends_at` (set by Booking in the delivery transaction). Add a `booking.sweep-completion` owned by Booking. It completes the booking when the window has passed and no active DISPUTE hold or open dispute exists, learning dispute state via a listed subscription. It also calls Payments to make that booking's items ELIGIBLE. Keep `payable_item.window_ends_at` only as a derived copy, or drop it.

---

#### AUD2-006 — Restriction cascade conflicts with the booking state table and the orchestration rule

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Domain design / PRD alignment |
| Section | 8.1 Identity, sessions and recovery ("Restriction cascade"); 7.1 State machines; 6.3 ("Booking is the orchestrator") |
| Related | AUD-017, AUTH-005, KYC-003, DISP-005, TRIP-006 |
| Approval needed | Product owner for the suspended-traveler pickup question (Q2-08) |

**Problem.**
- The cascade "system-cancels REQUESTED and ACCEPTED_AWAITING_PAYMENT bookings", but §7.1 has **no ACCEPTED_AWAITING_PAYMENT → CANCELLED_BY_SYSTEM** transition. The conditional update would fail.
- "Routes withdraws published trips" and "Shipments withdraws OPEN parcels" run as separate handlers outside Booking. TRIP-006 has no WITHDRAWN state. If it means CANCELLED, a trip with CONFIRMED or IN_FULFILMENT bookings ends up cancelled while its bookings are "not cancelled automatically" (AUTH-005). §6.3 says every flow that changes more than one aggregate is a Booking use case. SHIP-004 withdrawal is listed in §7.2 as a Booking flow (`shipment → bookings`), yet here Shipments does it alone.
- Whether a suspended traveler with a CONFIRMED booking may still perform pickup while ops decides is not stated. Pickup does not lock or check the user row.
- The cascade runs "one booking per transaction". If a booking changed state between enumeration and processing (for example AAP → CONFIRMED by payment), the handler must re-route it rather than fail. This is not stated.

**Impact.** Suspension can leave AAP bookings alive (then confirmed by payment), trips in an undefined state, and a suspended traveler taking custody of new parcels.

**Recommended correction.** Add AAP → CANCELLED_BY_SYSTEM (`cancel_reason = ACCOUNT_RESTRICTED`). Make the cascade one Booking-owned handler per affected trip or shipment, using the existing "Trip edit or cancel" and "Withdraw parcel" flows. Define a trip state for "withdrawn by restriction, live bookings pending ops" (or keep the trip PUBLISHED but unmatchable). Have each per-booking step re-read and re-route. Decide Q2-08 and, if pickup is blocked, add the user `FOR SHARE` check to pickup.

---

#### AUD2-007 — Posting table gaps: COD refusal, missed-handover share account, zero-net payouts, sender-dues write-off, contingent allocation

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Payments / Ledger |
| Section | 8.4.4 Posting rules; 8.4.5 Ledger integrity; 8.6 Payouts |
| Related | AUD-002, BR-32, BR-38, PAY-004, PAY-010, LEDGER-001 (acceptance criterion "dues ₹600, earning ₹400 → payout ₹0"), T-6 |
| Approval needed | CA review (T-6); product owner for sender-dues write-off (Q2-11) |

**Problem.**
1. **COD refusal (PAY-004, §8.7 row "Not collected / COD refused")** — "price + return cost as sender dues; traveler outbound earning as ContingentCredit" has no posting. It needs: which revenue or liability accounts are credited for the price and the return cost, how C and G are recognised, and which account funds the contingent outbound earning.
2. **E23 funded "as E15"** — E15 debits `cancellation_fee_revenue`, but E23 credited `missed_handover_fee_revenue`. Funding the traveler share from the wrong revenue account misstates both lines.
3. **Zero-net payout.** When dues ≥ payable items, `N = 0`. The PRD acceptance criterion requires both entries to exist. v0.2 does not say whether a `payout` with N = 0 is created, whether the provider is called, or how items reach PAID (E6a is triggered by a provider PAID).
4. **Write-off of sender dues** is not defined (E21 covers traveler dues only). §8.4.5 says contingent credits are voided "if the dues are written off", but contingent credits are linked to **sender** dues. Unpaid sender dues and their OPEN contingent credits therefore persist forever.
5. **Allocation rule conflict:** E15 funds contingent credits "in proportion to dues paid", but §8.4.5 says "funded oldest first". With several dues sources per sender, the two give different traveler credits.
6. The initial reserve funding (`bank_reserve` / `owner_funding`) has no rule. This is minor.

**Impact.** Implementers will invent postings for a frequent cash case (COD refusal). Revenue lines will be misstated. A PRD acceptance criterion cannot be met as written. Ledger property tests have no specification to test.

**Recommended correction.** Add rows for COD refusal, overpaid-dues refund (see AUD2-009), zero-net payout (no provider call; items → PAID; E5 with N = 0; no E6), sender-dues write-off with contingent-credit voiding, and reserve funding. Fix E23 funding to debit `missed_handover_fee_revenue`. Choose one allocation rule. Send the extended table to the CA.

---

#### AUD2-008 — Escrow never clears: retained cancellation revenue and withheld tax have no release movement; the commission-release trigger is undefined

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Payments / Reconciliation |
| Section | 8.4.2 Every money movement; 8.4.4 Posting rules (E4, E5, E12); 8.5 ("Reconciliation") |
| Related | AUD-001, AUD-031, AUD-049, BR-31, T-1, T-6 |
| Approval needed | CA; vendor capability (T-1) |

**Problem.**
- **E12** credits `cancellation_fee_revenue (K − S)` but the cash stays in escrow. M1–M11 contain no movement that releases retained cancellation money from escrow to the merchant balance, and no posting moves `provider_escrow` for it. Compare E4 for commission.
- **E5** credits `tds_payable W`. For escrow-funded items, E6a/E6b relieve escrow only by `N_e + D_e`, so `W_e` stays in escrow with no release.
- **M4** ("settlement of the commission portion") has no stated **instruction**: who tells the provider to release C+G, and when (at delivery, at window close, with the payout)? E4 is posted from the settlement report, but in a split product GoParcel normally has to instruct the release.
- §8.4.1 says escrow holds booking money "until delivery or refund". Retained fees contradict this.

**Impact.** `provider_escrow` in the ledger and the provider's escrow balance drift apart by every retained fee and every withholding. Reconciliation raises recurring items, revenue sits in escrow indefinitely, and BR-31 compliance cannot be demonstrated cleanly.

**Recommended correction.** Add M12 "release retained cancellation / missed fee from escrow to merchant" and M13 "release withheld tax from escrow", with postings (Dr `provider_merchant_balance` / Cr `provider_escrow`). Define the trigger and reference (`release:<id>`) for M4, M12 and M13. Add a ledger property test: for every booking whose lifecycle is terminal and whose payouts are PAID, the booking's escrow sub-ledger is zero.

---

#### AUD2-009 — Dues payment links can overpay; the automatic overpayment refund has no posting (D-21)

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Payments / Concurrency |
| Section | 8.4.5 Ledger integrity ("No spendable balance"); 8.5 ("Dues"); 8.6 Payouts |
| Related | AUD-019, D-21, BR-32, PAY-009, E14, E16 |
| Approval needed | None |

**Problem.** Links are "created under the party_balance lock for at most the amount owed". This does not prevent overpayment:
- two links created one after the other (owed is unchanged until one is paid) can both be paid;
- a link created for ₹600 is followed by a payout run that nets ₹400 of earnings against dues (E5), and the traveler then pays the ₹600 link;
- a cancellation-review or appeal reversal reduces dues after a link is issued.

E14/E16 then credit the receivable beyond zero (a credit balance in the party's favour). M9 refunds the excess, but **no posting rule exists for M9**, so between capture and refund the party has a positive balance, and the posting that clears it is undefined.

**Impact.** D-21 ("no positive spendable balance") is breached in the ledger. Overpayments are refunded ad hoc. With E7 reversals, a traveler can end up owing dues they already paid and then had refunded.

**Recommended correction.** Allow only one open link per party: creating a new link or netting dues in a payout expires the open link at the provider first (or caps the link at "owed minus amounts already committed"). Post an overpayment on capture as Dr `provider_merchant_balance` (paid) / Cr receivable (owed) and Cr `refunds_in_transit` (excess), then clear on refund success. Add the overpayment case to the ledger property tests.

---

#### AUD2-010 — Check-before-expire: the 15-minute default expires without a provider answer (not allowed by PD-08); the expiry transaction does not re-check local payment state

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Payments / PRD alignment |
| Section | 8.5 Payments ("Check before expiring", "When an attempt reaches SUCCEEDED"); 7.2 (row "Payment-window expiry") |
| Related | PD-08, AUD-005, BOOK-004, PAY-002, WF-B |
| Approval needed | **Product owner** (Q2-03) |

**Problem.**
1. PD-08: "before expiring an unpaid booking, the system asks the provider for status. Captured before the deadline → booking confirmed." v0.2 step 3: if the provider is unreachable for `CFG-PAY-EXPIRY-MAX-DEFER` (proposed 15 min), "it expires, and any later capture is refunded automatically" — including captures made **before** the deadline. This is a product rule change presented as a config default, not flagged for approval.
2. The expiry transaction "re-checks that the booking is still awaiting payment". It does not say it checks, under the `payment_attempt` lock it takes, that no attempt is already SUCCEEDED (the captured event may still be in the outbox; relay lag alert is 2 min) or that a SUCCEEDED attempt has `captured_at ≤ payment_due_at`. The booking stays ACCEPTED_AWAITING_PAYMENT until Booking processes the event, so a booking-only re-check can expire an on-time payment that has already been recorded locally.
3. WF-B and §10.7 require amount and order to match; §8.5's SUCCEEDED handling never states the amount check, nor that a mismatch goes to reconciliation without confirming.

**Impact.** On-time payers can be refunded and lose their booking (capacity released to someone else) during a provider outage, or in a narrow local race. That is the failure AUD-005 targeted.

**Recommended correction.** In the expiry transaction, lock the attempts and refuse to expire if any attempt is SUCCEEDED with `captured_at ≤ payment_due_at`; confirm instead. Either get product-owner approval for the max-defer behaviour, or keep the booking reserved until the provider answers (bounded by departure − cutoff, with an ops alert). State the amount/order check and the mismatch route.

---

#### AUD2-011 — The sweeper claim pattern contradicts itself and the canonical lock order

| Field | Value |
|---|---|
| Severity | MEDIUM (kernel, build step 1) |
| Category | Infrastructure / Database |
| Section | 7.7 Deadlines, sweepers and the worker ("Rules"); 8.5 ("Check before expiring") |
| Related | AUD-005, AUD-006, AUD-013, AUD2-010 |
| Approval needed | None |

**Problem.** "Each sweeper selects due rows `ORDER BY deadline LIMIT n FOR UPDATE SKIP LOCKED` and handles each row in its own transaction, re-checking state first." Taken literally:
- if the selecting transaction stays open, each per-row transaction (another connection) blocks on the row lock the selector holds. Postgres cannot detect this as a deadlock, because the selector is waiting on the application, not on a lock;
- if the selecting transaction commits first, the `SKIP LOCKED` protection is gone, and both workers can pick the same rows;
- the payment sweeper must call the provider **outside any transaction** (§8.5 step 1), so it cannot hold the claim lock across the call;
- the selector locks `booking` (rank 4) before the per-row flow locks `shipment` (rank 3).

**Impact.** Hung sweepers or double processing in the component that guards every deadline. Double processing is mostly harmless because of the conditional re-checks, but it doubles provider status calls and notifications.

**Recommended correction.** Use a lease column (`claimed_by`, `claimed_until`) set by a short `UPDATE … WHERE id IN (SELECT … FOR UPDATE SKIP LOCKED) RETURNING`, committed immediately. Then do provider calls with no transaction open, then run the canonical-order flow in a new transaction that re-checks state. Prove it in the spike.

---

#### AUD2-012 — Recipient token is stored only as a hash, but the link must be sent after commit and re-sent by support

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Security / Domain design |
| Section | 9.3 Recipient access ("1. Courier creates a 256-bit token at confirmation and stores only its SHA-256 hash"); 7.6 Integration events and the outbox ("Payloads carry IDs and non-personal facts"); 8.12 |
| Related | AUD-010 (same pattern), AUD-026, HAND-006, SUP-002 ("resend link"), PD-12 |
| Approval needed | None |

**Problem.** The recipient link is sent by Notifications, a consumer that runs after commit from outbox events whose payloads carry only IDs. With only the hash stored, Notifications cannot build the URL. The only ways are to put the raw token in the outbox payload (a bearer secret stored in `platform.outbox` and pg-boss job tables, retained and visible to database administrators) or to have Courier rotate the token, which needs a Courier command that Notifications, with read-only contracts, cannot call. SUP-002 "resend link" has the same problem.

**Impact.** Implementers will most likely put the token in the payload. PD-12 OTP reduces the damage (the token alone no longer shows the code), but the design states something that cannot be built as written.

**Recommended correction.** Add a Courier command `issueRecipientLink(deliveryId)` that rotates the token (new hash, old one revoked) and returns the raw value to the caller only. Have Booking or a dedicated handler call it and pass the URL to `platform/messaging` directly, or let Notifications call it through a listed exception. Alternatively, store the token encrypted (AD-16) like the codes.

---

#### AUD2-013 — A false DELIVERED cannot be corrected, and cash/COD deliveries get no plausibility hold

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Security / Domain design |
| Section | 7.1 State machines (delivery states; "Booking moves the shipment to CLOSED …"); 8.8 Handover and custody ("Plausibility") |
| Related | AUD-027, scenario I, DISP-001/002/005, BR-11, BR-12 |
| Approval needed | **Product owner** (Q2-05) |

**Problem.** Once a delivery is DELIVERED, the shipment is CLOSED in the same transaction, and no delivery edge leaves DELIVERED. If the recipient disputes "not delivered" and the parcel is shown to be still with the traveler (phished code, wrong person), there is no state path to recover it. Custody resolution (DISP-005) forbids forcing state without a verified handover, but the only remaining handover (return) needs EXCEPTION. Separately, the EARLY_DELIVERY plausibility flag "holds the payable item". Cash-at-pickup and COD bookings have no payable item, so the control does nothing for them.

**Impact.** In the main false-delivery fraud case, the only possible outcome is money (compensation), with no way to get the parcel back. The plausibility control does not apply to the cash part of the pilot.

**Recommended correction.** Product owner decides whether ops may move DELIVERED → EXCEPTION (custody disputed) under a dispute with reason and two-person approval, reopening the shipment's custody chain. If not, state explicitly that resolution is money-only. For cash and COD, make EARLY_DELIVERY a RISK flag with an ops review item (and optionally a hold on the traveler's other payable items).

---

#### AUD2-014 — UNKNOWN resolution is incomplete; `createOrder` idempotency is not a vendor criterion

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Payments / Reliability |
| Section | 8.5 Payments ("Refund states", port `PaymentProvider`); 8.4.2 ("Vendor criteria"); 8.6 Payouts ("Payout states") |
| Related | AUD-004, AUD2-002, scenario B |
| Approval needed | None (vendor questions under T-1) |

**Problem.**
- A refund or payout in UNKNOWN "is resolved only by `fetchRefund(ref)` / `fetchPayout(ref)`". There is no rule for when the fetch returns **not found** (the request never reached the provider), when the fetch itself keeps failing, or how long an item may stay UNKNOWN before finance is alerted.
- Retrying a FAILED refund "with the same reference": many providers bind an idempotency key to its first result, so a retry with the same key may just return the old failure (unverified per vendor).
- Vendor criteria require merchant references for refunds and payouts but not for **order creation**. A timed-out `createOrder` that is retried can create two live provider orders for one attempt. Only one is recorded; a capture on the other becomes an ORPHAN_CAPTURE found at the next daily reconciliation.
- Payout "repeated failure asks the traveler to re-verify their beneficiary". When the cause is an insufficient merchant balance (manual top-up, M11), this sends the traveler the wrong instruction.

**Impact.** Refunds or payouts stuck indefinitely. A sender double-charged until the next day. Misleading messages to travelers.

**Recommended correction.** Define "not found after T minutes since submission → safe to resubmit with the same reference" (only if the vendor guarantees reference idempotency; otherwise a new reference after a confirmed not-found). Add an UNKNOWN-age alert. Add order-reference idempotency or fetch-by-reference to the hard vendor criteria. Classify payout failure causes (beneficiary vs funding vs provider) before messaging the traveler.

---

#### AUD2-015 — Lost/returned money outcomes are asymmetric and ambiguous

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Payments / PRD alignment |
| Section | 8.7 Cancellations, missed handovers and outcomes (rows "Returned", "Lost"); 18 Open technical items (T-15) |
| Related | PD-17, DISP-002, PAY-005, HAND-010, AUD2-007 |
| Approval needed | **Product owner** (extend T-15; Q2-07) |

**Problem.** For an online booking that ends LOST or RETURNED, what happens to `customer_funds_held` is correctly left TBD (T-15). For cash, the Lost row says "compensation E17 (cash booking: E20)": it is unclear whether E20 (refund of the cash price, recovered from the traveler) replaces or adds to E17 (compensation up to the declared value). If it adds, v0.2 has decided "refund price + compensation" for cash while leaving the same question open for online. That is an unflagged product rule. RETURNED for cash and COD bookings (the outbound price the sender paid in cash, or owes for COD refusal) is also not addressed, and the return-leg price on the traveler's own return trip (HAND-010) has no source.

**Impact.** Inconsistent treatment of senders by payment method, decided in code. T-15 is scheduled "before step 8", but Payments/Ledger are built in step 6.

**Recommended correction.** Widen T-15 to cover cash, COD and online for LOST, RETURNED and ABANDONED, and the return-leg price. Rewrite the Lost row as "E17, and additionally E20 if …" once decided. Move the decision before step 6.

---

### 3.3 LOW

#### AUD2-016 — Storing deterministic 4xx responses contradicts the in-transaction idempotency record

| Field | Value |
|---|---|
| Severity | LOW |
| Category | API / Kernel |
| Section | 7.5 Idempotency; 7.3 ("Any of them rolls back the whole transaction") |
| Related | AUD-018 |
| Approval needed | None |

**Problem.** The record is inserted inside the business transaction, and business errors roll back the whole transaction, so the record rolls back with it and the 4xx cannot be "stored". **Correction:** use a savepoint (roll back to the savepoint after the idempotency insert, then store the 4xx and commit), or state that 4xx responses are not stored, which is harmless for deterministic errors. Also state that the losing concurrent duplicate must read the stored response in a new transaction after the unique-violation error.

#### AUD2-017 — The per-flow lock table omits several multi-row flows

| Field | Value |
|---|---|
| Severity | LOW |
| Category | Database / Concurrency |
| Section | 7.2 Transactions, lock order and the orchestrated flows ("Locks by flow") |
| Related | AUD-006, AUD2-005, AUD2-007 |
| Approval needed | None |

**Problem.** The table claims every flow but omits:
- **Delivery**: E2 posting, the payable-item insert and the traveler party balances (only "COD commission" is listed);
- **Payout PAID/FAILED/UNKNOWN handling**: started from the payout row (rank 10) by a webhook, but must touch items (rank 9) and party balances;
- **Dispute decision** (booking COMPLETED, hold release, E17/E20, sender payable item);
- **CancellationReview decision**;
- **contingent-credit rows** (not in the canonical list at all) in dues payment and write-off;
- **missed-delivery choice**;
- **ops `regenerateCodes`**;
- **sender payout run**.

No deadlock was found among the listed flows (§4.1). The risk is that implementers lock payout before items, or contingent credits in different orders in "dues paid" vs "write-off". **Correction:** add the rows, put `ledger.contingent_credit` in the canonical order (before `party_balance`), and require payout status handlers to lock items before the payout.

#### AUD2-018 — A dual messaging outage blocks every delivery because recipient OTP is mandatory

| Field | Value |
|---|---|
| Severity | LOW |
| Category | Operational readiness |
| Section | 9.3 Recipient access; 8.12 Notifications and masked calling |
| Related | PD-12, scenario J, R-08 |
| Approval needed | Product owner (Q2-10) |

**Problem.** Both the recipient link and the recipient OTP depend on WhatsApp or SMS. In a dual-channel outage, the recipient can neither receive the link nor pass the OTP, so neither code direction works and no delivery can complete. v0.1 had the same channel dependency for the link only. **Correction:** define an ops-assisted fallback, for example ops verifies the recipient by masked call and the traveler enters an ops-issued one-time code, audited. Alternatively, record the risk as accepted in the runbook.

#### AUD2-019 — CancellationReview lifecycle is incomplete

| Field | Value |
|---|---|
| Severity | LOW |
| Category | Domain design |
| Section | 8.7 (row "Mismatch decline at pickup"; "CancellationReview") |
| Related | RV-03, HAND-004, BOOK-005 |
| Approval needed | Product owner (Q2-09) |

**Problem.** The booking state after the decision (does it stay CANCELLED_AT_PICKUP with `cancel_reason` updated?) is not stated. An online sender's money stays in `customer_funds_held` for up to `CFG-DISPUTE-SLA-DAYS` even when the outcome will be no-fault, while the parcel is already OPEN for a new booking and payment. "Sender fault → late-cancel row" chooses the late band without saying so. **Correction:** state the post-decision state and cause, and ask whether a provisional no-fault refund is wanted.

#### AUD2-020 — Sender payouts never net dues; same-person-both-roles and run keys are undefined

| Field | Value |
|---|---|
| Severity | LOW |
| Category | Payments |
| Section | 8.6 Payouts ("Sender payouts") |
| Related | D-01, D-21, PD-07, PD-09 |
| Approval needed | Product owner (Q2-06) |

**Problem.** "Sender payouts … no dues netting" is a product rule not found in the PRD. One account can be both sender and traveler (D-01), and party IDs are user IDs, but the singleton key is `payout:<travelerId>` and E5 vs E18 differ. It is unclear whether a person owed sender compensation, with traveler dues, gets one payout or two, and whether they run concurrently. **Correction:** ask the product owner; key singleton runs per party ID and payout kind; document the netting rule.

#### AUD2-021 — Recycled-number login for accounts without a KYC face reference

| Field | Value |
|---|---|
| Severity | LOW |
| Category | Security |
| Section | 8.1 ("Recovery and recycled numbers") |
| Related | AUTH-004 step 5, PD-13 |
| Approval needed | None |

**Problem.** The inactive-number check requires a selfie re-match, but users who never did KYC (D-02: KYC only at first publish) have no reference. AUTH-004 says to route them to support. v0.2 does not say so for the recycled-number login. **Correction:** state that such accounts are routed to support (or offered a fresh account) and that, having no KYC, they cannot have payouts.

#### AUD2-022 — Cross-module key relationships without foreign keys are undocumented

| Field | Value |
|---|---|
| Severity | LOW |
| Category | Database |
| Section | 11.1 Schemas and ownership ("Foreign keys"); 11.5 Key fields |
| Related | AD-17, BR-05 |
| Approval needed | None |

**Problem.** Under the FK rule, `courier.delivery.booking_id`, `payments.payment_attempt.booking_id` and `payments.payable_item.booking_id` cannot have foreign keys (Courier and Payments may not call Booking), so integrity of the core custody and money links relies on application code. **Correction:** list these relationships, state who guarantees them (Booking creates both in one transaction), and add a nightly orphan check.

#### AUD2-023 — Minor specification gaps

| Field | Value |
|---|---|
| Severity | LOW |
| Category | Documentation |
| Section | 7.3; 7.1; 11.3; 8.1; 9.4; 8.8; 8.5; 1 Product decisions applied |
| Related | various |
| Approval needed | None |

1. **Time guard basis (§7.3):** scheduled or latest estimated departure from the pickup stop is not stated.
2. **Post-confirmation trip edits (TRIP-003):** v0.2 states PD-16 for pending requests but not the TRIP-003 guard that only delay updates and added capacity are allowed after a confirmation.
3. **Overlap exclusion (§11.3)** covers PUBLISHED and IN_PROGRESS but not PENDING_VERIFICATION, so the constraint error surfaces when ops approves a ticket.
4. **CONFIRMED → CANCELLED_BY_SYSTEM (§7.1)** has no listed trigger (presumably SUP-002 support cancel).
5. **`Identity.requireReverify`** implies Identity sets REVERIFY_REQUIRED, which is a KYC state (BR-01).
6. **Staff authentication transport (§9.4):** the CSRF reasoning covers user tokens only; staff session transport (header token or cookie) is not stated.
7. **Code regeneration** has no cap (`regenerated_count` exists).
8. **"The provider order expiry is updated to match" (§8.5)** after an extension is unverified per vendor; the fallback is a new attempt/order.
9. **§1 table** lists PD-01..PD-19; PD-20 is applied in §0.2 but not listed.

**Correction:** one line each.

### 3.4 INFO

#### AUD2-024 — Strengths to preserve

Provider-as-source-of-truth with conditional transitions and the single automatic-refund rule. Deadlines on aggregates with outbox-only writes in the business transaction. Booking as the single orchestrator of multi-aggregate flows. The canonical lock order with re-verify-and-retry for flows starting from a lower-ranked row. Payable items separated from payout batches with a clearing account. Cash commission posted in the custody transaction. Two-way codes with no on-device verification. Recipient OTP with tap-to-reveal and no-store. Selfie re-match plus cooling periods on payout-account changes. Contingent credits for BR-38. Reconciliation item types with post-restore replay. The foundation spike gating every [VERIFY]. A refusal to invent prices, fees or vendor costs, with explicit TBD owners.

#### AUD2-025 — Status of feasibility claims marked [VERIFY] (not checked online in this round)

| Claim | Auditor's understanding (unverified) | Fallback in v0.2 credible? |
|---|---|---|
| pg-boss started with migrations disabled; schema installed by an owner role in CD | pg-boss is understood to offer a "no migrate" start option and to expose its schema SQL; unverified for the pinned version | Yes: if not available, CD can run pg-boss once with the owner role before starting workers |
| pg-boss singleton keys give one active job per traveler | Singleton semantics differ between pg-boss major versions (queue policies in recent versions); unverified | Yes, because the payable-item partial unique index is the real guarantee |
| pg-boss over Supabase connections | Needs a session or direct connection; Supabase direct connections are reported IPv6-only without an add-on, and the session-mode pooler is reported to support IPv4; unverified | Yes (session pooler, or IPv4 add-on in the cost model) |
| Prisma multi-schema | Believed to be generally available in recent Prisma 6.x; unverified | Yes (single schema with table prefixes; rules unchanged) |
| Prisma drift with raw-SQL objects | Commonly reported to flag unknown partial indexes; unverified | Partly: "ignored by rule" needs a concrete mechanism (for example shadow-DB diff filtered by the registry) |
| Supabase Storage not covered by DB backups | Believed true; v0.2 already assumes not | Yes (replica store) |
| Provider can update an order's expiry; refund/payout idempotency by merchant reference; one transfer combining escrow and merchant funds | Vendor-specific; unknown until OQ-19 | Partly; see AUD2-002, AUD2-014, AUD2-023 |

### 3.5 Optional suggestions (not defects)

| ID | Suggestion |
|---|---|
| S-01 | Add a ledger property test: every terminal booking's `customer_funds_held` sub-ledger is zero once refunds and payouts are final (catches T-15-type gaps automatically). |
| S-02 | Persist the offline handover queue in IndexedDB with an expiry; it holds only the counterparty's one-time code. |
| S-03 | Optionally process payment webhooks synchronously in the API when the database is healthy (fetch-from-provider, then apply), keeping the outbox path as fallback; shortens the confirmation path at the payment deadline. |
| S-04 | Give the cash/COD part of the pilot its own risk rules (EARLY_DELIVERY, repeated cash disputes) since payout holds do not apply to it. |
| S-05 | Publish a one-page "money-movement × posting × provider reference × reconciliation source" matrix generated from §8.4 for the CA and the vendor RFP. |

---

## 4. Consistency checks performed

### 4.1 Lock order (§7.2 vs every flow in §7.3, §8.1, §8.5–§8.10)

Canonical ranks: 1 user, 2 traveler_profile, 3 shipment, 4 booking, 5 trip, 6 trip_weight → trip_capacity, 7 delivery, 8 payment_attempt, 9 payable_item → hold, 10 payout, 11 party_balance.

| Flow | Ranks in order | Follows canonical order? |
|---|---|---|
| Request booking | 1(s) 3 5(s) | Yes |
| Withdraw parcel | 3 4 | Yes |
| Accept (§7.3 sequence matches §7.2) | 1(s) 2 3 4 5(s) 6 [7 insert] | Yes |
| Payment confirmed | 3 4 5(s) 7(ins) 8 | Yes |
| Payment-window expiry | 3 4 6 8 | Yes (but see AUD2-011 for the sweeper's claim lock) |
| Sender/traveler cancel, no-show | 3 4 6 7 8 [11] | Yes |
| Trip edit/cancel | 3* 4* 5(u) 6 7* 8* [11] | Yes, with re-verify of the affected set |
| Pickup | 3 4 5(s) 7 [11] | Yes |
| Mismatch decline | 3 4 6 7 | Yes |
| Delivery | 3 4 7 [11] | Yes, but omits payable-item insert and E2 balances (AUD2-017) |
| Exception outcome | 3 4 7 9 [11] | Yes |
| Dispute window close | 4 9 9b | Yes (ownership problem, AUD2-005) |
| Dispute raised | 4 7 9 9b(ins) | Yes |
| Payout run | 9 9b 10 11 | Yes |
| Dues payment | 11 | Yes, but contingent-credit rows are unranked (AUD2-017) |
| Suspend | 1(u) | Yes |
| Payout PAID/FAILED handler, dispute decision, review decision | not listed | Unknown (AUD2-017) |

**Deadlock pairs.** For every pair of listed flows, the shared rows are acquired in the same relative order, so no wait cycle is possible on the listed rows. The closest cases checked: accept vs trip cancel (both lock the shipment first, and trip cancel retries if its affected set changed); pickup vs sender cancel (shipment → booking in both); request vs trip edit (no shared row is taken in opposite order); payout run vs dispute raised (payable item → hold in both; the payout does not lock booking). The listed flows are deadlock-free; the remaining risk is in the unlisted flows and the sweeper claim (AUD2-011, AUD2-017).

### 4.2 Allowed-dependency list (§6.3)

- **Acyclic:** yes. Same-layer edges are only KYC → Users and Travelers → KYC; every other edge points downward.
- **Complete against described calls and subscriptions:** no; twelve contradictions are listed in AUD2-004.
- **Read views vs §6.2 rule 2:** `routes.v_matchable_stop`, `ledger.v_party_dues`, `users.v_reputation`, `identity.v_user_status` and `kyc.v_kyc_status` can be built from each owner's own tables. `travelers.v_traveler_eligibility` "custody headroom" and "completed count" cannot (AUD2-004 row 8; AUD2-001).

### 4.3 Posting rules (§8.4.4)

| Event | Debits | Credits | Balances? | Note |
|---|---|---|---|---|
| E1 | escrow P | funds_held P | Yes | |
| E2 | funds_held P | pending E + C + G | Yes (P = E + C + G) | |
| E3 | pending E | payable E | Yes | |
| E4 | merchant (C+G−fee) + fees | escrow (C+G) | Yes | Trigger undefined (AUD2-008) |
| E5 | payable X | dues D + tds W + clearing N | Yes (N = X−D−W) | N = 0 case (AUD2-007) |
| E6a/E6b | clearing N; merchant D_e | escrow N_e + merchant N_m; escrow D_e | Yes | Composite legs (AUD2-002); W_e never released (AUD2-008) |
| E7 | reversal of E5 | — | Yes | Partial failure (AUD2-002) |
| E8/E9 | dues (C+G) | commission C + gst G | Yes | |
| E10/E11 | funds_held P; in_transit R | in_transit P; escrow R | Yes | |
| E12 | funds_held P | in_transit R + payable S + fee revenue (K−S) | Yes (R+K = P) | K−S stays in escrow (AUD2-008) |
| E13/E14/E15 | sender dues F; merchant A; fee revenue S | fee revenue F; sender dues A; payable S | Yes | Allocation rule conflict (AUD2-007) |
| E16 | merchant A | traveler dues A | Yes | Overpayment (AUD2-009) |
| E17 | comp expense X + traveler dues X | sender_payable X + recovery X | Yes | |
| E18/E19 | sender_payable Y; clearing Y | clearing Y; merchant Y | Yes | |
| E20 | commission C + gst G + traveler dues (P−C−G) | sender_payable P | Yes | Replaces or adds to E17? (AUD2-015) |
| E21–E24 | bad debt; merchant; merchant; escrow/merchant | traveler dues; reserve; sender dues; suspense | Yes | E23 funding account (AUD2-007) |

**Double counting checked:**
- **E2 vs E12:** mutually exclusive, because E12 happens only before pickup and E2 only at delivery.
- **E2 vs E8/E9:** mutually exclusive, because online and cash are disjoint.
- **E5/E6a/E6b:** the dues portion is released once, if legs are modelled (AUD2-002).
- **E17/E20:** not double-counted if both apply, since E20 is the price and E17 the goods value, but whether both apply is undecided (AUD2-015).
- **Contingent credits:** posted only on dues payment (E15), never at the fee event. Correct for BR-38.

**Can a party balance go positive (D-21)?** Yes, transiently via dues overpayment, with no posting to clear it (AUD2-009). `sender_payable` awaiting a beneficiary is a liability to the sender but not spendable. Acceptable, but worth stating.

**Coverage of the §8.7 table:** every row maps to a rule except COD refusal (AUD2-007), online RETURNED/LOST (T-15) and ABANDONED (counsel).

### 4.4 State machines vs PRD and PD-17

| Check | Result |
|---|---|
| PD-17 booking end states CLOSED_RETURNED / LOST / ABANDONED | Present (AD-18) |
| PD-17 EXCEPTION → DELIVERED with claim | Present; the "claim attached" check needs Disputes data Courier/Booking cannot read (AUD2-004) |
| PD-17 traveler withdraws while awaiting payment | Present, with reliability mark |
| AAP → CANCELLED_BY_SYSTEM (used by restriction cascade) | **Missing** (AUD2-006) |
| CONFIRMED → CANCELLED_BY_SYSTEM | Present but no trigger listed (AUD2-023) |
| Overdue not a state change | Consistent with HAND-013 and BR-11 |
| Shipment CLOSED with terminal delivery | Consistent; no exit from DELIVERED (AUD2-013) |
| Booking COMPLETED owner | Inconsistent (AUD2-005) |
| Trip COMPLETED (TRIP-006) | Requires delivery data Routes cannot read (AUD2-004) |
| "Verification pending" (PD-11) | Client-side only; no server state (AUD2-003) |

### 4.5 Confirmed decisions and invented rules

| Item | Finding |
|---|---|
| PD-08 vs max-defer expiry | Deviation without approval (AUD2-010) |
| PD-09 "one transfer" vs two provider operations | Unacknowledged (AUD2-002) |
| PD-10 custody cap excludes AAP | Weakens the decision (AUD2-001) |
| PD-11 "verification pending" | Partially implemented (AUD2-003) |
| "Sender payouts: no dues netting" | Invented rule (AUD2-020) |
| Cash LOST → refund price (E20) | Possibly invented rule (AUD2-015) |
| `CFG-PAY-EXPIRY-MAX-DEFER` proposed 15 min | A number with product effect (AUD2-010) |
| Prices, fees, shares, reserve | Correctly left as TBD config keys; no invented values found |
| ARCH-01..12, D-01..22, DEC-051, BR-01..38 | No contradiction found, apart from the PD items above and the BR-31 escrow residual (AUD2-008) |

---

## 5. Stress-scenario re-traces (v0.2)

### A. Two travelers accept the same shipment at the same time
- **Path:** `POST /bookings/{id}/accept` ×2 → Booking.acceptBooking. Each locks users (share) → own traveler_profile → **shipment** (T2 waits) → bookings → trip (share) → weight → capacity. T1 commits: booking ACCEPTED/CONFIRMED, siblings CANCELLED_BY_SYSTEM, shipment LOCKED. T2 resumes and sees shipment LOCKED → `ALREADY_BOOKED`, full rollback. Backstop: partial unique index on live bookings.
- **Same traveler, different shipments:** serialised on traveler_profile. **Different shipments, same capacity:** serialised on trip_weight/capacity with CHECK.
- **Verdict: Safe.** Remaining gap: the custody cap sum ignores AAP bookings (AUD2-001).

### B. Provider processes the payment but the API call times out
- **createOrder timeout:** the reference `order:<attemptId>` is sent, but order idempotency is not a vendor criterion, so a retry may create a second provider order. A capture on it is an ORPHAN_CAPTURE found next day → refund (AUD2-014).
- **Refund timeout:** UNKNOWN → `fetchRefund(ref)`; no blind resubmit. Safe against double refunds; "not found" is unresolved (AUD2-014).
- **Payout timeout:** UNKNOWN → `fetchPayout(ref)`. With composite legs under one reference, a partial outcome cannot be represented, and the E7 reversal can lead to a double pay (AUD2-002).
- **Verdict: Mostly safe for payments and refunds; unsafe for composite payouts.**

### C. Duplicate webhook
- Same `event_id` → unique `(provider, event_id)` → 200, no work.
- Different event types, or webhook plus poll → both fetch the provider status and apply a conditional transition; only the winner posts E1 and emits `captured`; Booking's handler is idempotent (`processed_event`). An automatic refund is unique per attempt.
- Out of order: FAILED → SUCCEEDED allowed; the provider fetch makes order irrelevant.
- **Verdict: Safe.**

### D. Payment succeeds at the provider but the local DB transaction fails
- **Webhook while the DB is down:** 503 → the provider retries; the poll job fetches PENDING attempts; ingestion is one transaction.
- **Worker down past the deadline:** on recovery, the sweeper checks the provider first and applies a capture made before the deadline (PD-08).
- **Gaps:** if the provider is unreachable for longer than `CFG-PAY-EXPIRY-MAX-DEFER`, the booking expires and an on-time capture is refunded (not allowed by PD-08). The expiry transaction does not state that it checks a locally recorded SUCCEEDED attempt (AUD2-010).
- **Verdict: Safe except the max-defer path and the unstated local re-check.**

### E. Traveler cancels after accepting
- **AAP:** CANCELLED_BY_TRAVELER plus reliability mark (PD-17); capacity released; shipment OPEN; a later capture is refunded automatically.
- **CONFIRMED:** cancel flow (shipment → booking → weight → capacity → delivery → attempt) → CANCELLED_BY_TRAVELER, delivery CANCELLED, E10 full refund (online) or nothing (cash), mark, re-match event.
- **Trip cancel with several bookings:** locks all affected shipments ascending, retries if the set changed; requests CANCELLED_BY_SYSTEM, live bookings CANCELLED_BY_TRAVELER.
- **Verdict: Safe.** Edge: a sender-side offline-queued pickup followed by a no-show/cancel report (AUD2-003).

### F. Accept too close to departure (online and cash)
- **Online:** departure − now ≥ PAY-WINDOW + CUTOFF, else `JOURNEY_TOO_SOON` with full rollback (PD-01). **Cash:** departure − now ≥ CUTOFF. Request expiry is checked directly in accept.
- **Extension:** keeps departure − new due ≥ CUTOFF.
- **Delay after accept:** departure later, so safe. An earlier time while requests are pending is caught by PD-16 cancellation and the `assertStillMatches` re-check.
- **Verdict: Safe.** Minor gaps: scheduled vs estimated departure basis; the post-confirmation edit guard is not restated (AUD2-023).

### G. Trip edit after match, request or confirmation
- **After match only:** nothing to protect; matching re-runs.
- **After request:** a material edit system-cancels REQUESTED bookings (PD-16) under the trip-edit lock flow with re-verify. A concurrent accept serialises on the shipment and then re-checks the match on the locked trip.
- **After confirmation:** TRIP-003 allows only delay and added capacity. Delays do not hit the overlap constraint (scheduled times). Capacity reductions below reserved are blocked by CHECK.
- **Verdict: Safe,** provided the TRIP-003 post-confirmation guard is written into the trip-edit use case (AUD2-023).

### H. Parcel damaged, lost or not delivered
- **Damaged:** EXCEPTION → DELIVERED with claim. E2 at delivery; dispute hold blocks the payable item; decision E17 → sender payout via merchant (M7); recovery netted from future earnings; write-off after `CFG-WRITEOFF-AFTER-DAYS` (PD-10).
- **Lost:** CLOSED_LOST; E17 (cash: E20?); the online price in `customer_funds_held` is TBD (T-15).
- **Not delivered, traveler unreachable:** custody resolution; overdue alerts only (BR-11).
- **Gaps:** the online LOST/RETURNED money outcome is undecided, and cash is ambiguous (AUD2-015); the dispute-decision lock flow is not listed (AUD2-017); Booking cannot learn the dispute outcome to complete (AUD2-004/005).
- **Verdict: Partially safe.** Money can stay parked in escrow with no rule.

### I. Traveler submits false proof of delivery
- The delivery code is required. The link cannot be forwarded usefully (recipient OTP, PD-12). Messages say "never share your code by phone". Server receipt times are used, and browser location is ignored. EARLY_DELIVERY flag plus hold for online bookings. 48 h dispute window before payout.
- **Remaining:** phishing by phone is still possible (inherent). No hold for cash/COD bookings. No state path back from DELIVERED if the parcel is later shown to be with the traveler (AUD2-013).
- **Verdict: Partially safe.** Money exposure is controlled for online bookings; custody recovery is not designed.

### J. Geospatial or notification provider unavailable
- **Geospatial:** none at runtime. Matching is city-ID based and distance comes from `city_pair_distance`. Safe.
- **WhatsApp down:** SMS fallback for OTP and critical events; OTP budget breaker.
- **Both channels down:** logins fail (accepted risk R-08). Recipients get neither the link nor the OTP, so **no delivery can complete** (AUD2-018).
- **Train status down:** ETA hidden. EARLY_DELIVERY then uses the schedule; this is not stated, so false flags are possible.
- **Verdict: Safe for geospatial; unsafe for delivery in a dual messaging outage** unless an ops fallback is defined.

### K. Sender submits the same shipment several times
- Same key → idempotent replay. A different key creates a separate draft (a product choice, as in round 1). The 3-request limit is checked under the shipment lock. One request per shipment–trip pair has a partial unique index.
- **Verdict: Safe** (once AUD2-016 wording is fixed).

### L. Provider records do not match the ledger
- Daily reconciliation by provider reference. Item types for orphan capture, missing at provider, amount mismatch, settlement timing, fee variance and payout status. Reconciliation holds on payable items, suspense account, post-restore replay before reopening payouts.
- **Gaps:** retained cancellation revenue and withheld tax never leave escrow in the model, so escrow-level reconciliation raises recurring items (AUD2-008). Composite payout legs cannot be matched one-to-one to provider transfers (AUD2-002).
- **Verdict: Partially safe.** The mechanism is right; the model will produce systematic false items until AUD2-002 and AUD2-008 are fixed.

---

**End of re-audit report.**
