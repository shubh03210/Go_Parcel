# GoParcel — Architecture Remediation Plan

Audit date 22 November 2026. This plan proposes changes to the architecture document; it does not make them. Nothing here changes an approved decision; where a change touches one, "Approval required" says so.

**Priorities**
- **P0 — before development starts.** Affects the platform kernel (build step 1), the money model, or a contradiction that would be copied into every module.
- **P1 — before the affected feature is built** (build-sequence step named).
- **P2 — before Stage P (pilot).**
- **P3 — improvement; schedule when convenient.**

## Summary

| Priority | Items | Findings |
|---|---|---|
| P0 | 8 | AUD-001, 002, 006, 010, 013, 014/015/016, 018, 036 |
| P1 | 15 | AUD-003, 004, 005, 007, 008, 009, 012, 017, 019, 020, 021, 022, 024, 025, 033/034 |
| P2 | 14 | AUD-011, 023, 026, 027, 028, 029, 030, 031, 032, 035, 037, 038, 039, 040, 041, 042 |
| P3 | 6 | AUD-043, 044, 045, 046, 047, 048, 049 |

(Some rows group several findings.)

---

## P0 — before development

### P0-1 Define the end-to-end funds flow (AUD-001)
- **Original design:** §8.4 provider port (order, refund, split/transfer, beneficiary); "funds never sit in GoParcel's bank account"; §8.5 accounts without a GoParcel operating account.
- **Problem:** compensation (≤ ₹5,000), cash-booking refunds, dues collection and dues-funded payouts are impossible or unaccounted in this model.
- **Proposed design:** a "Funds flow" section listing every money movement with source/destination account, provider API, ledger posting and reconciliation source. Add `payoutToParty` to the port, sender beneficiary capture (reuse KYC-004 flow), and operating-bank / reserve-bank accounts. Add "payouts from merchant balance to arbitrary verified beneficiaries" to vendor criteria.
- **Reasoning:** money model drives vendor choice, legal posture, ledger and reconciliation; changing it after build is expensive.
- **Impact on other docs:** PRD PAY-005, PAY-006, DISP-002, BR-31 wording may need clarifying (e.g. "online **booking** funds are held by the provider; GoParcel's own funds pay compensation"); OQ-11, OQ-19, OQ-20 inputs.
- **Implementation implications:** new port method, new beneficiary type, additional accounts; Stage A sandbox adapter must simulate it.
- **Approval required:** **Product owner + counsel** (BR-31 interpretation).

### P0-2 Write the ledger posting rules and complete the chart (AUD-002)
- **Original:** §8.5 accounts and one cash example.
- **Problem:** no rules for online capture, pending earnings, commission recognition, refunds, cancellation fees/shares, payouts, compensation; missing fee/clearing/suspense/withholding accounts; BR-38 contingent credits have no data model.
- **Proposed:** a posting table per event; `contingent_credit` record for BR-38; accounts for gateway fees (+GST), payout clearing, refunds in transit, sender compensation payable, suspense, TDS payable (zero until CA).
- **Reasoning:** reconciliation and "every rupee traceable" depend on it.
- **Impact on other docs:** PRD LEDGER-001 table (add rows); T-6.
- **Implementation:** property tests (already planned) get concrete rules to test.
- **Approval required:** CA review; product owner for RV-02/BR-38 status.

### P0-3 Fix the lock order and flow orchestration (AUD-006, AUD-015)
- **Original:** §7.2 global order; §7.4 pickup via Courier → transactional event → Booking → Shipments.
- **Problem:** pickup and other flows contradict the order; trip, payment attempt, hold rows missing.
- **Proposed:** Booking orchestrates pickup, confirmation, cancellation and expiry: lock shipment → booking(s) → trip (share) → capacity → delivery → payment attempt → payable/ledger rows; Courier exposes `verifyCode` inside the caller's transaction. One deadlock-retry wrapper.
- **Reasoning:** one rule that the code can actually follow.
- **Impact on other docs:** AD-12 and AD-13 text.
- **Implementation:** lock helper becomes "lock set by id in canonical order" API; concurrency suite adds pickup-vs-cancel, trip-cancel-vs-accept, expiry-vs-payment.
- **Approval required:** No (AD-12/13 are still Proposed).

### P0-4 Make handover codes displayable (AUD-010)
- **Original:** §10.2 HMAC-only codes.
- **Problem:** PRD requires showing codes later.
- **Proposed:** AES-GCM-encrypted code (AD-16) for display to the authorised party + HMAC for verification; attempt counter unchanged.
- **Impact on other docs:** none in PRD (still "never plain text").
- **Approval required:** No.

### P0-5 Prove the job/outbox kernel and connectivity (AUD-013, part of AUD-046)
- **Original:** §7.7 jobs "in the same transaction"; API on transaction pooler; pg-boss self-migrating; worker on direct connection.
- **Problem:** pg-boss is a separate client; DDL vs least privilege; IPv4/IPv6 and connection limits unverified.
- **Proposed:** store deadlines on aggregates (`payment_due_at`, `expires_at`, `window_ends_at`); sweepers query those columns; outbox is the only in-transaction write and the worker creates pg-boss jobs from it (or prove the pg-boss db adapter with a test); run pg-boss migrations in CD with migrations disabled at runtime; foundation spike verifying Supabase pooler modes, IPv4 path and connection budget.
- **Impact on other docs:** AD-03, AD-09, §14.2.
- **Approval required:** No.

### P0-6 Publish an explicit, consistent dependency model (AUD-014, AUD-016)
- **Original:** §6.3 strict layers contradicted by its own table; StateEvent ownership conflict.
- **Proposed:** an allowed-dependency list (acyclic, same-layer allowed where listed); Notifications and Tracking timeline as event consumers at the top; per-module `state_event` tables; per-module `anonymise` and hold ports; config storage in the kernel.
- **Impact:** AD-11 lint rule source; §11.2 entity placement.
- **Approval required:** No (AD-11 still Proposed); AD-10 unchanged.

### P0-7 Specify idempotency-record semantics (AUD-018)
- **Proposed:** record inserted in the business transaction; 2xx and deterministic 4xx stored; 5xx never stored; lease for any out-of-transaction use.
- **Approval required:** No.

### P0-8 Separate architecture approval from product decisions (AUD-036)
- **Original:** §0.2 approve-all; §1 assumed answers.
- **Proposed:** approval record states that OQ-02/05/06/07/09/21, D-15 and RV-01..04 are not decided by approving the architecture; ideally answer the six P1 OQs first (decision log lists them).
- **Approval required:** **Product owner.**

---

## P1 — before the affected feature

| # | Findings | Before build step (§16) | Original → Proposed | Approval |
|---|---|---|---|---|
| P1-1 | AUD-003 | 6 Payments | Success only handled after PAYMENT_EXPIRED → provider-truth status fetch on every event/poll; FAILED/EXPIRED → SUCCEEDED allowed; one rule "succeeded but booking not awaiting payment, or already paid → exactly one refund"; unique automatic refund per attempt; provider order expiry ≤ window | No (policy Q-07 optional) |
| P1-2 | AUD-004 | 6 Payments | Retries "for idempotent operations only" → stable references on every money call; UNKNOWN state; fetch-before-retry; refund state machine | No |
| P1-3 | AUD-005 | 5–6 | Single worker, plain expiry → check-before-expire; two workers; queue-age alert | No |
| P1-4 | AUD-007, AUD-041 | 5 Booking | Partial re-check → `assertStillMatches` at request and accept; cash cutoff guard; trip lock; edits system-cancel pending requests; quote rule | **Yes** for edit rule and quote rule (Q-05, Q-06) |
| P1-5 | AUD-008 | 5 Booking / 6 Payments | Absent → cancellation design table incl. mismatch-decline review record and traveler cancel before payment | No for design; amounts stay TBD (OQ-10/16); PRD state machine gap (Q-10) |
| P1-6 | AUD-009 | 9 Payouts | Post-then-call, per-record key → payable items + payout batches, locks, clearing, reversal, singleton | **Yes** for per-delivery vs aggregated (Q-09) |
| P1-7 | AUD-012 | 2 Identity / 3 KYC | Not designed → recovery and recycled-number flows; KYC reference image or provider face reference; phone/beneficiary change events place holds | **Yes**: threshold value, selfie on beneficiary change (Q-08); counsel for image retention |
| P1-8 | AUD-017 | 2 Identity / 5 Booking | `restrictAccount` only → suspension/restriction/re-KYC cascade events; user row lock in accept/request | No |
| P1-9 | AUD-019 | 5–6 | Missing constraints → shipment lock on request; unique cash confirmation, automatic refund, pending attempt, payable item; dues link locking | No |
| P1-10 | AUD-020 | 6–7 | Debit at cash tap → debit in pickup transaction, or reversal on decline/dispute | No |
| P1-11 | AUD-021 | 4 Trips | Overlap on live period → overlap on scheduled period only | No |
| P1-12 | AUD-022 | 4 Trips | Weight per class row → trip-level weight row | Segment reuse is a PO question (Q-11) |
| P1-13 | AUD-024 | 5–7 | No owner → terminal-transition handlers (booking COMPLETED, shipment CLOSED, delivery CANCELLED; outcomes for RETURNED/LOST) | **Yes** for EXCEPTION → DELIVERED and booking outcomes (Q-10) |
| P1-14 | AUD-025 | 5 / 10 Admin | Fixed job → `payment_due_at`; extension re-checks DEC-051 | No |
| P1-15 | AUD-033, AUD-034 | Each module | Conventions only → endpoint inventory and schema doc per module, starting with Booking, Payments, Ledger, Courier | No |

---

## P2 — before Stage P (pilot)

| # | Findings | Proposed change | Approval |
|---|---|---|---|
| P2-1 | AUD-011 | Decide and design offline/degraded handover | **Product owner** (Q-01) |
| P2-2 | AUD-023 | City/station matching semantics | **Product owner** (Q-03) |
| P2-3 | AUD-026 | Recipient page hardening; optional recipient OTP | Product owner for OTP (Q-04) |
| P2-4 | AUD-027 | Delivery plausibility checks; evidence treated as user-supplied; risk rules | No |
| P2-5 | AUD-028 | Evidence replication/versioning; integrity sweep; export design (T-9) | No |
| P2-6 | AUD-029 | Global OTP budget, edge limits, circuit breaker | No |
| P2-7 | AUD-030 | Admin subdomain, CSP, token storage and CSRF decision | No |
| P2-8 | AUD-031 | Reconciliation types, keys, tolerance, post-restore replay | Finance |
| P2-9 | AUD-032 | Threat model document (STRIDE per boundary + GoParcel threats) incl. compensation exposure caps | Product owner for caps (Q-12) |
| P2-10 | AUD-035 | Raw-SQL object registry and CI existence test; verify Prisma drift behaviour and multi-schema | No |
| P2-11 | AUD-037 | Provider-agnostic duplicate identity detection | Counsel |
| P2-12 | AUD-038 | KMS choice and key hierarchy (with AD-09) | No |
| P2-13 | AUD-039 | Pilot cost model | Product owner |
| P2-14 | AUD-040, AUD-042 | Notification data access; location and recipient retention; code-lockout recovery | Counsel for retention |

---

## P3 — improvements

| # | Findings | Proposed change |
|---|---|---|
| P3-1 | AUD-043 | Fix module count text; add `travelers` schema; decide cash opt-in ownership |
| P3-2 | AUD-044 | Mark PRD companion files as superseded copies (product owner) |
| P3-3 | AUD-045 | Maintained dues balance; SRID/type and GiST index; estimated vs scheduled times in matching |
| P3-4 | AUD-046 | Collapse outbox + pg-boss if P0-5 proves in-transaction enqueue |
| P3-5 | AUD-047 | Atomic event + job insert; 5xx when event cannot be stored; direct webhook routing |
| P3-6 | AUD-048, AUD-049 | Failure-path tests; withholding step in payout model |

---

## Exit criteria for the next readiness review

The architecture can be re-rated **CONDITIONALLY READY** when all P0 items are reflected in a v0.2 document and the six P1 open questions in PRD F.8 are answered or explicitly deferred by the product owner. **DEVELOPMENT READY** additionally needs the P1 items for build steps 1–5 (foundation through Booking) and no open CRITICAL finding.
