# GoParcel — Architecture Production Readiness Report

Audit date 22 November 2026. Subject: `docs/architecture/GOPARCEL_ARCHITECTURE.md` v0.1.

> **Scope note.** This report rates whether the **architecture** is ready to start development. It is **not** a launch-readiness assessment, a security certification, or legal, tax or regulatory compliance advice. No code exists and nothing was tested; all ratings are reasoned from documents. Vendor and library capabilities were not verified online (web search unavailable).

## 1. Overall verdict

# NOT READY

**Why:** one CRITICAL finding (AUD-001, funds-flow model) and eight P0 items are open. Two of the P0 items (job/transaction kernel, lock order and orchestration) affect build step 1 and every transactional module, and the money model affects vendor selection and the ledger schema. Starting development now would bake those gaps into the foundation.

**How far from ready:** the architecture's overall shape is sound and should be kept. The gaps are design omissions and contradictions, not a wrong stack. A focused v0.2 that closes the P0 items in the remediation plan, plus product-owner answers to the six P1 open questions in PRD Appendix F.8, should move the verdict to **CONDITIONALLY READY** (foundation, identity and master-data work may start while P1 designs are completed per feature).

## 2. Finding totals

| Severity | Count | Of which in a P0 remediation item |
|---|---|---|
| CRITICAL | 1 | 1 (AUD-001) |
| HIGH | 12 | 4 (AUD-002, 006, 010, 013) |
| MEDIUM | 29 | 5 (AUD-014, 015, 016, 018, 036) |
| LOW | 7 | 0 |
| INFO | 2 | — |

## 3. Area assessments

Rating scale: **Strong** (ready) · **Adequate** (ready with minor fixes) · **Weak** (material gaps; fix before the affected build step) · **Not ready** (blocks development).

| Area | Rating | Summary | Key findings |
|---|---|---|---|
| PRD alignment | **Weak** | 16 of 98 Section 4 requirements fully covered, 60 partial, 18 missing, 4 incorrect. Confirmed decisions are respected in letter, but cancellation, recovery, suspension, trip edit and several admin/support features have no design. | AUD-007, 008, 012, 017; coverage audit |
| Domain / module design | **Weak** | Good module map and internal layout. The layer rule is contradicted by its own examples; transactional events hide synchronous cycles; terminal transitions lack owners. | AUD-014, 015, 016, 024 |
| Database | **Adequate** | Strong use of conditional updates, partial unique indexes, CHECKs, append-only triggers. Gaps: missing invariants, overlap constraint vs delays, weight model, Prisma's limits on raw-SQL objects, no physical schema. | AUD-019, 021, 022, 034, 035 |
| Payments / financial | **Not ready** | Happy path only. Funds flow not executable for compensation/cash refunds/dues-funded payouts; posting rules missing; payout run unsafe; out-of-order and late events unhandled; refunds not idempotent. | AUD-001, 002, 003, 004, 005, 009, 020, 031 |
| Matching and pricing | **Adequate** | Read-only SQL over published views is appropriate for pilot scale; pricing in integer paise with snapshots. Gaps: city/station semantics, accept-time re-check, quote TTL ambiguity; AD-14 correctly flagged. | AUD-007, 023, 041, 045 |
| Security | **Weak** | Solid basics (OTP design, refresh-family reuse detection, field encryption, masked reveals, audit log, Data API disabled). Missing: threat model, recovery/recycled-number design, recipient-link and false-delivery threats, KMS, same-origin admin, OTP pumping. | AUD-012, 026, 027, 029, 030, 032, 038 |
| API | **Weak** | Conventions are good (problem+json, explicit commands, cursor pagination, idempotency keys, no PII in URLs). No endpoint inventory, authorisation matrix per endpoint, error catalogue or rate-limit table. | AUD-018, 033 |
| Infrastructure | **Adequate** (for development) / **Weak** (for pilot) | Right-sized choices and Stage A guards. Unverified pooler/pg-boss/IPv4 behaviour, single worker, no KMS, no evidence backup, no cost estimate. | AUD-005, 013, 028, 038, 039 |
| Scalability | **Adequate** | Comfortable at PRD pilot sizing. First bottlenecks are operational (manual reviews) and, later, connections and ledger aggregation — all reasoned, none measured. No case for microservices, Kafka, Redis or Kubernetes now. | AUD-045, report §6 |
| Operational readiness | **Weak** | Good observability list and CI pipeline. Missing: offline handover decision, code-lockout recovery, worker HA, post-restore reconciliation, runbook inputs. | AUD-005, 011, 031, 042 |

## 4. What must be true before re-review

1. All P0 items in the remediation plan reflected in architecture v0.2.
2. OQ-02, OQ-05, OQ-06, OQ-07, OQ-09, OQ-21 answered or explicitly deferred (PRD Appendix F.8).
3. Counsel has seen the funds-flow model (P0-1), or the product owner explicitly accepts building Stage A on it pending counsel.
4. A foundation spike plan exists for P0-5 (pg-boss in transaction, pooler modes, IPv4, connection budget, Prisma multi-schema and drift behaviour).

## 5. What this report does not cover

Launch readiness, pilot readiness gate items (PRD §12.5), legal or tax compliance (DPDP, IT Act, GST, TDS, payment-aggregator rules, Aadhaar), vendor due diligence, and penetration testing. These remain open in the PRD and must be handled separately.
