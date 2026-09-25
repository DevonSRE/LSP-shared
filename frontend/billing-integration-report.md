# Frontend → Backend: Billing & Payments Integration Report

**Module:** Court Subscription Billing, Court Invoices, Court Payment Configuration, Court → Lawyer payments
**Raised by:** Frontend team
**Date:** 2026-09-25
**Status:** Open
**Source documents:** `openapi.yaml` (Court Subscription Billing, Court Invoices, Court Payment Configuration tags), `changelog.md` (entries up to 2026-09-25, "Platform Admin subscription views"), `guide/PRD-court-subscription-billing.md`

The frontend has integrated every Court Subscription Billing, Court Invoices and Court Payment Configuration endpoint documented in `openapi.yaml` as of the 2026-09-25 morning releases, and ran an end-to-end test through the real UI against staging (`https://api.staging1.lsp.devontech.io/api/v1`) with a Platform Admin and a court (Judge) account. Every request below was recorded with its full body and response; the raw evidence is in the [Appendix](#appendix--raw-requestresponse-evidence).

**Result in one line:** the routes are deployed and accept our tokens, but **every call fails because the database migrations have not run on staging**. Nothing beyond that point can be verified until they do. Separately, some screens need endpoints that don't exist yet (section 3).

---

## 1. Failing on staging (blocking)

### BUG-01 — Billing and payment-config tables missing on staging

Every billing endpoint returns **400** with `ERROR: relation "<table>" does not exist (SQLSTATE 42P01)`. The routes exist (a Judge calling an admin route gets the correct 401 "required PLATFORM_ADMIN"), so this is a migration issue, not a deployment one.

| # | Endpoint | Called as | Status | Missing table | Evidence |
|---|---|---|---|---|---|
| 1 | `GET /platform/court-subscription-plans` | Platform Admin | 400 | `court_subscription_plans` | [E1](#e1-get-platformcourt-subscription-plans--platform-admin) |
| 2 | `POST /platform/court-subscription-plans` | Platform Admin | 400 | `court_subscription_plans` | [E2](#e2-post-platformcourt-subscription-plans--platform-admin) |
| 3 | `POST /platform/court-invoices` | Platform Admin | 400 | `invoices` | [E3](#e3-post-platformcourt-invoices--platform-admin) |
| 4 | `GET /court/subscription` | Judge | 400 | `court_subscriptions` | [E4](#e4-get-courtsubscription--court-user-judge) |
| 5 | `GET /court/payment_config` | Judge | 400 | `court_payment_configs` | [E5](#e5-get-courtpayment_config--court-user-judge) |
| 6 | `GET /court/subscription-plans` | Judge | 400 | `court_subscription_plans` | [E6](#e6-get-courtsubscription-plans--court-user-judge) |
| 7 | `GET /court/invoices` | Judge | 400 | `invoices` | [E7](#e7-get-courtinvoices--court-user-judge) |
| 8 | `POST /court/subscription/verify` | Judge | 400 | `invoices` | [E8](#e8-post-courtsubscriptionverify--court-user-judge) |

**Ask:** run the migrations on staging for `court_subscription_plans`, `court_subscriptions`, `invoices` and `court_payment_configs`, then tell the frontend team so the test can be rerun.

**Re-tested 2026-09-25 13:04 UTC (court endpoints #4–#8, as a Judge): unchanged.** All five still return 400 with the same missing-table errors, and `verify` still returns the SQL error in `message` (BUG-02). The Platform Admin endpoints (#1–#3) were not re-tested in this pass.

**Impact while unfixed:** `GET /court/subscription` runs on every court page load (it decides whether a court is read-only). It fails on every page, so the frontend falls back to "unrestricted" by design — the subscription lockout cannot work on staging until this is fixed.

### BUG-02 — `POST /court/subscription/verify` returns a raw database error in `message`

```json
{ "status": false, "message": "ERROR: relation \"invoices\" does not exist (SQLSTATE 42P01)", "data": null }
```

Every other billing endpoint returns a user-safe `message` ("Error fetching plans", "Error raising invoice", …) and keeps the database detail in `data.error`. `verify` puts the SQL error in `message`. The contract says backend messages are safe to show users verbatim, so the frontend was displaying this to a court on the payment confirmation screen.

**Frontend mitigation (shipped):** messages that look technical (`SQLSTATE`, `ERROR:`, `relation "…" does not exist`, …) are replaced with plain text on every billing and payments screen.
**Ask:** return a user-safe `message` from `verify` in every error path (and audit the other billing handlers for the same pattern).

---

## 2. Not yet exercised (blocked by BUG-01)

These need a plan, invoice, subscription or saved configuration to exist first. The frontend calls are implemented and will be tested once BUG-01 is fixed.

| Endpoint | Frontend UI action |
|---|---|
| `PATCH /platform/court-subscription-plans/{id}` | Billing → Plans → Edit |
| `POST /platform/court-subscription-plans/{id}?action=publish\|archive\|duplicate` | Plans → Publish / Archive / Duplicate |
| `DELETE /platform/court-subscription-plans/{id}` | Plans → Delete (drafts only) |
| `POST /platform/court-invoices/{id}/remind` | Invoices → Remind |
| `POST /platform/court-invoices/{id}/mark-paid` | Invoices → Mark paid |
| `POST /platform/court-invoices/{id}/waive` | Invoices → Waive (asks for a reason) |
| `PUT /court/payment_config` | Payments → Payment Configuration → Save |
| `POST /court/subscription/checkout` | Settings → Billing → Pay now |
| `POST /court/subscription/renewal-mode` | Settings → Billing → Automatic renewal |
| `GET /court/invoices/{id}/receipt` | Not wired into the UI yet (frontend gap) |
| `POST /public/court-billing/paystack/webhook` | Called by Paystack, not the frontend |

---

## 3. Missing endpoints (requests)

Screens designed for the billing and payments modules that have no documented endpoint. Each request lists what the frontend shows in the meantime. Items marked **acknowledged** are already listed as known limitations in `changelog.md`; they're included so the full list lives in one place.

### REQ-B01 — List invoices across all courts (Platform Admin)

**Screen:** Platform Admin → Billing → Invoices.
**Problem:** `GET /court/invoices` is scoped to the caller's own court. `POST /platform/court-invoices` exists, but there is no list, so the admin Invoices tab can't show existing invoices, and Remind / Mark paid / Waive can only be used on invoices raised in the current browser session.

**Requested endpoint:**

```yaml
/platform/court-invoices:
  get:
    tags: [Court Invoices]
    summary: List invoices across all courts
    description: Platform Admin only. Newest first.
    parameters:
      - { name: court_id, in: query, required: false, schema: { type: integer, format: int64 } }
      - { name: status, in: query, required: false, schema: { $ref: "#/components/schemas/InvoiceStatus" } }
      - { name: search, in: query, required: false, schema: { type: string }, description: Court name or invoice ref }
      - { name: page, in: query, required: false, schema: { type: integer } }
      - { name: size, in: query, required: false, schema: { type: integer } }
    responses:
      "200":
        description: Paginated invoice list; each row includes court_name
```

**Frontend until then:** the tab lists only invoices raised in the current session, with a note explaining why.

### REQ-B02 — Put a court on a plan without Paystack (backfill / offline payment)

**Screen:** Platform Admin → Billing → Subscriptions → "+" (assign a court to a plan).
**Problem:** Courts already paying through the old invoice-and-transfer process need to be brought onto a plan. `mark-paid` only activates a subscription when the invoice is tied to a subscription cycle, and the only way to create such an invoice is a court-initiated checkout. An ad-hoc invoice (`POST /platform/court-invoices`) has no `subscription_id`, so marking it paid does not start a subscription.
**Requested:** a Platform Admin endpoint that creates or updates a court's subscription for a given plan and period (e.g. `POST /platform/court-subscriptions/{court_id}` with `plan_id`, `period_start`, `period_end`, `paid_via: MANUAL`, optional `reference`/`note`), recorded in the audit trail.
**Frontend until then:** the "+" on the Subscriptions tab is disabled.

### REQ-B03 — Billing history / audit trail (Platform Admin)

**Screen:** Platform Admin → Billing → History (date, court, event, detail, actor).
**Problem:** Events such as plan created/edited/published, invoice raised/waived/marked paid, subscription activated/renewed, charge failed and suspension are recorded server-side ("recorded in the audit trail") but can't be read.
**Requested:** `GET /platform/billing-events` (or `/auditlog` filterable by entity type), paginated, filterable by court and event type.
**Frontend until then:** layout only, with a pending note.

### REQ-B04 — Billing settings and dunning rules (acknowledged — Workflow 5)

**Screens:** Platform Admin → Billing → Dunning and → Settings.
**Needed:** read/update reminder days, grace length, retry count and suspension behaviour; payment provider status and default settlement split; invoice numbering and terms; a "pause all billing" switch.
**Frontend until then:** Dunning shows the rules as the backend currently applies them (07:00 billing run, 3 daily retries, 06:30 waiver check); Settings is layout only.

### REQ-B05 — Automatic suspension (acknowledged)

`SUSPENDED` is documented as not yet reachable. The frontend already treats `SUSPENDED` as read-only (with a banner and renew link) and `GRACE` as a warning only. See Q2 below — this needs a product decision.

### REQ-B06 — Court → Lawyer payments: transactions, settlements, payments audit

**Screens:** Court dashboard → Payments → Transactions, Settlements, Audit.
**Problem:** Only `GET/PUT /court/payment_config` exists for the Court → Lawyer side. Nothing records or lists what lawyers pay.
**Requested:**

| Endpoint | Purpose |
|---|---|
| `POST /public/court-payments/checkout` (or similar) | Backend-priced checkout for a proceeding copy, CTC or virtual hearing, using the court's `payment_config` prices. Today the public cause-list page charges a **flat ₦5,000 from a frontend env var, directly from the browser** — the price should come from the court's configuration and be verified server-side. |
| `GET /court/payments/transactions` | Court-scoped list: case no., case title, designation, service, amount, status (`PENDING`/`CONFIRMED`/`FAILED`), date; filter by service/status/date; plus a detail view (reference, lawyer). |
| `GET /court/payments/settlements` | Court-scoped settlement log: gross, commission, court share, status (`PENDING`/`COMPLETED`/`MANUAL_REVIEW`), date. |
| `GET /court/payments/audit` | Court-scoped events: fees updated, bank account registered, payment confirmed, settlement completed; actor and role. (Or `/court/auditlog` filterable by entity type.) |

**Frontend until then:** all three pages render their table layout with an empty state.

### REQ-B07 — Payment configuration additions

| Item | Detail |
|---|---|
| Virtual hearing fee | The design has a third price (Virtual Hearing fee); `CourtPaymentConfigRequest` only has `price_per_page` and `ctc_price`. |
| Bank list + account name lookup | So the bank is picked from a list and the account can show as "Verified" (e.g. Paystack resolve). Bank name is free text today. |
| Settlement account change request | The account is set-once with "contact support" as the only path. The design has "Request change" → Platform Admin approval. Needs a request endpoint plus an admin approve/reject. |
| Paystack subaccount per court | Settlements (REQ-B06) depend on a subaccount/split per court, presumably created when the account is saved. |

### REQ-B08 — Receipt document (acknowledged)

`GET /court/invoices/{id}/receipt` returns invoice details only. A downloadable receipt (PDF) would let courts file proof of payment.

---

## 4. Questions and confirmations

| # | Question |
|---|---|
| Q1 | **Paystack `callback_url`.** Checkout must send courts back to `${APP_URL}/subscription/callback`. The session cookie is `SameSite=Strict`, so returning to any `/dashboard/*` URL lands the court on the login page and the reference is lost. What does checkout currently send? |
| Q2 | **Which statuses are read-only?** The PRD makes a court read-only from the start of the grace period (old `EXPIRED`); the backend flow shows lockout only at `SUSPENDED`. The frontend follows the backend: `GRACE` = warning, writes allowed; `SUSPENDED` = read-only. Please confirm with product, and confirm the backend will reject writes (402 `SUBSCRIPTION_READ_ONLY`) from `SUSPENDED` courts on cases, case files, evidence and schedules — the frontend guard alone is not a security boundary. |
| Q3 | **Auth header on the new `/platform/court-*` routes.** The frontend sends `Authorization: Bearer <token>` (same as `/court/*`); the older `/platform/*` routes accept a bare token. Confirm Bearer is correct for the new routes. |
| Q4 | **Response envelopes.** The frontend reads single resources and plan/invoice lists as **bare** (per the spec), and tolerates a `{status, message, data}` envelope. Confirm lists stay bare arrays (not paginated) for `GET /court/subscription-plans`, `GET /platform/court-subscription-plans` and `GET /court/invoices`. |
| Q5 | **Invoice reminders for courts that have never paid** return 400 (no billing contact). Is there a plan to fall back to the court's Judge/Registrar emails? Admins will hit this on the first invoice for every new court. |

---

## 5. Frontend status (for visibility)

**Implemented and waiting on BUG-01:** every endpoint in sections 1–2 except the receipt.

**Frontend follow-ups from the 2026-09-25 afternoon releases** (not backend asks — listed so both teams see the same picture):

- **Courts and Court Configuration moved under `/platform`.** `PlatformService` still calls `/courts…` and `/court_config`, which now return 404 — this is why the Billing invoice court picker was empty during the test (worked around by reading courts from `/platform/analytics/group`). The Courts screens must move to `/platform/courts` and `/platform/court-configurations`.
- **New Platform Admin subscription views** (`GET /platform/court-subscriptions`, `/{court_id}`, `/{court_id}/attempts`) — to be wired into Billing → Subscriptions, the KPI tiles and the court drawer. This closes the frontend's earlier "no cross-court subscription list" gap.
- **`paid_via` values** — `PAYSTACK_ONE_TIME` replaced by `PAYSTACK_CARD` / `PAYSTACK_TRANSFER` / `PAYSTACK_OTHER`; labels to update. The auto-renew switch's disabled hint will mention that bank-transfer payments can't be renewed automatically.
- **Receipt** — add "Receipt" on paid invoices in Settings → Billing.

---

## Appendix — raw request/response evidence

Recorded by the frontend's development-only API logger during the UI test on 2026-09-25 (times in UTC). Authorization headers are never recorded.

### E1. `GET /platform/court-subscription-plans` — Platform Admin

- **URL:** `https://api.staging1.lsp.devontech.io/api/v1/platform/court-subscription-plans`
- **Time:** 2026-09-25T12:09:16.919Z · 634 ms
- **Auth:** `Authorization: Bearer <redacted>`

Request body:

```json
null
```

Response (HTTP 400):

```json
{
  "status": false,
  "message": "Error fetching plans",
  "data": {
    "error": "ERROR: relation \"court_subscription_plans\" does not exist (SQLSTATE 42P01)"
  }
}
```

### E2. `POST /platform/court-subscription-plans` — Platform Admin

- **URL:** `https://api.staging1.lsp.devontech.io/api/v1/platform/court-subscription-plans`
- **Time:** 2026-09-25T12:09:52.200Z · 410 ms
- **Auth:** `Authorization: Bearer <redacted>`

Request body:

```json
{
  "name": "E2E TEST – delete me",
  "price": 100,
  "billing_cycle": "MONTHLY",
  "is_waiver": false,
  "waiver_cap_days": null,
  "reverts_to_plan_id": null
}
```

Response (HTTP 400):

```json
{
  "status": false,
  "message": "Error creating plan",
  "data": {
    "error": "ERROR: relation \"court_subscription_plans\" does not exist (SQLSTATE 42P01)"
  }
}
```

### E3. `POST /platform/court-invoices` — Platform Admin

- **URL:** `https://api.staging1.lsp.devontech.io/api/v1/platform/court-invoices`
- **Time:** 2026-09-25T12:12:49.178Z · 461 ms
- **Auth:** `Authorization: Bearer <redacted>`

Request body:

```json
{
  "court_id": 5,
  "period": "E2E TEST",
  "amount": 100,
  "due_date": "2026-10-09T00:00:00.000Z"
}
```

Response (HTTP 400):

```json
{
  "status": false,
  "message": "Error raising invoice",
  "data": {
    "error": "ERROR: relation \"invoices\" does not exist (SQLSTATE 42P01)"
  }
}
```

### E4. `GET /court/subscription` — Court user (Judge)

- **URL:** `https://api.staging1.lsp.devontech.io/api/v1/court/subscription`
- **Time:** 2026-09-25T12:16:30.808Z · 440 ms
- **Auth:** `Authorization: Bearer <redacted>`

Request body:

```json
null
```

Response (HTTP 400):

```json
{
  "status": false,
  "message": "Error fetching subscription",
  "data": {
    "error": "ERROR: relation \"court_subscriptions\" does not exist (SQLSTATE 42P01)"
  }
}
```

### E5. `GET /court/payment_config` — Court user (Judge)

- **URL:** `https://api.staging1.lsp.devontech.io/api/v1/court/payment_config`
- **Time:** 2026-09-25T12:16:55.780Z · 610 ms
- **Auth:** `Authorization: Bearer <redacted>`

Request body:

```json
null
```

Response (HTTP 400):

```json
{
  "status": false,
  "message": "Error fetching payment configuration",
  "data": {
    "error": "ERROR: relation \"court_payment_configs\" does not exist (SQLSTATE 42P01)"
  }
}
```

### E6. `GET /court/subscription-plans` — Court user (Judge)

- **URL:** `https://api.staging1.lsp.devontech.io/api/v1/court/subscription-plans`
- **Time:** 2026-09-25T12:17:04.427Z · 473 ms
- **Auth:** `Authorization: Bearer <redacted>`

Request body:

```json
null
```

Response (HTTP 400):

```json
{
  "status": false,
  "message": "Error fetching plans",
  "data": {
    "error": "ERROR: relation \"court_subscription_plans\" does not exist (SQLSTATE 42P01)"
  }
}
```

### E7. `GET /court/invoices` — Court user (Judge)

- **URL:** `https://api.staging1.lsp.devontech.io/api/v1/court/invoices`
- **Time:** 2026-09-25T12:17:04.450Z · 490 ms
- **Auth:** `Authorization: Bearer <redacted>`

Request body:

```json
null
```

Response (HTTP 400):

```json
{
  "status": false,
  "message": "Error fetching invoices",
  "data": {
    "error": "ERROR: relation \"invoices\" does not exist (SQLSTATE 42P01)"
  }
}
```

### E8. `POST /court/subscription/verify` — Court user (Judge)

- **URL:** `https://api.staging1.lsp.devontech.io/api/v1/court/subscription/verify`
- **Time:** 2026-09-25T12:17:20.330Z · 416 ms
- **Auth:** `Authorization: Bearer <redacted>`

Request body:

```json
{
  "reference": "E2E-TEST-NONEXISTENT"
}
```

Response (HTTP 400):

```json
{
  "status": false,
  "message": "ERROR: relation \"invoices\" does not exist (SQLSTATE 42P01)",
  "data": null
}
```
