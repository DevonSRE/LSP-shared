# Subscription Grace and Suspension — Frontend Integration Guide

**Module:** Court subscription status, grace and suspension
**Date:** 2026-09-25
**Status:** Ready to integrate
**Source documents:** `openapi.yaml` (`Court Subscription Billing` tag), `flow/06-dunning-suspension.html`

## What a court can see

`GET /court/subscription` (Judge, Legal Aide or Registrar, scoped to their own court) now also returns:

| Field | Meaning |
|---|---|
| `status` | `ACTIVE`, `GRACE` or `SUSPENDED`. A court that has never subscribed has no status (the response is empty), and is not restricted. |
| `grace_started_at` | When the court entered its grace period. `null` unless `GRACE` or `SUSPENDED`. |
| `grace_ends_at` | When grace runs out. Use it for "N days left in your grace period". `null` unless `GRACE` or `SUSPENDED`. |
| `read_only` | `true` while creating, editing and deleting is paused (`SUSPENDED`). Use it to disable the Create and Edit buttons and to show the banner. |
| `next_billing_date` | End of the paid period. |

## What each status means

- **ACTIVE** — paid up. Nothing is restricted.
- **GRACE** — the paid period ended with no payment, or an automatic charge failed. **Nothing is restricted.** Show a warning banner with the days left and a Renew button.
- **SUSPENDED** — grace ran out (if the platform has suspension switched on). Creating, editing and deleting is paused on cases, casefiles, evidence, schedules, comments and uploads. **Reading and downloading always work.** Show the red banner and disable those buttons.

A payment moves the court back to `ACTIVE` immediately. Refresh `GET /court/subscription` after a confirmed payment. (If the platform has turned auto-reactivation off, a suspended court stays suspended after paying until an admin restores it. `status` will tell you.)

## The suspended error

Any create, edit or delete on those record types from a suspended court returns:

```
HTTP 403
{
  "status": false,
  "message": "Your court's subscription is suspended. You can still view everything; creating and editing is paused until the subscription is renewed.",
  "data": { "code": "SUBSCRIPTION_SUSPENDED" }
}
```

Match on `data.code === "SUBSCRIPTION_SUSPENDED"`, not on the message. On this code, show the renew prompt rather than a generic error. Don't rely on the API alone: use `read_only` to stop the user reaching the action in the first place.

Not affected: all `GET` requests, Platform Admin users, courts with no subscription, and the subscription and payment endpoints themselves.

## Renewal reminders

Courts are emailed before the paid period ends (by default 14 days and 3 days before), at the email addresses of the court's active Judges, Registrars and Legal Aides. The email's Renew button links to `<site>/subscription?plan_id=<id>` for the court's current plan. The Subscription screen should read `plan_id` from the query string and preselect that plan. The billing cycle is not in the link yet.

## Platform Admin screens

- **Dunning settings:** `GET` and `PUT /platform/dunning-settings`. `PUT` replaces the whole ruleset, so send every field; a switch you leave out is saved as `false`. `is_default: true` means nobody has saved it yet.
- **Overrides on one court:** `POST /platform/court-subscriptions/{court_id}?action=set-status | extend | remind`. Send `reason` unless `require_override_reason` is off. The reply is the court's updated subscription row.
- **To waive a cycle:** use `POST /platform/court-invoices/{id}/waive` on the court's open invoice. Find the invoice with `GET /platform/court-subscriptions/{court_id}/attempts`.
