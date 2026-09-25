# Changelog

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
- `POST /public/court-billing/paystack/webhook` — Paystack calls this directly (no auth middleware, signature-verified). Drives recurring/`AUTO` billing only.

**Invoices**
- `GET/POST /court/invoices` — list (court-scoped) / raise an ad-hoc invoice (Platform Admin only).
- `GET /court/invoices/{id}/receipt` — paid invoice details, tenant-scoped.
- `POST /court/invoices/{id}/remind` — on-demand reminder (Platform Admin only).
- `POST /court/invoices/{id}/mark-paid` — record a manual/offline payment, idempotent (Platform Admin only).
- `POST /court/invoices/{id}/waive` — zero an invoice with a logged reason (Platform Admin only).

**Known limitations, not oversights:**
- Recurring billing (`AUTO`) has no admin-configurable retry/grace settings yet — a hardcoded retry limit is used until Dunning settings (Workflow 5) ship.
- `SUSPENDED` status is not yet reachable — automatic suspension after grace expires isn't built.
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
