# **Product Requirements Document**

## **Court Subscription Billing**

**Module:** Platform Administration + Court Portal **Product:** JudicAI

| Field | Detail |
| :---- | :---- |
| Product | JudicAI |
| Module | Platform Administration, Court Portal |
| Feature Type | Billing / Revenue |
| Priority | High |
| Status | Approved for build (engineering plan reviewed) |
| Prepared By | Product Team |
| Related | `docs/designs/court-subscription-billing.md` (engineering design doc) |

## **01 — Overview**

Court Subscription Billing lets Devon Technology sell JudicAI platform access to courts as a subscription — Basic, Essential, or Standard plans, billed quarterly, biannually, or annually — collected on-platform via Paystack instead of manual invoice and bank transfer. A court that lets its subscription lapse loses write access to the platform (read-only) until it renews.

This is the **Devon → Court** side of JudicAI's billing. It's a separate feature from **Court → Lawyer** billing (the price-per-page/CTC pricing a court already sets to charge lawyers for proceeding copies, with a Paystack split payment where Devon takes a percentage). The two features share no data and don't block each other — a court's subscription status has no effect on whether it can configure or collect lawyer payments, and vice versa.

## **02 — Problem Statement**

Courts already pay Devon for JudicAI access today, but entirely outside the product:

* Devon manually invoices a court  
* The court pays by bank transfer  
* Someone at Devon manually confirms the payment and tracks it — nowhere in the system, no audit trail, no reminders  
* There is no way to see, at a glance, which courts are paid up and which are overdue  
* There is no consequence in-app for a court that doesn't renew — access continues regardless

This is real revenue Devon is already collecting, just untracked and manual. The goal is to bring it on-platform, add a real subscription record per court, and give lapsed subscriptions a consequence.

## **03 — Goals**

### **Primary Goals**

* Platform Admin can define and edit subscription plans (Basic, Essential, Standard) with a price per billing cycle  
* A court can view available plans, subscribe, and pay via Paystack — on-platform, no manual step from Devon  
* Every court, including ones already paying via the old manual process, has one subscription record Devon can see  
* A court that lets its subscription lapse loses write access until it renews  
* Courts get automatic renewal reminder emails before their subscription expires

### **Secondary Goals**

* Give Devon a single place to see who's paid, who's overdue, and by how much  
* Reduce Devon's manual invoice/reconciliation workload  
* Lay the access-gate groundwork the future Court → Lawyer marketplace depends on (a court has to be an active, paying customer before its lawyers can transact through it)

## **04 — Scope**

### **In Scope (Phase 1\)**

* Subscription plan creation and management (Platform Admin)  
* Court-facing plan browsing and subscription checkout via Paystack  
* Payment confirmation (Paystack transaction verification) that reliably activates a subscription  
* Platform Admin can manually record a payment for a court (used both for the initial backfill of already-paying courts, and for any future edge case — e.g. a court that paid by bank transfer because Paystack was briefly down)  
* Subscription status: Active, Expired (grace period), Locked  
* Read-only access enforcement once a court's subscription is Locked  
* Renewal reminder emails  
* Audit log entry for every plan change and every payment recorded (online or manual)

**Delivery note:** Engineering ships this in two back-to-back releases — plan/checkout/payment first, reminders + access lockout immediately after (target: within 1–2 weeks of the first release). Both are described together in this document because they're one continuous feature to the court and to Devon; the split is an engineering sequencing decision, not a product one.

### **Out of Scope (Phase 1\)**

* Auto-renewing/recurring billing — a court re-initiates payment each cycle, it does not renew automatically  
* Prorated upgrades/downgrades mid-cycle  
* A bulk migration tool for backfilling existing courts — Platform Admin backfills each existing court's current billing state one at a time, using the same "record a payment" action used for any manual payment  
* A dedicated communication/support channel for a court that needs a payout-account-style correction — not applicable to this feature (see Court → Lawyer PRD for that), no analog exists here  
* Multi-currency support — Naira only  
* Detailed payment/accounting ledger beyond the audit log (a full financial ledger is a later phase)

## **05 — User Stories**

### **US-01 — Create and Manage Subscription Plans**

As a **Platform Admin**, I want to create and edit subscription plans with a price per billing cycle so that courts have clear, consistent options to subscribe to.

#### **Process Flow**

1. Platform Admin navigates to Subscription Plans in Platform Admin.
2. Platform Admin clicks 'New Plan'.
3. Platform Admin enters the plan name (e.g. Basic, Essential, Standard) and sets a price for each billing cycle: Quarterly, Biannual, Annual.
4. Platform Admin saves the plan.
5. Plan appears in the Plan list and immediately becomes visible to courts.
6. Platform Admin can later edit an existing plan's name or per-cycle pricing.
7. System logs the create/edit action in the Audit Trail.

#### **Acceptance Criteria**

| # | Criteria |
| ----- | ----- |
| AC01 | The Create Plan form requires a plan name and a price for at least one billing cycle before it can be saved. |
| AC02 | A plan can have a different price set for each of the three billing cycles (Quarterly, Biannual, Annual). |
| AC03 | Saving a plan makes it immediately visible on the court-facing plan list — no separate "publish" step. |
| AC04 | Editing an existing plan's price does not change the price already locked in for a court's current active cycle — only future subscribe/renew actions use the new price. |
| AC05 | Plan create and edit actions are captured in the Audit Trail with the admin's ID, action type, and timestamp. |

### **US-02 — Court Subscribes to a Plan**

As a **Court Registrar/Judge**, I want to view available subscription plans and pay for one so that my court gets or keeps access to JudicAI.

#### **Process Flow**

1. Court user navigates to Subscription in the court portal.
2. Court user sees the available plans (Basic, Essential, Standard) with pricing for each billing cycle.
3. Court user selects a plan and a billing cycle (Quarterly, Biannual, Annual).
4. Court user clicks 'Pay Now'.
5. System redirects the court user to Paystack's hosted checkout page, pre-filled with the correct amount.
6. Court user completes payment on Paystack.
7. Paystack redirects the court user back to JudicAI.
8. System verifies the payment with Paystack before activating anything.
9. On confirmed success, the court's subscription becomes Active with an end date set from the billing cycle chosen.
10. Court user sees a confirmation with their new subscription status and end date.
11. System logs the payment in the Audit Trail.

#### **Acceptance Criteria**

| # | Criteria |
| ----- | ----- |
| AC01 | The plan list shows all admin-defined plans with pricing for every billing cycle, in Naira. |
| AC02 | Clicking 'Pay Now' always starts a fresh Paystack transaction — it never re-shows a stale/expired payment link from a prior abandoned attempt. |
| AC03 | The subscription only becomes Active after the backend independently verifies the payment with Paystack — a redirect back to JudicAI is never, by itself, treated as proof of payment. |
| AC04 | If the verified payment amount doesn't match the selected plan/cycle's price, the subscription is not activated and the court sees an error, not a false success. |
| AC05 | If the court re-hits the payment confirmation page (e.g. browser refresh, back button) after a successful payment, the subscription is not double-extended — the confirmation is idempotent. |
| AC06 | If payment fails, is declined, or is abandoned on Paystack, the court sees a plain "payment not completed, try again" message and can restart checkout immediately. |
| AC07 | Every completed payment is captured in the Audit Trail with the court's ID, plan, cycle, amount, and timestamp. |

### **US-03 — Court Renews Before Expiry**

As a **Court Registrar/Judge**, I want to renew my subscription before it expires so that my court's access is never interrupted.

#### **Process Flow**

1. Court user sees their current subscription status and end date on the Subscription page at any time.
2. As the end date approaches, court user receives reminder emails (see US-06).
3. Court user returns to the Subscription page and clicks 'Renew'.
4. Court user selects a billing cycle (may be the same plan or a different one) and pays via Paystack, same flow as US-02.
5. On confirmed payment, the new cycle is added on top of the current end date (not from today) — the court doesn't lose any time it already paid for.
6. Court user sees the updated end date.

#### **Acceptance Criteria**

| # | Criteria |
| ----- | ----- |
| AC01 | A court with an Active subscription can renew (pay again) at any time before expiry — renewal is not blocked while a subscription is still valid. |
| AC02 | Renewing while still Active extends the end date from the current end date, not from the renewal date — no paid time is lost. |
| AC03 | A court can switch plans when renewing; the new plan's price for the selected cycle is what's charged. |

### **US-04 — Court's Subscription Expires**

As a **Court Registrar/Judge**, I want a clear, honest warning period after my subscription lapses so that I have a real chance to renew before losing access, instead of being cut off the instant a deadline passes.

#### **Process Flow**

1. A court's subscription end date passes without renewal.
2. Subscription status changes from Active to Expired. The court enters a grace period (3 days).
3. During the grace period, the court can still view all existing cases, casefiles, schedules — read-only — but cannot create or edit records.
4. The court sees a persistent banner: subscription expired, X days remaining before further restriction, with a 'Renew Now' action.
5. If the court renews at any point during the grace period, status returns to Active immediately and the read-only restriction lifts.
6. If the grace period passes without renewal, status changes to Locked. Access remains read-only (Locked does not add further restriction beyond Expired — it's a status distinction for Devon's visibility into how overdue a court is, not a harsher lockout).
7. The court can still renew at any time from Locked, which immediately restores full access.

#### **Acceptance Criteria**

| # | Criteria |
| ----- | ----- |
| AC01 | A court moves from Active to Expired exactly at its subscription end date, and from Expired to Locked exactly 3 days later if not renewed. |
| AC02 | Both Expired and Locked states allow full read access to existing records and block create/edit/delete actions on cases, casefiles, evidence, and schedules. |
| AC03 | The read-only banner is visible on every page while a court is Expired or Locked, with a direct path to renew. |
| AC04 | Paying at any point during Expired or Locked immediately restores full access — there is no additional delay or admin approval step. |
| AC05 | A court is only ever evaluated for expiry/lockout if it has an actual subscription record — a court with no record yet (not onboarded into this system) is never locked out by this feature. |

### **US-05 — Platform Admin Records a Manual Payment**

As a **Platform Admin**, I want to manually record that a court has paid so that I can migrate an existing court's current billing history into the system, or handle a payment that happened outside Paystack.

#### **Process Flow**

1. Platform Admin navigates to a specific court's Subscription record in Platform Admin.
2. Platform Admin clicks 'Record Payment'.
3. Platform Admin selects the plan, billing cycle, and enters the end date reflecting what the court has actually already paid for (for backfilling an existing court) or the cycle being newly paid (for a genuine offline payment).
4. Platform Admin saves.
5. Court's subscription status updates immediately (Active, with the entered end date) — reminders and lockout timing follow from this date going forward, same as an online payment.
6. System logs the action in the Audit Trail, distinguishing it as a manually-recorded payment.

#### **Acceptance Criteria**

| # | Criteria |
| ----- | ----- |
| AC01 | Only Platform Admin can record a manual payment — no other role has access to this action. |
| AC02 | Recording a manual payment for a court with no existing subscription record creates one; for a court that already has a record, it updates it (e.g. extends the end date). |
| AC03 | This is a one-court-at-a-time action — there is no bulk/CSV-import backfill flow in Phase 1. |
| AC04 | Every manually-recorded payment is captured in the Audit Trail with the admin's ID, court, plan, cycle, and timestamp, distinguishable from an online (Paystack) payment. |

### **US-06 — Court Receives Renewal Reminders**

As a **Court Registrar/Judge**, I want to be reminded before my subscription expires so that I can renew in time and never get surprised by a lockout.

#### **Process Flow**

1. As a court's subscription end date approaches, the system sends a reminder email 14 days before expiry.
2. The system sends a second reminder email 3 days before expiry.
3. Each email states the plan, the end date, and a direct link to renew.
4. If the court renews before either reminder is due, no further reminders are sent for that cycle.

#### **Acceptance Criteria**

| # | Criteria |
| ----- | ----- |
| AC01 | Reminder emails go out automatically at 14 and 3 days before a court's subscription end date, with no manual trigger from Devon. |
| AC02 | Each reminder email names the specific plan, exact end date, and includes a working link that takes the court directly to the renewal flow. |
| AC03 | A court that has already renewed does not receive a reminder for a cycle it no longer needs to worry about. |

## **06 — Functional Requirements**

### **6.1 Subscription Plans (Admin)**

| Field | Description |
| ----- | ----- |
| Plan Name | e.g. Basic, Essential, Standard |
| Price — Quarterly | Amount charged for a 3-month cycle |
| Price — Biannual | Amount charged for a 6-month cycle |
| Price — Annual | Amount charged for a 12-month cycle |

### **6.2 Court Subscription**

| Field | Description |
| ----- | ----- |
| Plan | Which plan the court is on |
| Billing Cycle | Quarterly / Biannual / Annual |
| Status | Active / Expired / Locked |
| End Date | When the current paid period ends |
| Payment Method | Online (Paystack) / Manual (Admin-recorded) |

### **6.3 Subscription Statuses**

| Status | Meaning |
| ----- | ----- |
| Active | Fully paid, full access |
| Expired | Past end date, within the 3-day grace period — read-only |
| Locked | Past the grace period — read-only (same restriction as Expired, distinguished for Devon's reporting) |
| No Record | Court has never subscribed or been backfilled — treated as unrestricted/not enforced, not as a payment failure |

### **6.4 Access Enforcement**

Read-only restriction applies to create/edit/delete actions on: Cases, Case Files, Evidence, Schedules. Viewing existing records remains available in every status.

## **07 — Permissions & Access**

| Action | Platform Admin | Judge / Registrar (own court) |
| ----- | ----- | ----- |
| Create/edit subscription plans | ✅ | ❌ (view only) |
| View plan list | ✅ | ✅ |
| Subscribe / renew (pay) | ❌ (not applicable — Devon isn't a court) | ✅ |
| Record a manual payment | ✅ | ❌ |
| View own court's subscription status | ✅ (any court) | ✅ (own court only) |

## **08 — Audit Trail**

Every action below is logged with the acting user's ID, action type, and timestamp.

| Action | Trigger |
| ----- | ----- |
| Plan Created | New subscription plan saved |
| Plan Edited | Existing plan's name or pricing changed |
| Payment Recorded (Online) | Court completes and verifies a Paystack payment |
| Payment Recorded (Manual) | Platform Admin records a payment on a court's behalf |

## **09 — Technical Requirements**

### **9.1 Backend APIs**

| Endpoint | Function |
| ----- | ----- |
| Create/Edit Plan | Admin creates or updates a subscription plan |
| List Plans | Court-facing read of available plans and pricing |
| Checkout | Court initiates a Paystack payment for a plan/cycle |
| Verify Payment | Backend confirms a Paystack transaction and activates the subscription |
| Record Manual Payment | Admin records an offline/backfill payment for a court |
| Get Subscription Status | Court or admin views current status, plan, end date |

### **9.2 Database Entities**

**Subscription Plan**

| Field | Description |
| ----- | ----- |
| ID | Unique identifier |
| Name | Plan name |
| Price (Quarterly / Biannual / Annual) | Per-cycle pricing, in kobo |

**Court Subscription**

| Field | Description |
| ----- | ----- |
| ID | Unique identifier |
| Court ID | Foreign key to Court |
| Plan ID | Foreign key to Subscription Plan |
| Billing Cycle | Quarterly / Biannual / Annual |
| Status | Active / Expired / Locked |
| End Date | Current paid-through date |
| Payment Method | Online / Manual |

### **9.3 Frontend**

**Required Pages**

* Subscription Plans (Platform Admin)
* Court Subscription Detail (Platform Admin — per court)
* Subscription (Court Portal)

**Required Screens**

* Plan List / Create / Edit (Admin)
* Plan Selection + Checkout (Court)
* Payment Confirmation (success and failure states)
* Subscription Status view (Active / Expired banner / Locked banner)
* Manual Payment Recording form (Admin)

**Required States per Screen (for design):**

* Empty (no plans defined yet — Admin side)
* No subscription yet (court has never subscribed — distinct from Expired/Locked, should not look like an error state)
* Active — normal state
* Expired — persistent warning banner, days-remaining countdown, renew CTA
* Locked — persistent restriction banner, renew CTA, read-only indicators on write actions
* Payment in progress (redirected to Paystack, awaiting return)
* Payment failed/declined
* Payment succeeded — confirmation with new end date

## **10 — Future Enhancements (Phase 2\)**

| Enhancement | Description |
| ----- | ----- |
| Auto-renewing subscriptions | Tokenized/recurring Paystack charges — no manual re-pay each cycle |
| Prorated upgrades/downgrades | Mid-cycle plan changes with fair pricing adjustment |
| Structured financial ledger | Full invoice/transaction history beyond the audit log, for accounting/reconciliation |
| Dunning | Automated retry attempts and escalating notices for failed recurring charges |
| Bulk backfill tooling | If manually backfilling existing courts one at a time proves too slow in practice |

## **11 — Success Metrics**

| Metric | Target |
| ----- | ----- |
| Courts with a subscription record | 100% of active courts within 30 days of launch (via backfill + new signups) |
| On-platform payment rate | 100% of new/renewal payments collected via Paystack, not manual invoice, within 60 days |
| Manual reconciliation workload | Measurable reduction in Devon's manual invoice-tracking effort post-launch |
| Reminder-driven renewals | Majority of renewals happen before expiry (Active → Active), not after (Expired/Locked → Active) |
