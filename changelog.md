# Changelog

## 2026-09-01 — Court Payment Configuration

**Tag:** Web · NON-BREAKING

New endpoints — no existing contract changed.

- `GET /court/payment_config` — fetch the caller's court payment configuration (price per page, CTC price, payout bank account). Accessible to Judge, Legal Aide, Registrar, and Platform Admin users.
- `PUT /court/payment_config` — set or update it. Prices are always editable; the payout `account_number` can only be set once per court — a second attempt to change it is rejected with a 400.

See `openapi.yaml` (`Court Payment Configuration` tag) for the full contract, and `frontend/court-payment-config-api.md` for a frontend-facing walkthrough with request/response examples.
