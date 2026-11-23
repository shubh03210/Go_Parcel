# GoParcel — Architecture Production Readiness Report V2

Re-audit date 22 November 2026. Subject: `docs/architecture/GOPARCEL_ARCHITECTURE.md` v0.2. Previous verdict (v0.1): **NOT READY**.

> **Scope note.** This report rates whether the **architecture** is ready for development to begin. It is **not** a launch- or pilot-readiness assessment (PRD §12.5 gate), not a security certification or penetration test, and **not legal, tax, regulatory or payment-compliance advice**. DPDP, IT Act, GST/TDS, payment-aggregator rules, Aadhaar handling and the funds-flow model still need counsel and a CA (OQ-20, T-1, T-6, T-12). No code exists and nothing was tested. Ratings are reasoned from documents. Vendor and library behaviour was not checked online in this round.

## 1. Overall verdict

# CONDITIONALLY READY

**What this means:**
- **May start now:** the foundation spike (§14.1). It is investigation, not module code, and it does not depend on the open findings.
- **May start after small text fixes to v0.2** (no new product decisions needed): the platform kernel and the Identity, Users, Routes master data, Pricing, KYC stub and Travelers steps (build steps 1–3). The fixes needed first are AUD2-004 (dependency list, the CI lint source), AUD2-011 (sweeper claim pattern) and AUD2-016 (idempotency wording).
- **Not before the named fixes:** Booking, Payments/Ledger, Courier and Payouts (build steps 5–9). See §4.

**Why not DEVELOPMENT READY:**
- Four HIGH findings are open (AUD2-001..004).
- One of them, AUD2-004, is a P0-class item from round 1 (AUD-014 / P0-6): the allowed-dependency list is the source of the CI rule that build step 1 creates.
- Two kernel-level items are open (AUD2-011, AUD2-016).
- CA review of the posting rules and counsel sign-off on the funds-flow model (PD-07) are still pending.

**Why no longer NOT READY:**
- No CRITICAL finding is open. AUD-001 is answered structurally.
- All eight round-1 P0 items are reflected in v0.2, though P0-6 and P0-7 only partly.
- The six P1 open questions the round-1 exit criteria required are answered by PD-01..PD-20.
- The remaining defects are text-level corrections inside a sound design, not a re-architecture.

## 2. Finding totals

**Round-1 defects (AUD-001..049):** Fixed 33 · Partially fixed 15 · Not fixed 0 · No longer applicable 1.

**New findings (AUD2-001..025):**

| Severity | Count | IDs |
|---|---|---|
| CRITICAL | 0 | — |
| HIGH | 4 | AUD2-001 custody cap ignores bookings awaiting payment; AUD2-002 composite payout under one reference/state; AUD2-003 offline queue vs cancellation, memory-only, no server state; AUD2-004 dependency list contradicted |
| MEDIUM | 11 | AUD2-005 … AUD2-015 |
| LOW | 8 | AUD2-016 … AUD2-023 |
| INFO | 2 | AUD2-024, AUD2-025 |

## 3. Area ratings

Scale: **Strong** (ready) · **Adequate** (ready with minor fixes) · **Weak** (material gaps; fix before the affected build step) · **Not ready** (blocks development).

| Area | v0.1 | v0.2 | Summary | Key open findings |
|---|---|---|---|---|
| PRD alignment | Weak | **Adequate** | PD-01..PD-20 are applied and traceable (§1, §0.3). Four places weaken or quietly change a decision: PD-08 (max-defer expiry), PD-09 (two provider operations), PD-10 (cap excludes AAP), PD-11 (pending state only on the client). One rule appears that is not in the PRD (sender payouts never net dues). | AUD2-001, 002, 003, 010, 015, 020 |
| Domain / module design | Weak | **Weak** | Booking-as-orchestrator and no in-transaction events are the right model. But the exhaustive dependency list breaks against twelve described flows, the booking-completion owner is inconsistent, and the restriction cascade conflicts with the state table. | AUD2-004, 005, 006 |
| Database | Adequate | **Adequate** | Good invariants, canonical lock order, raw-SQL registry, trip-level weight, scheduled-period overlap. Gaps: the sweeper claim pattern, an incomplete lock table, cross-module links with no foreign key. | AUD2-011, 017, 022 |
| Payments / financial | Not ready | **Weak** | Large improvement: funds flow, chart, balanced posting rules, provider truth, UNKNOWN states, payable items, clearing account, reconciliation. Before Payments is built, fix composite payouts (double-pay path), escrow residuals, dues overpayment, the posting gaps, lost/returned outcomes and the PD-08 deviation. CA and counsel pending. | AUD2-002, 007, 008, 009, 010, 014, 015 |
| Matching and pricing | Adequate | **Adequate** | City-level matching (PD-14), whole-trip capacity, re-check at accept, PD-15 quote rule, cached dues. The custody-headroom view cannot be built by Travelers. AD-14 still needs product-owner approval (T-5). | AUD2-001, 004 |
| Security | Weak | **Adequate** | Recipient OTP, encrypted codes, recovery with selfie re-match, cooling periods, separate admin host, KMS hierarchy, OTP budgets, threat-model starter. Gaps: PD-10 bypass, recipient token handling, no correction path after a false delivery, full STRIDE before Stage P. | AUD2-001, 012, 013, 021, 023 |
| API | Weak | **Adequate** | Core inventory, idempotency inside the transaction, problem+json, header token plus Strict cookies. Small wording conflict on stored 4xx responses. Per-module tables come at P1 and some endpoints are missing from the core list (ratings, deletion, data export). | AUD2-016; AUD-033 |
| Infrastructure | Adequate / Weak | **Adequate** (development) | Two workers, worker-only pg-boss, migrations in CD, spike with a connection budget, KMS, replica store, cost structure. The sweeper pattern must be fixed in the kernel. Spike results are pending. | AUD2-011, 025 |
| Scalability | Adequate | **Adequate** | Unchanged: comfortable at PRD pilot sizing (5,000 users, 500 bookings/day); cached balances remove the `SUM(postings)` hotspot. Reasoned, not measured. | — |
| Operational readiness | Weak | **Weak** | Better alerts, restore and outage drills, essential admin console. Offline handover is not operable as designed (lost queue entries, cancellation conflicts). No ageing alerts for UNKNOWN money states. A dual messaging outage blocks all deliveries. Payout failure messaging can be wrong. | AUD2-003, 014, 018 |

## 4. What remains before development (by build step, §16)

| Gate | Must be done | Owner |
|---|---|---|
| **Before kernel code (step 1, after the spike)** | AUD2-004 dependency list corrected and re-checked acyclic. AUD2-011 sweeper lease pattern. AUD2-016 idempotency 4xx wording. Spike results for every [VERIFY] (T-7). | Architect |
| **Before Booking (step 5)** | AUD2-001 custody cap counts AAP. AUD2-006 AAP → CANCELLED_BY_SYSTEM and cascade orchestration. AUD2-005 booking-completion owner. AUD2-017 lock table complete. Q2-02, Q2-08 answered. | Architect; product owner |
| **Before Payments and Ledger (step 6)** | AUD2-007, 008, 009, 010, 014, 015 fixed in the posting table and §8.5. T-15 decided for online, cash and COD (Q2-07). Q2-03 answered. CA review of the extended posting table (T-6). Counsel view on the funds-flow model, or the product owner explicitly accepts building Stage A on it pending counsel (PD-07, T-1). | Architect; product owner; CA; counsel |
| **Before Courier (step 7)** | AUD2-003 offline queue rules (Q2-01). AUD2-012 recipient token issuance. AUD2-013 false-delivery correction policy (Q2-05). | Architect; product owner |
| **Before Payouts (step 9)** | AUD2-002 payout legs (Q2-04 plus a vendor question). AUD2-020 sender payout netting (Q2-06). | Architect; product owner; vendor |
| **Before Stage P** | AUD2-018 messaging-outage fallback. AUD2-021, 022, 023. Full STRIDE threat model (T-13). Vendor choices (OQ-19). Config values (T-11). | Architect; product owner; ops |

**Re-rating rule for the next review:** DEVELOPMENT READY when (a) no CRITICAL or HIGH finding is open, (b) AUD2-004, 011 and 016 are closed, (c) the Q2 questions for steps 1–5 are answered, and (d) spike results confirm or replace every [VERIFY] used by the kernel. Money-module approval (step 6 onwards) additionally needs the CA review and counsel's view, or explicit product-owner acceptance of the Stage A-only risk.

## 5. What this report does not cover

- Launch or pilot readiness (PRD §12.5 gate items: DLT registration, vendor contracts, support staffing, restore drill results, pen test).
- Legal, tax or regulatory compliance of any kind, including whether GoParcel's handling of customer money needs a payment-aggregator licence or is acceptable under BR-31 / PD-07.
- Vendor due diligence or capability confirmation.
- Security testing of any implementation.
- Performance measurement.

These remain open in the PRD and the architecture's §18 and must be handled separately.
