# Changelog

## 2026-09-25 — Invoice list across courts, reminders reach a court's Judges, safe error messages

**Tag:** Web · NON-BREAKING for request shapes. One new endpoint, one behaviour change to reminders, and a change to error message text (below).

- **New: `GET /platform/court-invoices`** (Platform Admin) — every invoice across all courts, newest first, paginated, with `court_name` and `plan_name` on each row. Filters: `court_id`, `status` (`PENDING`, `OVERDUE`, `PAID`, `WAIVED`; anything else is a 400), `search` (invoice reference or court name), `page`, `size`. The admin Invoices tab can now list existing invoices instead of only ones raised in the current session.
- **Invoice reminders no longer fail for a court that has never paid.** `POST /platform/court-invoices/{id}/remind` goes to the court's billing contact (the email from its last payment) if it has one; otherwise to the court's Judges: every active Judge with an email, once each. It is a 400 only if the court has neither. It counts as sent if at least one address was reached.
- **Error messages are now safe to show.** A billing endpoint used to put the database error text in `message` when something unexpected went wrong (for example `POST /court/subscription/verify` returned `ERROR: relation "invoices" does not exist (SQLSTATE 42P01)`). Now `message` is a plain sentence ("Something went wrong on our side. Please try again.") and the technical detail is in `data.error`, as on the other billing endpoints. This covers the subscription, plan, invoice and dunning endpoints. Errors the backend raises on purpose ("Plan not found", "A reason is required") are unchanged, except that they **no longer start with "Error: "**. A failed reminder no longer includes the mail provider's error text.
- **Migrations run on every start.** The `RUN_MIGRATIONS` setting is gone: a redeploy always creates any missing table. This is what the staging "relation … does not exist" errors needed.

See `openapi.yaml` (`Court Invoices` tag).

## 2026-09-25 — Court subscription screen, recording offline payments, ledger and confirmation email

**Tag:** Web · NON-BREAKING for request shapes. Additive fields and new endpoints, plus two behaviour changes flagged below.

**Court side (the Subscription screen)**
- `GET /court/subscription-plans?view=cards` — plans grouped into one card per plan family, with a price per billing cycle and `offered: false` ("Not offered") for a missing cycle. Each offered cycle carries the `plan_id` to send to checkout. Without `view` the flat list is unchanged.
- `GET /court/subscription/payments` — the court's own payment history, paginated, newest first: plan and cycle, date, method (`method_label` is words such as "Paystack · card" or "Manual · recorded by Devon"), amount and the period end ("through"). Confirmed subscription payments only.
- `POST /court/subscription/verify` — a failed verify now returns `data.code`: `NO_PAYMENT_FOUND`, `PAYMENT_PENDING`, `PAYMENT_LINK_EXPIRED`, `PAYMENT_DECLINED`, `AMOUNT_MISMATCH` or `VERIFICATION_UNAVAILABLE`, with a message fit to show the court. A success now includes `already_confirmed` (true for a reload or a second caller).
- `GET /court/subscription` — new fields: `billing_cycle`, `start_date`, `days_left`, `days_overdue`, `last_payment_method`, `last_payment_label`, `last_paid_at`, `reminder_sent_at`.
- **Plan family and description.** Plans take an optional `family` and `description` (create, edit and duplicate). Plans that share a family appear as one card. Two published plans cannot share a family and billing cycle.

**Platform Admin side**
- `POST /platform/court-subscriptions/{court_id}?action=record-payment` — record a payment a court made outside the platform, including one from before it joined JudicAI. Body: `plan_id` (required), optional `amount`, `paid_at`, `period_start`, `period_end`, `reference`, `reason`. It creates a paid, manual invoice and audit entry. If the period runs past the court's current one, the subscription starts or extends. A payment whose period has already ended is history only, and for a court with no subscription it creates none (a subscription that is already lapsed would only get the court suspended).
- `GET /platform/subscription-ledger` — everything that has happened to courts' subscriptions and billing, newest first: payments, renewals, failures, grace, suspensions, overrides, reminders, invoice actions. Filters: `court_id`, `action`, `search`, `from`, `to`, `page`, `size`.
- Confirmed payments now write a "Payment Recorded (Online)" or "Payment Recorded (Manual)" line to the audit trail, with the plan, cycle, amount, method and period end.

**Email.** After a successful Paystack payment (card, transfer or automatic charge), every active Judge, Registrar and Legal Aide of the court receives a payment confirmation in the approved HTML style. It is sent once, in the background, and a failure never affects the payment. A payment an admin records by hand is not announced. The email shows VAT and Paystack-fee rows only when those amounts are recorded; today they are not, so it shows a single "Amount paid". Renewal reminders now link with `?plan_id=…&cycle=…`.

**Behaviour changes**
- **Verify is now limited to the caller's own court.** Before, a user could verify another court's payment reference and see that court's subscription. Another court's reference now gets `NO_PAYMENT_FOUND`.
- **Bin.** A suspended court can no longer restore or permanently delete items from the bin.

**Flowchart:** new `flow/07-court-subscription-screen.html`. Frontend guide: `frontend/court-subscription-screen-api.md`.

See `openapi.yaml` (`Court Subscription Billing` tag).

## 2026-09-25 — Paystack payments are confirmed by verifying, not by webhook

**Tag:** Web · **BREAKING** for one endpoint: `POST /public/court-billing/paystack/webhook` is removed. If you registered that URL in the Paystack dashboard, delete it: nothing calls it any more. Everything else keeps its shape.

**Why.** The Paystack account is shared with other apps. An account has one webhook URL per mode, so this app could not rely on receiving events. Payments are now confirmed only by asking Paystack about the payment's reference (`GET /transaction/verify/{reference}`), which only ever returns this app's own transactions.

**What changed**
- **Automatic charges** are verified straight after they are made. Confirmed, failed, or (if Paystack is still processing) left pending for the reconcile job.
- **New reconcile job**, hourly and again daily at 07:25 (before the 07:30 dunning job), verifies every pending subscription attempt with Paystack. It confirms a bank transfer that landed after the court left the page, and settles automatic charges that were still processing.
- **Unresolved automatic charges:** still pending after 24 hours counts as a failed attempt. A charge Paystack has no record of is closed without counting against the court, because the card was not declined.
- **A court that has already paid can no longer be moved to grace** while its payment waits to be confirmed: the reconcile job runs before the dunning job.
- **Confirming is now atomic.** If the court's page and the job confirm the same payment at once, the subscription is extended once.

**For the frontend.** While a bank transfer is pending, keep calling `POST /court/subscription/verify` (every few seconds at first, then about every 30 seconds). A `400 Payment not completed` means not paid yet. It is safe to call repeatedly. The server also confirms the payment within the hour if the court stops polling.

See `openapi.yaml` (`/court/subscription/verify`) and `flow/03-recurring.html`.

## 2026-09-25 — Dunning settings, grace and suspension (Workflow 5)

**Tag:** Web · NON-BREAKING for request and response shapes (new endpoints and additive fields). **Behaviour change:** a court whose subscription becomes `SUSPENDED` now gets `403` on creating, editing and deleting case records (see below). Reading is never restricted, and courts with no subscription are unaffected.

**New endpoints (Platform Admin only)**
- `GET /platform/dunning-settings` and `PUT /platform/dunning-settings` — the platform-wide rules: `reminder_days`, `final_reminder_days`, `grace_days`, `retry_attempts`, `suspend_on_grace_expiry`, `auto_reactivate_on_payment`, `require_override_reason`. `PUT` replaces the whole set. Until first saved the defaults apply (14, 3, 3, 3, all three switches on) and `is_default` is `true`.
- `POST /platform/court-subscriptions/{court_id}?action=set-status | extend | remind` — manual overrides on one court, each audit-logged with the admin and a reason. `set-status` forces `ACTIVE`, `GRACE` or `SUSPENDED`; `extend` pushes the paid period out by `days` (1 to 365); `remind` emails the court's staff a renewal reminder now.

**Subscription lifecycle** — a daily job (07:30) now drives the status. Before this, a manual subscription stayed `ACTIVE` forever after its end date and `SUSPENDED` could not happen.
- A paid period that ends with no payment moves the court to `GRACE`. An automatic subscription is given a day for its charge to land first.
- `GRACE` is a warning window and restricts nothing. When `grace_days` run out, and `suspend_on_grace_expiry` is on, the court moves to `SUSPENDED`.
- A confirmed payment returns the court to `ACTIVE` at once. If `auto_reactivate_on_payment` is off, the payment is still recorded and the period extended, but the court stays suspended until an admin sets it back.
- The retry limit for automatic charges is now `retry_attempts` (it was a fixed 3).

**Suspension: what is paused.** While `SUSPENDED`, `POST`, `PUT`, `PATCH` and `DELETE` on cases, casefiles, evidence, schedules, case comments and media uploads return `403` with `data.code = "SUBSCRIPTION_SUSPENDED"`. `GET` requests, downloads, Platform Admins and courts that never subscribed are never blocked.

**Renewal reminders.** The daily job emails every active Judge, Registrar and Legal Aide of a court `reminder_days` before its paid period ends, and again `final_reminder_days` before (`0` = no final reminder), once per paid period each. These are plain HTML emails in the approved platform style, not Mailjet templates. The button links to `COURT_BILLING_RENEW_URL` (default `https://judicai.devontech.io/subscription`) with `?plan_id=`.

**New fields on `GET /court/subscription`:** `grace_started_at`, `grace_ends_at` and `read_only`. All three are additive.

**New setting:** `COURT_BILLING_RENEW_URL` (optional).

**Not included:** a global "pause all billing" switch (the Settings tab), and a separate override to waive a cycle's fee: use `POST /platform/court-invoices/{id}/waive`.

**Flowchart:** new `flow/06-dunning-suspension.html`. Frontend guide: `frontend/subscription-suspension-api.md`.

See `openapi.yaml` (`Court Subscription Billing` tag; `DunningSettings`, `SubscriptionOverrideRequest`, `SubscriptionStatus`).

## 2026-09-25 — Platform Admin subscription views

**Tag:** Web · NON-BREAKING — three new read-only endpoints for Platform Admins, plus two additive fields. No existing contract changed.

- `GET /platform/court-subscriptions` — every court, subscribed or not, with its current plan, `start_date`, `end_date`, `status`, `renewal_mode` and whether a card is saved. A court that has never subscribed has `status: NONE` and empty subscription fields. Filters: `search` (court name or code), `status` (`ACTIVE`, `GRACE`, `SUSPENDED` or `NONE`), `plan_id`, `page`, `size`. An unknown `status` returns 400.
- `GET /platform/court-subscriptions/{court_id}` — one court's current subscription plus its history: every confirmed subscription payment, newest first, with the plan, amount, `paid_via`, `paid_at` and the period it covered. Unknown court returns 400.
- `GET /platform/court-subscriptions/{court_id}/attempts` — every checkout and automatic charge the court has made, paid or not, newest first and paginated, with a `summary` (total, paid, pending, overdue) across the whole history. `kind` is `CHECKOUT` or `AUTOMATIC`. `PENDING` means the court abandoned checkout, or a bank transfer hasn't landed yet; `OVERDUE` means an automatic charge failed. Ad-hoc invoices are not attempts.

**Data now recorded (needed for the dates above).** A confirmed subscription payment now stores when it was paid and the period it covers (`paid_at`, `period_start`, `period_end` on the invoice), and the subscription stores the start of its current paid period. Payments confirmed before this release have none of these, so their history entries show `null` dates, and a subscription without a recorded period start reports `start_date` as `subscribed_since`.

**Flowchart:** `flow/04-invoices.html` is now `flow/04-platform-subscriptions.html` and covers these views as well as invoices.

See `openapi.yaml` (`Court Subscription Billing` tag; `CourtSubscriptionOverview`, `CourtSubscriptionDetailResponse`, `CourtSubscriptionAttemptsResponse`).

## 2026-09-25 — Courts and Court Configuration moved under `/platform`

**Tag:** Web · **BREAKING** — the old paths return 404. The Platform Admin panel (`LSP-frontend`, `PlatformService`) must be updated together with this release.

This completes the convention that every endpoint only a Platform Admin can call lives under `/api/v1/platform/...`. Roles, request bodies and responses are unchanged.

| Old | New |
|---|---|
| `GET /courts` | `GET /platform/courts` |
| `POST /courts` | `POST /platform/courts` |
| `GET /courts/constants` | `GET /platform/courts/constants` |
| `POST /courts/onboard` | `POST /platform/courts/onboard` |
| `GET /courts/{id}` | `GET /platform/courts/{id}` |
| `PATCH /courts/{id}` | `PATCH /platform/courts/{id}` |
| `DELETE /courts/{id}` | `DELETE /platform/courts/{id}` |
| `GET /court_config` | `GET /platform/court-configurations` |
| `POST /court_config` | `POST /platform/court-configurations` |
| `GET /court_config/{id}` | `GET /platform/court-configurations/{id}` |

Note: `openapi.yaml` already documented the configuration routes as `/courts/configurations`, which the server never served. The spec now matches the server.

**Not moved:** `/users` and the court-facing groups (`/court/...`, `/metric`, `/bin`, cases, casefiles, evidence, uploads, schedules), which court staff also use.

## 2026-09-25 — Bank transfer payments and `paid_via`

**Tag:** Web · **BREAKING** for one field value: `paid_via` no longer returns `PAYSTACK_ONE_TIME`. Nothing has consumed it yet; treat any stored or hard-coded `PAYSTACK_ONE_TIME` as `PAYSTACK_CARD`, `PAYSTACK_TRANSFER` or `PAYSTACK_OTHER`.

- **`paid_via` now says how the payment was made.** Checkout payments are recorded by the Paystack channel the court used:

| Value | Meaning |
|---|---|
| `PAYSTACK_CARD` | Checkout, paid by card |
| `PAYSTACK_TRANSFER` | Checkout, paid by bank transfer |
| `PAYSTACK_OTHER` | Checkout, USSD, QR, mobile money or another channel |
| `PAYSTACK_RECURRING` | Automatic charge of a saved card |
| `MANUAL` | A Platform Admin recorded the payment (unchanged) |
| `WAIVED` | No money collected (unchanged) |

- **Courts can pay by bank transfer.** Nothing extra is needed. Only a card can be saved for auto-renewal, so a court that pays by transfer has `has_saved_card: false`, and `POST /court/subscription/renewal-mode` with `AUTO` is refused (400). It stays on manual renewal and pays each cycle through checkout.
- **A transfer that lands after the court leaves the page** is now confirmed by the Paystack webhook, the same way as any other payment, and the subscription activates on its own. Previously the webhook labelled these as recurring charges. ~~**Superseded 2026-09-25:** there is no webhook now; the reconcile job confirms these by verifying with Paystack.~~
- **Clearer error** when a court tries to switch to `AUTO` without a card: "Auto-renew needs a saved card. Pay once by card to enable it — bank transfer payments can't be renewed automatically".

See `openapi.yaml` (`PaymentMethod` schema, `renewal-mode` and the webhook).

## 2026-09-25 — Platform Admin billing endpoints moved under `/platform`

**Tag:** Web · **BREAKING** — every Platform Admin-only billing URL changes. Requests to the old paths return 404. Platform Admin billing screens (plans, invoices) must be updated together with this release; no external consumer is known.

Convention going forward: **every endpoint only a Platform Admin can call lives under `/api/v1/platform/...`.** This release applies it to the billing endpoints built so far; the older Platform Admin endpoints (`/courts`, `/court_config`, ...) will be migrated in a follow-up and are unchanged here.

| Old | New |
|---|---|
| `POST /court/subscription-plans` | `POST /platform/court-subscription-plans` |
| `GET /court/subscription-plans/admin` | `GET /platform/court-subscription-plans` (lists every status; the `/admin` suffix is gone) |
| `PATCH /court/subscription-plans/{id}` | `PATCH /platform/court-subscription-plans/{id}` |
| `POST /court/subscription-plans/{id}?action=publish\|archive\|duplicate` | `POST /platform/court-subscription-plans/{id}?action=publish\|archive\|duplicate` |
| `DELETE /court/subscription-plans/{id}` | `DELETE /platform/court-subscription-plans/{id}` |
| `POST /court/invoices` | `POST /platform/court-invoices` |
| `POST /court/invoices/{id}/remind` | `POST /platform/court-invoices/{id}/remind` |
| `POST /court/invoices/{id}/mark-paid` | `POST /platform/court-invoices/{id}/mark-paid` |
| `POST /court/invoices/{id}/waive` | `POST /platform/court-invoices/{id}/waive` |

**Unchanged (court-facing):** `GET /court/subscription-plans` (published plans), `GET /court/invoices`, `GET /court/invoices/{id}/receipt`, `/court/subscription`, `/court/subscription/checkout|verify|renewal-mode`, and the public Paystack webhook. Request and response bodies, roles and behavior are identical.

**Flowcharts renamed:** `flow/platform-admin-plans-workflow.html` is now `flow/01-platform-admin-plans-workflow.html` and `flow/02-checkout.html` is now `flow/02-court-checkout.html`.

See `openapi.yaml` (`Court Subscription Billing` and `Court Invoices` tags).

## 2026-09-25 — Plan `created_by` and duplicate as a query action

**Tag:** Web · **BREAKING** for `POST /court/subscription-plans/{id}/duplicate` only (Platform Admin plan screens; no external consumer known). The `created_by` field is additive.

- **Duplicate moved to the query parameter.** Removed: `POST /court/subscription-plans/{id}/duplicate`. Now: `POST /court/subscription-plans/{id}?action=duplicate`, still `201` with the new draft plan. `action` now accepts `publish`, `archive` or `duplicate`; anything else is a 400.
- **New: `created_by` on the plan response.** A snapshot of the Platform Admin who created the plan (id, first/last name, email, court_id, user_type), taken at that moment; it does not follow later changes to that user. A duplicate records the admin who duplicated it. Edit, publish and archive leave it alone. Plans created before this change have no `created_by` (the field is omitted).

See `openapi.yaml` (`Court Subscription Billing` tag, `CourtSubscriptionPlanResponse`).

## 2026-09-25 — Plans workflow fixes

**Tag:** Web · **BREAKING** for anything calling the two publish/archive URLs below (Platform Admin plan screens only; no external consumer known)

- **Publish and archive moved from path segments to a query parameter.**
  - Removed: `POST /court/subscription-plans/{id}/publish`
  - Removed: `POST /court/subscription-plans/{id}/archive`
  - Now: `POST /court/subscription-plans/{id}?action=publish` and `?action=archive`. `action` is required; a missing or unknown value returns 400.
- **Publish approval removed.** Publishing a plan priced above ₦200,000 no longer needs a second, different Platform Admin. Publishing is one call from one admin at any price. The `pending_approval` field is gone from the plan response.
- **New: `DELETE /court/subscription-plans/{id}`** — deletes a plan, but only while it is still a `DRAFT` (soft delete). A plan that has been published, even if archived since, can only be archived.
- **Flowchart:** `flow/01-plans.html` is now `flow/01-platform-admin-plans-workflow.html` and covers only the Platform Admin side of a plan (create, edit, publish, duplicate, archive, delete). The court-facing browse/checkout steps live in the checkout flowchart.

Unchanged: `POST /court/subscription-plans`, `GET /court/subscription-plans`, `GET /court/subscription-plans/admin`, `PATCH /court/subscription-plans/{id}`. (`/duplicate` moved later the same day — see the entry above.)

See `openapi.yaml` (`Court Subscription Billing` tag).

## 2026-09-24 — Court Subscription Billing (Workflow 4: Plans lifecycle)

**Tag:** Web · NON-BREAKING (behavioral change to `POST /court/subscription-plans` — see below)

**Plan lifecycle** — plans are no longer flat CRUD:
- `POST /court/subscription-plans` — **behavior changed**: now always creates a `DRAFT` plan (previously created a live, immediately-selectable plan). Not selectable by any court until published.
- `GET /court/subscription-plans` — **behavior changed**: now only returns `PUBLISHED` plans (previously returned all statuses).
- `GET /court/subscription-plans/admin` — new. All statuses, Platform Admin only, for plan management screens.
- `PATCH /court/subscription-plans/{id}` — new. Edit name/price/cycle/waiver fields; never retroactively changes an already-locked-in subscriber's fee.
- ~~`POST /court/subscription-plans/{id}/publish` — new. Idempotent. Plans priced above ₦200,000 require a second, different Platform Admin to confirm before going live (the first call records the request and returns `pending_approval: true`).~~ **Superseded 2026-09-25** — publishing is now `POST /court/subscription-plans/{id}?action=publish`, one call at any price, no approval. See the entry above.
- ~~`POST /court/subscription-plans/{id}/archive`~~ **Moved 2026-09-25** to `POST /court/subscription-plans/{id}?action=archive`. Stops new selection; never force-migrates existing subscribers, who may still renew the same archived plan.
- ~~`POST /court/subscription-plans/{id}/duplicate`~~ **Moved 2026-09-25** to `POST /court/subscription-plans/{id}?action=duplicate`. Copies a plan into a new draft.
- Time-capped/waiver plans (`is_waiver`, `waiver_cap_days`, `reverts_to_plan_id`) — a subscriber on a waiver plan auto-reverts to the fallback plan once the cap elapses (daily cron).
- `POST /court/subscription/checkout` — **behavior changed**: now rejects a non-`PUBLISHED` plan unless the caller's current subscription is already on that exact plan (renewal case).

**Why this is tagged NON-BREAKING despite the behavior changes:** no consumer has integrated against the old create/list behavior yet (Workflows 1-3 shipped same-day, 2026-09-24) — flagging the shape of the change here for the record, not because it broke a live integration.

See `openapi.yaml` (`Court Subscription Billing` tag, `PlanStatus`/`CourtSubscriptionPlanRequest`/`CourtSubscriptionPlanResponse` schemas) for the full contract.

## 2026-09-24 — Court Subscription Billing (Workflows 1–3)

**Tag:** Web · NON-BREAKING

New endpoints — no existing contract changed. This is the **Devon → Court** subscription/billing system — distinct from Court Payment Configuration above, which is **Court → Lawyer**. They share no data and don't block each other.

**Subscription Plans & Checkout**
- `GET/POST /court/subscription-plans` — list (Judge/Legal Aide/Registrar/Platform Admin) / create (Platform Admin only) plans.
- `GET /court/subscription` — the caller's own court subscription status.
- `POST /court/subscription/checkout` — start paying for a plan/cycle; backend-initiated Paystack transaction, returns a redirect URL.
- `POST /court/subscription/verify` — confirm payment after the Paystack redirect; idempotent, amount-checked.
- `POST /court/subscription/renewal-mode` — switch between `MANUAL` and `AUTO` renewal. Never charges the court — `AUTO` only reuses a card already on file.
- ~~`POST /public/court-billing/paystack/webhook`~~ **Removed 2026-09-25** — payments are confirmed by verifying with Paystack instead. See the entry above.

**Invoices**
- `GET/POST /court/invoices` — list (court-scoped) / raise an ad-hoc invoice (Platform Admin only).
- `GET /court/invoices/{id}/receipt` — paid invoice details, tenant-scoped.
- `POST /court/invoices/{id}/remind` — on-demand reminder (Platform Admin only).
- `POST /court/invoices/{id}/mark-paid` — record a manual/offline payment, idempotent (Platform Admin only).
- `POST /court/invoices/{id}/waive` — zero an invoice with a logged reason (Platform Admin only).

**Known limitations, not oversights:**
- ~~Recurring billing (`AUTO`) has no admin-configurable retry/grace settings yet~~ **Superseded 2026-09-25** — see Workflow 5 above.
- ~~`SUSPENDED` status is not yet reachable — automatic suspension after grace expires isn't built.~~ **Superseded 2026-09-25** — see Workflow 5 above.
- ~~Plan lifecycle (Draft/Publish/Archive/Duplicate) isn't built — `POST /court/subscription-plans` always creates a live, immediately-selectable plan.~~ **Superseded** — see the Workflow 4 entry above, same date.
- Invoice reminders only reach a court that has paid at least once (recipient is captured from the last successful payment) — no broader court-staff lookup yet.
- No receipt PDF generation — the receipt endpoint returns invoice details only.

See `openapi.yaml` (`Court Subscription Billing` and `Court Invoices` tags) for the full contract.

## 2026-09-01 — Court Payment Configuration

**Tag:** Web · NON-BREAKING

New endpoints — no existing contract changed.

- `GET /court/payment_config` — fetch the caller's court payment configuration (price per page, CTC price, payout bank account). Accessible to Judge, Legal Aide, Registrar, and Platform Admin users.
- `PUT /court/payment_config` — set or update it. Prices are always editable; the payout `account_number` can only be set once per court — a second attempt to change it is rejected with a 400.

See `openapi.yaml` (`Court Payment Configuration` tag) for the full contract, and `frontend/court-payment-config-api.md` for a frontend-facing walkthrough with request/response examples.
