# GoParcel — Decision Register

> **SUPERSEDED (22 Nov 2026, PD-20).** This file is kept for history only and is no longer maintained. The authoritative version is the matching appendix of `GOPARCEL_MASTER_PRD.md` (decision register: Appendix C; open questions: Appendix D; traceability: Appendix E; audit: Appendix F). Decisions PD-01..PD-20 and the resolved open questions appear only there.

Version 2.0 draft · 22 November 2026 · Companion to `GOPARCEL_MASTER_PRD.md`

Status values: **Confirmed** · **Provisional** · **Proposed** · **Superseded**. Only the product owner changes a status.

## 1. Product decisions (PRD review, Nov 2026)

| ID | Decision | Status | Notes |
|---|---|---|---|
| D-01 | One person, one account, one verified identity for both sender and traveler roles | Confirmed | |
| D-02 | KYC once, before the first publish of a parcel or trip (not at signup) | Confirmed | Method subject to counsel |
| D-03 | WhatsApp OTP primary; SMS fallback; SMS required for recovery and payout changes | Confirmed | |
| D-04 | Platform-set price from size class × distance band | Confirmed | Table values TBD |
| D-05 | Sender may request several travelers; first acceptance locks the parcel and cancels the rest | Confirmed | Limit of 3 proposed |
| D-06 | S/M/L size classes plus weight cap per mode | Confirmed | Dimensions and caps TBD |
| D-06a | Train, bus and car at launch; bike and air later | Confirmed | Car is accepted risk R-01 |
| D-07 | Handover at the traveler's seat (train/bus); coach and seat shared, never PNR | Confirmed | Sender/recipient responsible for reaching seat |
| D-08 | Ticket proof (PNR, seat) required for train and bus trips | Confirmed | |
| D-09 | Intermediate-stop route matching | Confirmed | |
| D-10 | Mobile-first website first; mobile app after | Confirmed | Funding constraint |
| D-11 | Online prepaid at acceptance; funds held by payment provider; payout after delivery | Confirmed | |
| D-11a | Cash at pickup and cash on delivery (delivery price only) added | Confirmed (revised) | Supersedes earlier "no cash" recommendation; safeguards Proposed |
| D-12 | Contents photo + sealed photo with booking code; weight check; traveler right to ask; refusal = sender cancellation | Confirmed | No routine inspection (R-02) |
| D-13 | Masked calling; real numbers never shown | Confirmed | Supersedes PRD v1.0 "template-only, no phone sharing" |
| D-14 | Train running status; "30 min away / arrived" alerts; live location only in app, final stretch only | Confirmed | |
| D-15 | Missed-handover fallback (later stop or final destination collection, fee, return, abandonment) | **Proposed** | Awaiting approval (OQ-09) |
| D-16 | In-page camera only; custody on code verification; photo upload retry 10 min | Confirmed | Resolves PRD v1.0 UD-03 |
| D-17 | Fault-based liability by seal; GoParcel compensates sender first, then recovers | Confirmed for initial launch | |
| D-18 | Payout per delivery, 48–72 h after delivery | Confirmed | |
| D-19 | Car trips: public meeting points; licence, registration, insurance checks; no app use while driving | Confirmed | |
| D-20 | Declared-value cap ₹5,000 for pilot | Confirmed | |
| D-21 | Traveler ledger: commission debits on cash, throttle ₹500, block ₹1,000 or 30 days, provider netting, no spendable balance; subscription pass later | Confirmed | Supersedes PRD v1.0 "no wallet / no ledger" |
| D-22 | Commission: higher of 15% or minimum fee, plus GST, deducted from traveler share; online and cash | Confirmed | Minimum amount TBD; GST to CA |
| PC-01 | Small businesses out of scope until after pilot | Confirmed | Policies §12 |
| PC-02 | Legal counsel consultation to be arranged | Confirmed | Topics in PRD §10.2 |
| PC-03 | Real money from first real parcel: PRD v1.0 gates G and M merge into one controlled pilot | Confirmed | Stage A retention: OQ-08 |

## 2. Review recommendations (22 Nov 2026) awaiting approval

| ID | Recommendation | Status | PRD location |
|---|---|---|---|
| RV-01 | Recipient may be the sender (sending to yourself is allowed) | Proposed | SHIP-001 |
| RV-02 | GoParcel never credits a traveler with money it has not collected (BR-38) | Proposed | BOOK-005, PAY-004, PAY-006, LEDGER-001 |
| RV-03 | Traveler mismatch decline: no-fee cancellation, ops decides fault | Proposed | HAND-004 |
| RV-04 | Payouts include compensation credits as well as online earnings | Proposed | PAY-006 |

## 3. Recorded decisions from the master prompt

| ID | Decision | Status | Notes |
|---|---|---|---|
| DEC-051 | Reject acceptance with `JOURNEY_TOO_SOON` when the minimum payment hold cannot be met | Confirmed | Meaning of "payment hold" and cash applicability TBD (OQ-02) |
| DEC-052 | Resumable exceptions | Provisional | Text not available; PRD models EXCEPTION → IN_TRANSIT resume (OQ-03) |
| DEC-057 | Dispute appeal | Provisional | Text not available; PRD proposes one appeal to a different senior admin (OQ-03) |

The full DEC register was not available. Other DEC entries may exist and are **unverified**.

## 4. Architecture decisions (from master prompt)

| ID | Decision | Status |
|---|---|---|
| ARCH-01 | Monorepo with pnpm and Turborepo | Confirmed |
| ARCH-02 | `apps/api` and `apps/web` with shared packages | Confirmed |
| ARCH-03 | NestJS modular monolith | Confirmed |
| ARCH-04 | Supabase-hosted PostgreSQL with Prisma and PostGIS | Confirmed (supersedes PRD v1.0 "hosting unresolved") |
| ARCH-05 | Versioned API under `/api/v1` | Confirmed |
| ARCH-06 | Domain modules with contracts-only public boundaries | Confirmed |
| ARCH-07 | Provider ports and adapters for external integrations | Confirmed |
| ARCH-08 | Extraction-ready; no premature microservices | Confirmed |
| ARCH-09 | No Kafka or Kubernetes without demonstrated need | Confirmed |
| ARCH-10 | Domains: Identity, Users, Travelers, KYC, Routes, Shipments, Booking, Matching, Pricing, Payments, Ledger, Tracking, Notifications, Courier, Disputes, Fraud, Operations, Admin | Confirmed (Courier meaning OQ-04) |
| ARCH-11 | Swagger API documentation | Confirmed |
| ARCH-12 | Rule-based matching and pricing engines | Confirmed |

## 5. Carried from PRD v1.0

| ID | Decision | Status |
|---|---|---|
| PQ-1 | Photos mandatory at sealing, handover and delivery | Confirmed |
| v1.0 BR-01..BR-20 | Core business rules | Confirmed; BR-08 amended by D-11a |

## 6. Superseded

| Earlier position | Superseded by |
|---|---|
| PRD v1.0 non-goal "no wallet"; "98-table design has no ledger" | D-21 (ledger, no spendable balance) |
| PRD v1.0 separate gates G and M | PC-03 |
| PRD v1.0 "template messages only, no phone sharing" | D-13 masked calling (templates retained) |
| PRD v1.0 OTP and hosting providers unresolved | D-03, ARCH-04 |
| Review recommendation "prepaid only, no cash" | D-11a |
| Review recommendation "launch train and bus only" | D-06a (car included) |
| Initial proposal of monthly traveler settlement | D-18 per-delivery payout |
| PRD v1.0 "verification for travelers only" | D-01/D-02 KYC for all users |
