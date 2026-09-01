# Court Payment Configuration — Frontend Integration Guide

**Module:** Court Payment Configuration
**Date:** 2026-09-01
**Status:** Ready to integrate
**Source documents:** `openapi.yaml` (`Court Payment Configuration` tag)

Lets a court set the price it charges per page of a proceeding (casefile) copy, the price for a Certified True Copy (CTC), and the bank account its share of payments should be paid into. This is configuration only — no live payment/payout processing is wired up yet, so nothing here triggers an actual money movement.

## Who can call these endpoints

**Judge, Legal Aide, Registrar, or Platform Admin** — send the same bearer token you use for other authenticated court endpoints. There's no separate permission to request; if the logged-in user has one of those four roles, they can call both endpoints. Any other role gets a `401`.

Both endpoints are scoped to the **caller's own court** — there is no court ID in the URL or payload. The backend resolves the court from the caller's token.

## The one-time-lock rule (read this before building the form)

Prices (`price_per_page`, `ctc_price`) can be changed anytime — no restrictions.

The payout `account_number` (+ `bank_name`) can only be **set once**. Once a court has a non-empty `account_number` on file, any later request that tries to submit a *different* account number is rejected with a `400`. There is currently no in-app or support flow to change it after that — so:

- Once `GET` returns `account_locked: true`, **disable the account number / bank name fields in the UI**. Don't rely on the API's `400` alone — a disabled field avoids the user filling it in and then hitting a confusing error.
- Show the currently-set `account_number`/`bank_name` as read-only reference once locked.
- There's no "request a change" button to wire up yet — if you want to communicate the "contact us" part of this rule to court staff, it's just static copy for now (e.g. "To change your payout account, contact support"), not a real flow.

## `GET /court/payment_config`

Fetches the caller's court's current configuration.

**Request:** no body, no params.

**Response `200`** — always `200`, even if the court hasn't configured anything yet (that's a normal first-time state, not an error):

```json
{
  "id": 0,
  "court_id": 12,
  "price_per_page": 0,
  "ctc_price": 0,
  "account_number": "",
  "bank_name": "",
  "account_locked": false,
  "created_at": "0001-01-01T00:00:00Z",
  "updated_at": "0001-01-01T00:00:00Z"
}
```

Treat `account_locked: false` + empty `account_number` as "not configured yet" — show the form as fully editable/empty, not as an error state. `id: 0` and the zero-value timestamps are the tell for "no record exists yet" if you need to distinguish it from a real (but zeroed) saved record.

Once configured:

```json
{
  "id": 4,
  "court_id": 12,
  "price_per_page": 5000,
  "ctc_price": 200000,
  "account_number": "0123456789",
  "bank_name": "First Bank",
  "account_locked": true,
  "created_at": "2026-09-01T10:15:00Z",
  "updated_at": "2026-09-01T10:15:00Z"
}
```

## `PUT /court/payment_config`

Creates the config on first call, updates it on every call after.

**⚠️ Units: `price_per_page` and `ctc_price` are in kobo, not naira.** ₦50.00 is `5000`. This is an easy off-by-100 bug — divide/multiply by 100 at the form boundary, don't send naira directly.

**Request — first time (no config exists yet):**

```json
{
  "price_per_page": 5000,
  "ctc_price": 200000,
  "account_number": "0123456789",
  "bank_name": "First Bank"
}
```

`account_number` and `bank_name` are optional even on the first call — a court can set prices without setting a payout account yet, and add the account in a later call (which will still count as the "first set" and lock it).

**Request — updating prices only, after the account is already locked:**

```json
{
  "price_per_page": 7500,
  "ctc_price": 250000
}
```

Omit `account_number`/`bank_name` entirely (don't send empty strings vs. omit — either is fine, the backend only acts on `account_number` if it's non-empty in the payload).

**Response `200`** — same shape as the `GET` response, reflecting the new state.

**Response `400` — validation error** (missing/invalid prices, or `account_number` provided without `bank_name`):

```json
{
  "status": false,
  "message": "Validation Error, invalid value provided",
  "data": { "error": "..." }
}
```

**Response `400` — locked-account violation** (attempting to change an already-set `account_number`):

```json
{
  "status": false,
  "message": "Payout account has already been set. Contact support to change it.",
  "data": null
}
```

This `message` is safe to show directly to the user — surface it as-is in an error toast/banner rather than writing your own copy for this case, so the wording stays consistent with what the backend team documents/changes over time.
