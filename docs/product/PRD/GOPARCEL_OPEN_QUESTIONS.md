# GoParcel — Open Questions

> **SUPERSEDED (22 Nov 2026, PD-20).** This file is kept for history only and is no longer maintained. The authoritative version is the matching appendix of `GOPARCEL_MASTER_PRD.md` (decision register: Appendix C; open questions: Appendix D; traceability: Appendix E; audit: Appendix F). Decisions PD-01..PD-20 and the resolved open questions appear only there.

Version 2.0 draft · 22 November 2026 · Companion to `GOPARCEL_MASTER_PRD.md`

Priority: **P1** blocks the pilot or core build · **P2** needed before the affected feature ships · **P3** can wait.

| ID | Pri | Question | Why it matters | Options | Recommendation | Status |
|---|---|---|---|---|---|---|
| OQ-01 | P2 | Do F1–F9, ADRs or a full DEC register exist anywhere? The repository contains only `README.md`. | Without them, other earlier decisions cannot be checked | Copy into repo / confirm they do not exist | If they exist, copy them into `docs/`; otherwise this PRD becomes the baseline | Open |
| OQ-02 | P1 | What does "minimum payment hold" in DEC-051 mean, and does it apply to cash bookings? | Shapes the acceptance rule and `CFG-MIN-PAYMENT-HOLD` | (a) minimum time for payment before departure; (b) provider fund hold; (c) card auth hold | (a): pay window + booking cutoff; for cash, apply booking cutoff only | Open |
| OQ-03 | P2 | Current text of DEC-052 (resumable exceptions) and DEC-057 (dispute appeal) | Both are provisional; PRD modelled them as assumptions | Provide text / approve PRD's model | Approve PRD model or send text | Open |
| OQ-04 | P2 | What does the "Courier" domain module mean? | Module boundaries | Custody and handover / something else | Custody and handover, separate from Travelers | Open |
| OQ-05 | P1 | Confirm OTP-only login (no passwords, no email login) | Master prompt mentions password recovery and email; D-03 is OTP | OTP only / add passwords | OTP only | Open |
| OQ-06 | P1 | Confirm the recipient is a scoped role without an account | Master prompt omits recipient | Scoped link / require account | Scoped link (PRD v1.0) | Open |
| OQ-07 | P1 | Team size and timeline | MVP sizing and sequencing | — | Even a rough answer | Open |
| OQ-08 | P2 | Keep Stage A (staff simulated alpha) before the pilot? | Cheap insurance against state and money bugs | Keep / skip | Keep | Open |
| OQ-09 | P1 | Approve D-15 missed-handover fallback; choose 12 h or 24 h final-destination window | HAND-008 is MVP but still Proposed | Approve / change | Approve with 24 h | Open |
| OQ-10 | P2 | Sender cancellation fee: policies say "minus platform fee", but commission is borne by the traveler | Refund calculation would be ambiguous | Fixed fee / % of price / none | Fixed fee, set in config | Open |
| OQ-11 | P2 | How are refunds paid to cash-paying senders with no UPI or bank account? | Traveler-fault refunds on cash bookings | Require UPI/bank for claims / cash via ops | Require a UPI or bank account to receive refunds | Open |
| OQ-12 | P1 | Values: S/M/L dimensions, weight caps per mode, distance bands, price table, minimum commission | Pricing and matching cannot run without them | — | Set after per-parcel cost estimate | Open |
| OQ-13 | P2 | Pilot cities and routes | Master data, ops staffing, liquidity | — | 1–3 dense city pairs with frequent trains and buses | Open |
| OQ-14 | P2 | Support hours and channels | Handover issues happen early morning and late night | Business hours / extended / 24×7 | Extended hours covering train arrival peaks | Open |
| OQ-15 | P3 | Pilot success targets | Exit criteria | — | Set after 4 weeks of baseline | Open |
| OQ-16 | P2 | Late-cancel refund %, traveler compensation share, missed-handover fee and traveler share | Cancellation and fallback flows | — | — | Open |
| OQ-17 | P1 | Compensation reserve amount for the pilot | GoParcel pays senders before recovery | — | Size from expected parcels × loss rate × ₹5,000 cap | Open |
| OQ-18 | P1 | Final prohibited-items list | Legal exposure | — | Use PRD baseline, confirm with counsel | Open |
| OQ-19 | P1 | Vendor choices: payment, KYC, WhatsApp BSP, SMS, masked calling, train data | Integrations and cost | — | Shortlist and compare on capability, cost and terms | Open |
| OQ-20 | P1 | Counsel and CA review of PRD §10.2 topics | Legal and tax viability | — | Single consultation covering all topics | Open |
| OQ-21 | P2 | Approve review recommendations RV-01 to RV-04 (decision register §2) | Affect shipments, cancellations, payouts and pickup declines | Approve / change | Approve all four | Open |
