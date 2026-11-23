# GoParcel — Architecture Risk Register

Audit date 22 November 2026. Risks arising from the architecture as documented (v0.1). Likelihood and impact are the auditor's judgement for the controlled pilot if the design is built **as written**. Scale: Low / Medium / High. Owners are roles; the product owner assigns people.

Product risks already in PRD §13.2 (R-01..R-13) are not repeated unless the architecture changes them.

## 1. Financial

| ID | Risk | Likelihood | Impact | Mitigation | Owner | Findings |
|---|---|---|---|---|---|---|
| AR-F01 | Compensation, cash refunds and dues-funded payouts cannot run through the designed provider model; done manually outside the ledger or in a way counsel has not approved | High | High | Define end-to-end funds flow; extend provider port; counsel sign-off; vendor criterion | Product owner + counsel + architect | AUD-001 |
| AR-F02 | Ledger posting rules invented in code; inconsistent revenue/GST recognition; daily false reconciliation mismatches from gateway fees | High | High | Posting-rules table reviewed by CA before Ledger build | Architect + CA | AUD-002, AUD-031 |
| AR-F03 | Double payout (overlapping runs) or payout of disputed money | Medium | High | Payable items linked to one payout; locks; singleton run; re-check holds under lock | Architect | AUD-009 |
| AR-F04 | Ledger shows payouts that failed at the provider | Medium | Medium | Clearing account; post on provider confirmation; reversal on FAILED | Architect | AUD-009 |
| AR-F05 | Double refund after refund timeout, or webhook + poll | Medium | Medium | Stable references, UNKNOWN state, fetch-before-retry, unique automatic refund per attempt | Architect | AUD-003, AUD-004 |
| AR-F06 | Captured payment with no booking and no refund (out-of-order events, success after cancellation, duplicate attempt) | Medium | High | Provider-truth status fetch; FAILED→SUCCEEDED allowed; single refund rule | Architect | AUD-003 |
| AR-F07 | Traveler debited commission for a cash booking that never moved | Medium | Medium | Post commission in pickup transaction or reverse on decline | Architect | AUD-020 |
| AR-F08 | Dues overpayment creating a positive balance (D-21 breach) | Low | Low | Lock dues account when issuing links; auto-refund excess | Architect | AUD-019 |
| AR-F09 | Collusive loss claims drain the compensation reserve; recovery by ledger debit fails when the traveler abandons the account | Medium | High | Exposure caps per traveler/pair, risk rules, reserve monitoring, product decision on recovery | Product owner + ops | AUD-032 |
| AR-F10 | TDS/withholding confirmed late forces payout redesign | Medium | Medium | Withholding step in payout model now (zero until CA confirms) | Architect + CA | AUD-049 |

## 2. Technical / correctness

| ID | Risk | Likelihood | Impact | Mitigation | Owner | Findings |
|---|---|---|---|---|---|---|
| AR-T01 | Deadlocks between pickup and cancellation, trip cancel and accept | Medium | Medium | Single orchestrator per flow; extended lock order; deadlock retry; concurrency tests | Architect | AUD-006 |
| AR-T02 | Bookings created on cancelled or edited trips, or cash bookings after cutoff | Medium | High | Full match re-check at accept; trip row lock; edit invalidates requests | Architect | AUD-007 |
| AR-T03 | Timers silently lost because pg-boss jobs are not in the business transaction | Medium | High | Deadlines stored on aggregates + sweepers; outbox-driven job creation; prove in foundation spike | Architect | AUD-013 |
| AR-T04 | Handover impossible because codes are hash-only | High (certain if built as written) | High | Encrypted code storage | Architect | AUD-010 |
| AR-T05 | Idempotency keys stuck IN_PROGRESS block users for 24 h | Medium | Medium | Same-transaction record or lease | Architect | AUD-018 |
| AR-T06 | Prisma drift tooling drops raw-SQL indexes/constraints, removing DB invariants | Medium | High | Raw-SQL object registry + CI existence test | Architect | AUD-035 |
| AR-T07 | Module-boundary CI rules relaxed on day one because the layer model is self-contradictory | High | Medium | Publish explicit allowed-dependency list | Architect | AUD-014, AUD-015, AUD-016 |
| AR-T08 | Missing cancellation workflows implemented inconsistently | High | High | Cancellation design table | Architect | AUD-008 |
| AR-T09 | Delay updates rejected by the overlap constraint | Medium | Medium | Constraint on scheduled period only | Architect | AUD-021 |
| AR-T10 | Weight overbooking due to per-class weight cap | Medium | Medium | Trip-level weight row | Architect | AUD-022 |

## 3. Security

| ID | Risk | Likelihood | Impact | Mitigation | Owner | Findings |
|---|---|---|---|---|---|---|
| AR-S01 | Account takeover via recycled number leads to payout redirection | Medium | High | Recycled-number check, selfie re-match on beneficiary change, threshold ≤ carrier recycle period | Architect + product owner | AUD-012 |
| AR-S02 | Forwarded or previewed recipient link lets a stranger collect the parcel | Medium | High | No SSR of code, no-referrer, optional recipient OTP | Architect + product owner | AUD-026 |
| AR-S03 | Traveler obtains delivery code by phone and marks delivered without handover | Medium | High | Plausibility checks, recipient messaging, risk rules | Architect + ops | AUD-027 |
| AR-S04 | SMS pumping / OTP flood costs money and loads the primary DB | Medium | Medium | Global OTP budget, edge limits, circuit breaker | Architect | AUD-029 |
| AR-S05 | XSS in user content hijacks staff sessions (same origin) | Low | High | Admin subdomain, CSP, separate cookies | Architect | AUD-030 |
| AR-S06 | Insider with Supabase dashboard access reads or alters data outside app audit | Low | High | Restrict and log dashboard access; KMS keys outside Supabase | Product owner + architect | AUD-032, AUD-038 |
| AR-S07 | Field encryption keys stored beside DB credentials | Medium | High | KMS with key hierarchy | Architect | AUD-038 |
| AR-S08 | One person verifies several accounts with different documents | Medium | Medium | Provider identity token, face-template similarity, risk flag | Architect + counsel | AUD-037 |
| AR-S09 | Fabricated evidence photos uploaded through the API | Medium | Medium | Treat evidence as user-supplied; server times; gap rules | Architect | AUD-027 |
| AR-S10 | No threat model; pen test finds design-level issues late | High | Medium | Threat model before Stage P | Security lead | AUD-032 |

## 4. Operational / reliability

| ID | Risk | Likelihood | Impact | Mitigation | Owner | Findings |
|---|---|---|---|---|---|---|
| AR-O01 | Worker outage causes wrongful payment expiry and mass refunds | Medium | High | Check-before-expire; two workers; queue-age alert | Architect + ops | AUD-005 |
| AR-O02 | No network at the seat; handovers fail at short halts | High | High | Product decision on offline verification | Product owner | AUD-011 |
| AR-O03 | Handover-code lockout with no recovery path | Medium | Medium | Regenerate command (sender, ops) | Architect | AUD-042 |
| AR-O04 | Evidence lost (storage not covered by PITR) | Low | High | Replication/versioning; integrity sweep | Architect + ops | AUD-028 |
| AR-O05 | PITR restore leaves DB behind the provider; money state wrong | Low | High | Post-restore provider replay and reconciliation runbook | Ops + finance | AUD-031 |
| AR-O06 | Supabase connectivity (IPv4/IPv6, pooler limits) blocks deployment late | Medium | Medium | Connectivity spike before choosing host | Architect | AUD-013 |
| AR-O07 | Manual KYC/ticket review becomes the first scaling bottleneck | High (beyond pilot) | Medium | OCR auto-approval thresholds, queue SLAs, staffing plan | Ops | Report §6 |
| AR-O08 | Unbudgeted infrastructure and per-booking costs break contribution margin | Medium | Medium | Cost model feeding `CFG-MIN-COMMISSION` | Product owner | AUD-039 |

## 5. Scalability

| ID | Risk | Likelihood | Impact | Mitigation | Owner | Findings |
|---|---|---|---|---|---|---|
| AR-SC01 | Dues computed by SUM(postings) per candidate in matching slows search | Low (pilot) / Medium (100k users) | Medium | Maintained balance proven nightly | Architect | AUD-045 |
| AR-SC02 | Postgres rate-limit and idempotency writes contend under bursts | Low (pilot) | Medium | Edge limits; Redis only on measured need | Architect | AUD-029 |
| AR-SC03 | Connection exhaustion through the pooler as instances grow | Medium (at 100k users) | Medium | Size pools; monitor; vertical scale | Architect | AUD-013 |
| AR-SC04 | Single primary DB CPU at 1M users | Low (not in horizon) | High | Read replica for matching/reporting; extraction only on evidence | Architect | Report §6 |

## 6. Governance / documentation

| ID | Risk | Likelihood | Impact | Mitigation | Owner | Findings |
|---|---|---|---|---|---|---|
| AR-G01 | Architecture approval is read as approval of open product decisions (OQ-02/05/06/07/09/21, D-15, RV-01..04) | Medium | Medium | Explicit statement in approval record | Product owner | AUD-036 |
| AR-G02 | Endpoints and schema emerge from code without review | High | Medium | API inventory and schema doc per module before build | Architect | AUD-033, AUD-034 |
| AR-G03 | PRD companion files drift from appendices | Low | Low | Mark superseded | Product owner | AUD-044 |
