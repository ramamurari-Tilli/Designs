# Nudge Dynamic Merge Tags — QA Test Plan

Feature: Dynamic (admin-configurable) merge tags for **Payment Reversal**, **Payment Failure**, and **Payment Link Request** notifications.
Branch: Dev_RamaMurari (PR #1218 → dev). Services: nodejs_gps (backend), gps_admin (Admin Portal → Nudge Configuration), reactjs_gps / gps_payments_app (merchant & payment flows that trigger the notifications).

## 1. What this feature does

A Nudge notification (email/SMS) is built from a template that has "blanks" called **merge tags** (e.g. `{{customer_name}}`). Previously the list of blanks for these notifications was **hardcoded in the app (static)**. Now the blanks can be **configured by staff from the Admin → Nudge Configuration page (dynamic)** and stored in the `nudge_merge_tags` table.

Three notifications are covered:
1. **Payment Reversal** (`notify_payment_reversal`) — refund/reversal email — newly dynamic.
2. **Payment Failure** (`notify_payment_failure`) — payment-failed email — was already dynamic; verified + a log-label fix.
3. **Payment Link Request** (`payment_link_request`) — "here's your payment link" email — newly dynamic (all send paths, US + India).

**Backward compatible:** if a merchant has **no** merge tags configured, the notification sends the **same content as before** (a built-in fallback list), so nothing breaks.

**Backend change summary (for reference):** new helper `resolveTagsWithFallback` in `nudge.helper.js`; `notifyOnPaymentRevarsal` (`http.js`) → dynamic; payment-link email → dynamic in `sendUserPaymentLinkV2`, `notifyOnSharePaymentLink`, `sendPayLLink` (`billinginfo.service.js`) and `notifyOnSharePaymentLinkIND` (`billinginfo.service.ind.js`); failure log `type` corrected `PAYMENT_REVERSAL` → `PAYMENT_FAILURE`.

**Migrations:** **None required.** The `nudge_merge_tags` table and the `notify_payment_reversal` / `notify_payment_failure` / `payment_link_request` columns already exist.

## 2. How to use this document

- Execute the test cases in order within each section.
- Each test case has **Preconditions → Steps → Expected Result → Screenshot**.
- The core check for "is it dynamic": **change a tag in Admin → trigger the notification → if the email changes, it is dynamic; if it ignores the config, it is static.**
- Test against a backend running this branch, using a merchant where payments actually work.

## 3. Glossary

- **Merge tag** — a blank in the Nudge template, e.g. `{{customer_name}}`.
- **Tag Name** — the blank's name in the Nudge template (the Nudge side).
- **Tag Value** — the TilliPay field the blank is filled from (entered in Admin).
- **Static** — merge tags are hardcoded in the app's code.
- **Dynamic** — merge tags come from the `nudge_merge_tags` DB table, edited in the Admin Nudge Config page.
- **Fallback** — when no tags are configured, the code uses the original hardcoded list, so behaviour is unchanged.

---

# SECTION A — Admin: Nudge Configuration (setup)

Where: Admin Portal → Sub-merchants → select the test merchant → **Nudge Configuration**.

### TC-NDT-A1 — Merge-tag icon appears for a notification
- Preconditions: Logged into Admin Portal; the notification's template ID is filled in.
- Steps:
  1. Open Nudge Configuration for the test merchant.
  2. Enter a template ID in the notification field (e.g. Notify payment reversal / Payment link request).
  3. Look next to that field.
- Expected Result: A tag (🏷️) icon appears next to the field. It is **red** when no merge tags are configured.
- Screenshot :

### TC-NDT-A2 — Add merge tags and Save
- Preconditions: TC-NDT-A1 done for the chosen notification.
- Steps:
  1. Click the 🏷️ icon → (Edit mode) → click **Add Tag**.
  2. Enter a **Tag Name** and **Tag Value** row (see the section for the notification).
  3. Add all required rows, then click **Save**.
- Expected Result: Tags are saved; the 🏷️ icon turns **green** (tags configured).
- Screenshot:

### TC-NDT-A3 — View saved merge tags
- Steps:
  1. Click the green 🏷️ icon.
- Expected Result: The dialog opens in View mode and lists each **Tag Name / Tag Value** pair that was saved.
- Screenshot:

### TC-NDT-A4 — Edit / remove merge tags
- Steps:
  1. Click 🏷️ → Edit → change a Tag Value, or remove a row (⊝), or Add Tag.
  2. Click **Save**.
- Expected Result: The change is saved; reopening shows the updated list (icon stays green while any tag exists, turns red when all are removed).
- Screenshot:

---

# SECTION B — Payment Reversal (`notify_payment_reversal`)

Where: Admin Nudge Config (to set tags) + a refund/reversal on a paid transaction (to trigger).
Merge tags to configure (Tag Name → Tag Value):
- `customer_name` → `customer_name`
- `Refund_amount` → `amount`
- `Payment_amount` → `amount`
- `Refund_days` → `refundDays`
- `currency` → `currency`

Template must contain: `{{customer_name}}`, `{{Refund_amount}}`, `{{Payment_amount}}`, `{{Refund_days}}`, `{{currency}}`

### TC-NDT-B1 — Configure reversal merge tags
- Preconditions: Notify payment reversal has a template ID; template contains the placeholders above.
- Steps:
  1. Admin → Nudge Config → **Notify payment reversal** → 🏷️ → Add the tags listed above → **Save**.
- Expected Result: Tags saved; 🏷️ icon green.
- Screenshot:

### TC-NDT-B2 — Reversal email uses the configured tags (dynamic)
- Preconditions: TC-NDT-B1 done.
- Steps:
  1. Process a **refund / reversal** on a paid transaction for the merchant.
  2. Check the customer's inbox.
- Expected Result: The reversal email is received and its blanks show the real values (customer name, refund amount, currency).
- Screenshot:

### TC-NDT-B3 — Prove it is dynamic
- Steps:
  1. In Admin, change a Tag Value (e.g. `customer_name` → `customer_number`) or add a new tag → **Save**.
  2. Process another reversal.
- Expected Result: The email changes to match the new config — no code change / redeploy. (Proves dynamic.)
- Screenshot:

### TC-NDT-B4 — Fallback (no tags = old behaviour)
- Steps:
  1. In Admin, remove all reversal tags → **Save** (🏷️ turns red).
  2. Process a reversal.
- Expected Result: The reversal email still sends, using the built-in default content (identical to before). Nothing breaks.
- Screenshot:

---

# SECTION C — Payment Failure (`notify_payment_failure`)

Where: Admin Nudge Config + a payment that is declined/fails (to trigger).
Merge tags to configure (Tag Name → Tag Value):
- `customer_name` → `customerName`
- `Payment_amount` → `Payment_amount`
- `last_four_digits_of_card` → `last_four_digits_of_card`
- `currency` → `currency`

Template must contain: `{{customer_name}}`, `{{Payment_amount}}`, `{{last_four_digits_of_card}}`, `{{currency}}`

### TC-NDT-C1 — Configure failure merge tags
- Preconditions: Notify payment failure has a template ID; template contains the placeholders above.
- Steps:
  1. Admin → Nudge Config → **Notify payment failure** → 🏷️ → Add the tags above → **Save**.
- Expected Result: Tags saved; 🏷️ icon green.
- Executed: `notify_payment_failure` (Nudge ID 25140) is **already dynamic**; 3 tags configured, with swaps to prove dynamic — `customer_name → amount`, `Payment_amount → customerName`, `last_four_digits_of_card → last_four_digits_of_card`.
- Screenshot:

![Notify payment failure — merge tags configured in Admin (already dynamic, 3 tags)](<Screenshot 2026-08-31 194109.png>)

### TC-NDT-C2 — Failure email uses the configured tags (dynamic)
- Steps:
  1. Make a payment that is **declined / fails**.
  2. Check the customer's inbox.
- Expected Result: The "payment failed" email is received with the real values (customer name, amount, last 4 digits).
- Screenshot:

### TC-NDT-C3 — Prove it is dynamic
- Steps:
  1. Change/add a tag in Admin → **Save**.
  2. Trigger another failed payment.
- Expected Result: The email changes to match the new config (dynamic).
- Screenshot:

### TC-NDT-C4 — Log label check
- Steps:
  1. After a failed payment, inspect the `nudge_logs` row for this send.
- Expected Result: `nudge_logs.type` reads **`PAYMENT_FAILURE`** (previously mislabeled `PAYMENT_REVERSAL`).
- Screenshot:

### TC-NDT-C5 — Fallback (no tags = old behaviour)
- Steps:
  1. Remove all failure tags → **Save**.
  2. Trigger a failed payment.
- Expected Result: The failure email still sends with the built-in default content.
- Screenshot:

---

# SECTION D — Payment Link Request (`payment_link_request`)

Where: Admin Nudge Config + the merchant **Share Payment** page (to trigger).
Merge tags to configure (Tag Name → Tag Value):
- `requested` → `email`
- `amountUnit` → `currency`
- `requestedAmount` → `amount`
- `tilliPayPaymentUrl` → `uri`
- `customer` → `business_legal_name`
- `payment_for` → `payment_for`
- `Customer_Name` → `customer_Name`
- `customer_number` → `customer_number`
- `customField3` → `customFieldsObj?.customField3 || ''`
- `customField4` → `customFieldsObj?.customField4 || ''`
- `customField5` → `customFieldsObj?.customField5 || ''`

Template must contain the matching `{{placeholders}}`.

### TC-NDT-D1 — Configure payment-link merge tags
- Preconditions: Payment link request has a template ID; template contains the placeholders.
- Steps:
  1. Admin → Nudge Config → **Payment link request** → 🏷️ → Add the tags above → **Save**.
- Expected Result: Tags saved; 🏷️ icon green.
- Executed: 5 tags configured for `payment_link_request` (Nudge ID 25140), with deliberate swaps to prove dynamic behaviour — `Customer_Name → customer_number` and `customer_number → customer_name`.
- Screenshot:

![Payment Link Request — merge tags configured in Admin (5 tags, values swapped)](<Screenshot 2026-08-31 171823.png>)

### TC-NDT-D2 — Payment-link email uses the configured tags (dynamic)
- Steps:
  1. Merchant portal → **Share Payment** → enter a customer email + amount (e.g. `56`) → create/share the link.
  2. Check the customer's inbox.
- Expected Result: The payment-link email is received; the amount blank shows `56` and the link works.
- Screenshot:

### TC-NDT-D3 — Prove it is dynamic (local/dynamic vs dev/static)
- Steps:
  1. Keep the swapped config from TC-NDT-D1 (`Customer_Name → customer_number`, `customer_number → customer_name`).
  2. Share the same payment link from the **local backend** (dynamic code) and from the **dev backend** (static code) with the same data (name "manay", number "12222").
  3. Compare both emails.
- Expected Result: The **local (dynamic)** email reflects the admin config (swapped), while the **dev (static)** email ignores the config and uses the hardcoded order.
- Result (PASS ✅):
  - **Local (dynamic):** *"Dear **12222** with **manay** … for **124** …"* — name and number are swapped, matching the admin config → the DB tags drove the email.
  - **Dev (static):** *"Dear **manay** with **12222** … for **124.00** …"* — normal order, config ignored → uses hardcoded tags.
  - Same config, different result per environment → **confirms dynamic on local, static on dev.**
- Screenshots:

Local — dynamic (reflects the swapped config):
![Payment link email from LOCAL — dynamic, values swapped per config](<Screenshot 2026-08-31 171920.png>)

Dev — static (ignores the config):
![Payment link email from DEV — static, config ignored](<Screenshot 2026-08-31 172024.png>)

### TC-NDT-D4 — Fallback (no tags = old behaviour)
- Steps:
  1. Remove all payment-link tags → **Save**.
  2. Share a payment link.
- Expected Result: The email still sends with the built-in default content.
- Screenshot:

### TC-NDT-D5 — Coverage: resend, bulk, India
- Steps:
  1. **Resend** an existing payment link (`sendPayLLink`).
  2. **Bulk** share links (`sendUserPaymentLinkV2`).
  3. Repeat D1–D2 for an **India** merchant (`notifyOnSharePaymentLinkIND`).
- Expected Result: All paths send the payment-link email using the configured tags (or the fallback when none) — behaviour is consistent across all four paths.
- Screenshot:

---

# SECTION E — Backward compatibility / regression

### TC-NDT-E1 — Un-configured merchant is unchanged
- Steps:
  1. Use a merchant with **no** merge tags configured for these notifications.
  2. Trigger each of the three notifications.
- Expected Result: All three send **exactly the same content as before** (fallback). No errors, no visible change.
- Screenshot:

### TC-NDT-E2 — No UI / flow breakage
- Steps:
  1. Exercise the normal payment, refund/reversal, and share-link flows (single, resend, bulk).
- Expected Result: All flows work as before; only the merge tags sent to Nudge differ. No route/screen changes.
- Screenshot:

---

## Sign-off

- [ ] Branch deployed to QA; no new migration run (confirmed not needed).
- [ ] Section A (Admin config) passes.
- [ ] Section B (Reversal) — dynamic proof + fallback pass.
- [ ] Section C (Failure) — dynamic proof + fallback pass; log type = `PAYMENT_FAILURE`.
- [ ] Section D (Payment Link) — dynamic proof + fallback pass; resend/bulk/India verified.
- [ ] Section E — un-configured merchant unchanged; no UI/flow breakage.
