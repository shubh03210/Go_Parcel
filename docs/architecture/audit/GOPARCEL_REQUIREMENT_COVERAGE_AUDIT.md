# GoParcel — Requirement Coverage Audit

Audit date 22 November 2026. For every PRD v2.0 Section 4 requirement, every business rule BR-01..BR-38 and every NFR, this table records which architecture component covers it and how well.

**Coverage key**
- **Full** — module, data ownership, workflow, failure handling and testability are all addressed (at architecture level).
- **Partial** — a module owns it and some design exists, but a workflow, failure path, data element or security aspect is missing.
- **Missing** — no design beyond a name in the module map.
- **Incorrect** — the design contradicts the requirement.

A similarly named module is **not** counted as coverage. Section references are to `GOPARCEL_ARCHITECTURE.md`.

## Summary

| Group | Items | Full | Partial | Missing | Incorrect |
|---|---|---|---|---|---|
| Section 4 requirements | 98 | 16 | 60 | 18 | 4 |
| Business rules BR-01..38 | 38 | 22 | 13 | 1 | 2 |
| NFRs | 27 | 11 | 15 | 1 | 0 |

## 1. Section 4 functional requirements

| PRD ID | Requirement | Architecture component | Coverage | Finding / note |
|---|---|---|---|---|
| AUTH-001 | Request login OTP | Identity §8.1, §8.7 (sync send + SMS fallback), §10.4 | Partial | Equal response and fallback designed; OTP pumping/global budget missing (AUD-029). |
| AUTH-002 | Verify OTP, create account | Identity §8.1 (atomic UPDATE … RETURNING) | Full | Account creation + consent record implied. |
| AUTH-003 | Sessions and logout | Identity §8.1 (refresh families, revocation cache) | Partial | Token storage/CSRF undecided (AUD-030); idle/absolute lifetimes not mapped to keys. |
| AUTH-004 | Recovery, phone change | — | Missing | No recovery/recycled-number flow; KYC photo not stored (AUD-012). |
| AUTH-005 | Restriction and suspension | Identity `restrictAccount` §8.10 | Partial | Cascade (withdraw, cancel, route to ops) missing (AUD-017). |
| AUTH-006 | Account deletion | §11.5 anonymisation | Partial | Blocker checks (dues, bookings, holds) and grace state not designed; cross-module anonymisation path (AUD-014). |
| USER-001 | Profile | Users §6.1 | Partial | Ownership only; display rules (first name + initial) not designed. |
| USER-002 | Notification preferences, consent | Users §6.1; Notifications §8.7 | Partial | Channel routing uses opt-in; consent versioning store not described. |
| USER-003 | Role dashboard switching | Web §13 | Full | UI-level. |
| KYC-001 | Identity verification | KYC §8.9 | Partial | Duplicate-identity method uncertain (AUD-037); resume-after-verify not designed. |
| KYC-002 | Manual KYC review | KYC + Operations queues | Partial | Queue ownership only; document image storage/retention unspecified. |
| KYC-003 | Re-verification | — | Missing | Triggers and blocking not designed (AUD-017). |
| KYC-004 | Payout beneficiary verification | Payments `validateBeneficiary` §8.4 | Partial | Beneficiary-change hold and selfie protection missing (AUD-012). |
| KYC-005 | Car documents | Travelers §6.1 | Partial | Insurance-expiry check at publish not designed. |
| TRIP-001 | Create trip | Routes §6.1; §11.3 overlap constraint; §8.6 ticket upload | Partial | Overlap constraint conflicts with delays (AUD-021); weight cap model (AUD-022). |
| TRIP-002 | Ticket verification | Worker OCR adapter §3 | Partial | Flow, name-match, queue not designed. |
| TRIP-003 | Edit a trip | — | Missing | No edit rules; pending requests not invalidated (AUD-007). |
| TRIP-004 | Cancel a trip | — | Missing | Multi-booking cancel and lock order missing (AUD-006, AUD-008). |
| TRIP-005 | Delay update | §8.8 train status; §7.7 | Partial | Free-cancel trigger and overlap interaction missing (AUD-021). |
| TRIP-006 | Trip lifecycle | §7.7 `routes.trip-progress` | Partial | COMPLETED only when all deliveries terminal — no handler (AUD-024). |
| ROUTE-001 | Station/city master list | Routes; §11.4 seeds; T-8 | Partial | SRID/type and spatial index unspecified (AUD-045); city grouping (AUD-023). |
| ROUTE-002 | Meeting points | Routes | Partial | Mapping to matching stops undefined (AUD-023). |
| SHIP-001 | Create parcel draft | Shipments | Partial | Fields owned; recipient data encryption listed (§10.2); draft retention job not listed. |
| SHIP-002 | Eligibility, declaration | Shipments; §11.3 value CHECK | Full | |
| SHIP-003 | Publish parcel | Shipments | Partial | Dues-block check needs Ledger read; not described. |
| SHIP-004 | Withdraw/edit published parcel | — | Partial | Lock order lists shipment first (good) but no workflow (AUD-008). |
| SHIP-005 | Parcel expiry | §7.7 `shipments.expiry` | Full | |
| SHIP-006 | Parcel lifecycle | Shipments | Partial | CLOSED transition has no owner (AUD-024). |
| MATCH-001 | Find matches | Matching §8.2 | Partial | Exact station-ID semantics (AUD-023); re-check at request/accept (AUD-007). |
| MATCH-002 | Ranking | §8.2 step 3 | Full | Formula TBD in PRD; architecture follows it. |
| MATCH-003 | No match | Web | Full | |
| MATCH-004 | Re-match after cancellation | — | Partial | Depends on missing cancellation workflows (AUD-008). |
| MATCH-005 | Trip alerts | — (Phase 2) | Full (deferred) | Correctly out of scope. |
| PRICE-001 | Quote | Pricing §8.3 | Partial | AD-14 changes input (flagged); quote TTL ambiguity (AUD-041). |
| PRICE-002 | Commission, earning | Pricing §8.3 | Full | Integer paise, half-up. |
| PRICE-003 | Price display and lock | §8.3 quote kept on request; §12 snapshots | Partial | Expired quote at accept undefined (AUD-041). |
| BOOK-001 | Request booking | Booking | Partial | Max-3 race (AUD-019); re-check of match (AUD-007). |
| BOOK-002 | Accept/reject | §7.3 | Partial | Missing trip lock, cash cutoff, match re-check (AUD-007). |
| BOOK-003 | Confirmation | §8.4, §7.3 cash branch | Partial | Confirmation path vague (transactional vs integration, AUD-015); duplicate attempt (AUD-003). |
| BOOK-004 | Expiry | §7.7 jobs | Partial | Expiry can overtake payment (AUD-005); deadline field missing (AUD-025). |
| BOOK-005 | Sender cancels | — | Missing | AUD-008. |
| BOOK-006 | Traveler cancels/no-show | — | Missing | AUD-008. |
| BOOK-007 | Concurrency and idempotency | §7.2, §7.5, §11.3 | Partial | Lock-order contradiction (AUD-006); idempotency semantics (AUD-018). |
| PAY-001 | Online checkout | Payments §8.4 | Partial | Multiple attempts/orders per booking unhandled (AUD-003). |
| PAY-002 | Event verification, late events | §9.5, §8.4 | Partial | Out-of-order, poll vs webhook (AUD-003). |
| PAY-003 | Cash at pickup | §7.4 cash confirmation | Incorrect | Commission debited before pickup with no reversal on decline (AUD-020). |
| PAY-004 | Cash on delivery | Payments | Partial | Refusal → return → dues + contingent earning credit not designed (AUD-002). |
| PAY-005 | Refunds | Port `refund` | Incorrect | Cash-booking refunds by GoParcel not executable in model (AUD-001); no refund idempotency (AUD-004). |
| PAY-006 | Payouts with netting | §8.4 payout run | Incorrect | Ledger before provider call; double-pay and hold races (AUD-009); dues-funded credits (AUD-001). |
| PAY-007 | Commission, GST, invoices | §8.5; T-6 | Partial | Invoicing TBD (CA). |
| PAY-008 | Reconciliation | §8.4, §7.7 | Partial | AUD-031. |
| PAY-009 | Traveler dues throttle/block | §7.7 `ledger.dues-age`; §7.3 throttle check | Partial | Dues payment flow and overpayment (AUD-019). |
| PAY-010 | Sender dues | Ledger accounts | Partial | Payment link flow, publish/request block wiring not designed. |
| LEDGER-001 | Party ledger | §8.5 | Partial | Immutability and balance rules good; posting rules missing (AUD-002). |
| HAND-001 | Reveal location | §7.7 `courier.seat-reveal` | Partial | Home-address mutual consent not designed. |
| HAND-002 | Approach info | §8.8, §13 location | Partial | Location retention unstated (AUD-040). |
| HAND-003 | Packing evidence | §8.6 | Partial | Gap check basis (server vs device time) and authenticity (AUD-027). |
| HAND-004 | Pickup verification | §7.4 | Partial | Lock order (AUD-006); codes not displayable (AUD-010); decline flow (AUD-008); lockout recovery (AUD-042); offline (AUD-011). |
| HAND-005 | In transit | Tracking, Courier | Full | Status-based. |
| HAND-006 | Delivery verification | Courier, §9.3 | Partial | Delivery code display (AUD-010); link leakage (AUD-026); false delivery (AUD-027). |
| HAND-007 | Evidence storage, integrity | §8.6 | Partial | Read-time checksum, export, backups (AUD-028). |
| HAND-008 | Recipient misses delivery | §7.4 exception exits | Partial | Stop change, fee, contingent share not designed; D-15 assumed. |
| HAND-009 | Sender misses pickup | — | Missing | AUD-008. |
| HAND-010 | Return to sender | §7.4 return code | Partial | Return-as-new-booking linkage missing (AUD-034). |
| HAND-011 | Abandoned parcels | §7.4 exit only | Missing | Counsel-dependent; process absent. |
| HAND-012 | Lost or damaged | Courier → Disputes | Partial | Compensation path not executable (AUD-001); booking outcome (AUD-024). |
| HAND-013 | Overdue escalation | §7.7 `courier.overdue` | Full | Never completes (BR-11). Minor ambiguity AUD-024. |
| TRACK-001 | Status timeline | Tracking | Full | From outbox events. |
| TRACK-002 | Event history | §7.1 StateEvent | Partial | Ownership contradiction (AUD-016). |
| NOTIF-001 | Event notifications | §8.7, outbox | Partial | Data access for message composition (AUD-014, AUD-040); codes not loadable (AUD-010). |
| NOTIF-002 | Retry, dead letters | §7.6, §8.7 | Full | |
| NOTIF-003 | Masked calling | §8.7 | Full | Vendor TBD. |
| NOTIF-004 | Template coordination messages | — | Missing | No design (small). |
| DISP-001 | Raise claim | Disputes | Partial | Auto-hold race with payout (AUD-009). |
| DISP-002 | Investigation and decision | Disputes → Payments, Ledger | Incorrect | Investigation design present, but compensation cannot be paid via the documented port (AUD-001). |
| DISP-003 | Holds | Disputes; `PayoutHoldChecker` | Partial | Reconciliation hold crosses layers (AUD-014); hold-vs-payout race (AUD-009). |
| DISP-004 | Appeal | — (T-10) | Missing | Provisional; reversal effects TBD. |
| DISP-005 | Custody resolution | Disputes `CustodyResolutionCase` | Partial | Entity only. |
| RATE-001 | Mutual ratings | Users | Missing | Entity only. |
| RATE-002 | Reliability record | Users | Partial | Mark creation depends on missing cancellation flows; auto-restriction path (AUD-017). |
| FRAUD-001 | Risk rules | §8.10 | Partial | Collusion/compensation rules missing (AUD-032). |
| FRAUD-002 | Manual review | Operations queues | Partial | Commands not listed. |
| SUP-001 | Support cases | Operations | Missing | Entity only; state machine named in §7.1. |
| SUP-002 | Controlled intervention | — | Missing | Pay-window extension etc. not designed (AUD-025, AUD-042). |
| ADMIN-001 | Staff auth, 2FA, RBAC | §8.1, §10.1 | Full | Same-origin risk noted (AUD-030). |
| ADMIN-002 | Ops dashboard | Operations views | Missing | Named only. |
| ADMIN-003 | User search, reveal | §10.2 reveal endpoint | Partial | Commands depend on AUD-017. |
| ADMIN-004 | KYC/ticket queues | Operations | Partial | |
| ADMIN-005 | Monitoring with history | Tracking StateEvent | Partial | AUD-016. |
| ADMIN-006 | Finance console | Admin | Missing | Depends on AUD-001/002. |
| ADMIN-007 | Configuration with two approvers | §12, §10.1 | Full | |
| ADMIN-008 | Audit log viewer | §10.5 | Full | |
| ADMIN-009 | Reports | Admin | Missing | Named only. |
| ADMIN-010 | Master data | Admin / Routes | Partial | |
| PRIV-001 | Consent and notices | Users | Partial | Versioning/re-consent not designed. |
| PRIV-002 | Data rights requests | Privacy module | Missing | Entity only. |
| PRIV-003 | Retention, legal hold | §11.5, §7.7 `privacy.retention` | Partial | Cross-module deletion path (AUD-014); evidence objects (AUD-028). |
| INC-001 | Security incident record | Privacy module | Missing | Entity only (process is mostly operational). |


## 2. Business rules

| BR | Rule | Architecture component | Coverage | Finding / note |
|---|---|---|---|---|
| BR-01 | Account and KYC status separate | Identity vs KYC modules | Full | |
| BR-02 | Only eligible users in matching | §8.2 view flags | Full | |
| BR-03 | Eligibility re-checked at acceptance | §7.3 | Partial | Incomplete re-check (AUD-007). |
| BR-04 | Separate state machines | §7.1 | Full | |
| BR-05 | Exactly one delivery per confirmed booking | §11.3 UNIQUE | Full | |
| BR-06 | No client-side confirmation | §8.4, G-2 | Full | |
| BR-07 | Concurrency-safe capacity | §7.2, §7.3, §11.3 | Partial | Lock-order contradiction; weight model (AUD-006, AUD-022). |
| BR-08 | Online verified before confirmation; cash exception | §7.3, §8.4 | Full | |
| BR-09 | Code verification by booked traveler | §7.4 | Partial | Codes not displayable (AUD-010). |
| BR-10 | Photos mandatory | §8.6, photo-check job | Full | Authenticity caveat (AUD-027). |
| BR-11 | Timers never complete fulfilment | §7.4, §7.7 | Full | |
| BR-12 | Cancellation only before pickup | — | Partial | No cancellation design (AUD-008). |
| BR-13 | Post-custody failure enters EXCEPTION | §7.4 | Full | |
| BR-14 | Recipient scoped to one delivery | §9.3 | Partial | Link = collection right (AUD-026). |
| BR-15 | Evidence access authorised and logged | §8.6, EvidenceAccess | Full | |
| BR-16 | Idempotent commands and provider events | §7.5, §9.5 | Partial | AUD-003, AUD-004, AUD-018. |
| BR-17 | Any active hold blocks payout | `PayoutHoldChecker` | Partial | Race (AUD-009). |
| BR-18 | Staff cannot bypass state machines | §7.1 | Full | |
| BR-19 | Important actions auditable | §10.5 | Full | |
| BR-20 | One person, one account | §8.9 blind index | Partial | Cross-document evasion (AUD-037). |
| BR-21 | KYC before first publish | KYC + Shipments/Routes | Full | |
| BR-22 | Declared value ≤ ₹5,000 | §11.3 CHECK | Full | |
| BR-23 | Price from size × band | §8.3 | Full | Band source AD-14 flagged. |
| BR-24 | Commission formula | §8.3 | Full | |
| BR-25 | First acceptance locks, cancels others | §7.3 | Full | |
| BR-26 | `JOURNEY_TOO_SOON` | §7.3 | Partial | Cash scope and extension (AUD-007, AUD-025). |
| BR-27 | PNR and phones never shown | §10.2, §8.7 masked calling | Full | |
| BR-28 | Inspection refusal = sender cancellation | — | Missing | AUD-008. |
| BR-29 | Custody on code verification | §7.4 | Full | Offline gap is a PRD issue (AUD-011). |
| BR-30 | Car handovers only while parked | Web §13 | Partial | UX rule; no design detail. |
| BR-31 | Online funds held by provider | §8.4 | Incorrect | Model conflicts with required GoParcel-funded movements (AUD-001). |
| BR-32 | Ledger append-only; no spendable balance | §8.5 | Partial | Overpayment of dues can create positive balance (AUD-019). |
| BR-33 | Dues throttle/block | §7.3, §7.7 | Full | |
| BR-34 | Seal-based liability; compensate then recover | Disputes | Incorrect | Compensation not executable (AUD-001). |
| BR-35 | Payout after dispute window | §7.7 | Partial | AUD-009. |
| BR-36 | COD collects only delivery price | Payments | Full | |
| BR-37 | Train, bus, car | Routes | Full | |
| BR-38 | Never credit uncollected money (Recommended) | §8.5 | Partial | No contingent-credit data model (AUD-002). |

## 3. Non-functional requirements

| NFR | Requirement | Architecture component | Coverage | Finding / note |
|---|---|---|---|---|
| NFR-PERF-01 | API p95 ≤ 500 ms read / 1 s write | §14.2, §15 load test | Partial | Next.js proxy hop and sync OTP send add latency; unmeasured. |
| NFR-PERF-02 | Matching p95 ≤ 1.5 s | §8.2 | Partial | Dues SUM per candidate (AUD-045). |
| NFR-PERF-03 | FCP ≤ 2.5 s | §13 SSR | Full | |
| NFR-PERF-04 | OTP within 30 s | §8.7 | Full | |
| NFR-SCALE-01 | Pilot sizing; vertical first | §14.2 scaling path | Full | |
| NFR-AVAIL-01 | 99.5% | 2 API instances | Partial | Single worker and single DB (AUD-005). |
| NFR-REL-01 | No lost transitions | §7.1–7.2 | Partial | Jobs outside transaction (AUD-013). |
| NFR-REL-02 | At-least-once jobs, dead letters | §7.6–7.7 | Partial | AUD-013. |
| NFR-SEC-01 | TLS, HSTS | §10.2 | Full | |
| NFR-SEC-02 | Encryption at rest, field-level | §10.2 | Partial | KMS undefined (AUD-038). |
| NFR-SEC-03 | OWASP ASVS L2 | — | Missing | No ASVS mapping or reference; same-origin admin (AUD-030); no threat model (AUD-032). |
| NFR-SEC-04 | Secrets in managed store | §10.2, §12 | Partial | Store vendor TBD (AUD-038). |
| NFR-SEC-05 | Rate limits | §10.4 | Partial | AUD-029. |
| NFR-PRIV-01 | Visibility per §3.5; reveals logged | §10.1, §10.2 | Full | |
| NFR-PRIV-02 | No PII in logs/URLs/analytics | §9.1, §10.3 | Partial | Recipient token in URL; Referer/preview leakage (AUD-026). |
| NFR-DATA-01 | Integer paise; DB constraints | §11.1, §11.3 | Partial | Missing constraints (AUD-019). |
| NFR-DATA-02 | Ledger immutability in DB | §8.5 | Full | |
| NFR-BACKUP-01 | PITR, RPO 1 h, RTO 8 h | §10.6 | Partial | Storage objects; post-restore reconciliation (AUD-028, AUD-031). |
| NFR-OBS-01 | Logs, metrics, alerts | §14.5 | Full | Add queue-age alert (AUD-005). |
| NFR-OBS-02 | Error tracking with release | §14.4 | Full | |
| NFR-OBS-03 | Analytics events | §10.3, §14.5 | Full | Sink TBD (T-4). |
| NFR-MAINT-01 | Rules in domain modules, contracts only | §6.2 | Partial | Layer rule self-contradictory (AUD-014). |
| NFR-MAINT-02 | Unit + concurrency tests | §15 | Partial | Failure-path gaps (AUD-048). |
| NFR-COMPAT-01 | Browsers, 320 px | §13 | Full | |
| NFR-ACC-01 | WCAG 2.1 AA | §13 axe checks | Full | |
| NFR-EXT-01 | Modes/providers via config and adapters | Ports §8.4, §8.7 | Partial | New modes affect capacity/matching rules in code. |
| NFR-COST-01 | Minimal budget | §2, §10.4 | Partial | No estimate (AUD-039). |

