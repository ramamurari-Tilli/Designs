# Nudge Configuration — Simple Guide & Plan

This document explains: what the Nudge notification system is, how it is set up in TilliPay, what is already built, what still needs to be built and how, a full list of every notification (the "inventory"), and where each one is triggered from in the code.

## 1. The basics

**What is Nudge?**
Nudge is an outside service that sends the actual emails and text messages to customers. Nudge stores the message templates — for example: *"Hi {{customer_name}}, your payment of {{payment_amount}} was received."* TilliPay does not write the message. TilliPay only tells Nudge **which template to use** and **what real values to put into the blanks**.

**What is a "merge tag"?**
A merge tag is a **blank in the template**. In *"Hi {{customer_name}}"*, the blank is `customer_name`. Each blank has two sides:
- **Tag name** = the name of the blank in the Nudge template (the Nudge side), e.g. `customer_name`.
- **Tag value** = the field on the **TilliPay side** that holds the real value (our customer's actual name). TilliPay looks up that value and hands it to Nudge, which drops it into the blank.

**What is "Nudge configuration"?**
It is the settings that answer two questions for every notification:
1. **Which Nudge template do we use?** (a template ID number, called the "Nudge ID").
2. **What blanks does it have, and where do their values come from?** (the merge tags).
---

## 2. Static vs. Dynamic

Both words describe **one thing only: where the list of blanks comes from.**

- **Static** = the blanks are **hardcoded in the app's code**. Fixed for everyone. Changing them needs a developer + redeploy. Like a **printed form**.
- **Dynamic** = the blanks come from a **database table that staff edit from the admin screen**. No developer, no redeploy, and each merchant can have its own. Like a **fillable digital form**.

**Gotcha:** if the dynamic list is empty (nobody set up blanks yet), the email would go out with empty blanks. So the safe rule is: *use the database list if it exists, otherwise fall back to the old hardcoded list.*
---

## 3. What is already implemented, and how

**Five notifications already work the dynamic way** — all the payment notifications:
1. Payment success → merchant email
2. Payment success → customer email
3. Payment success → merchant SMS
4. Payment success → customer SMS
5. Payment failed → customer email

**How it is done (the pattern that we copy for the rest):**
1. Decide which settings to use — the merchant's own settings, or the shared default; US or India.
2. Work out the merchant key (`accountId`), or `default` if the merchant has no custom settings.
3. Gather all the facts about the event into one bundle (amount, date, names, etc.), and mask card/account numbers down to the last 4 digits.
4. Look up the blanks for this notification from the database table and fill them from the bundle.
5. Send it to Nudge, and write a log record of the send.

That is the entire mechanism. Making a notification "dynamic" means switching **step 4** from a hardcoded list to a database lookup.
---

## 4. How this impacts the database

This is important: **the change is low-impact on the database. No tables are dropped and no columns are removed.**

Three tables are involved:

| Table | Role | Impact of this work |
|---|---|---|
| `nudge_config` (and `nudge_config_ind` for India) | Holds each merchant's Nudge IDs (which template to use for each notification). One row per merchant; each column is a notification. | **No change.** The columns for reversal, failure, and payment link already exist. |
| `nudge_merge_tags` | Holds the editable list of blanks for the dynamic notifications. Columns: `account_id`, `nudge_id` (the notification name), `merge_tags` (the list of blank → source-field pairs). | **New rows added.** One row per (merchant, notification) that we make dynamic. This is where the new configuration data lives. |
| `nudge_logs` | Records every send (who, success/failure, errors). | **No change** (a small label fix only — see step 4 in section 6). |

So the database footprint of this task is essentially: **insert some rows into `nudge_merge_tags`.** No schema migration is needed for the notifications themselves, because their Nudge-ID columns already exist. (Adding "default" rows can be done through the admin page or a small one-time data seed.)
---

## 5. The Nudge IDs to integrate, and how

These are the three notifications targeted for dynamic merge tags — the same three shown on the Nudge configuration page:

| Nudge ID | Notification | State today |
|---|---|---|
| `notify_payment_reversal` | Refund / reversal email to the customer | Static (hardcoded blanks) |
| `notify_payment_failure` | Payment-failed email to the customer | Sending code is already dynamic; only needs its blanks configured |
| `payment_link_request` | "Here's your payment link" email to the customer | Static (hardcoded blanks) |

**How to integrate a Nudge ID into the dynamic system (same three steps for each):**
1. **Template is set** — confirm the notification's Nudge ID (template) is filled in on the configuration page. For these three it already exists in `nudge_config`.
2. **Configure the blanks** — add a `nudge_merge_tags` entry that lists each blank (tag name) and the TilliPay field it pulls from (tag value). This is done on the admin configuration page.
3. **Point the code at the table** — change the sending code so it reads the blanks from the table instead of using the hardcoded list (with a fallback to the old list, so nothing breaks).
---

## 6. Steps to implement

**Step 0 — Add a safety helper.** A small shared helper that tries the database list first and, if it is empty, returns the old hardcoded list. This is the single most important step for not breaking anything.

**Step 1 — Payment reversal → dynamic.** In the reversal notification function, build the facts bundle and switch the blanks from the hardcoded list to the database lookup (with fallback). This one function feeds every refund/reversal path in the app, so fixing it in one place upgrades them all at once.

**Step 2 — Payment failure → verify + tiny fix.** The sending code is already dynamic. Just: (a) fix a copy-paste bug where the log is labelled `PAYMENT_REVERSAL` instead of `PAYMENT_FAILURE`, and (b) confirm its blanks are configured in the database.

**Step 3 — Payment link request → dynamic.** The payment-link notification is built in a few places (normal send, resend, bulk send, and the India version). Update each to use the database lookup (with fallback), ideally through the same shared helper so they stay consistent.

**Step 4 — Seed the default blanks.** Add "default" rows to `nudge_merge_tags` for reversal and payment-link (and confirm failure) that reproduce today's blanks. This makes the default behaviour dynamic-by-data, with the hardcoded list acting only as a safety net.

**Step 5 — Test** (see section 8).
---

## 7. What happens after implementing

- **Staff can edit the blanks** for reversal, failure, and payment-link emails directly from the configuration page — add, rename, or remove blanks — with no developer involvement and no redeploy.
- **Each merchant can customise** these notifications for themselves, instead of everyone sharing one hardcoded set.
- **Existing merchants are unaffected** — if a merchant has not configured any blanks, they keep receiving exactly the same emails as before (because of the fallback).
- The only new data created is rows in `nudge_merge_tags`; nothing else in the system changes shape.

**How the admin uses it (step by step, after implementation):**
1. Open the **Nudge configuration page** — for a specific merchant, or for the shared "default" that applies to everyone.
2. Find the notification to edit (e.g. *Payment reversal*, *Payment link request*).
3. For each blank in the Nudge template, add a row with two values:
   - **Tag name** = the placeholder in the Nudge template (e.g. `customer_name`).
   - **Tag value** = the TilliPay field the real value comes from (e.g. `customer_name`).
4. Add, rename, or remove rows as needed, then **Save**.
5. The change takes effect on the **next notification sent** — no code change, no redeploy.

*What the admin should keep in mind:* a tag value must be a field the system actually provides for that notification (the available fields are the same facts the message already knows — customer name, amount, date, etc.). If a blank is left unconfigured for a merchant, that merchant keeps getting the original default content (the safety fallback).
---

## 8. Making sure existing functionality is not broken

- **Fallback to the old list.** If the database has no blanks for a notification, the code uses the original hardcoded list. So an un-configured merchant sees no change.
- **One function per notification.** Reversal flows through a single function, so there is one place to change and one place to test — no risk of missing a caller.
- **Keep US vs India logic.** The existing rule that picks India templates/settings for India accounts stays exactly as-is.
- **Keep masking.** Card/account numbers are still reduced to the last 4 digits before filling blanks.
- **Test both cases:**
  1. A merchant **with** configured blanks → email uses the new custom blanks.
  2. A merchant **without** blanks → email is **identical to today** (fallback).
  3. Exercise reversal through all its paths (refund, reversal, US, India, SDK) and confirm none break.
  4. Confirm the failure log now reads `PAYMENT_FAILURE`.
---

## 9. Full inventory of notifications

There are **28 notifications** (Nudge IDs) in the configuration, plus **4 non-notification setup fields** (a login token and web-push keys) — **28 + 4 = 32 fields** on the configuration tab.

**Status** shows whether dynamic merge-tag support is in place: **Dynamic ✅** or **Static ❌**.
For dynamic notifications, "Current merge tags" is "configured in the database" (whatever staff set); the available runtime fields are noted.

| # | Nudge ID | Notification | Triggered when | Channel | Current merge tags | Status |
|---|---|---|---|---|---|---|
| 1 | `notify_via_email` | Payment success → merchant | A payment completes successfully | Email | Configured in DB (v2). Old static list: `Client_request_ID, submerchant_name, Remittance_ID, Transaction_ID, customer_name, payment_amount, Payment_date` | Dynamic ✅ |
| 2 | `notify_via_email_customer` | Payment success → customer | A payment completes successfully | Email | Configured in DB (v2). Old: `Client_request_ID, Remittance_ID, Transaction_ID, customer_name, payment_amount, Payment_date` | Dynamic ✅ |
| 3 | `notify_via_sms` | Payment success → merchant | A payment completes successfully | SMS | Configured in DB (v2). Old: `submerchant_name, customer_name, payment_amount` | Dynamic ✅ |
| 4 | `notify_via_sms_customer` | Payment success → customer | A payment completes successfully | SMS | Configured in DB (v2). Old: `customer_name, payment_amount` | Dynamic ✅ |
| 5 | `notify_via_email_ind` | Payment success → merchant (India) | A payment completes successfully (India account) | Email | `submerchant_name, Transaction_ID, customer_name, payment_amount, Payment_date` | Static ❌ |
| 6 | `notify_via_email_customer_ind` | Payment success → customer (India) | A payment completes successfully (India account) | Email | `Transaction_ID, customer_name, payment_amount, Payment_date` | Static ❌ |
| 7 | `notify_via_sms_ind` | Payment success → merchant (India) | A payment completes successfully (India account) | SMS | `submerchant_name, Transaction_ID, customer_name, payment_amount, Payment_date` | Static ❌ |
| 8 | `notify_via_sms_customer_ind` | Payment success → customer (India) | A payment completes successfully (India account) | SMS | `Transaction_ID, customer_name, payment_amount, Payment_date` | Static ❌ |
| 9 | `notify_payment_failure` | Payment failed → customer | A payment attempt is declined or errors out | Email | Configured in DB; runtime provides `Payment_amount, last_four_digits_of_card` (+ request fields) | Dynamic ✅ |
| 10 | `notify_payment_reversal` | Refund / reversal → customer | A refund or reversal is processed | Email | `customer_name, Refund_amount, Payment_amount, Refund_days, currency` | Static ❌ **(to do)** |
| 11 | `payment_link_request` | "Here's your payment link" → customer | A merchant shares a payment link with a customer | Email | `requested, amountUnit, requestedAmount, paymenturl, customer, payment_for, Customer_Name, customer_number` (+ `customField3–5`) | Static ❌ **(to do)** |
| 12 | `add_wallet_id` | Card/bank added to wallet | A card or bank account is saved to a wallet | Email | `customerName, walletType, walletNumber, walletHolderName` | Static ❌ |
| 13 | `update_wallet_id` | Saved card/bank updated | A saved card/bank's details are updated | Email | `customerName, walletType, walletNumber, walletHolderName, walletHolderUpdatedName` | Static ❌ |
| 14 | `delete_wallet_id` | Saved card/bank removed | A saved card/bank is removed | Email | `customerName, walletType, walletNumber, walletHolderName` | Static ❌ |
| 15 | `save_wallet_id` | Wallet saved confirmation | A wallet is saved (Freeman flow) | Email | large static set | Static ❌ |
| 16 | `notify_wallet_failure` | Google/Apple Pay payment failed | A Google/Apple Pay payment fails | Email | `customer_name, Payment_amount, wallet` | Static ❌ |
| 17 | `phone_otp` | One-time code by SMS | A login/verification code is requested by SMS | SMS | `sms_otp` | Static ❌ |
| 18 | `email_otp` | One-time code by email | A login/verification code is requested by email | Email | `otp, type` | Static ❌ |
| 19 | `email_verify` | "Verify your email" link | A new user or sub-user registers | Email | `name, url_new` | Static ❌ |
| 20 | `forget_password` | Password reset link | A user (or admin) requests a password reset | Email | `name, URL` | Static ❌ |
| 21 | `wrong_password_email_nudge_id` | Wrong-password attempt alert | A wrong-password login attempt occurs | Email | `CustomerName, PlatformName, CurrentDate` | Static ❌ |
| 22 | `share_to_owner` | Share an agreement to the owner | An agreement is shared with the owner | Email | `user_name, owner_name` | Static ❌ |
| 23 | `notify_on_kyc_submit` | "KYC submitted" confirmation | A merchant finishes KYC | Email | `Submerchant_name` | Static ❌ |
| 24 | `notify_on_live_mode` | "You're now in live mode" | A merchant switches to live mode | Email | `Submerchant_name` | Static ❌ |
| 25 | `notify_to_admin` | Internal alert to admin (DB problem) | A database problem occurs at startup | Email | (none) | Static ❌ |
| 26 | `notify_to_csr` | New support ticket → CSR | A new support ticket is opened | Email | `Ticket_Id, Priority, Type, Subject, Email` | Static ❌ |
| 27 | `notify_to_assignee` | Ticket assigned → assignee | A support ticket is assigned to someone | Email | `Ticket_Id, Priority, Type, Subject` | Static ❌ |
| 28 | `notify_to_customer_on_resolve` | Ticket resolved → customer | A support ticket is marked resolved | Email | `ticketId, resolutionNotes, customerEmail` | Static ❌ |

*The 4 non-notification setup fields:* `nudge_token` (login token), `nudge_api_key`, `nudge_client_id`, `nudge_web` (client-side web push — not templated emails/SMS).

**Summary:** 5 dynamic · 23 static · 3 in scope (rows 9, 10, 11).
---

## Appendix — code references (kept for future development)

Not needed to understand the plan; useful when doing the implementation.

**Configuration layer**
- Settings tables/models: `nudge_config`, `nudge_config_ind` → `nodejs_gps/src/db/models/`.
- Loader (settings → memory): `loadNudgeConfig()` → `nodejs_gps/src/utils/bin/global.js:36` (runs at `nodejs_gps/src/bin/index.js:112`).
- Blanks table: `nudge_merge_tags` → model `nodejs_gps/src/db/models/nudge_merge_tags.js`, migration `nodejs_gps/src/db/migrations/20260421110530-create-nudge-merge-tags.js`.
- Send function: `sendNudgeCommunication(...)` → `nodejs_gps/src/helpers/nudge.helper.js:21`.
- Dynamic lookup: `resolveTagsFromDb(...)` → `nodejs_gps/src/helpers/nudge.helper.js:74`.
- Admin screens: `gps_admin/src/layouts/admin/sub-merchants/components/nudgeConfiguration/` and `gps_admin/src/layouts/admin/for-developers/thirdParty/Nudge/`.
- Admin endpoints: `POST /v1/admin/nudge/merge-tags`, `DELETE /v1/admin/nudge/merge-tags/:id` (`nodejs_gps/src/modules/admin/admin.controller.js:1238, 1266`; routes `nodejs_gps/src/routes/admin/index.js:1883`).

**Reference implementation (already dynamic)**
- Payment success: `notifyOnSuccessPayment` → `nodejs_gps/src/modules/billing-info/v2/billinginfo.service.js:1278` (sends at `:1433, 1462, 1496, 1530`).
- Payment failure: `notifyOnFailedpayment` → `nodejs_gps/src/services/http.js:540` (send at `:573`; log-label bug at `:580`).

**To implement**
- Reversal: `notifyOnPaymentRevarsal` → `nodejs_gps/src/services/http.js:485` (static tags at `:507`; debug logs to remove at `:492, 499`). Called from ~18 places incl. `nodejs_gps/src/modules/sdk/v1/reversal.service.js:119`, `nodejs_gps/src/modules/sdk/v1/refund.service.js:193`, and v1/v2/v3/India billing services.
- Payment link: `notifyOnSharePaymentLink` (`nodejs_gps/src/modules/billing-info/billinginfo.service.js:1204`), `sendPayLLink` (`:1294`), `sendUserPaymentLinkV2` (`:1032`, bulk), India `notifyOnSharePaymentLinkIND` (`nodejs_gps/src/modules/billing-info/billinginfo.service.ind.js:561`).
