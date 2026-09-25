# Backend → Frontend: Reply to the Billing & Payments Integration Report

**In reply to:** `billing-integration-report.md` (2026-09-25)
**From:** Backend
**Date:** 2026-09-25
**Status:** Bugs fixed in code, waiting on a staging deploy. Several of your requests were already built after you wrote the report.

## 1. The two bugs

**BUG-01, missing tables on staging: fixed, needs a deploy.** The cause was that migrations only ran when the environment variable `RUN_MIGRATIONS=true` was set, and staging never had it. That gate is gone. Migrations now run every time the backend starts, so the next staging deploy creates `court_subscription_plans`, `court_subscriptions`, `invoices`, `court_payment_configs` and `dunning_settings` (and any other missing table) with no manual step. Startup takes a few seconds longer. After the deploy the logs show `[migrate] all 22 tables are up to date`, or name any table that failed. **Please rerun your test once staging has redeployed** and tell us what you see.

**BUG-02, `verify` returned a raw SQL error in `message`: fixed.** You were right, and other billing handlers had the same pattern. Across the subscription, plan, invoice and dunning endpoints, an unexpected error now returns a plain `message` ("Something went wrong on our side. Please try again.") with the technical detail in `data.error`. Two related changes to know about: errors we raise on purpose ("Plan not found", "A reason is required") no longer start with `Error: `, and a failed invoice reminder no longer includes the mail provider's error text. Keep your mitigation; it is harmless.

## 2. Your requests

| Request | Status |
|---|---|
| **B01** Invoices across all courts | **Built.** `GET /platform/court-invoices?court_id=&status=&search=&page=&size=`. Paginated; every row has `court_name` and `plan_name` (`plan_id` 0 for an ad-hoc invoice). Search matches the invoice reference or the court name. |
| **B02** Put a court on a plan without Paystack | **Built**, with a different route from the one you proposed: `POST /platform/court-subscriptions/{court_id}?action=record-payment`. Body: `plan_id` (required), optional `amount`, `paid_at`, `period_start`, `period_end`, `reference`, `reason`. It creates a paid, manual invoice and starts or extends the subscription if the period runs past the current one. A payment whose period has already ended is history only. Use it for a court's older payments too. |
| **B03** Billing history | **Built** as `GET /platform/subscription-ledger` (`court_id`, `action`, `search`, `from`, `to`, `page`, `size`): time, court, action, detail and actor. Court-level events only. |
| **B04** Dunning and Settings | **Dunning built:** `GET`/`PUT /platform/dunning-settings` (reminder days, final reminder, grace days, retries, suspend on/off, auto-reactivate on/off, require a reason on overrides). `PUT` replaces the whole set. **Settings tab not built:** payment provider status, default split, invoice numbering and terms, and the pause-all switch are still open. The dunning rules are no longer fixed at "3 daily retries": show what the endpoint returns. |
| **B05** Automatic suspension | **Built.** A daily job (07:30) moves a court to `GRACE` when its paid period ends, then to `SUSPENDED` when grace runs out. Admin overrides: `POST /platform/court-subscriptions/{court_id}?action=set-status\|extend\|remind`. |
| **B06** Court → Lawyer payments | **Not built.** This is a separate module and needs product decisions first (commission split, settlement flow, subaccounts). We agree the flat ₦5,000 from a browser env var on the public cause-list page is a problem: the price must come from the court's configuration and be checked on the server. |
| **B07** Payment configuration additions | **Not built.** Virtual hearing fee, bank list and account lookup, account-change requests and Paystack subaccounts wait on the B06 decisions. |
| **B08** Receipt PDF | **Not built.** The receipt endpoint still returns invoice details as JSON. |

## 3. Your questions

- **Q1, Paystack callback URL.** Checkout sends the value of the backend setting `COURT_BILLING_CALLBACK_URL` exactly as configured and adds nothing to it. Staging must have it set to `${APP_URL}/subscription/callback`. We will confirm the current value with whoever manages the staging environment. Paystack appends `?reference=…` (and `trxref`) itself.
- **Q2, read-only statuses.** The backend does enforce it: while `SUSPENDED`, create, edit and delete are refused on cases, casefiles, evidence, schedules, comments, media uploads and the bin. The response is **403** with `data.code = "SUBSCRIPTION_SUSPENDED"` (not 402 `SUBSCRIPTION_READ_ONLY`), so match on `data.code`. `GET` requests are never refused. `GRACE` is a warning only, as you have it. Whether read-only should start at the beginning of grace (as the PRD shows) is still a product decision. It is a one-line backend change once decided.
- **Q3, auth header.** `Authorization: Bearer <token>` is correct on every route. A bare token also works everywhere.
- **Q4, response envelopes.** `GET /court/subscription-plans` (flat list), `GET /platform/court-subscription-plans` and `GET /court/invoices` stay bare, unpaginated arrays. The newer list endpoints are paginated (`data`, `total_rows`, `total_pages`, `page`, `size`): `GET /court/subscription/payments`, `GET /platform/court-invoices`, `GET /platform/court-subscriptions` and its `/attempts`, and `GET /platform/subscription-ledger`.
- **Q5, reminders for courts that have never paid.** Fixed. A reminder goes to the court's billing contact if it has one, otherwise to the court's Judges (every active Judge with an email, once each). It is a 400 only if the court has neither.

## 4. Changes since you wrote the report

You have most of these in the changelog. The ones that affect the frontend:

- **The Paystack webhook is removed.** `POST /public/court-billing/paystack/webhook` no longer exists. Payments are confirmed by calling `POST /court/subscription/verify`, and while a bank transfer is pending the page should keep calling it (every few seconds at first, then about every 30 seconds). A server job also confirms it within the hour.
- **Verify returns `data.code`** on failure (`PAYMENT_PENDING`, `PAYMENT_LINK_EXPIRED`, `PAYMENT_DECLINED`, `AMOUNT_MISMATCH`, `NO_PAYMENT_FOUND`, `VERIFICATION_UNAVAILABLE`) and `already_confirmed` on success. It now only accepts the caller's own court's references.
- **New for the Subscription screen:** `GET /court/subscription-plans?view=cards`, `GET /court/subscription/payments`, and new fields on `GET /court/subscription`. See `court-subscription-screen-api.md`.
- **Grace and suspension fields** (`grace_started_at`, `grace_ends_at`, `read_only`): see `subscription-suspension-api.md`.
- **`paid_via`** is now `PAYSTACK_CARD`, `PAYSTACK_TRANSFER`, `PAYSTACK_OTHER`, `PAYSTACK_RECURRING`, `MANUAL` or `WAIVED`.
- **Courts and Court Configuration** are under `/platform` (your follow-up list has this).

## 5. One thing for you to check

The backend's CORS setting allows only the methods `GET, POST, PATCH, DELETE` and the headers `Origin, Content-Type, Accept`. It does not list `PUT` or `Authorization`. If the frontend calls the API straight from the browser, a `PUT` (payment configuration, dunning settings) or a request with an `Authorization` header could be blocked by the browser's preflight check. If your calls go through a server-side proxy this does not apply. Tell us which it is, and we will widen the setting if needed.
