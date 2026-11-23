# GoParcel — Audit Decision Log

Audit date 22 November 2026. Records the assumptions and judgement calls made during the architecture audit, the items that need the product owner, and the questions for the product owner.

## 1. Audit method decisions

| ID | Decision / assumption | Reason |
|---|---|---|
| AL-01 | The PRD v2.0 appendices are authoritative; the four separate PRD files were read and found to match them (minor wording differences only). | PRD header says the appendices replace the companion files (raised as AUD-044). |
| AL-02 | The only architecture source is `GOPARCEL_ARCHITECTURE.md` v0.1. The separately named system/domain/database/API/security/infrastructure/scalability/decisions/review files and ADRs do not exist (confirmed by Glob over `docs/**`). Their absence is a finding only where it blocks a decision or build step (AUD-032, AUD-033, AUD-034). | Brief instruction; avoid treating missing documents as automatic defects. |
| AL-03 | F1–F9 and a full DEC register are assumed not to exist (same as PRD OQ-01 and architecture §1). | No evidence they exist. |
| AL-04 | No code, test or vendor check was executed. Web search was attempted and was unavailable, so library and vendor behaviour (pg-boss transactional enqueue, Supavisor transaction-mode limits, Supabase IPv6-only direct connections and add-on pricing, Supabase Storage exclusion from backups, Prisma multi-schema GA status and drift behaviour, provider refund idempotency, carrier number-recycling period, DigiLocker/offline e-KYC data returned) is labelled **unverified** in findings. | Honesty about evidence. |
| AL-05 | Severity reflects impact if the design is built **as written**, for the controlled pilot with real money. | Pilot is the first real-money stage (PC-03). |
| AL-06 | Only one finding is rated CRITICAL (AUD-001). Other money-path issues are HIGH because they are design corrections within an otherwise workable model, and daily reconciliation (PAY-008) would surface many of them within a day. | Calibrate; avoid inflating severity. |
| AL-07 | Verdict NOT READY rather than CONDITIONALLY READY, because the open P0 items affect build step 1 (platform kernel) and the money model drives vendor choice. | Readiness rule: never DEVELOPMENT READY with open CRITICAL/P0; conditional readiness would still require P0 first. |
| AL-08 | Coverage "Full" means architecture-level coverage of module, data, workflow, failure and testability — not implementation detail. A named module with no workflow is "Missing". | Brief instruction. |
| AL-09 | Assumed answers in architecture §1 (OQ-02, 05, 06, 07, 09, 21) were **not** treated as approved decisions; the audit checked the design both under those assumptions and for consistency with them (AUD-007 found the OQ-02 cash assumption not implemented). | Do not silently change or adopt decision statuses. |
| AL-10 | No approved decision (ARCH-xx, D-xx, DEC-051) is changed by this audit. Where a finding touches one (BR-31/D-11 in AUD-001; D-18 in AUD-009; D-16 in AUD-027), it is raised for product-owner approval. | Brief instruction. |
| AL-11 | Matching formulas, prices, fees and thresholds were not invented. Where the audit needed a number (e.g. carrier recycling period) it is flagged unverified and raised as a question. | Brief instruction. |
| AL-12 | Scalability figures assume roughly one booking per ten users per day at the top end; they are illustrative, not forecasts. | No demand data exists (OQ-15). |

## 2. Items pending product-owner approval

| ID | Item | Related findings / decisions | Blocks |
|---|---|---|---|
| PA-01 | Funds-flow model: which movements may use GoParcel's own funds (compensation, cash-booking refunds, dues-funded payouts) and how this fits BR-31 / D-11 — with counsel | AUD-001; BR-31, D-11, D-17, RV-04, OQ-19, OQ-20 | Payments, Ledger, vendor choice |
| PA-02 | Statement that architecture approval does not answer OQ-02/05/06/07/09/21 or approve D-15 and RV-01..RV-04 | AUD-036 | Architecture approval |
| PA-03 | Answers to the six P1 open questions in PRD Appendix F.8 | AUD-036 | Re-review |
| PA-04 | AD-10 (Privacy module) and AD-14 (distance source) — already flagged by the architecture | §17 | Unchanged by audit |
| PA-05 | Status of BR-38 / RV-02, since the ledger design depends on it | AUD-002 | Ledger |
| PA-06 | Payout granularity (per delivery per D-18, or aggregated net of dues per PAY-006) | AUD-009; D-18 | Payouts |
| PA-07 | `CFG-RECYCLED-NUMBER-DAYS` value and selfie re-match on beneficiary change | AUD-012 | Identity, Payments |
| PA-08 | Whether the separate PRD companion files should be marked superseded or removed | AUD-044 | Documentation |

## 3. Questions for the product owner

| ID | Question | Why it matters | Options (auditor's recommendation in bold) | Findings |
|---|---|---|---|---|
| Q-01 | What happens when the traveler and sender/recipient are together but the phone has no network (or the API is down) at the handover? | Custody requires server code verification; halts can be 5 minutes. | **(a) offline-verifiable code with server confirmation shortly after**; (b) "verification pending" custody state; (c) accept the risk and use the missed-handover flow | AUD-011 |
| Q-02 | Who funds compensation and cash-booking refunds, and through which account? May GoParcel pay senders from its own account? | Decides vendor, ledger and legal model. | **GoParcel operating/reserve account pays via provider payouts, subject to counsel**; or insurance/third party later | AUD-001 |
| Q-03 | When a sender picks an origin/destination, is it a **city** (any station/bus stand/meeting point in that city matches) or an **exact station**? | Exact-ID matching may give near-zero matches across train, bus and car. | **City-level matching with the trip's stop as the handover point**; or exact station | AUD-023 |
| Q-04 | Should the recipient confirm with an OTP to their phone before the delivery code is revealed on the link? | A forwarded link currently lets anyone collect the parcel. | **Yes, for parcels above a value threshold or always**; or no (accept risk) | AUD-026 |
| Q-05 | A quote is valid 30 min but a request lives up to 6 h. At acceptance, is the request's quote honoured even if expired? | Accept behaviour and price lock. | **Honour the quote attached to the request until the request ends** (TTL only limits creating requests); or re-price at accept | AUD-041 |
| Q-06 | If a traveler edits a trip while requests are pending, what happens to those requests? | Bookings could be accepted on changed stops or times. | **System-cancel pending requests on any change to stops, times, seat or capacity reductions**; or block edits while requests are pending | AUD-007 |
| Q-07 | If a payment is captured at the provider before the window ends but the confirmation arrives after the window, should the booking be honoured if capacity is still free? | Fairness to senders paying at the last second; refund volume. | **Honour if captured before the deadline and still possible; otherwise refund**; or always refund after the window | AUD-003, AUD-005 |
| Q-08 | Lower `CFG-RECYCLED-NUMBER-DAYS` to at most the carrier recycling period, and require selfie re-match for every beneficiary change? | Prevents payout redirection after number recycling. | **Yes to both** | AUD-012 |
| Q-09 | Is a payout one transfer per delivery (D-18) or one transfer per run netting all credits and dues (PAY-006)? | Payout data model and provider usage. | **Per run netting, with per-delivery payable items for traceability** | AUD-009 |
| Q-10 | Booking outcome when the delivery ends RETURNED, LOST or ABANDONED; may a delivery go EXCEPTION → DELIVERED (damage reported, then delivered with claim, as HAND-012 says)? Also: may a traveler cancel while the booking is ACCEPTED_AWAITING_PAYMENT? | PRD state machines have gaps the architecture inherits. | **Add the edges and define booking terminal states explicitly** | AUD-024, AUD-008 |
| Q-11 | Should a capacity slot freed at an intermediate stop be reusable for a later segment of the same trip? | Capacity model complexity vs supply. | **No for the pilot (whole-trip capacity)**; revisit later | AUD-022 |
| Q-12 | Should there be caps on compensation exposure per traveler and per sender–traveler pair, and what happens when recovery from a traveler fails? | Collusive claims can drain the reserve. | **Yes: caps plus automatic review; unrecovered amounts written off against the reserve with reporting** | AUD-032 |
| Q-13 | Is cash acceptance a per-traveler setting, a per-trip setting, or both? | Ownership between Travelers and Routes modules. | **Per-traveler opt-in, per-trip toggle defaulting to the opt-in** | AUD-043 |
| Q-14 | Should the separate PRD companion files be marked superseded? | Avoid drift. | **Yes** | AUD-044 |

Existing PRD open questions that this audit depends on and that remain open: OQ-02, OQ-05, OQ-06, OQ-07, OQ-09, OQ-10, OQ-11, OQ-12, OQ-16, OQ-17, OQ-19, OQ-20, OQ-21.
