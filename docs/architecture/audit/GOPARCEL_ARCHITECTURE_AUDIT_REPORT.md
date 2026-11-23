# GoParcel — Architecture Audit Report

| Item | Value |
|---|---|
| Document under audit | `docs/architecture/GOPARCEL_ARCHITECTURE.md` v0.1 (Draft, 22 Nov 2026) |
| Audit date | 22 November 2026 |
| Auditor role | Independent principal software / security / database architect (did not author the architecture) |
| Method | Document review and reasoning only. **No code exists, no test was executed, no vendor capability was verified online** (web search was unavailable during this audit; every vendor or library claim below is labelled "unverified" where it matters). |
| Companion files | Conflict matrix, requirement coverage audit, risk register, remediation plan, production readiness report, audit decision log (same folder) |

This audit does not modify the architecture, the PRD or the decision register. Where it recommends a change to an approved decision it says so and marks it as needing product-owner approval.

---

## 1. Source inventory

### 1.1 Sources read in full

| File | Version / size | Notes |
|---|---|---|
| `docs/architecture/GOPARCEL_ARCHITECTURE.md` | v0.1 draft, 825 lines | The only architecture document. |
| `docs/product/PRD/GOPARCEL_MASTER_PRD.md` | v2.0 draft (reviewed), 2,302 lines | Section 4 requirements, §6 business rules, §7 NFRs, Appendices A–F. |
| `docs/product/PRD/GOPARCEL_DECISION_REGISTER.md` | 93 lines | Same content as PRD Appendix C. |
| `docs/product/PRD/GOPARCEL_OPEN_QUESTIONS.md` | 29 lines | Same content as PRD Appendix D (wording differs slightly in OQ-01, OQ-20). |
| `docs/product/PRD/GOPARCEL_PRD_AUDIT.md` | 82 lines | Same content as PRD Appendix F. |
| `docs/product/PRD/GOPARCEL_REQUIREMENTS_TRACEABILITY.md` | 155 lines | Same content as PRD Appendix E. |

### 1.2 Expected sources that do not exist

A Glob over `docs/**` confirms that none of the following exist. The architecture is one consolidated document.

| Expected file | Status | Does its absence block implementation? |
|---|---|---|
| GOPARCEL_SYSTEM_ARCHITECTURE.md | Missing; content partly in §3–§6 of the consolidated doc | No |
| GOPARCEL_DOMAIN_*.md (domain model) | Missing; module map in §6.1 only | Partly: aggregates and invariants per module are not written down (AUD-034) |
| GOPARCEL_DATABASE_*.md (physical schema) | Missing | **Yes, for Booking, Payments, Ledger and Courier** (AUD-034) |
| GOPARCEL_API_*.md (endpoint inventory) | Missing; only conventions in §9 | **Yes, before the affected feature** (AUD-033) |
| GOPARCEL_SECURITY_*.md / threat models | Missing; §10.7 defers to a pen test | Yes, before Stage P (AUD-032) |
| GOPARCEL_INFRASTRUCTURE_*.md | Missing; §14 covers the basics | No for development; yes for pilot (cost, KMS, backups) |
| GOPARCEL_SCALABILITY_AND_PERFORMANCE.md | Missing; one paragraph in §8.2 and §14.2 | No for pilot |
| GOPARCEL_ARCHITECTURE_DECISIONS.md / ADRs | Missing; AD-01..17 table in §17, ADRs promised "once approved" | No (the table is enough to review) |
| GOPARCEL_ARCHITECTURE_REVIEW.md | Missing | No (this audit replaces it) |
| F1–F9, full DEC register (named in the PRD) | Missing (OQ-01) | Assumed not to exist, as the PRD and architecture both do |

Missing documents are only raised as findings where their absence actually blocks a decision or an implementation step.

---

## 2. Decisions identified

| Group | IDs | Status as recorded | How the architecture treats them |
|---|---|---|---|
| Architecture | ARCH-01..12 | Confirmed | Applied. ARCH-10 extended by AD-10 (adds Privacy module) — correctly flagged as needing approval. |
| Product | D-01..D-22, D-06a, D-11a | Confirmed except D-15 (Proposed) | Applied. D-15 is **assumed approved** (§1, OQ-09). |
| Review recommendations | RV-01..RV-04 | Proposed | **Assumed approved** (§1, OQ-21); BR-38 "enforced in Ledger" (§8.5). |
| Master-prompt decisions | DEC-051 (Confirmed), DEC-052 / DEC-057 (Provisional) | | DEC-051 guard in accept (§7.3); interpretation assumed from OQ-02. DEC-052/057 left to T-10. |
| Business rules | BR-01..37 Confirmed, BR-38 Recommended | | §17 says it applies BR-01..37; §8.5 also applies BR-38. |
| New architecture decisions | AD-01..AD-17 | Proposed | "Approving this document approves all AD-xx unless struck out" (§0.2). |
| Open questions assumed | OQ-01, 02, 04, 05, 06, 07, 09, 21 | Open | Built on each question's recommended answer (§1). |

---

## 3. Initial assessment

The architecture is a competent, pragmatic starting point. It chooses a sensible shape for a small team (modular monolith, Postgres for queue and rate limits, no Redis/Kafka/K8s), applies most confirmed decisions faithfully, and gets several hard things right: conditional-update state transitions, append-only ledger enforced in the database, provider-event deduplication, server-authoritative confirmation, Stage A guards, the Supabase Data API disabled, and explicit flags where it departs from the PRD (AD-10, AD-14).

Its weaknesses are concentrated where real money and real parcels meet failure:

1. **The money model stops at the happy path.** The funds flow for compensation, cash-booking refunds, dues collection and dues-funded payouts is not defined and is not executable through the documented provider port and "funds held by provider" model (AUD-001). The ledger lists accounts but not the posting rules for the online lifecycle, refunds or payouts (AUD-002). Payout runs post to the ledger before the provider call and have no documented protection against paying the same credit twice (AUD-009).
2. **Failure paths of payments are thin.** Out-of-order and late provider events, duplicate attempts, refunds after timeouts and worker outages can each leave captured money with no booking or trigger a wrong refund (AUD-003, AUD-004, AUD-005).
3. **The concurrency design contradicts itself.** The global lock order (§7.2) is broken by the doc's own pickup flow (§7.4), and the accept transaction never locks or re-checks the trip (AUD-006, AUD-007).
4. **Whole workflows are missing**: every cancellation, no-show and pickup-decline path (AUD-008), suspension cascades (AUD-017), recovery and recycled numbers (AUD-012), trip edits after a request (AUD-007).
5. **One design detail cannot meet a PRD requirement**: handover codes are stored only as hashes, but the PRD requires the sender's screen and the recipient's page to show the codes (AUD-010).
6. **Feasibility claims about the job system and Prisma are unverified** and at least partly wrong as written (AUD-013, AUD-035).

The verdict (see the production readiness report) is **NOT READY** for development approval. The fixes are mostly design text, not a re-architecture: one focused revision (v0.2) covering the P0 items in the remediation plan should be enough to reach CONDITIONALLY READY.

### 3.1 Finding counts

| Severity | Count |
|---|---|
| CRITICAL | 1 |
| HIGH | 12 |
| MEDIUM | 29 |
| LOW | 7 |
| INFO | 2 |
| **Total** | **51** |

Defects are AUD-001 to AUD-049. AUD-050 and AUD-051 are informational. Inside each finding, "Recommended correction" is the fix for a defect; anything labelled "Optional" is a recommendation, not a defect.

---

## 4. Findings

### AUD-001 — Funds-flow model is not defined end-to-end; several PRD money movements cannot be executed

| Field | Value |
|---|---|
| Severity | **CRITICAL** |
| Category | Payments |
| Affected document | Architecture; PRD |
| Section | Architecture §8.4 Payments, §8.5 Ledger (LEDGER-001), §18 Open technical items (T-1); PRD PAY-005, PAY-006, PAY-009, PAY-010, DISP-002, BR-31 |
| Product-owner approval | **Yes**, plus counsel (OQ-20) and provider capability (OQ-19) |

**Problem.** The architecture adopts the PRD model "funds are held by the provider via escrow or split settlement, never in GoParcel's bank account" (§8.4, BR-31) and a `PaymentProvider` port with `createOrder`, `refund`, `createTransfer / releaseSplit`, `validateBeneficiary`. But the PRD also requires money movements that are not refunds of a captured online payment and not a split release:

- **Compensation up to ₹5,000** to a sender when the traveler is at fault, paid "from the compensation reserve" (DISP-002). A provider refund is normally limited to the captured amount (₹100–₹500 typical price), and cash senders have no captured payment at all. The port has no "pay a sender" operation; senders have no verified beneficiary (Beneficiary is a traveler concept, KYC-004).
- **Refunds for cash bookings** "refunded by GoParcel to the sender's UPI or bank account" (PAY-005) — money that must come from GoParcel's own funds.
- **Dues collection** (PAY-009/010) by UPI/payment link. The architecture does not say where this money lands. If it lands in GoParcel's account (the natural outcome), then
- **Dues-funded credits** (traveler compensation share on cash bookings, COD earnings, RV-04) must be paid out from GoParcel's own money, not from escrow. §18 T-1 notes this but treats it as a vendor question only.
- The ledger has `platform:compensation_reserve` and `provider:escrow` but **no account for GoParcel's own bank/settlement balance**, so none of these movements can be posted in full.

**Why it matters.** This is the legal and financial spine of the product. Whether GoParcel touches customer money at all decides payment-aggregator exposure (PRD §10.2), vendor selection (OQ-19), the chart of accounts, the provider port and reconciliation. Building Payments and Ledger before this is settled means rebuilding them.

**Real-world impact.** Compensation cannot be paid through the designed path, so ops pays manually from a bank account outside the ledger, breaking "every rupee traceable" (G-1, PG-4). Or GoParcel ends up holding customer funds in a way counsel has not approved.

**Recommended correction.** Add a "Funds flow" section that, for each money movement in the PRD (online payment, online refund full/partial, late-payment refund, duplicate-payment refund, cash-booking refund, compensation, recovery, dues payment, dues overpayment refund, payout of escrow earnings, payout of dues-funded credits, missed-handover fee and share, provider fees), states: source account (escrow / GoParcel operating / reserve), destination, provider API used, ledger posting, and reconciliation source. Extend the port with an explicit `payoutToParty` (or equivalent) operation and a sender beneficiary capture path. Add GoParcel operating-bank and reserve-bank accounts to the chart. Make "provider supports payouts from merchant balance to arbitrary verified beneficiaries" an explicit vendor criterion alongside split settlement. Get counsel to confirm the model before Payments is built.

**Related:** BR-31, BR-34, BR-38, D-11, D-17, D-21, RV-02, RV-04, PAY-005, PAY-006, PAY-009, PAY-010, DISP-002, OQ-11, OQ-17, OQ-19, OQ-20, T-1.

---

### AUD-002 — Ledger posting rules are missing for most money events; chart of accounts is incomplete

| Field | Value |
|---|---|
| Severity | HIGH |
| Category | Payments / Database |
| Affected document | Architecture |
| Section | §8.5 Ledger (LEDGER-001) [PROPOSED AD-15] |
| Product-owner approval | Yes (CA review, T-6) |

**Problem.** §8.5 gives one worked example (cash commission debit). It does not say what is posted for: online payment captured; the period between capture and the end of the dispute window (PRD says the traveler's earning is credited "after the dispute window", LEDGER-001 table — so where does the money sit until then?); when commission and GST are recognised (capture, delivery or window close); full and partial refunds and their effect on commission/GST; sender cancellation fee and traveler compensation share on online bookings ("from the retained amount"); late-payment refund; payout initiation, success and failure; dues netting; compensation and recovery; adjustments. Missing accounts include: provider/payment-gateway fees (and GST on them), payout in-transit/clearing, sender compensation payable, refunds in transit, unallocated/suspense (for unmatched provider money), TDS payable (TBD by CA), and per-booking "earnings pending window". BR-38 says compensation funded by sender dues "is posted only on `ledger.dues.paid`", but there is no entity that remembers the pending (not yet credited) compensation and which dues it depends on.

**Why it matters.** "Every rupee traceable" and reconciliation depend on deterministic posting rules. Without a fee account, every online payment will show a reconciliation mismatch (provider settles net of fees).

**Real-world impact.** Daily false reconciliation mismatches; commission revenue recognised inconsistently; GST reporting errors; finance cannot answer "how much do we owe travelers right now".

**Recommended correction.** Add a posting-rules table (event → debit/credit accounts → amounts → cause type → trigger) for every money event in AUD-001's list, a "contingent credit" record for BR-38 (created at the fee event, posted when the linked dues are paid), and the missing accounts. Have a CA review it before the Ledger module is built (T-6).

**Related:** LEDGER-001, PAY-005..010, BOOK-005, BR-24, BR-32, BR-38, RV-02, D-21, D-22.

---

### AUD-003 — Payment event handling misses out-of-order, duplicate and late cases; money can be captured with no booking

| Field | Value |
|---|---|
| Severity | HIGH |
| Category | Payments |
| Affected document | Architecture |
| Section | §8.4 Payments ("Confirmation"), §9.5 Provider webhooks, §7.5 Idempotency |
| Product-owner approval | No (design correction); one policy question (late payment grace, see decision log Q-07) |

**Problem.**
1. **Out-of-order statuses.** PAY-001 defines attempt states CREATED → PENDING → SUCCEEDED | FAILED | EXPIRED, with SUCCEEDED/FAILED treated as terminal. UPI payments commonly report a failure and later a success (or events arrive out of order). If FAILED is processed first, a later SUCCEEDED cannot transition (conditional update finds `status = FAILED`), so captured money gets neither a confirmation nor a refund.
2. **Success after other terminal booking states.** §8.4 only handles success "after PAYMENT_EXPIRED → automatic refund". Success can also arrive after CANCELLED_BY_SENDER (allowed from ACCEPTED_AWAITING_PAYMENT), after trip cancellation, or on a second attempt/order for a booking already CONFIRMED (PAY-001 allows retry; a sender with two tabs or two UPI apps can pay twice).
3. **Webhook vs poll.** Provider events are deduplicated by `(provider, event_id)`, but poll results (`payments.poll-pending`) have no event ID, and providers often emit several event types for one payment (authorised, captured, order paid). Deduplication by event ID does not stop the same payment being applied twice; only a business-level conditional transition does, and the doc does not state which transition triggers refund creation.
4. **What happens to the attempt at window expiry** (EXPIRED or left PENDING) is not stated; the choice decides whether a late success can be processed at all.

**Why it matters.** BR-16 requires idempotent provider events; BR-31 and customer trust require that captured money always ends in either a confirmed booking or a refund.

**Real-world impact.** Senders charged with no booking until finance notices in the next day's reconciliation; or two refunds for one late payment (one from webhook, one from poll).

**Recommended correction.** (a) Treat the provider as the source of truth: on every webhook or poll, fetch the payment's current status from the provider and apply it; allow FAILED → SUCCEEDED (and EXPIRED → SUCCEEDED) as transitions. (b) Define a single rule: "a SUCCEEDED attempt whose booking is not in ACCEPTED_AWAITING_PAYMENT, or whose booking already has another SUCCEEDED attempt, creates exactly one refund". Enforce with a unique constraint on `refund(payment_attempt_id, reason)` for automatic refunds. (c) Refund creation happens only in the transaction that wins the attempt's conditional transition to SUCCEEDED. (d) Set the provider-side order/UPI-collect expiry to end at or before the payment window to reduce late payments.

**Related:** PAY-001, PAY-002, PAY-005, BOOK-004, BR-06, BR-16, BR-31; stress scenarios B, C, D.

---

### AUD-004 — Refund and provider-call retries are not idempotent; provider timeouts can double-refund

| Field | Value |
|---|---|
| Severity | HIGH |
| Category | Payments / Reliability |
| Affected document | Architecture |
| Section | §8.4 Payments, §14.6 Provider resilience |
| Product-owner approval | No |

**Problem.** §14.6 allows "bounded retries for idempotent operations only". Payouts carry an idempotency key `payout:<id>`; refunds, orders and beneficiary validations have none documented. When a refund call times out, the provider may already have processed it. Retrying blindly can refund twice; not retrying leaves the refund in limbo. There is no refund state machine in the architecture (PRD gives REQUESTED → PROCESSING → SUCCEEDED | FAILED), no "unknown outcome" state and no rule to query the provider before retrying.

**Why it matters / impact.** A double refund is a direct loss; a lost refund is a customer complaint and possibly a consumer-protection issue.

**Recommended correction.** Every outbound money call (order, refund, transfer, payout) gets a stable GoParcel reference (`refund:<id>`, etc.) sent as the provider's idempotency key or merchant reference. On timeout, set the record to `UNKNOWN`/`PROCESSING`, then reconcile by fetching by that reference before any retry. Document which operations the chosen vendor makes idempotent (unverified for any vendor until OQ-19).

**Related:** PAY-005, PAY-006, BR-16, NFR-REL-02; stress scenario B.

---

### AUD-005 — Single worker is a single point of failure for money correctness; expiry can overtake payment confirmation

| Field | Value |
|---|---|
| Severity | HIGH |
| Category | Reliability |
| Affected document | Architecture |
| Section | §7.7 Scheduled work, §9.5 Provider webhooks, §14.2 Runtime topology |
| Product-owner approval | No |

**Problem.** Webhooks are stored and "processed asynchronously" by the worker (§9.5). Payment-window expiry (§7.7) also runs in the worker. The pilot runs **one** worker instance (§14.2). If the worker is down or backlogged for longer than `CFG-PAY-WINDOW` (15 min), on recovery both the queued expiry jobs (their `startAfter` has passed) and the queued "payment succeeded" processing run in no guaranteed order. The expiry handler "re-checks state" — but the booking is still ACCEPTED_AWAITING_PAYMENT, so it expires it, releases capacity and reopens the shipment. The payment is then refunded as "late", although the sender paid on time.

**Why it matters / impact.** A 20-minute worker outage at a busy time turns every in-flight online booking into a refund and a lost booking; another sender may grab the released capacity.

**Recommended correction.** (a) "Check before expire": the expiry handler must, in its transaction, look for a SUCCEEDED attempt or an unprocessed `provider_event` for the booking, and query the provider for PENDING attempts before expiring. (b) Run at least two worker instances (pg-boss supports concurrent workers; the doc already says this is safe). (c) Alert on queue age, not only depth. (d) Optionally process payment webhooks synchronously in the API request when the database is healthy, keeping the async path as fallback.

**Related:** BOOK-004, PAY-002, NFR-REL-01, NFR-REL-02, NFR-AVAIL-01; stress scenarios B, D.

---

### AUD-006 — The global lock order is contradicted by the document's own flows

| Field | Value |
|---|---|
| Severity | HIGH |
| Category | Architecture Conflict / Database |
| Affected document | Architecture |
| Section | §7.2 Transactions and lock order (AD-12), §7.4 Delivery and custody transitions, §7.7 Scheduled work |
| Product-owner approval | No |

**Problem.** §7.2 fixes the order shipment → bookings → trip_capacity → delivery → payout/ledger and says "a helper acquires locks only in this order and is used by every multi-row command". But:
- **Pickup (§7.4):** Courier locks and updates the **delivery**, then a transactional event makes Booking update the **booking**, which then calls Shipments to update the **shipment**. That is order 4 → 2 → 1, the exact reverse. A sender cancellation (shipment → booking → capacity → delivery) running at the same moment can deadlock with it. Postgres will abort one transaction, so this is not a hang, but the documented guarantee is false and the user sees an error at the handover.
- **Payment-window expiry (§7.7):** "PAYMENT_EXPIRED, release capacity, shipment → OPEN" — the natural implementation starts from the booking; the doc does not say it locks the shipment first.
- **Trip cancellation with several bookings (TRIP-004):** starts from the trip, touches many shipments. To follow the order it must read the booking list unlocked, lock shipments in ascending id, then re-verify. Not described.
- **Rows missing from the order:** `trip` (status), `payment_attempt`, `hold`/`dispute`, `user` (status), `refund`. Flows that lock these have no defined position.
- Because Courier (a lower module) starts the pickup transaction, the "helper" cannot see the whole lock set up front; the event-driven design makes a central lock order hard to enforce.

**Why it matters / impact.** Deadlocks under load at exactly the moments that matter (handover at a 5-minute halt, payment confirmation), plus retries that may re-send notifications. The concurrency test suite listed in §7.2 does not include pickup vs cancel.

**Recommended correction.** Either (a) make Booking the orchestrator of pickup (`Booking.recordPickup` locks shipment → booking → delivery, then calls Courier to verify the code inside), or (b) lock the full set up front by id at the start of every multi-aggregate command. Extend the lock order to include trip, payment_attempt and hold rows. Add pickup-vs-cancel, trip-cancel-vs-accept and expiry-vs-payment to the concurrency suite. Retry transactions on deadlock/serialization errors in one place.

**Related:** AD-12, BOOK-002, BOOK-007, BR-07, NFR-REL-01, PRD F.6 #8 (DR-07); stress scenarios A, E.

---

### AUD-007 — The accept transaction re-checks too little; trip status, edits and cutoffs can be bypassed

| Field | Value |
|---|---|
| Severity | HIGH |
| Category | Domain Design / PRD Misalignment |
| Affected document | Architecture (and a PRD gap) |
| Section | §7.3 The accept transaction (BOOK-002, WF-A); §1 (OQ-02 assumption) |
| Product-owner approval | Partly (trip edit rules while requests are pending, quote validity — see decision log Q-05, Q-06) |

**Problem.** The accept sequence re-checks traveler status/KYC, cash opt-in/dues, the DEC-051 guard (online only), shipment OPEN and capacity. It does **not** re-check:
- **Trip status**: the trip row is never read or locked. A trip cancellation committing concurrently can leave a CONFIRMED booking on a CANCELLED trip.
- **Booking cutoff for cash**: §1 says "for cash bookings only the booking cutoff applies", but §7.3 applies no time guard to cash at all. Cutoff is enforced only in matching and by the request-expiry job, which may run late (sweeper every 1–5 min).
- **Match still valid**: stops still on the trip and still eligible (halt ≥ minimum), delivery window still satisfied after a delay update, mode/size still allowed. TRIP-003 lets a traveler edit all fields "before any booking request is accepted", i.e. while requests are pending; nothing in the architecture invalidates pending requests on edit.
- **Sender status** (suspended or dues-blocked after the request).
- **Quote validity**: Quote has `valid_until = CFG-QUOTE-TTL` (30 min) but a request lives up to `CFG-REQUEST-TTL` (6 h), and PRICE-003 says the request's quote is honoured. The architecture does not say which rule wins.

**Why it matters / impact.** Bookings that violate confirmed rules (BR-02, BR-03, BR-26, D-09) can be created: a booking whose pickup stop was removed from the trip, a cash booking 20 minutes before departure, a booking on a cancelled trip.

**Recommended correction.** Make accept call one `assertStillMatches(booking)` guard that re-runs the MATCH-001 conditions against locked trip and trip-stop data, plus the time guard for both methods (DEC-051 for online, cutoff for cash), plus sender eligibility. Add `trip` to the lock order and lock it `FOR SHARE` (or `FOR UPDATE`) in accept and trip cancel. Define what a trip edit does to pending REQUESTED bookings (recommended: system-cancel them with reason "trip changed"). Ask the product owner which quote rule wins.

**Related:** BOOK-001, BOOK-002, TRIP-003, TRIP-004, PRICE-003, MATCH-001, BR-02, BR-03, BR-26, DEC-051, OQ-02; stress scenarios F, G.

---

### AUD-008 — Cancellation, no-show and pickup-decline workflows are absent from the architecture

| Field | Value |
|---|---|
| Severity | HIGH |
| Category | PRD Misalignment / Payments |
| Affected document | Architecture |
| Section | §7 Core runtime patterns and §8 Domain-specific design (no section covers them) |
| Product-owner approval | No for the design; amounts remain TBD (OQ-10, OQ-16) |

**Problem.** The architecture designs accept, payment, pickup and delivery, but none of: sender cancellation (BOOK-005, three time bands, online vs cash), traveler cancellation/no-show (BOOK-006), trip cancellation with several bookings (TRIP-004), sender refuses inspection (BR-28), traveler mismatch decline with later ops fault decision (HAND-004, RV-03), sender no-show (HAND-009), parcel withdrawal (SHIP-004), delay-triggered free cancellation (TRIP-005). Each needs: which booking terminal state, refund amount calculation from snapshot config, fee and compensation-share postings (incl. BR-38 contingent credit for cash), capacity release under the lock order, delivery → CANCELLED, masked-call teardown, reliability mark, re-match. The mismatch decline also needs a "fault pending" state or record so that fees are posted only after ops decides.

**Why it matters / impact.** These are the most frequent money events after payment itself. Leaving them to implementers means refund amounts and ledger entries get decided ad hoc in code.

**Recommended correction.** Add a cancellation section with one table: trigger → actor → allowed booking states → resulting states (booking, shipment, delivery, capacity) → refund/fee/compensation calculation → ledger posting → notifications → reliability mark. Add a `cancellation_review` record (or booking sub-state) for mismatch declines.

**Related:** BOOK-005, BOOK-006, TRIP-004, TRIP-005, SHIP-004, HAND-004, HAND-009, BR-12, BR-28, BR-38, RV-02, RV-03; stress scenario E.

---

### AUD-009 — Payout run can double-pay, pay held money, or show money as paid that never left

| Field | Value |
|---|---|
| Severity | HIGH |
| Category | Payments |
| Affected document | Architecture; PRD |
| Section | Architecture §8.4 Payments ("Payouts"), §7.7 Scheduled work (`payments.payout-run`, `payments.dispute-window-close`); PRD PAY-006, §6.3 Payout state machine |
| Product-owner approval | Partly (payout granularity, decision log Q-09) |

**Problem.**
1. **Ledger posted before the provider call** ("computes net, posts ledger entries, then calls the provider"). If the call fails or times out, the ledger says the traveler was paid. There is no payout-clearing account and no reversal rule on FAILED.
2. **No guarantee a credit is paid once.** The idempotency key is per payout record. Two overlapping runs (cron overlap, retry, a second worker) can each create a payout record covering the same credits. Nothing says credits are linked to at most one non-failed payout (no unique constraint), or that the run locks the traveler's ledger account.
3. **Hold race.** `PayoutHoldChecker` is called, then a dispute is raised (auto-hold, DISP-001) before the provider call; the payout still goes. No lock or re-check is defined.
4. **Netting race.** A cash confirmation posting new dues during a run is not serialised with it.
5. **Granularity conflict.** The PRD payout state machine (§6.3) is per delivery (PENDING_WINDOW at delivery), while PAY-006 nets all credits minus all dues into one transfer. The architecture uses both (dispute-window job moves a payout to ELIGIBLE; the run aggregates credits) without defining a "payable item" vs "payout batch" model.

**Why it matters / impact.** Double payouts are direct loss; paying disputed money defeats BR-17; a ledger that claims a payout that failed breaks reconciliation and traveler trust.

**Recommended correction.** Separate `payable_item` (per earning/credit, carries hold and window state) from `payout` (one transfer). A payable item references at most one payout in a non-FAILED state (unique partial index). The run locks the traveler's ledger account row, re-checks holds under lock, posts `Dr earnings / Cr payout_clearing` on initiation and `Dr payout_clearing / Cr provider_or_bank` on PAID, reverses on FAILED. Use a pg-boss singleton key so only one run per traveler is active. Ask the product owner to confirm per-delivery vs aggregated payouts (D-18 says "per delivery").

**Related:** PAY-006, PAY-008, BR-17, BR-35, D-18, D-21, DISP-003, RV-04.

---

### AUD-010 — Handover codes stored only as hashes, but the PRD requires them to be displayed later

| Field | Value |
|---|---|
| Severity | HIGH |
| Category | Architecture Conflict / Security |
| Affected document | Architecture vs PRD |
| Section | Architecture §10.2 Encryption and secrets ("Codes and tokens"), §7.4, §8.7, §9.3; PRD HAND-004 step 6, HAND-006 step 2, §9.3 Data rules |
| Product-owner approval | No (design choice within PRD's "never stored in plain text") |

**Problem.** §10.2: "handover codes are stored as HMACs with a pepper ... Plain values exist only in memory and in the outgoing message." But HAND-004 step 6 says "Sender's screen shows the 6-digit pickup code" at the seat, possibly days after confirmation, and HAND-006 step 2 says the recipient link shows "the delivery code" for as long as the link is valid. A hash cannot be shown again. Integration-event handlers "load what they need through contracts" (§7.6), so a notification sent later cannot include the code either.

**Why it matters / impact.** As written, pickup and delivery codes are only available at creation time; the core handover flow cannot work.

**Recommended correction.** Store codes encrypted with the field-encryption key (AD-16) in addition to (or instead of) the HMAC used for verification; decrypt only for the authorised viewer (sender for pickup/return codes, recipient-token holder for the delivery code). Keep the attempt counter and lockout. This still satisfies the PRD rule "never stored in plain text".

**Related:** HAND-004, HAND-006, HAND-010, BR-09, BR-14, §9.3 Data rules, AD-16.

---

### AUD-011 — Handover depends on live network at the seat; no offline or degraded mode

| Field | Value |
|---|---|
| Severity | HIGH |
| Category | Reliability / PRD Misalignment |
| Affected document | Architecture and PRD |
| Section | Architecture §7.4, §13 Web application ("States"); PRD HAND-004, HAND-006, §5.7 ("Offline") |
| Product-owner approval | **Yes** (product decision, decision log Q-01) |

**Problem.** Custody transfers only when the server verifies the code (BR-29). The PRD's offline requirement covers only photos (§5.7). Mobile coverage inside trains and at many Indian stations is patchy, halts can be 5 minutes (`CFG-MIN-HALT-MINUTES`), and a database or API outage has the same effect. Neither document defines what happens when the traveler has the parcel in hand and the code cannot be verified.

**Why it matters / impact.** Missed handovers (HAND-008 fees, return trips) caused by connectivity, not by people; or travelers taking parcels without verification, which voids the evidence model.

**Recommended correction.** Product decision needed. Options: (a) offline-verifiable codes (the traveler's device holds a signed verifier issued at confirmation; verification is queued and confirmed by the server when back online, with a short deadline); (b) a "verification pending" custody state with server confirmation within N minutes; (c) accept the risk and rely on HAND-008. The architecture should then design it explicitly.

**Related:** HAND-004, HAND-006, HAND-008, BR-09, BR-11, BR-29, R-04, NFR-AVAIL-01.

---

### AUD-012 — Recovery and recycled-number protection are not designed; account takeover can redirect payouts

| Field | Value |
|---|---|
| Severity | HIGH |
| Category | Security |
| Affected document | Architecture (and PRD config) |
| Section | Architecture §8.1 Identity and sessions, §8.9 KYC; PRD AUTH-004, KYC-004, Appendix A `CFG-RECYCLED-NUMBER-DAYS` |
| Product-owner approval | Yes for the config value; no for the design |

**Problem.** §8.1 covers OTP and sessions but not AUTH-004: no "last active" tracking, no recycled-number check at login, no selfie re-match flow. §8.9 stores the KYC face score but **not the KYC selfie or a provider reference usable for re-match**, so "fresh selfie matched to the KYC photo on file" is impossible unless the provider supports re-match by reference (unverified). The phone-change payout hold needs Identity (L0) to place a hold owned by Disputes (L4), which the layering forbids. Separately, the PRD's `CFG-RECYCLED-NUMBER-DAYS` = 180, while Indian operators can reassign a disconnected number earlier (commonly cited as 90 days; unverified). A new holder of a recycled number inside that gap gets full access, and a beneficiary change is protected only by an SMS OTP to that same number (KYC-004).

**Why it matters / impact.** Takeover of a traveler account leads directly to payouts redirected to the attacker; takeover of a sender account exposes recipients' data.

**Recommended correction.** Design AUTH-004: track `last_seen_at`; at login from a number inactive beyond the threshold, or on any beneficiary change, require selfie liveness matched against a stored KYC reference (store an encrypted reference image or a provider face-template reference, subject to counsel). Use an event (`identity.phone.changed`) consumed by Disputes/Payments to place the hold. Ask the product owner to lower the threshold to at most the carrier recycle period and to require selfie re-match for beneficiary changes.

**Related:** AUTH-004, KYC-003, KYC-004, D-03, FRAUD-001, PRD §10.4 (account takeover).

---

### AUD-013 — "Jobs created in the same transaction" is not how pg-boss works by default; pooler and privilege claims unverified

| Field | Value |
|---|---|
| Severity | HIGH |
| Category | Infrastructure / Reliability |
| Affected document | Architecture |
| Section | §7.7 Scheduled work, §7.6 Events, §6.4 Platform kernel (`jobs`), §14.2 Runtime topology ("Database connections"), §10.6 Supabase hardening |
| Product-owner approval | No |

**Problem.**
1. §7.7: "Deadlines use pg-boss jobs with `startAfter`, created in the same transaction as the state that needs them", and §7.3 schedules the payment-window job inside the accept transaction. pg-boss uses its own `pg` connection pool by default, so `send()` is **not** part of the Prisma interactive transaction. pg-boss offers a custom `db`/`executeSql` adapter that could route the insert through the Prisma transaction (unverified for the pinned version, and Prisma's `$executeRawUnsafe` inside `$transaction` would be needed). If not done, a crash between commit and `send()` loses the deadline (the sweeper mitigates this only if it exists for every job type).
2. The API connects through Supabase's transaction-mode pooler (§14.2). Any pg-boss call from the API therefore runs through transaction pooling; features that need session state (session-level advisory locks, LISTEN/NOTIFY, prepared statements) are unavailable there (Supavisor behaviour; unverified for the current version).
3. pg-boss creates and migrates its own schema at `start()` unless told not to. That needs DDL rights, which contradicts §10.6 ("the API connects with a dedicated least-privilege role; migrations run with a separate owner role used only in CI and CD").
4. "The worker uses a direct or session connection": Supabase direct connections are reported to be IPv6-only unless an IPv4 add-on is bought (unverified); the hosting vendor (TBD) may not support IPv6.
5. Connection budget: two API instances (Prisma pools) + worker (pg-boss pool + Prisma pool) + migrations against the pooler/direct limits of the chosen Supabase compute size is not sized.

**Why it matters / impact.** The platform kernel is build step 1 (§16). If the transactional-job claim is wrong, every timer (payment window, request expiry, dispute window) can silently go missing.

**Recommended correction.** Use the outbox as the only thing written in the business transaction (it already is in the same transaction), and have the worker turn outbox rows into pg-boss jobs; or adopt pg-boss's db adapter and prove it with a test in step 1. Store the deadline itself on the aggregate (`payment_due_at`, `expires_at`) and make the sweepers query those columns, so jobs are an optimisation, not the source of truth. Run pg-boss schema migrations in CD with the owner role and start pg-boss with migration disabled. Verify Supabase connectivity (IPv4/IPv6, pooler modes, limits) in a spike before AD-09 is decided.

**Related:** AD-03, AD-09, AD-12, AD-13, NFR-REL-01, NFR-REL-02, T-7.

---

### AUD-014 — Module layering in §6.3 is violated by the document's own dependency table and workflows

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Domain Design / Architecture Conflict |
| Section | §6.3 Module dependencies; §8.4; §8.7; §8.10; §11.5; §12 |
| Product-owner approval | No |

**Problem.** §6.3 says a module may call only modules **below** it. Yet: Booking → Payments (same layer L3); Matching → Routes, Shipments, Pricing (same layer L2); Notifications and Tracking sit at L0 but must load booking/shipment/user data to compose messages and timelines ("handlers load what they need through contracts", §7.6) — upward calls; Payments (L3) places reconciliation holds owned by Disputes (L4) (§8.4); Identity (L0) restriction must withdraw trips/parcels and cancel requests in L2/L3 (AUTH-005); Privacy (L4) retention deletes data in every module's tables (§11.5), against rule 2; `platform/config` (kernel) reads `admin.config_value` (§12). AD-11 promises CI will fail on these.

**Impact.** Either CI rules are relaxed on day one, or developers invent workarounds per case.

**Recommended correction.** Allow same-layer calls with an explicit acyclic list; move Notifications and Tracking timeline composition to a top "consumers" layer (they only consume events); give each module an `anonymise(subjectId)` and `placeHold`-style port where a higher module must act; move config storage into the kernel. Publish the allowed-dependency list as the lint rule source.

**Related:** ARCH-06, AD-11, NFR-MAINT-01.

---

### AUD-015 — In-transaction "transactional events" create hidden synchronous coupling and block extraction

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Domain Design |
| Section | §7.6 Events: transactional and integration (AD-13); §7.4; §8.4 ("Confirmation") |
| Product-owner approval | No |

**Problem.** Transactional events run upper-module code synchronously inside a lower module's transaction (Courier → Booking → Shipments). Booking also calls Courier directly (createDelivery). This is a runtime cycle hidden behind an event bus; a handler failure in Booking rolls back Courier's command. Extraction (ARCH-08, G-4) would need a distributed transaction. §8.4 does not say whether `payments.attempt.succeeded` → booking CONFIRMED is transactional or an outbox event, and §7.6's list of integration-event uses does not include it.

**Recommended correction.** Optional simplification: replace transactional events with explicit orchestration in the top module of each flow (Booking orchestrates accept, confirm, pickup, cancel). State explicitly that payment confirmation is an integration event processed idempotently by Booking (and design it as such), or explicitly transactional.

**Related:** AD-13, ARCH-06, ARCH-08.

---

### AUD-016 — StateEvent: "same transaction" vs "only through outbox"

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Architecture Conflict |
| Section | §7.1 Server-authoritative state machines vs §6.3 ("Everyone → Tracking ... only through integration events (outbox), never direct calls"); §11.2 (`StateEvent` owned by Tracking) |

**Problem.** §7.1 says every transition writes a `StateEvent` in the same transaction; §6.3 says modules reach Tracking only through the outbox; §6.2 rule 2 says a module writes only its own tables. All three cannot hold.

**Recommended correction.** Each module writes its own `<module>.state_event` table in the transaction (audit-grade, TRACK-002); Tracking builds the user timeline (TRACK-001) from outbox events.

**Related:** TRACK-001, TRACK-002, BR-19.

---

### AUD-017 — Suspension, restriction and re-verification cascades are not designed

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | PRD Misalignment |
| Section | §8.1 Identity and sessions; §8.10 Fraud |

**Problem.** AUTH-005 requires that suspension withdraws published trips and parcels, system-cancels pending requests and routes confirmed bookings to ops without cancelling them; KYC-003 blocks publish/accept and routes bookings to ops; PAY-009 hard block restricts. §8.10 only says a high flag calls `restrictAccount`. Who runs the cascade, synchronously or by event, is undefined. Also, accept re-checks traveler status without a row lock (READ COMMITTED), so an accept can commit just after the cascade enumerated the user's bookings.

**Recommended correction.** Define `identity.account.restricted/suspended` events with handlers in Routes, Shipments, Booking, Operations; lock the user row `FOR SHARE` in accept and request (or re-check in the cascade after a short delay). Add an acceptance test for AUTH-005's first criterion.

**Related:** AUTH-005, KYC-003, PAY-009, DISP-005, BR-03.

---

### AUD-018 — Idempotency-record semantics can lock a user out for 24 hours

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | API / Reliability |
| Section | §7.5 Idempotency |

**Problem.** The record has status IN_PROGRESS/DONE and a key "kept across retries" (§13). The doc does not say whether the record is inserted in the business transaction or separately. If separately and the business transaction fails or the process crashes, the key stays IN_PROGRESS and every retry returns `409 REQUEST_IN_PROGRESS` until the 24 h cleanup. It also does not say whether business errors (e.g. `JOURNEY_TOO_SOON`) are stored and replayed.

**Recommended correction.** Insert the record in the same transaction as the business change (concurrent duplicates then wait on the unique index and see the result), or add a lease timeout to IN_PROGRESS. Store 2xx and deterministic 4xx responses; never store 5xx.

**Related:** BOOK-007, BR-16.

---

### AUD-019 — Several PRD invariants have no database or locking protection

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Database |
| Section | §11.3 Database-enforced invariants |

**Problem.** Not covered: (a) max `CFG-MAX-CONCURRENT-REQUESTS` live requests per shipment — two simultaneous requests can both see 2 and create a 4th (BOOK-001 acceptance criterion); (b) one cash confirmation per (delivery, stage); (c) one automatic refund per attempt and reason (AUD-003); (d) one payment attempt in PENDING per booking; (e) dues payments capped at amount owed — two payment links paid concurrently overpay and create a positive balance (D-21); (f) one payable item per earning (AUD-009).

**Recommended correction.** Lock the shipment row when creating a request; add the unique/partial indexes listed; create dues payment links with a lock on the party's dues account and auto-refund any overpayment.

**Related:** BOOK-001, PAY-003, PAY-009, LEDGER-001, BR-32, D-21.

---

### AUD-020 — Cash confirmation is posted before pickup with no reversal

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Payments |
| Section | §7.4 ("Cash confirmation ... run before code entry") |

**Problem.** The commission + GST debit is posted in its own transaction when the traveler taps "cash received". If pickup then does not happen (traveler declines for mismatch, wrong code lockout, sender walks away, or the sender disputes within `CFG-CASH-DISPUTE-MINUTES`), the traveler keeps a dues debit for a booking that never moved. No reversal or link to the outcome is defined.

**Recommended correction.** Post the commission debit in the pickup (custody) transaction, with the cash confirmation recorded beforehand as a non-financial fact; or define a reversal posting on decline/no-pickup and on an upheld cash dispute.

**Related:** PAY-003, PAY-004, LEDGER-001, HAND-004.

---

### AUD-021 — The trip-overlap exclusion constraint can block delay updates

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Database |
| Section | §11.3 ("No overlapping published trips per traveler") |

**Problem.** The `tstzrange` exclusion constraint uses the trip period. A delay update (TRIP-005, or automatic from train status) that extends the arrival can overlap the traveler's next trip and be rejected by the database, so the traveler cannot report a real delay. Which statuses count as "active" and whether the period uses scheduled or estimated times is not defined.

**Recommended correction.** Apply the constraint to the scheduled period set at publish only; store estimates in separate columns not covered by the constraint; flag (not block) overlaps caused by delays.

**Related:** TRIP-001, TRIP-005, HAND-002.

---

### AUD-022 — Capacity model puts the weight cap on each size-class row

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Database / Domain Design |
| Section | §7.2 (lock "trip_capacity row(s) ... then size class"), §7.3, §11.3 (`CHECK (weight_used_g BETWEEN 0 AND weight_cap_g)`) |

**Problem.** TRIP-001 defines "number of S, M, L slots and **total** weight cap". Per-size-class rows each with a `weight_cap_g` check do not enforce a trip-level total. Also, capacity is whole-trip: a slot used A→B is unavailable B→C even though intermediate-stop matching (D-09) makes reuse possible — a product question, not a defect.

**Recommended correction.** One trip-level row for weight (locked first), plus per-class slot rows; or one row with columns per class. Ask the product owner whether segment reuse matters (decision log Q-11).

**Related:** TRIP-001, BOOK-002, BR-07, NFR-DATA-01.

---

### AUD-023 — Exact station-ID matching may not match how senders and travelers choose places

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Geospatial / PRD Misalignment |
| Section | §8.2 Matching and search ("Station matching uses master-list station IDs") |
| Product-owner approval | **Yes** (decision log Q-03) |

**Problem.** Parcel origin/destination come from a "station / city master list" (SHIP-001); train stops are railway stations, bus stops are bus stands, car stops are meeting points (TRIP-001). Exact ID equality means a parcel from "Indore" (city) or "Indore Junction" never matches a bus trip from "Sarwate bus stand" or a car meeting point in Indore. The architecture does not define a city/station grouping.

**Recommended correction.** Product owner decides whether the parcel picks a city (and the handover point comes from the trip) or an exact station. Architecture then defines `station.city_id` matching or explicit equivalence groups and how car meeting points map to cities.

**Related:** D-09, MATCH-001, ROUTE-001, ROUTE-002, SHIP-001.

---

### AUD-024 — Terminal transitions for booking, shipment and delivery have no owner

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Domain Design |
| Section | §7.4; §7.7 Scheduled work |

**Problem.** Nothing moves booking IN_FULFILMENT → COMPLETED ("delivery terminal and dispute window closed / dispute resolved"), shipment IN_DELIVERY → CLOSED, or delivery PENDING_PICKUP → CANCELLED. Bookings whose delivery ends RETURNED, LOST or ABANDONED have no defined booking outcome. HAND-012 says "LOST or DELIVERED-with-claim", but the PRD delivery state machine has no EXCEPTION → DELIVERED edge (a PRD inconsistency). §7.4 lists "overdue" as an EXCEPTION cause while also saying overdue jobs "only raise alerts".

**Recommended correction.** Add handlers for each terminal transition; ask the product owner about EXCEPTION → DELIVERED and booking outcomes for RETURNED/LOST (decision log Q-10).

**Related:** BOOK state machine, SHIP-006, HAND-012, HAND-013, TRIP-006, BR-11.

---

### AUD-025 — Payment-window extension and the payment deadline are not designed

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Payments |
| Section | §7.3; §7.7 (`booking.payment-window`) |

**Problem.** SUP-002 lets support extend the window once by up to `CFG-PAY-WINDOW-MAX-EXTENSION`. The architecture stores no `payment_due_at` on the booking, the expiry job is scheduled for a fixed time, and an extension could push payment past the point DEC-051 protects (pay window + cutoff before departure).

**Recommended correction.** Store `payment_due_at`; the expiry handler compares against it; the extension command re-checks `departure − new_due ≥ CFG-BOOKING-CUTOFF`; set provider order expiry accordingly.

**Related:** SUP-002, BOOK-004, DEC-051, BR-26.

---

### AUD-026 — Recipient link equals the right to collect the parcel

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Security |
| Section | §9.3 Recipient access |
| Product-owner approval | Yes (step-up option, decision log Q-04) |

**Problem.** The link shows the delivery code (HAND-006), so anyone who sees a forwarded WhatsApp message can collect the parcel. Link-preview crawlers fetch the URL; if the page server-renders the code, it is sent to a third party. The token in the path can leak through `Referer` headers or third-party scripts.

**Recommended correction.** Never server-render the code; reveal it only after a tap, via an API call with `Cache-Control: no-store`; set `Referrer-Policy: no-referrer` and no third-party scripts on `/r/*`; optionally require an OTP to the recipient's phone before revealing the code (product decision).

**Related:** BR-14, HAND-006, NFR-PRIV-02, PRD §10.4 (recipient link leakage).

---

### AUD-027 — False delivery and code phishing; evidence authenticity is overstated

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Security |
| Section | §7.4; §8.6 Courier evidence; §13 ("Camera", "Location") |

**Problem.** A traveler can phone the recipient ("I'm at the coach door, tell me the code"), enter it hours before or without a handover, and the delivery is DELIVERED. No plausibility check (time vs expected arrival at the delivery stop, train running status) exists. "In-page camera only" is a client-side rule: the API accepts any image with a matching client-computed checksum, and device timestamps are client-supplied. Browser geolocation is trivially spoofable (the architecture rightly does not use it as evidence; this should be stated).

**Recommended correction.** Reject or flag delivery verification far from the expected arrival window; record server receipt times only as evidence times; state in the threat model that photos are user-supplied evidence; add risk rules for deliveries verified long before arrival. Coach recipients in notifications never to share the code by phone.

**Related:** HAND-003, HAND-006, BR-10, BR-29, FRAUD-001.

---

### AUD-028 — Evidence objects have no backup, and checksum-on-read is not designed

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Reliability |
| Section | §8.6 Courier evidence; §10.6 Supabase hardening (PITR) |

**Problem.** PITR protects the database. Supabase's documentation is understood to say database backups do not include Storage objects (unverified during this audit). Evidence is the basis of liability decisions (D-17). HAND-007 requires checksum verification on read and an export with integrity manifest (T-9); §8.6 verifies only at upload.

**Recommended correction.** Replicate evidence to a second bucket/provider or enable object versioning if available; run a periodic integrity sweep (sample or full) instead of hashing on every read; design the export (T-9) before pilot.

**Related:** HAND-007, NFR-BACKUP-01, BR-15, D-17.

---

### AUD-029 — OTP abuse and Postgres rate limiting

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Security / Performance |
| Section | §10.4 Rate limiting; §8.1 ("Same response for every number") |

**Problem.** Fixed windows allow double bursts at window edges; every limited request becomes a database write on a shared row, so an OTP flood loads the same database that confirms payments. SMS-pumping fraud (bots requesting OTPs to premium or many numbers) is a known cost attack on OTP endpoints; per-number and per-IP limits do not stop distributed attacks, and there is no global daily spend cap or circuit breaker.

**Recommended correction.** Add a global and per-prefix OTP budget with a circuit breaker and alert; use the host edge/WAF limit as the primary flood control; consider a sliding window or token bucket in one statement; keep Redis as the documented escalation if counters show up in DB metrics.

**Related:** AUTH-001, NFR-SEC-05, NFR-COST-01, R-08.

---

### AUD-030 — Admin console shares the user app's origin; CSRF posture is unspecified

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Security |
| Section | §9.4 Web ↔ API connection; §8.1 ("Sessions"); AD-04 |

**Problem.** Staff and users share one origin. Any XSS in user-supplied content (parcel descriptions, names, dispute text) rendered in the admin console runs with staff cookies. The access token is "in memory or a cookie" — undecided; cookie-authenticated POSTs with `SameSite=Lax` rely on browser behaviour for CSRF protection.

**Recommended correction.** Serve admin on a separate subdomain with its own cookies and a strict CSP; decide token storage; add a CSRF token or `SameSite=Strict` for state-changing cookie-authenticated calls.

**Related:** ADMIN-001, NFR-SEC-03, AD-04.

---

### AUD-031 — Reconciliation design is incomplete

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Payments |
| Section | §8.4 ("Reconciliation"); §7.7 (`payments.reconcile`) |

**Problem.** The daily job "compares provider settlement and payout reports with ledger journals". Not covered: matching key (journals need the provider payment/refund/transfer ID), settlement timing (T+1/T+2) producing false mismatches, fee lines (AUD-002), refund reports, orphan captures (provider has a payment GoParcel never recorded), and re-reconciliation after a PITR restore (provider facts after the restore point must be replayed). The reconciliation hold "on the related booking" is a Disputes-owned hold placed by Payments (AUD-014).

**Recommended correction.** Define reconciliation item types, matching keys, tolerance windows, and a post-restore replay procedure (re-fetch provider events since the restore point).

**Related:** PAY-008, DISP-003, NFR-BACKUP-01, R-10.

---

### AUD-032 — No threat model; GoParcel-specific threats are not analysed

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Security / Documentation |
| Section | §10.7 Threats |

**Problem.** §10.7 maps to PRD §10.4 and defers to a pen test. There is no STRIDE analysis per trust boundary (browser, recipient link, provider webhooks, staff console, worker, Supabase dashboard). Not addressed: collusion between sender and traveler to extract ₹5,000 compensation from the reserve and abandon the traveler account (recovery by ledger debit is weak); one person holding several accounts via different documents (DL vs Aadhaar) defeating D-01; insiders with Supabase dashboard/SQL access bypassing app audit (field encryption helps only if keys are elsewhere); refund abuse via repeated borderline cancellations; staff reveal misuse patterns. PRD §10.3 asks for a threat model per release.

**Recommended correction.** Produce a threat model document before Stage P covering the threats in the brief and above, with an owner and mitigation per threat; add risk rules for compensation exposure per pair and per traveler; restrict and log Supabase dashboard access.

**Related:** PRD §10.3, §10.4, FRAUD-001, D-17, BR-20, NFR-SEC-03.

---

### AUD-033 — No API inventory for the PRD journeys

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | API / Documentation |
| Section | §9 API design |

**Problem.** §9 sets conventions and gives three example commands. There is no list of endpoints per journey (PRD §5.2–5.5), no per-endpoint actor/ownership rule, no error-code catalogue, no per-endpoint rate limits or pagination contract. AD-05 generates the client from OpenAPI, so the inventory will otherwise emerge from code.

**Recommended correction.** Before each feature is built (P1), add an endpoint table per module: method, path, actor, ownership check, idempotency, rate limit, errors, state transition. Start with Booking, Payments and Courier.

**Related:** ARCH-05, ARCH-11, PRD §5, §5.7.

---

### AUD-034 — No physical data model for money and custody modules

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Database / Documentation |
| Section | §11 Data architecture; §11.2 Entity placement |

**Problem.** Entities are placed by module but key fields and records needed by findings above are missing: `payment_due_at`, booking terminal reason and fault-pending state, return booking ↔ original delivery link (HAND-010), changed delivery stop after a missed delivery (HAND-008), contingent credits (BR-38), payable items vs payouts, refund state, sender beneficiary, provider references on journals.

**Recommended correction.** Write a schema document (tables, keys, constraints, indexes) for Booking, Payments, Ledger and Courier before they are built.

**Related:** PRD §9, NFR-DATA-01.

---

### AUD-035 — Prisma limitations for the claimed constraints and locks

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Maintainability / Database |
| Section | §11.1 Schemas and ownership; §11.4 Migrations; §7.2 |

**Problem.** Prisma has no `SELECT ... FOR UPDATE` API, so the lock helper and most core commands will be raw SQL (losing type safety). Partial unique indexes, exclusion constraints, CHECK constraints, triggers and deferred constraint triggers are outside the Prisma schema; they must live in raw-SQL migrations. Prisma's migration diff may then report drift or generate statements that drop indexes it does not know about (a commonly reported issue; unverified for the pinned version). The doc's own `prisma migrate diff` CI check could fail or be ignored. Multi-schema support: the doc correctly says to verify (T-7); it is believed to have become generally available in recent Prisma 6.x releases (unverified).

**Recommended correction.** Keep a "raw SQL objects" registry and a CI test that asserts each expected index/constraint/trigger exists after migrations; define the lock helper and conditional updates as typed raw-SQL functions; verify the drift behaviour in the foundation spike.

**Related:** ARCH-04, AD-17, NFR-DATA-01, NFR-DATA-02, T-7.

---

### AUD-036 — Approving the architecture would implicitly adopt open product decisions

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Documentation |
| Section | §0.2 New architecture decisions; §1 Assumptions for the blocking open questions |
| Product-owner approval | Yes |

**Problem.** PRD Appendix F.8 says architecture should start once OQ-02, 05, 06, 07, 09 and 21 are answered; they are still open. The architecture proceeds on assumed answers (transparent and reasonable), but §0.2 says approving the document approves all AD-xx. The assumed answers include D-15 and RV-01..04 (still Proposed), and §8.5 says BR-38 "is enforced". There is a risk that approval is later read as approval of those product decisions.

**Recommended correction.** State in the approval record that architecture approval does not answer any OQ or approve D-15/RV-01..04; answer the six P1 questions first or approve them explicitly at the same time.

**Related:** OQ-02, OQ-05, OQ-06, OQ-07, OQ-09, OQ-21, D-15, RV-01..04, BR-38.

---

### AUD-037 — Duplicate-identity detection via Aadhaar blind index is uncertain

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Security / PRD Misalignment |
| Section | §8.9 KYC; §11.3 ("One verified KYC per identity document") |
| Product-owner approval | Counsel |

**Problem.** DigiLocker and Aadhaar offline e-KYC flows typically expose a masked number or a reference, not the full number (unverified per provider), so a blind index over "normalised doc number" may not be computable. Whether storing an HMAC of an Aadhaar number counts as storing it is a legal question. A person can also verify a second account with a different document type, defeating BR-20.

**Recommended correction.** Make duplicate detection provider-agnostic (provider's unique identity token if offered, plus face-template similarity, plus name+DOB fuzzy match feeding a risk flag); get counsel's view on hashing Aadhaar numbers.

**Related:** KYC-001, BR-20, D-01, FRAUD-001, OQ-20.

---

### AUD-038 — Field-encryption key management is undefined

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Security / Infrastructure |
| Section | §10.2 Encryption and secrets (AD-16) |

**Problem.** Envelope encryption needs a KMS for the key-encryption key; "a managed secret store or KMS" with a hosting vendor still TBD (AD-09) means keys may end up as environment variables next to the database credentials, which reduces envelope encryption to obfuscation. Rotation procedure, who can decrypt, and blind-index key separation in practice are not defined.

**Recommended correction.** Choose a KMS together with AD-09; document key hierarchy, rotation, break-glass and the rule that DB administrators never hold decryption keys.

**Related:** NFR-SEC-02, NFR-SEC-04, AD-16.

---

### AUD-039 — No cost estimate despite low cost being a top-three goal

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Infrastructure |
| Section | §2 (G-3), §14 Deployment and operations |

**Problem.** G-3 and NFR-COST-01 are central, but no monthly estimate exists. Likely paid items: Supabase plan plus compute size, PITR add-on, possibly IPv4 add-on (unverified pricing), container host for 2 API + 2 workers + web, error tracking, KMS, plus per-unit costs (WhatsApp templates, SMS/DLT, KYC checks, masked calling minutes, payment fees) that affect contribution margin (BG-3, R-13).

**Recommended correction.** Add a cost table for pilot sizing (NFR-SCALE-01) with fixed and per-booking costs; feed the per-booking costs into `CFG-MIN-COMMISSION` (OQ-12).

**Related:** NFR-COST-01, G-3, BG-3, OQ-12, R-13.

---

### AUD-040 — Notification data access and location/recipient retention not specified

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Domain Design / Privacy |
| Section | §8.7 Notifications; §13 ("Location"); §11.5 Retention |

**Problem.** Notification handlers must decrypt recipient phones and seat numbers owned by other modules; the access path and the audit of those decryptions are not defined. Live location sent in the final window: stored or not, and for how long, is not stated (D-14 says it stops at handover). Recipient personal data (a non-user data principal under DPDP) has no stated retention.

**Recommended correction.** Define contract methods that return decrypted contact data only to Notifications, logged; do not persist location beyond relay (or keep it for a short, stated period); add recipient data to the PRIV-003 retention table (counsel).

**Related:** NOTIF-001, HAND-002, PRIV-003, NFR-PRIV-01, D-14.

---

### AUD-041 — Quote validity vs request lifetime is ambiguous in the PRD and unresolved

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | PRD Misalignment |
| Section | Architecture §8.3 Pricing; PRD PRICE-001, PRICE-003, BOOK-001, Appendix A (`CFG-QUOTE-TTL` 30 min, `CFG-REQUEST-TTL` 6 h) |
| Product-owner approval | **Yes** (decision log Q-05) |

**Problem.** A quote expires in 30 minutes; a request carrying it lives up to 6 hours and PRICE-003 says the request's quote is honoured at acceptance. The architecture persists `valid_until` but does not say whether accept rejects an expired quote, re-prices, or honours it.

**Recommended correction.** Product owner chooses; recommended reading: the quote's TTL limits how long it can be used to create a request; once attached to a request it is honoured until the request ends.

**Related:** PRICE-001, PRICE-003, BOOK-001, BOOK-002.

---

### AUD-042 — No recovery path after handover-code lockout

| Field | Value |
|---|---|
| Severity | MEDIUM |
| Category | Domain Design |
| Section | §7.4 ("entry locks at CFG-HANDOVER-CODE-MAX-ATTEMPTS") |

**Problem.** After lockout "ops is alerted" (HAND-004), but there is no command to unlock or regenerate the code, no owner, and no time budget. At a short halt the handover is lost.

**Recommended correction.** Add a sender-initiated "regenerate code" (re-authenticated) and an ops command with reason code, both audited, with a risk flag on repeated lockouts.

**Related:** HAND-004, HAND-006, SUP-002, BR-18.

---

### AUD-043 — Minor internal inconsistencies in module and schema lists

| Field | Value |
|---|---|
| Severity | LOW |
| Category | Documentation |
| Section | §6.1 Module map; AD-10; §11.1 Schemas and ownership |

**Problem.** §6.1 says "plus two the PRD uses that ARCH-10 does not list" but adds only Privacy (Support is merged into Operations); AD-10 says a "19th module". The schema list in §11.1 has no `travelers` schema though Travelers owns tables. Cash opt-in is a "traveler capability" in Travelers but also `Trip.cash_enabled` per trip in Routes (PRD §9.1 and TRIP-001), so ownership is split.

**Recommended correction.** Fix the text; add the `travelers` schema; decide whether cash opt-in is per traveler, per trip, or both.

**Related:** ARCH-10, AD-10, AD-17, TRIP-001, D-21.

---

### AUD-044 — PRD claims there are no companion files, but they exist

| Field | Value |
|---|---|
| Severity | LOW |
| Category | Documentation |
| Section | PRD header table ("Structure"); change log entry "separate companion files retired"; companion files' headers ("Companion to GOPARCEL_MASTER_PRD.md") |

**Problem.** Four separate PRD files still exist and describe themselves as companions; their wording already differs slightly from the appendices (OQ-01, OQ-20). If they drift, it is unclear which is authoritative.

**Recommended correction.** Mark the separate files as superseded copies or delete them (product-owner decision); the appendices stay authoritative.

**Related:** OQ-01.

---

### AUD-045 — Matching query details

| Field | Value |
|---|---|
| Severity | LOW |
| Category | Performance / Geospatial |
| Section | §8.2 Matching and search; §8.3 Pricing (AD-14) |

**Problem.** Matching needs each candidate traveler's dues (cash rules) computed by `SUM(postings)`, which grows with history; the delivery-window check uses scheduled arrival rather than the latest estimate; no spatial index or SRID/type (geometry vs geography) is specified for station and meeting-point points (proximity queries for ROUTE-002 and admin tools). `ST_DistanceSphere` is fine for bands but AD-14 is correctly flagged as changing PRICE-001.

**Recommended correction.** Keep a per-party dues balance maintained in the posting transaction and proven nightly (§8.5 already allows this); store points as `geography(Point,4326)` or `geometry(Point,4326)` with a GiST index; decide scheduled vs estimated times for matching.

**Related:** MATCH-001, NFR-PERF-02, ROUTE-001, ROUTE-002, PRICE-001, AD-14.

---

### AUD-046 — Outbox table plus pg-boss queue is two mechanisms for one job

| Field | Value |
|---|---|
| Severity | LOW |
| Category | Maintainability |
| Section | §7.6; §6.4 (`outbox`, `jobs`) |

**Problem.** Integration events go to an outbox table, then are dispatched via pg-boss, which is itself a Postgres table queue. Two tables, two retry policies, two dead-letter paths.

**Recommended correction (optional).** If pg-boss can enqueue in the business transaction (AUD-013), use it as the outbox; otherwise keep the outbox and document a single retry/dead-letter policy.

**Related:** AD-03, AD-13, NOTIF-002.

---

### AUD-047 — Webhook ingestion edge cases

| Field | Value |
|---|---|
| Severity | LOW |
| Category | Reliability / API |
| Section | §9.5 Provider webhooks; §9.4 |

**Problem.** Inserting `provider_event` and enqueuing processing are separate steps (no recovery if the enqueue fails, except the poll job); the response when the database is down (must be non-2xx so the provider retries) is not stated; webhooks may pass through the Next.js proxy, adding a hop and body-size limits.

**Recommended correction.** Insert event and outbox/job row in one transaction (or sweep unprocessed events); return 5xx when the event cannot be stored; route `/api/v1/webhooks/*` directly to the API.

**Related:** PAY-002, BR-16.

---

### AUD-048 — Test strategy misses failure-path and recovery tests

| Field | Value |
|---|---|
| Severity | LOW |
| Category | Maintainability |
| Section | §15 Testing strategy |

**Problem.** Missing: provider timeout/unknown-outcome injection, out-of-order webhook sequences, worker-outage catch-up ordering (AUD-005), pickup-vs-cancel and trip-cancel-vs-accept deadlocks (AUD-006), reconciliation scenarios, PITR restore drill as a scripted test, clock-skew between DB `now()` and app clock.

**Recommended correction.** Add these to the concurrency and contract suites; use the injectable clock in SQL by passing time as a parameter.

**Related:** NFR-MAINT-02, NFR-BACKUP-01, pilot readiness gate item 4.

---

### AUD-049 — Payout calculation has no hook for TDS or other withholdings

| Field | Value |
|---|---|
| Severity | LOW |
| Category | Payments |
| Section | §8.4 ("net = credits − dues") |

**Problem.** If the CA confirms TDS on payments to travelers (PRD §10.1), payout math and the chart of accounts change. The formula has no withholding step.

**Recommended correction.** Model payout as credits − dues − withholdings (zero until CA confirms), with a TDS payable account.

**Related:** PAY-006, PAY-007, OQ-20, T-6.

---

### AUD-050 — Strengths worth preserving (INFO)

Conditional-update transitions with version columns; one transaction per command; append-only ledger and audit log enforced by trigger and grants; deferred balance check; provider-event table with unique key; Stage A adapter registry that refuses live credentials; Supabase Data API disabled and least-privilege DB role; injectable clock; refusal to add Redis/Kafka/K8s without evidence; explicit flags on AD-10 and AD-14; recipient token hashed and redacted from logs. These should survive revision.

### AUD-051 — Scalability outlook (INFO)

Reasoned, not measured. See §6 of this report and the production readiness report. The design is adequate for the PRD's pilot sizing (5,000 users, 500 bookings/day). No change to the stack is recommended now.

---

## 5. Stress-scenario traces

Each trace follows the documented design only. "Gap" lists what the design does not cover; "Correction" points to findings.

### A. Two travelers accept the same shipment at the same time
- **Path:** `POST /bookings/{id}/accept` ×2 (different booking IDs, same shipment) → Booking.acceptBooking → one transaction each.
- **DB:** both re-check traveler (no lock); both `SELECT shipment FOR UPDATE` → T2 waits; T1 locks siblings (including T2's booking row), capacity, sets T1 booking ACCEPTED/CONFIRMED, siblings CANCELLED_BY_SYSTEM, shipment LOCKED, commits. T2 resumes, sees shipment LOCKED → `ALREADY_BOOKED`, rolls back.
- **Backstop:** partial unique index "one live booking per shipment".
- **Result:** correct for this case.
- **Gap:** T2's conditional update of its own booking would also fail (now CANCELLED_BY_SYSTEM) — fine. Different-shipment, same-capacity accepts serialise on `trip_capacity` — fine. Accept vs trip cancel is **not** safe (trip not locked) → AUD-007. Accept vs pickup lock order is reversed → AUD-006.

### B. Provider processes the payment but the API call times out
- **createOrder timeout:** attempt stays CREATED/unknown; retry creates a second order unless the provider order is created with a merchant reference = attempt ID. Two live orders can both be paid → second success on a CONFIRMED booking has no handler → AUD-003.
- **Refund timeout:** no idempotency key or unknown-outcome state → retry may double-refund → AUD-004.
- **Payout timeout:** key `payout:<id>` protects the provider call, but the ledger already says paid → AUD-009.
- **Correction:** stable references on every money call, UNKNOWN state, fetch-before-retry.

### C. Duplicate webhook
- Same `event_id`: second insert hits `UNIQUE (provider, event_id)` → 200, no work. Correct.
- Same payment, different event types or webhook + poll: both reach processing; safety depends on the attempt's conditional transition PENDING → SUCCEEDED and on refund creation happening only in the winning transaction, neither of which is stated → AUD-003.
- Out of order (failed after succeeded, or succeeded after failed): not handled → AUD-003.

### D. Payment succeeds at the provider but the local DB transaction fails
- **Webhook path:** event stored, 200 returned, worker processing fails → pg-boss retries → succeeds later. If the event insert itself fails (DB down), the doc does not say to return 5xx so the provider retries → AUD-047. Poll job eventually picks up PENDING attempts.
- **Window interaction:** if retries span the window end, the expiry job may win and the payment is refunded although made on time → AUD-005.
- **Consistency:** ledger posting for the capture happens in the processing transaction (implied) so no half state; but posting rules are undefined → AUD-002.

### E. Traveler cancels after accepting
- **Not designed** (AUD-008). Expected path: Booking.cancelByTraveler → lock shipment → booking → capacity → delivery; booking → CANCELLED_BY_TRAVELER; capacity released; delivery → CANCELLED; shipment LOCKED → OPEN; full refund (online) via Payments; reliability mark (Users); masked-call teardown; re-match notification.
- **If still ACCEPTED_AWAITING_PAYMENT:** PRD state machine has no CANCELLED_BY_TRAVELER from ACCEPTED_AWAITING_PAYMENT — a gap in both documents; a payment arriving afterwards must auto-refund → AUD-003.
- **Trip cancel with several bookings:** lock order across shipments not defined → AUD-006.

### F. Accept too close to departure (DEC-051)
- **Online:** guard `departure_from_pickup − now ≥ CFG-MIN-PAYMENT-HOLD` → `JOURNEY_TOO_SOON`, rollback. Correct as designed (parameter TBD, OQ-02).
- **Cash:** the architecture's own assumption says the booking cutoff applies, but the sequence applies **no** guard; only matching and request expiry enforce the cutoff → AUD-007.
- **Extension:** a support extension can push payment past the protected point → AUD-025.
- **Delay after accept:** a delay makes departure later (safe); an earlier-time edit is blocked only by TRIP-003 rules that the architecture does not implement → AUD-007.

### G. Traveler edits the trip after a match or after a request/confirmation
- **After request, before accept:** TRIP-003 allows all edits. Pending requests are not invalidated; accept does not re-check stops, times or window → booking on a changed trip → AUD-007.
- **After confirmation:** only delay/capacity-add allowed by PRD; architecture has no trip-edit design. Delay may violate the overlap constraint → AUD-021. Adding capacity must lock capacity rows under the lock order.
- **Correction:** system-cancel pending requests on material edits; full match re-check at accept.

### H. Parcel damaged, lost or not delivered
- Traveler/sender reports → delivery IN_TRANSIT → EXCEPTION (Courier) → dispute (Disputes) → auto payout hold (BR-17).
- Decision traveler-at-fault → compensation to sender up to ₹5,000 from reserve → **no executable payment path** (AUD-001) → recovery debit to traveler dues (posting rules undefined, AUD-002).
- Not delivered with traveler unreachable → DISP-005 custody resolution; "no state forced without a verified handover" — consistent.
- Lost-claim timing is after delivery deadline + grace; payout for a non-delivered parcel never becomes eligible (correct).
- Booking outcome for LOST is undefined → AUD-024.

### I. Traveler submits false proof of delivery
- Cannot mark DELIVERED without the delivery code (good). Attack: obtain code by phone or from a leaked link, enter it without handing over → DELIVERED, dispute window starts, payout after 48 h unless the recipient disputes → AUD-026, AUD-027.
- Fake photos: API accepts any uploaded image; camera-only is client-side → AUD-027.
- Mitigation exists only through disputes and fault table; recommend plausibility checks and recipient confirmation messaging.

### J. Geospatial or notification provider unavailable
- **Geospatial:** PostGIS is in-database; no external geo provider at runtime (maps INT-07 only for admin). Distance bands from `city_pair_distance` table (AD-14) — no outage risk. Correct.
- **WhatsApp down:** OTP falls back to SMS synchronously; notifications retry then switch channel then dead-letter (§8.7). Correct.
- **Both channels down:** login impossible (R-08); recipient never gets the link → handover fails; no third channel (e.g. on-screen for logged-in users only). Accepted risk, should be explicit in runbooks.
- **Train-status down:** ETA hidden; traveler taps. Correct.

### K. Sender submits the same shipment several times
- Same intent, same key: idempotency returns one draft/publish result. Correct if AUD-018 semantics are fixed.
- Different keys (two tabs): two drafts/shipments — allowed by design; each can be booked with a different traveler for the same physical parcel. No duplicate-detection rule exists (product question; low risk). Requests: 3-request limit not race-safe → AUD-019.

### L. Provider records do not match the ledger
- Daily reconciliation → ReconciliationItem + reconciliation hold → payout blocked (BR-17). Correct shape.
- Gaps: fees will mismatch every payment (no fee account, AUD-002); settlement timing false positives; orphan captures; no matching key on journals; no post-restore replay → AUD-031. Hold placement crosses layers → AUD-014.

---

## 6. Scalability reasoning (theoretical, not demonstrated)

| Users | Approx. load (assumption: 1 booking per 10 users per day at the top end) | First bottleneck | Action |
|---|---|---|---|
| 1k | <100 bookings/day | None technical. Ops review queues (KYC, tickets) are the limit. | None |
| 10k | ~1k bookings/day, ~15k notifications/day | Manual ticket review per trip (TRIP-002) and KYC review — people, not machines. Single worker outage risk (AUD-005). | Two workers; OCR auto-approval thresholds |
| 100k | ~10k bookings/day, ~150k notifications/day, OTP bursts | DB connection limits through the pooler; rate-limit and idempotency writes; `SUM(postings)` for dues in matching; pg-boss/outbox table churn and autovacuum; Supabase compute size. | Maintained balances; edge rate limits; vertical DB scale; split worker queues (already in scaling path) |
| 1M | ~100k bookings/day | Database CPU/IO on one primary; matching read load; notification throughput and provider rate limits. | Read replica for matching and admin reports; consider Redis for rate limits **only if** metrics show it; module extraction only with evidence (ARCH-08) |

No measurement exists. The NFR-PERF targets should be load-tested before the pilot as §15 already plans.

---

**End of audit report.**
