# GoParcel — PRD Audit Report

> **SUPERSEDED (22 Nov 2026, PD-20).** This file is kept for history only and is no longer maintained. The authoritative version is the matching appendix of `GOPARCEL_MASTER_PRD.md` (decision register: Appendix C; open questions: Appendix D; traceability: Appendix E; audit: Appendix F). Decisions PD-01..PD-20 and the resolved open questions appear only there.

Version 2.0 draft · 22 November 2026 · Independent-style review of `GOPARCEL_MASTER_PRD.md`

This audit was written as if by a reviewer who did not author the PRD. It does **not** claim the PRD is error-free, legally cleared or complete against documents the author could not see.

## 1. Sources reviewed

PRD v1.0; the PRD review conversation (D-01 to D-22); Marketplace Policies Draft v0.1; master prompt (ARCH context, DEC-051/052/057); the local `Go_Parcel` repository (contains only `README.md`).

**Not available:** F1–F9, ADRs, full DEC register, module specs, the 98-table design. Rule 7 cross-document verification is therefore partial (OQ-01).

## 2. Decisions correctly preserved

All 24 product decisions (D-01..D-22, D-06a, D-11a), PC-01..03, ARCH-01..12, DEC-051 (as confirmed rule), PQ-1 and PRD v1.0 BR-01..20 appear in the PRD with matching status. DEC-052 and DEC-057 are marked Provisional. D-15 is marked Proposed. Review recommendations RV-01..04 are marked Proposed, not Confirmed.

## 3. Conflicts found and how they are handled

| # | Conflict | Handling in PRD |
|---|---|---|
| 1 | Master prompt lists password recovery and email; D-03 is OTP-only | OTP-only kept; confirmation requested (OQ-05) |
| 2 | Master prompt omits recipient role; PRD v1.0 has scoped recipient | Scoped recipient kept (OQ-06) |
| 3 | Master prompt's "pickup locations" implies addresses; D-07/D-19 seat and meeting points | D-07/D-19 kept |
| 4 | Master prompt's "authorisation/holds/capture"; D-11 provider-held prepaid | D-11 kept; DEC-051 "hold" meaning asked (OQ-02) |
| 5 | Master prompt's "route overlap" matching; D-09 stop-list matching | D-09 kept; PostGIS used for distance bands and coordinates |
| 6 | PRD v1.0 "no ledger / no wallet"; D-21 ledger | D-21 supersedes; recorded in decision register |
| 7 | PRD v1.0 separate G and M gates; PC-03 merges them | Merged; Stage A retained pending OQ-08 |
| 8 | PRD v1.0 "no phone sharing / templates only"; D-13 masked calling | D-13 supersedes; templates retained |
| 9 | Policies "full refund minus platform fee"; commission is borne by traveler | Replaced with named sender cancellation fee (OQ-10) |

## 4. Review of 22 November 2026: findings fixed

| # | Finding | Fix | Approval |
|---|---|---|---|
| F-1 | Four config keys referenced in features were only implied by combined Appendix A rows (`CFG-SESSION-MAX`, `CFG-DELIVERY-OVERDUE-GRACE`, `CFG-RELIABILITY-WINDOW-DAYS`, `CFG-MISSED-FEE-TRAVELER-SHARE`) | Split into explicit rows | Editorial |
| F-2 | SHIP-001 blocked the recipient number from equalling the sender's, blocking "send to myself" | Allowed; consent checkbox hidden when recipient is sender | RV-01 (OQ-21) |
| F-3 | Cash bookings: traveler compensation credited before GoParcel collected the sender's dues, exposing GoParcel | Credited only when the sender pays; new rule BR-38 | RV-02 (OQ-21) |
| F-4 | PAY-006 paid out only online earnings, leaving compensation credits unpaid | Payouts include all payable credits | RV-04 (OQ-21) |
| F-5 | HAND-004 had no outcome for a traveler declining over a mismatch | No-fee cancellation; ops decides fault | RV-03 (OQ-21) |
| F-6 | Section 11 KPIs had no instrumentation requirement | NFR-OBS-03 analytics events added | Recommended |

An automated check confirmed every requirement ID, integration ID and open-question reference in the PRD is defined.

## 5. Consistency checks performed

| Check | Result |
|---|---|
| Booking rules vs payment rules (BR-08 and D-11a cash exception) | Consistent |
| Matching vs eligibility (KYC, size per mode, dues throttle) | Consistent |
| Traveler eligibility vs KYC (one KYC for both roles) | Consistent |
| Cancellation vs refund (BOOK-005/006 vs PAY-005) | Consistent; amounts TBD |
| Delivery lifecycle vs tracking events | Consistent |
| Admin permissions vs authorisation (§3.5 vs ADMIN-001 and BR-18) | Consistent |
| Ledger entries vs cash, payouts, disputes, compensation | Consistent after F-3/F-4 |
| Payout timing vs dispute window | Consistent: proposed 48 h within confirmed 48–72 h |
| Phone/PNR exposure across features | Consistent: never exposed to users |

## 6. Issues still open

| # | Severity | Issue | Recommendation |
|---|---|---|---|
| 1 | High | DEC-051 applicability to cash bookings undefined | OQ-02 |
| 2 | High | HAND-008 (missed delivery) is MVP-critical but D-15 is still Proposed | OQ-09 |
| 3 | High | Core numbers missing: sizes, weight caps, price table, minimum commission, cancellation amounts | OQ-12, OQ-16 |
| 4 | High | Legal viability (car carriage, Aadhaar route, payment structure, abandoned parcels) unconfirmed | OQ-20 |
| 5 | Medium | Whether provider payouts can carry credits funded by sender dues collected by GoParcel | Counsel and provider capability (PAY-006) |
| 6 | Medium | Refund route for cash-paying senders without UPI/bank | OQ-11 |
| 7 | Medium | Booking states beyond v1.0's four, shipment and trip state machines are recommended, not verified against F3 | OQ-01 |
| 8 | Medium | Lock order for accept (PRD v1.0 DR-07) undecided | Technical design |
| 9 | Medium | Proposed capacity per mode not approved | OQ-12 |
| 10 | Medium | Compensation reserve size unknown | OQ-17 |
| 11 | Low | Hindi support proposed, not confirmed | Confirm |
| 12 | Low | NFR numbers are proposals | Validate in load testing |
| 13 | Low | PRD v1.0 open items UD-01 (TD-23 missing text), UD-02, UD-06, UD-36, UD-41, PA-15, DR-01..03 remain open | Track after OQ-01 |

## 7. Requirements that could not be verified

Anything depending on F1–F9, ADRs and the full DEC register: exact state names in F3; the 98-table schema; module boundaries in F4; threat model and release gates in F5; compliance research in F6; scalability targets in F7; backlog in F8. If these documents do not exist, this PRD becomes the baseline and these items are simply design work for the next phase.

## 8. Overall assessment

The PRD is complete enough to start **architecture and technical design** once OQ-02, OQ-05, OQ-06, OQ-07, OQ-09 and OQ-21 are answered. It is **not** ready for the controlled pilot until numeric values (OQ-12, OQ-16, OQ-17), vendor choices (OQ-19) and counsel/CA review (OQ-20) are done. No development should begin until the product owner says "START DEVELOPMENT".
