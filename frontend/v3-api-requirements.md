# Frontend → Backend: API requirements for the v3 court app

**Raised by:** Frontend team
**Date:** 2026-10-05
**Status:** Open
**Design source:** "JudicAI Court App v3" prototype (standalone HTML, shared by product on 2026-10-04)
**Related:** [`billing-integration-report.md`](billing-integration-report.md) (REQ-B01 – B08), [`billing-integration-reply.md`](billing-integration-reply.md)

The frontend has moved the court dashboard and the Platform Admin dashboard to the v3 design (branch `feature/court-ui-v3`). Every v3 screen that the current API can serve is built. This document lists **everything else the v3 design needs from the backend**, screen by screen, so nothing is left implicit.

Conventions used below:

- Money is in **kobo** on the wire (as in the billing contract), shown in Naira by the frontend.
- Lists are `PaginatedResult` (`data`, `total_rows`, `total_pages`, `page`, `size`) unless noted.
- Field and status names are taken from the prototype; rename freely, but please keep the **set of states**, since each one has its own UI.
- "Already requested" means the item is covered by an earlier REQ-B and is only cross-referenced here.

---

## Summary

| # | Area | Who sees it | Blocks |
|---|---|---|---|
| [V01](#v01--new-roles-lawyer-court_admin-finance) | New roles: `LAWYER`, `COURT_ADMIN`, `FINANCE` | — | Everything in V02, V05, V09 |
| [V02](#v02--lawyer-portal) | Lawyer portal: invoices, receipts, payment history, refunds, outstanding | Lawyer | 5 screens |
| [V03](#v03--records-of-proceedings-requests) | Records of Proceedings requests (court queue + lawyer side) | Court staff, Lawyer | 2 screens |
| [V04](#v04--court-calendar-and-virtual-proceeding-booking) | Court calendar: virtual cap, booking, moving a proceeding | Court staff, Lawyer | Calendar actions, lawyer booking screen |
| [V05](#v05--court-financials) | Court financial overview and the Judge's summary | Judge, Court Admin | Overview finance tabs, Financial Overview, Reports |
| [V06](#v06--platform-financial-operations) | Platform financial operations | Platform Admin | 8 screens |
| [V07](#v07--billing-overview-tab) | Billing overview tab | Platform Admin | Billing → Overview |
| [V08](#v08--platform-dashboard-money-figures) | Platform dashboard money figures | Platform Admin | Dashboard |
| [V09](#v09--finance-officer-reconciliation) | Finance Officer reconciliation | Finance | 3 screens |
| [V10](#v10--notifications) | Notifications feed | Everyone | Header bell |
| [V11](#v11--platform-settings) | Platform Settings | Platform Admin | Settings page |
| [V12](#v12--transcript-review-flags) | Transcript review flags | Court staff | Transcript editor flags panel |
| [Q](#questions-and-confirmations) | Confirmations needed | — | Correctness of existing screens |

Already requested and still open (not repeated here): **REQ-B06** Court → Lawyer payments (checkout, court transactions, settlements, payments audit), **REQ-B07** payment configuration additions (virtual hearing fee, bank list and account lookup, settlement-account change request, Paystack subaccount), **REQ-B08** receipt PDF.

---

## V01 — New roles: `LAWYER`, `COURT_ADMIN`, `FINANCE`

The v3 prototype has six personas. The backend has four roles (`PLATFORM_ADMIN`, `JUDGE`, `REGISTRAR`, `LEGAL_AIDE`).

| Persona in v3 | Role needed | What they can do |
|---|---|---|
| Lawyer (e.g. "Barr. A. Bello") | `LAWYER` | Not a court employee. Pays court fees (CTC, virtual hearing, records of proceedings), requests records, books virtual proceedings, asks for refunds. Sees only their own cases and payments, across any court. |
| Court Staff (Admin) | `COURT_ADMIN` | Court staff with the Registrar's menu plus Payments write access, Record Requests and the court financial overview. |
| Finance Officer | `FINANCE` | DevonTech staff: read-only reconciliation across courts (V09). |

**Needed:**

- The three roles in `user_type`, accepted by sign-in and returned in the session/profile.
- **Lawyer sign-up or invitation.** How does a lawyer get an account? Self sign-up with email verification, or invited by a court? This decides whether we need a public registration page.
- **Lawyer ↔ case link.** A lawyer must see "their" cases. Is this by the counsel records already on a case (`prosecutor_counsels` / `defense_counsels` emails), or an explicit link?
- User management: Platform Admin `POST /platform/users` and court `POST /court/users` accept the new roles where they apply.

---

## V02 — Lawyer portal

Screens: **Invoices, Receipts, Payment History, Refunds, Outstanding** (Lawyer → Payments). Checkout itself is REQ-B06.

| Endpoint | Purpose / fields |
|---|---|
| `GET /lawyer/invoices` | Invoices raised to the lawyer: `ref` (e.g. `INV-648120`), `service` (`CTC` \| `VIRTUAL_HEARING` \| `RECORD_OF_PROCEEDINGS`), `case_title`, `case_number`, `court_name`, `amount`, `description`, `status` (`AWAITING_PAYMENT` \| `PAID` \| `FAILED` \| `REFUNDED`), `created_at`, `paid_at`. Filter by status. |
| `GET /lawyer/invoices?status=AWAITING_PAYMENT` | **Outstanding** screen ("services are released once payment is confirmed"). |
| `POST /lawyer/invoices/{ref}/checkout` | Start Paystack for an invoice; returns `authorization_url`, `reference`. Same verify pattern as `POST /court/subscription/verify` (no webhook). |
| `POST /lawyer/payments/verify` | `{ reference }` → status codes as for the subscription verify (`PAYMENT_PENDING`, `PAYMENT_DECLINED`, …). |
| `GET /lawyer/receipts` | `ref` (`RCP-…`), `service`, `amount`, `date`, `invoice_ref`. Plus `GET /lawyer/receipts/{ref}` (and the PDF from REQ-B08). |
| `GET /lawyer/payments` | Payment history: every attempt with its outcome (`PAID` \| `PENDING` \| `FAILED`), reference, service, amount, date. |
| `GET /lawyer/refunds`, `POST /lawyer/refunds` | Request: `{ invoice_ref, reason }`. List: `ref` (`RFD-…`), `service`, `amount`, `reason`, `status` (`UNDER_REVIEW` \| `APPROVED` \| `REJECTED` \| `PAID`), `date`, `decision_note`. |

Unlocking rule shown in the design: a CTC document or a record of proceedings becomes downloadable only once its invoice is `PAID`. Please return the download URL (or a `released: true` flag) on the paid invoice.

---

## V03 — Records of Proceedings requests

A lawyer requests a case's record of proceedings; the court prepares it, sets the page count, and the lawyer pays `pages × price_per_page` (from `payment_config`).

Statuses (court label / lawyer label):

| Status | Court sees | Lawyer sees |
|---|---|---|
| `REQUESTED` | New request | Requested |
| `NEEDS_UPDATE` | Needs update (the record changed since it was prepared) | Court updating record |
| `PREPARING` | Preparing | Being prepared |
| `READY` | Awaiting payment | Ready to pay |
| `DELIVERED` | Paid & delivered | Delivered |

| Endpoint | Purpose |
|---|---|
| `POST /lawyer/rop-requests` | `{ case_id }` → creates a `REQUESTED` request (`ROP-2041`). |
| `GET /lawyer/rop-requests` | The lawyer's requests with status, pages, amount, and a download link when `DELIVERED`. |
| `GET /court/rop-requests` | Court queue: `id`, `case_number`, `case_title`, `lawyer_name`, `status`, `pages`, `amount`, `requested_at`, number of proceedings in the case. Filter by status. Also an open-count for the sidebar badge. |
| `PATCH /court/rop-requests/{id}` | Court sets `pages` and moves `REQUESTED/NEEDS_UPDATE → PREPARING → READY`. Moving to `READY` raises the lawyer's invoice (V02) and notifies them (V10). |
| `GET /court/rop-requests/{id}/document` | The stamped PDF preview (the design shows a stamped record before release). |

---

## V04 — Court calendar and virtual proceeding booking

The frontend Court Calendar is built today from `GET /causelist` (one month per request). It needs:

| Endpoint | Purpose |
|---|---|
| `GET /court/calendar/settings`, `PUT /court/calendar/settings` | `virtual_cap_per_day` (the design's "Virtual cap / day", default 4). Optionally court sitting days (the design greys out weekends as "Closed"). |
| `GET /court/calendar?month=2026-10` | **Optional but recommended.** Per-day counts `{ date, physical, virtual }` for a month. Today we pull up to 500 full cause-list rows just to count them. |
| `PATCH /court/virtual-proceedings/{id}` | Move a virtual proceeding to another day: `{ date, notify_attendees: boolean }`. Must respect the target day's cap. Notifies the lawyer (and attendees when asked). |
| `GET /lawyer/calendar?court_id=…&month=…` | Lawyer booking view ("Book a Virtual Proceeding"): per day `{ date, slots_left }` (cap minus booked), with closed and past days marked. |
| `POST /lawyer/virtual-proceedings` | `{ case_id, date }` → booking; may raise a virtual-hearing invoice (REQ-B07 virtual hearing fee). Notifies the court. |

---

## V05 — Court financials

These depend on REQ-B06 data existing; they are the read side.

| Screen | Endpoint | Fields |
|---|---|---|
| Overview → "Financial snapshot", "Payments", "Transactions" tabs (court staff) | `GET /court/payments/summary?from=&to=` | Totals: gross received, commission, court share, settled, pending settlement; counts by service; latest transactions (or reuse REQ-B06 list with `size=5`). |
| Court Financial Overview (`High Court of Justice · July 2026`) | `GET /court/payments/summary?month=2026-07` | Same totals plus a monthly series by service for the revenue chart: `[{ month, ctc, virtual_hearing, rop }]` for the last 6 months. |
| Judge financial summary ("Judges have no access to transaction detail") | Same summary endpoint, aggregate only | Judges get totals, no per-transaction rows. Please enforce on the backend (a Judge calling the transactions list gets 403, or the role check lives in the summary). |
| Reports → revenue and settlements | `GET /court/reports/revenue?from=&to=&format=pdf\|csv` | Downloadable revenue/settlement report for the court. NJC quarterly returns already exist. |

---

## V06 — Platform financial operations

Platform Admin → Financial Operations. Only **Billing** exists today (the subscription contract). The rest:

| Screen | Endpoint | Fields / actions |
|---|---|---|
| All Transactions | `GET /platform/payments/transactions` | `ref`, `court_name`, `service`, `gross`, `commission`, `status` (`PAID` \| `PENDING` \| `FAILED`), `date`. Filter by court, service, status, date. |
| Settlement Monitoring | `GET /platform/settlements`, `POST /platform/settlements/{ref}/retry` | `ref` (`STL-…`), `court_name`, `gross`, `court_share`, `status` (`COMPLETED` \| `PENDING` \| `MANUAL_REVIEW` \| `FAILED`), `attempts`, `last_error`, `date`. Retry for failed or manual-review rows. Sidebar badge = count needing action. |
| Flagged Transactions | `GET /platform/payments/flagged`, `POST /platform/payments/flagged/{ref}/resolve` | Payments whose amount didn't match the expected fee: `ref`, `court_name`, `expected`, `received`, `issue` (e.g. `AMOUNT_MISMATCH`). Resolve with `{ action: "accept" \| "refund" \| "investigate", note }`. Badge count. |
| Commission Rules | `GET /platform/commission-rules`, `PUT /platform/commission-rules/{id}` | Per court and service: `court_name`, `service`, `percent` (e.g. 5%, 7%). Changes go to the audit trail. |
| Refund Requests | `GET /platform/refunds`, `POST /platform/refunds/{ref}/approve`, `POST /platform/refunds/{ref}/reject` | `ref` (`RFD-…`), `court_name`, `lawyer_name`, `amount`, `reason`, `status`. Reject needs a reason; approve triggers the Paystack refund. Badge count. |
| Bank Account Change Requests | `GET /platform/bank-change-requests`, `…/{id}/approve`, `…/{id}/reject` | The approval side of REQ-B07's change request: `court_name`, masked old → new account, `requested_at`, `status`. Badge count. |
| Reports | `GET /platform/reports/{type}?from=&to=&format=pdf\|csv` | Types in the design: Commission Report, Cross-Court Transaction Summary, Settlement Monitoring, Subscription Report. |
| Audit Trail | `GET /platform/audit` | Every high-impact action: `event`, `actor_name`, `actor_role` (`PLATFORM_ADMIN` \| `SYSTEM` \| court role), `reference`, `time`. E.g. "Commission percentage changed", "Settlement failed (after 3 retries)", "Bank account change approved". Filter by event type, court, date. |
| Financial Operations overview ("Cross-court monitoring · July 2026") | `GET /platform/payments/summary?month=` | Totals across courts: gross, commission, settled, pending, failed; per-court breakdown. |

---

## V07 — Billing overview tab

Billing has tabs Subscriptions / Invoices / Plans / Dunning / History today. The v3 **Overview** tab needs:

| Endpoint | Fields |
|---|---|
| `GET /platform/billing/overview` | `active_mrr`, `contracted_value`, `at_risk_amount` (+ grace and suspended counts), `renewal_rate_90d`, **needs attention** list (`court_name`, `detail` e.g. "Suspended 24 Jul · ₦150,000 outstanding", `status`), **renewals in the next 30 days** (`court_name`, `plan_name`, `amount`, `due_date`, `reminder_sent_at`), **revenue by plan** (`plan_name`, `court_count`, `amount`). |
| `GET /platform/billing/export?format=csv` | The "Export" button on the Billing banner. |

The frontend computes MRR, contracted and at-risk from `GET /platform/court-subscriptions` today. Renewal rate, the 30-day renewal list and revenue by plan can't be derived reliably without history.

---

## V08 — Platform dashboard money figures

The Platform dashboard banner reads "Every court on JudicAI, with platform-wide activity and money". `GET /platform/analytics` returns activity counts only (`total_court`, `total_cases`, `total_recordings`, `total_transcript`, `total_virtual`). Please add money totals for the period: subscription revenue, court-fee gross, commission earned.

---

## V09 — Finance Officer reconciliation

Role `FINANCE` (V01), read-only.

| Screen | Endpoint | Fields |
|---|---|---|
| Reconciliation overview ("inbound commission & settlement reconciliation · July 2026") | `GET /finance/reconciliation?month=` | Commission expected vs received, settlements due vs completed, discrepancies count. |
| Commission Report | `GET /finance/commission?month=` | Per court: `court_name`, `service`, `percent`, `collected`. |
| Settlement Reconciliation | `GET /finance/settlements?month=` | Settlements matched against commission: `ref`, `court_name`, `gross`, `commission`, `court_share`, `status`, `matched: boolean`. |

These can share the V06 queries with a read-only role check.

---

## V10 — Notifications

The header bell is on every page (court and platform). It shows an empty state today.

| Endpoint | Purpose |
|---|---|
| `GET /notifications` | Current user's notifications: `id`, `text`, `created_at`, `read`, optional `link` (e.g. to a case or a request). Paginated, newest first. |
| `GET /notifications/unread-count` | For the red dot on the bell. |
| `POST /notifications/read` | `{ ids: [...] }`, or `{ all: true }` for "Mark all read". |

Events the design notifies on: virtual proceeding booked or moved (court and lawyer), record of proceedings ready to pay, refund decided, settlement failed, subscription reminder or grace (court admins), hearing reminders.

---

## V11 — Platform Settings

The page exists but is empty ("Manage your platform account and administrative preferences"). Needed: `GET/PUT /platform/profile` (name, email, phone, photo), change password (as courts have), and any platform-wide preferences you want admins to edit. Tell us the intended scope and we'll build the form.

---

## V12 — Transcript review flags

The v3 transcript editor shows a **flags panel**: low-confidence phrases with a suggested correction that the editor can accept or dismiss (e.g. "Chinedu Okafor", confidence 58%, suggestion "Chinedum Okafor").

Needed on the transcription payload: per flagged phrase `{ id, segment_index, phrase, confidence, suggestion }`, plus a way to save the resolution (`accepted` \| `dismissed`) with the edit. If word-level confidence already exists, a threshold field is enough and the frontend can build the list.

---

## Questions and confirmations

1. **`hearing_date` format.** Does `/causelist` return wall-clock Lagos time (`2026-10-05T09:00:00`) or a UTC instant (`…Z`)? The calendar buckets hearings by the date part. If it's UTC, a hearing just after midnight lands on the previous day and times show one hour off.
2. **`/causelist` maximum `size`.** The Cause List and Calendar now request `size=500` (the old default of 50 could drop newly registered cases on a busy day). Is 500 accepted, or is there a lower cap? Both pages show a notice when `total_rows` is larger than what came back.
3. **New case and schedule status.** When a case is registered from the Cause List with a hearing date, is its schedule created as `SCHEDULED` straight away? The Cause List filters on `schedule_status=SCHEDULED`; a case created in another status won't appear.
4. **`COURT_BILLING_CALLBACK_URL`** must be `${APP_URL}/subscription/callback` (from the billing report, still to confirm on staging).
5. **CORS** (asked by backend): every API call from the court and platform dashboards is made server-side (Next.js server components and server actions), so the browser never calls the API directly and CORS doesn't apply to them. The exceptions are Paystack's own hosted pages and the public cause-list page; if that page's payment moves to REQ-B06's backend checkout, it will be server-side too.

---

## Frontend status (for visibility)

| Area | Frontend today |
|---|---|
| Court dashboard (Judge, Registrar, Legal Aide) | Built in v3 against existing endpoints. Court Calendar built from `/causelist`. Payments pages show empty states until REQ-B06. |
| Platform Admin dashboard | Built in v3. Financial Operations has Billing only. |
| Lawyer, Court Admin, Finance personas | Not built; blocked on V01. |
| Record Requests, booking, notifications, financial overviews | Not built; blocked on V03 – V10. |
