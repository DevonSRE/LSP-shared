# Court Subscription Screen: Frontend Integration Guide

**Module:** Court Subscription screen (Judge, Legal Aide, Registrar)
**Date:** 2026-09-25
**Status:** Ready to integrate
**Source documents:** `openapi.yaml` (`Court Subscription Billing` tag), `flow/07-court-subscription-screen.html`, `design/JudicAI Court Subscription (PRD).dc.html`

All of these are scoped to the caller's own court. Base path `/api/v1`.

## 1. Status card and banner: `GET /court/subscription`

| Screen element | Field |
|---|---|
| Current plan | `plan_name` |
| Cycle · price · "paid by …" | `billing_cycle`, `fee` (kobo), `last_payment_label` (words, e.g. "Paystack · card") |
| PAID THROUGH / EXPIRED ON | `next_billing_date` |
| "N days left" | `days_left` (0 once ended) |
| "N days overdue" | `days_overdue` |
| "Renewal reminder sent" | `reminder_sent_at` (null until one has gone out for this period) |
| Grace banner ("N days left in your grace period") | `status = GRACE`, `grace_ends_at` |
| Read-only banner, disabled Create/Edit | `read_only` (true when `SUSPENDED`) |
| Sidebar red dot | `status` is `GRACE` or `SUSPENDED` |

A court that has never subscribed gets an empty response (no `plan_name`, `status` empty). Show "Choose a plan to get started"; nothing is restricted.

## 2. Plan cards: `GET /court/subscription-plans?view=cards`

```json
[
  { "name": "Basic", "description": "Recording, transcripts and cause list",
    "cycles": [
      { "billing_cycle": "QUARTERLY", "offered": true,  "plan_id": 3, "price": 45000000 },
      { "billing_cycle": "BIANNUAL",  "offered": true,  "plan_id": 4, "price": 85500000 },
      { "billing_cycle": "ANNUAL",    "offered": false, "plan_id": null, "price": null }
    ] }
]
```

- One card per plan. Every card has the same cycle columns. Show "Not offered" where `offered` is false.
- Prices are in kobo. **When the court picks a cycle, send that cycle's `plan_id` to checkout.** There is no separate cycle field.
- Preselecting from a reminder link: the reminder email opens `…/subscription?plan_id=<id>&cycle=<CYCLE>`. Select the card containing that `plan_id`, and that cycle.

## 3. Paying: `POST /court/subscription/checkout`, then `POST /court/subscription/verify`

Checkout takes `{ "plan_id": … }` and returns an `authorization_url`. Redirect the court to it. When Paystack sends the court back, call verify with `{ "reference": "…" }`.

**Success (200):** the subscription, plus `already_confirmed`.
- `already_confirmed: false` → "Payment confirmed" (plan, `fee` as the amount paid, `next_billing_date` as "Access until").
- `already_confirmed: true` → "This payment is already confirmed" (a reload, or a second call). Nothing was added again.

**Not confirmed (400):** match on `data.code`, not the message. `message` is a sentence you can show as it is.

| `data.code` | What it means | Screen |
|---|---|---|
| `PAYMENT_PENDING` | Not paid yet. A bank transfer may still land. | Keep the "Confirming your payment" state and call verify again: every few seconds at first, then about every 30 seconds. |
| `PAYMENT_LINK_EXPIRED` | The checkout was started but never completed. | "This payment link has expired": Start a new payment. |
| `PAYMENT_DECLINED` | Declined, failed or reversed. Nothing was charged. | "Payment not completed": Close / Try again. |
| `AMOUNT_MISMATCH` | Paystack confirmed a different amount. Nothing was activated. | "We couldn't confirm this payment": Try again / contact support. |
| `NO_PAYMENT_FOUND` | Unknown reference, or not this court's. | Treat as an error; start a new payment. |
| `VERIFICATION_UNAVAILABLE` | Paystack could not be reached. | "Try again in a moment": call verify again. |

Verify is safe to call as often as you like. A payment is never applied twice. If the court closes the page while a transfer is pending, the server confirms it within the hour anyway.

## 4. Payment history: `GET /court/subscription/payments?page=1&size=4`

Paginated (`page`, `size`; the response carries `total_rows` and `total_pages`), newest first. Confirmed payments only. Each row:

| Screen element | Field |
|---|---|
| "Plan · Cycle" | `plan_name`, `billing_cycle` |
| Date · method | `paid_at`, `method_label` |
| Amount | `amount` (kobo) |
| "through …" | `period_end` |

Payments an admin recorded for the court appear here as "Manual · recorded by Devon", dated when the court actually paid. For payments confirmed before this release, `paid_at` and `period_end` can be `null`: show the row without a date.

## 5. Emails the court receives

- **Payment confirmation** to every active Judge, Registrar and Legal Aide, after a Paystack payment. Nothing for the frontend to do.
- **Renewal reminders** at 14 and 3 days before the end (the admin can change the days). The Renew button links to the Subscription screen with `plan_id` and `cycle` in the query string (section 2).

## What is not here

- Auto-renew: the PRD screen has no toggle, but `POST /court/subscription/renewal-mode` exists if you want one.
- VAT and Paystack fee lines: not recorded yet, so the amount shown is the plan price.
