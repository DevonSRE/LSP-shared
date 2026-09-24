# Changelog

## 2026-09-24 — Court Subscription Billing (Workflow 4: Plans lifecycle)

**Tag:** Web · NON-BREAKING (behavioral change to `POST /court/subscription-plans` — see below)

**Plan lifecycle** — plans are no longer flat CRUD:
- `POST /court/subscription-plans` — **behavior changed**: now always creates a `DRAFT` plan (previously created a live, immediately-selectable plan). Not selectable by any court until published.
- `GET /court/subscription-plans` — **behavior changed**: now only returns `PUBLISHED` plans (previously returned all statuses).
- `GET /court/subscription-plans/admin` — new. All statuses, Platform Admin only, for plan management screens.
- `PATCH /court/subscription-plans/{id}` — new. Edit name/price/cycle/waiver fields; never retroactively changes an already-locked-in subscriber's fee.
- `POST /court/subscription-plans/{id}/publish` — new. Idempotent. Plans priced above ₦200,000 require a second, different Platform Admin to confirm before going live (the first call records the request and returns `pending_approval: true`).
- `POST /court/subscription-plans/{id}/archive` — new. Stops new selection; never force-migrates existing subscribers, who may still renew the same archived plan.
- `POST /court/subscription-plans/{id}/duplicate` — new. Copies a plan into a new draft.
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
