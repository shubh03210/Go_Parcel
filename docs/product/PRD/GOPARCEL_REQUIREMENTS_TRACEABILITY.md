# GoParcel — Requirements Traceability

> **SUPERSEDED (22 Nov 2026, PD-20).** This file is kept for history only and is no longer maintained. The authoritative version is the matching appendix of `GOPARCEL_MASTER_PRD.md` (decision register: Appendix C; open questions: Appendix D; traceability: Appendix E; audit: Appendix F). Decisions PD-01..PD-20 and the resolved open questions appear only there.

Version 2.0 draft · 22 November 2026 · Companion to `GOPARCEL_MASTER_PRD.md`

Acceptance criteria live with each feature in Section 4 of the master PRD. Journeys: S = sender (§5.2), T = traveler (§5.3), R = recipient (§5.4), O = admin/ops (§5.5). Phase: A = staff alpha, P = pilot/MVP, 2/3 = later.

## 1. Requirement → source → decision

| Req ID | Feature | PRD v1.0 source | Decisions | Journey | Phase | Status |
|---|---|---|---|---|---|---|
| AUTH-001 | Request OTP | FR-AUTH-01/02 | D-03 | S, T | A, P | Confirmed |
| AUTH-002 | Verify OTP / create account | FR-AUTH-03/04/05 | D-03 | S, T | A, P | Confirmed |
| AUTH-003 | Sessions, logout | FR-AUTH-05/06 | — | S, T | A, P | Confirmed |
| AUTH-004 | Recovery, phone change | FR-AUTH-07 | D-03 | S, T | P | Proposed |
| AUTH-005 | Restriction, suspension | FR-AUTH-08, FR-ADM-05 | — | O | A, P | Confirmed |
| AUTH-006 | Account deletion | FR-AUTH-09 | — | S, T | P | Confirmed |
| USER-001 | Profile | — | D-01 | S, T | A, P | Confirmed / Recommended fields |
| USER-002 | Notification consent | FR-PRV-01 | D-03, D-13 | S, T | A, P | Confirmed |
| USER-003 | Role tabs | — | D-01 | S, T | A, P | Recommended |
| KYC-001 | Identity verification | FR-VER-01..04 | D-01, D-02 | S, T | A, P | Confirmed; method Proposed |
| KYC-002 | Manual KYC review | FR-VER-05/06 | — | O | A, P | Confirmed |
| KYC-003 | Re-verification | — | — | T | P | Recommended |
| KYC-004 | Payout beneficiary | FR-PAY-06 | D-18 | T | P | Confirmed |
| KYC-005 | Car documents | — | D-19 | T | P | Confirmed |
| TRIP-001 | Create trip | FR-JRN-01..04 | D-06, D-06a, D-08, D-09, D-19 | T | A, P | Confirmed |
| TRIP-002 | Ticket verification | — | D-08 | T, O | A, P | Proposed |
| TRIP-003 | Edit trip | FR-JRN-05 | — | T | A, P | Recommended |
| TRIP-004 | Cancel trip | — | Policies §9 | T | A, P | Confirmed |
| TRIP-005 | Delay update | — | Policies §9 | T | A, P | Confirmed |
| TRIP-006 | Trip lifecycle | FR-JRN-05/07/08 | BR-11 | T | A, P | Confirmed |
| ROUTE-001 | Station/city master | — | D-09 | O | A, P | Recommended |
| ROUTE-002 | Meeting points | — | D-19 | O, T | P | Confirmed |
| SHIP-001 | Create parcel | FR-SHP-01..03 | D-06, D-20, RV-01 | S | A, P | Confirmed; RV-01 Proposed |
| SHIP-002 | Eligibility, declaration | FR-SHP-04 | D-20 | S | A, P | Confirmed; list Proposed |
| SHIP-003 | Publish | FR-SHP-05 | D-02 | S | A, P | Confirmed |
| SHIP-004 | Withdraw/edit | FR-SHP-06/07 | — | S | A, P | Confirmed |
| SHIP-005 | Expiry | — | — | S | A, P | Recommended |
| SHIP-006 | Lifecycle | FR-SHP-08 | BR-04 | — | A, P | Recommended |
| MATCH-001 | Find matches | FR-MCH-01..07 | D-09 | S | A, P | Confirmed |
| MATCH-002 | Ranking | — | ARCH-12 | S | A, P | Recommended |
| MATCH-003 | No match | — | — | S | A, P | Recommended |
| MATCH-004 | Re-match | — | — | S | A, P | Recommended |
| MATCH-005 | Trip alerts | — | — | S, T | 2 | Deferred |
| PRICE-001 | Quote | FR-PAY-01 | D-04 | S | A, P | Confirmed |
| PRICE-002 | Commission, GST | — | D-22 | T | A, P | Confirmed |
| PRICE-003 | Display, lock | — | D-04 | S, T | A, P | Confirmed |
| BOOK-001 | Request | FR-BKG-01 | D-05 | S | A, P | Confirmed |
| BOOK-002 | Accept/reject | FR-BKG-02/08 | D-05, DEC-051 | T | A, P | Confirmed |
| BOOK-003 | Confirmation | FR-BKG-04/05 | D-11, D-11a | S, T | A, P | Confirmed |
| BOOK-004 | Expiry | — | — | S | A, P | Confirmed |
| BOOK-005 | Sender cancel | FR-BKG-06 | Policies §9, RV-02 | S | A, P | Confirmed; amounts TBD |
| BOOK-006 | Traveler cancel | FR-BKG-06 | Policies §9 | T | A, P | Confirmed |
| BOOK-007 | Concurrency | FR-BKG-07/08 | BR-07, BR-16 | — | A, P | Confirmed |
| PAY-001 | Online checkout | FR-PAY-02/04 | D-11 | S | A, P | Confirmed |
| PAY-002 | Event verification | FR-PAY-03/08 | — | — | A, P | Confirmed |
| PAY-003 | Cash at pickup | — | D-11a | S, T | A, P | Confirmed; safeguards Proposed |
| PAY-004 | Cash on delivery | — | D-11a, RV-02 | R, T | A, P | Confirmed |
| PAY-005 | Refunds | FR-PAY-05 | — | S | A, P | Confirmed |
| PAY-006 | Payouts, netting | FR-PAY-06 | D-18, D-21, RV-04 | T | P | Confirmed; RV-04 Proposed |
| PAY-007 | Commission, invoices | — | D-22 | O | P | Confirmed; invoicing TBD |
| PAY-008 | Reconciliation | FR-PAY-07 | — | O | P | Confirmed |
| PAY-009 | Dues throttle/block | — | D-21 | T | P | Confirmed |
| PAY-010 | Sender dues | — | Policies §6, §8 | S | P | Confirmed |
| LEDGER-001 | Party ledger | (new) | D-21, RV-02 | T, O | A, P | Confirmed |
| HAND-001 | Reveal location | FR-PKP-01 | D-07, D-19 | S, T, R | A, P | Confirmed |
| HAND-002 | Approach info | — | D-14 | S, R | A, P | Confirmed |
| HAND-003 | Packing evidence | FR-PKP-04 | D-12, D-16, PQ-1 | S | A, P | Confirmed |
| HAND-004 | Pickup verification | FR-PKP-02/03/05/06 | D-12, D-16, RV-03 | S, T | A, P | Confirmed; RV-03 Proposed |
| HAND-005 | In transit | FR-DLV-02 | — | S, R | A, P | Confirmed |
| HAND-006 | Delivery verification | FR-DLV-03/04/05 | D-16 | R, T | A, P | Confirmed |
| HAND-007 | Evidence integrity | FR-DLV-08 | — | O | A, P | Confirmed |
| HAND-008 | Missed delivery | FR-DLV-06 | D-15, DEC-052 | R, T | P | Proposed / Provisional |
| HAND-009 | Sender missed pickup | FR-PKP-07 | Policies §8 | S, T | P | Confirmed |
| HAND-010 | Return to sender | FR-DLV-07 | Policies §8 | S, T | P | Confirmed |
| HAND-011 | Abandonment | — | Policies §8 | O | P | TBD (counsel) |
| HAND-012 | Lost/damaged | FR-DLV-07 | D-17 | S, T | P | Confirmed |
| HAND-013 | Overdue escalation | FR-JRN-08 | BR-11 | O | A, P | Confirmed |
| TRACK-001 | Status timeline | — | D-14 | S, T, R | A, P | Confirmed |
| TRACK-002 | Event history | FR-SHP-08, BR-19 | — | O | A, P | Confirmed |
| NOTIF-001 | Event notifications | FR-NTF-01/02 | D-03, D-13 | All | A, P | Confirmed |
| NOTIF-002 | Retry, dead letters | FR-NTF-03/04, FR-JOB-01..04 | — | — | A, P | Confirmed |
| NOTIF-003 | Masked calling | — | D-13 | S, T, R | P | Confirmed |
| NOTIF-004 | Template messages | WF-11 | — | S, T, R | A, P | Confirmed |
| DISP-001 | Raise claim | FR-ADM-01/02 | D-17 | S, T, R | A, P | Confirmed |
| DISP-002 | Decide | — | D-17 | O | P | Confirmed |
| DISP-003 | Holds | FR-ADM-03 | BR-17 | O | P | Confirmed |
| DISP-004 | Appeal | FR-ADM-06 | DEC-057 | S, T, O | P | Provisional |
| DISP-005 | Custody resolution | WF-1, WF-10 | — | O | P | Confirmed |
| RATE-001 | Mutual ratings | — | Policies §11 | S, T | P | Proposed |
| RATE-002 | Reliability record | — | Policies §8, §9 | T | P | Confirmed |
| FRAUD-001 | Risk rules | WF-10 | — | O | P | Recommended |
| FRAUD-002 | Manual review | FR-ADM-03/05 | — | O | P | Confirmed |
| SUP-001 | Support cases | FR-SUP-01..04 | — | All | A, P | Confirmed |
| SUP-002 | Controlled intervention | FR-SUP-05/06 | BR-18 | O | P | Confirmed |
| ADMIN-001..010 | Admin console | FR-ADM-04/07/08, v1.0 §12.4 | BR-18, BR-19 | O | A, P | Confirmed / Recommended |
| PRIV-001 | Consent, notices | FR-PRV-01 | — | All | A, P | Confirmed |
| PRIV-002 | Data rights requests | FR-PRV-02/03 | — | S, T | P | Confirmed |
| PRIV-003 | Retention, legal hold | FR-PRV-04/05 | — | O | P | Confirmed; periods TBD |
| INC-001 | Incidents | FR-INC-01..03 | — | O | P | Confirmed |
| NFR-OBS-03 | Analytics events | v1.0 §14 metrics | — | — | A, P | Recommended (review 22 Nov) |

## 2. PRD v1.0 requirement coverage

Every PRD v1.0 functional requirement is carried forward; none was removed.

| PRD v1.0 | Now covered by |
|---|---|
| FR-AUTH-01..10 | AUTH-001..006, TRACK-002 (audit) |
| FR-VER-01..06 | KYC-001..002 |
| FR-SHP-01..08 | SHIP-001..006 |
| FR-JRN-01..08 | TRIP-001..006 |
| FR-MCH-01..07 | MATCH-001..002 |
| FR-BKG-01..08 | BOOK-001..007 |
| FR-PKP-01..07 | HAND-001, HAND-003, HAND-004, HAND-009 |
| FR-DLV-01..08 | HAND-005..008, HAND-010, HAND-012, HAND-007 |
| FR-PAY-01..08 | PRICE-001, PAY-001..008 |
| FR-ADM-01..08 | DISP-001..004, AUTH-005, ADMIN-001..010 |
| FR-NTF-01..04, FR-JOB-01..04 | NOTIF-001..002 |
| FR-PRV-01..05, FR-INC-01..03 | PRIV-001..003, INC-001 |
| FR-SUP-01..06 | SUP-001..002 |
| BR-01..BR-20 | Master PRD §6.1 BR-01..BR-20 (BR-08 amended by D-11a) |

## 3. Decision → requirements

| Decision | Requirements |
|---|---|
| D-01 | USER-001, USER-003, KYC-001, BR-20 |
| D-02 | KYC-001, SHIP-003, TRIP-001, BR-21 |
| D-03 | AUTH-001, AUTH-002, AUTH-004, USER-002, NOTIF-001 |
| D-04 | PRICE-001, PRICE-003, BR-23 |
| D-05 | BOOK-001, BOOK-002, BR-25 |
| D-06 / D-06a | TRIP-001, SHIP-001, MATCH-001, BR-37 |
| D-07 | TRIP-001, HAND-001, BR-27 |
| D-08 | TRIP-001, TRIP-002 |
| D-09 | MATCH-001, ROUTE-001 |
| D-10 | §5.1, NFR-COMPAT-01 |
| D-11 / D-11a | BOOK-003, PAY-001..004, BR-08, BR-31, BR-36 |
| D-12 | HAND-003, HAND-004, BR-10, BR-28 |
| D-13 | NOTIF-003, BR-27 |
| D-14 | HAND-002, TRACK-001 |
| D-15 | HAND-008 |
| D-16 | HAND-003, HAND-004, HAND-006, BR-29 |
| D-17 | DISP-002, HAND-012, BR-34 |
| D-18 | PAY-006, BR-35 |
| D-19 | KYC-005, TRIP-001, ROUTE-002, HAND-002, BR-30 |
| D-20 | SHIP-001, SHIP-002, BR-22 |
| D-21 | LEDGER-001, PAY-006, PAY-009, BR-32, BR-33 |
| D-22 | PRICE-002, PAY-007, BR-24 |
| DEC-051 | BOOK-002, BR-26 |
| DEC-052 | HAND-008, delivery state machine |
| DEC-057 | DISP-004 |
| RV-01 | SHIP-001 |
| RV-02 | BOOK-005, PAY-004, PAY-006, LEDGER-001, BR-38 |
| RV-03 | HAND-004 |
| RV-04 | PAY-006 |
