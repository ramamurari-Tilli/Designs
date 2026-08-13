# Nudge Configuration — Inventory & Plan

This document explains: what the Nudge notification system is, how it is set up in TilliPay, what is already built, what still needs to be built and how, a full list of every notification (the "inventory"), and where each one is triggered from in the code.
---

## 1. The basics

**What is Nudge?**
Nudge is an outside service that actually sends the emails and text messages to customers. Nudge stores the message templates — for example: *"Hi {{customer_name}}, your payment of {{payment_amount}} was received."* TilliPay does not write the message. TilliPay only tells Nudge **which template to use** and **what real values to put into the blanks**.

**What is a "merge tag"?**
A merge tag is a **blank in the template**. In *"Hi {{customer_name}}"*, the blank is `customer_name`. Each blank has two sides:
- **Tag name** = the name of the blank inside the Nudge template (the Nudge side), e.g. `customer_name`.
- **Tag value** = the field on the **TilliPay side** that holds the real value (e.g. our customer's actual name). TilliPay looks up that real value and hands it to Nudge, which drops it into the blank.

**What is "Nudge configuration"?**
It's a settings table that answers two questions for every kind of notification:
1. **Which Nudge template should we use?** (stored as a template ID number, called the "Nudge ID").
2. *What blanks does that template have, and where do their values come from?* (the merge tags).
---

## 2. Static vs. Dynamic (the heart of the whole task)

Both words describe **one thing only: where the list of blanks comes from.** Everything else is the same.

**Static = the blanks are hardcoded in the app.**
A developer wrote the exact list of blanks directly into the program's code. It's fixed. Every merchant gets the same blanks. To change anything, a developer must edit the code and redeploy. It's like a **printed form** — you can't add or remove a field without reprinting it.

**Dynamic = the blanks come from a database table that staff can edit.**
Instead of hardcoding the list, the app asks the database: *"what blanks are set up for this notification, for this merchant?"* Staff manage that list from the admin screen — no developer, no redeploy. It's like a **fillable digital form** where an admin can add, remove, or rename fields whenever they want.

The technical difference is tiny in the code:
- Static looks like a hardcoded list: `mergeTags: [ { tagName:'customer_name', tagValue: ... }, ... ]`
- Dynamic looks like one line: `mergeTags: await resolveTagsFromDb(accountId, '<notification>', runtimeData)`

**One gotcha with dynamic:** if an admin hasn't set up any blanks yet for a notification, the dynamic lookup returns **nothing**, and the email would go out with empty blanks. The static way can't have that problem because the list is always in the code. That's why the safe plan is: *use the table if it has entries, otherwise fall back to the old hardcoded list.*
---

## 3. Where everything lives

**A. The settings table (the "Nudge IDs")**
- Two database tables: `nudge_config` (US) and `nudge_config_ind` (India).
  Models: `nodejs_gps/src/db/models/nudge_config.js` and `nodejs_gps/src/db/models/nudge_config_ind.js`.
- **Each row = one merchant's setup.** There's a shared default row (`account_id = 'default'` for US, `'default_ind'` for India), and any merchant can have its own row to customize things.
- **Each column = one type of notification**, and the value stored there is that notification's Nudge template ID.

**B. The loader (settings → memory)**
- `loadNudgeConfig()` in `nodejs_gps/src/utils/bin/global.js:36` runs once when the app starts (`nodejs_gps/src/bin/index.js:112`). It reads the two default rows and keeps them in memory so the app doesn't hit the database on every notification. Merchant-specific rows are looked up on demand.

**C. The blanks table (for dynamic notifications)**
- `nudge_merge_tags` — model `nodejs_gps/src/db/models/nudge_merge_tags.js`, created by migration `nodejs_gps/src/db/migrations/20260421110530-create-nudge-merge-tags.js`.
- Each row says: *for this merchant + this notification, here is the list of blanks and where each value comes from.* Columns: `account_id`, `nudge_id` (the notification name, e.g. `notify_via_email`), and `merge_tags` (the list of blank-name → source-field pairs).
- It's read at send time by `resolveTagsFromDb(...)` in `nodejs_gps/src/helpers/nudge.helper.js:74`.

**D. The admin screens (where staff edit all this)** — in the `gps_admin` app:
- Per-merchant: `gps_admin/src/layouts/admin/sub-merchants/components/nudgeConfiguration/controller.js`
- Global/default: `gps_admin/src/layouts/admin/for-developers/thirdParty/Nudge/`
- These call the backend endpoints (see section 7).

**E. The send function and the log**
- Everything is sent through one function: `sendNudgeCommunication(...)` in `nodejs_gps/src/helpers/nudge.helper.js:21`.
- Every send is recorded in the `nudge_logs` table (who, success/failure, errors).
---

## 4. What is ALREADY implemented (and how)

**Only 5 notifications currently use the flexible/dynamic way.** They were all built together in a single change: commit **`f659f357` (28 Apr 2026)** — *"replace hardcoded mergeTags with DB-driven lookups for payment notifications."*

The 5 already-dynamic notifications:
1. **Payment success → merchant email** (`notify_via_email`)
2. **Payment success → customer email** (`notify_via_email_customer`)
3. **Payment success → merchant SMS** (`notify_via_sms`)
4. **Payment success → customer SMS** (`notify_via_sms_customer`)
5. **Payment failed → customer email** (`notify_payment_failure`)

> Note: the payment-failure **sending code** already performs the dynamic table lookup (it was part of that same batch of 5). It still needs its blanks configured on the configuration page to actually produce dynamic content, so it stays one of the three in-scope items (see section 5).

**How the already-built version works (the reference pattern to copy):**
The four payment-success messages are built in the function `notifyOnSuccessPayment` in `nodejs_gps/src/modules/billing-info/v2/billinginfo.service.js:1278`. In plain steps:
1. Figure out which settings to use (this merchant's row, or the default; US or India).
2. Decide the merchant key: `accountId = the merchant's account, or 'default'` if none.
3. Gather all the facts about this payment into one bundle (called `runtimeData`) — the amount, the date, names, etc. — and hide sensitive card/account numbers down to the last 4 digits.
4. Ask the table for the blanks: `resolveTagsFromDb(accountId, '<notification>', runtimeData)`.
5. Send it to Nudge with `sendNudgeCommunication(...)`, then write a `nudge_logs` record.

The payment-failure message (`notifyOnFailedpayment` in `nodejs_gps/src/services/http.js:540`) does the exact same thing, just shorter.
---

## 5. What still needs to be implemented

These are the three notifications targeted for dynamic merge tags — the same three shown on the Nudge configuration page: **payment reversal, payment failure, and payment link request.**

| Notification | What the code does today | What's left to do |
|---|---|---|
| **Payment reversal / refund** | Uses a hardcoded (static) list of blanks | Convert the sending code to the dynamic table lookup, then configure its blanks on the config page |
| **Payment failure** | Sending code already does the dynamic table lookup | Configure/verify its blanks on the config page (the only code change is a small log-label fix) |
| **Payment link request** | Uses a hardcoded (static) list of blanks | Convert the sending code to the dynamic table lookup, then configure its blanks on the config page |

For payment reversal and payment link request the work is **code + configuration**; for payment failure the sending code is already in place, so the work is mainly **configuration**.
---

## 6. How to implement the remaining work

**Step 0 — Add a safety helper (so nothing breaks).**
Add a small helper next to the existing one, in `nodejs_gps/src/helpers/nudge.helper.js`, called something like `resolveTagsWithFallback(accountId, notification, runtimeData, oldStaticList)`. It tries the table first; if the table has no entries, it returns the old hardcoded list instead.
*Why:* this guarantees that merchants who haven't set up blanks yet keep getting exactly the same emails as today — no surprises, no blank emails.

**Step 1 — Payment reversal → dynamic.**
- File/function: `notifyOnPaymentRevarsal` in `nodejs_gps/src/services/http.js:485`.
- Today: it uses a hardcoded list of blanks (`customer_name`, `Refund_amount`, `Payment_amount`, `Refund_days`, `currency`) at `nodejs_gps/src/services/http.js:507`.
- Change: build a `runtimeData` bundle with those same facts, then replace the hardcoded list with `resolveTagsWithFallback(accountId, 'notify_payment_reversal', runtimeData, <the old list>)`.
- **Bonus:** this one function is called from ~18 different places (refunds, reversals, US/India, SDK). Because they all go through this single function, fixing it here upgrades **all** of them at once — no need to touch the 18 callers.
- Also remove two leftover debug `console.log` lines at `nodejs_gps/src/services/http.js:492` and `nodejs_gps/src/services/http.js:499`.

**Step 2 — Payment failure → just verify + tiny fix.**
- File/function: `notifyOnFailedpayment` in `nodejs_gps/src/services/http.js:540`. Already dynamic.
- Fix a copy-paste bug: it writes the log type as `'PAYMENT_REVERSAL'` at `nodejs_gps/src/services/http.js:580` — it should say `'PAYMENT_FAILURE'`.
- Optionally switch it to the same `resolveTagsWithFallback` helper for the empty-blanks safety net.
- Confirm the `nudge_merge_tags` table actually has rows for `notify_payment_failure`.

**Step 3 — Payment link request → dynamic (4 places).**
- Files/functions (all currently static):
  - `notifyOnSharePaymentLink` — `nodejs_gps/src/modules/billing-info/billinginfo.service.js:1159`
  - `sendPayLLink` — `nodejs_gps/src/modules/billing-info/billinginfo.service.js:1222`
  - `sendUserPaymentLinkV2` (builds the payload used by bulk sends) — `nodejs_gps/src/modules/billing-info/billinginfo.service.js:938`
  - India version `notifyOnSharePaymentLinkIND` — `nodejs_gps/src/modules/billing-info/billinginfo.service.ind.js:525`
- Change: in each, build a `runtimeData` bundle with the current fields (the payment URL, amount, customer name, etc.) and replace the hardcoded list with `resolveTagsWithFallback(accountId, 'payment_link_request', runtimeData, <the old list>)`.
- Using the shared helper keeps all four in sync and avoids copy-paste drift.

**Step 4 — Seed the default blanks (data, not code).**
Add the "default" rows to `nudge_merge_tags` for reversal and payment-link (and confirm failure) that reproduce today's blanks. This can be done with a small data migration or via the admin screen (`POST /v1/admin/nudge/merge-tags`). *Why:* so the default accounts also work the dynamic way, and the hardcoded fallback becomes just a safety net.

**Things to watch out for (risks):**
- *Empty blanks:* the #1 risk — a dynamic notification with no table rows sends empty blanks. The fallback helper (Step 0) prevents this.
- *Field-name mismatch:* the table points at TilliPay field names; if a field isn't in the `runtimeData` bundle, that blank comes out empty. Make sure each bundle includes every field the blanks reference.
- *Silent failures:* these notify functions quietly swallow errors, so a misconfiguration won't crash anything — it'll just produce a bad email. Check the `nudge_logs` table and test real sends.
- *US vs India:* keep the existing logic that picks the India settings/templates for India accounts.
- *Sensitive data:* mask card/account numbers to the last 4 digits before filling blanks, like the payment-success code already does.

**How to test:**
1. A merchant **with** table rows → the email uses the custom blanks.
2. A merchant **without** rows → the email is identical to today (the fallback kicks in).
3. Run reversal through all its paths (refund, reversal, US, India, SDK) and confirm none break.
4. Confirm the failure log now says `PAYMENT_FAILURE`.

---

## 7. How the configuration is edited (for reference)

- `POST /v1/admin/nudge/merge-tags` → add/update a notification's blanks (`nodejs_gps/src/modules/admin/admin.controller.js:1238`; route `nodejs_gps/src/routes/admin/index.js:1883`).
- `DELETE /v1/admin/nudge/merge-tags/:id` → remove a set of blanks (`nodejs_gps/src/modules/admin/admin.controller.js:1266`).
- The blanks list is returned together with the config by `getNudgeConfiguration` (`nodejs_gps/src/modules/admin/admin.service.js:4798`).
- Create/update/get/delete the settings row itself: `POST /v1/admin/nudgeConfig/create|update`, `GET .../get`, `DELETE .../delete` (`nodejs_gps/src/routes/admin/index.js:1519, 1679, 1839, 1862`). Editing the default row reloads the in-memory config automatically (`nodejs_gps/src/modules/admin/admin.service.js:4861`).
---

## 8. Full inventory of Nudge notifications

There are **28 notification types** stored as columns in the settings table (plus 4 non-message fields — the API token and web-push keys — which aren't notifications). **28 + 4 = 32 fields** on the configuration tab.

Column notes: **Current merge tags** = the tag names the notification uses today (for dynamic ones the list is "DB-configured", i.e. whatever an admin sets in `nudge_merge_tags`; the runtime fields available are noted). **Mode** = *Dynamic* (reads the editable table) or *Static* (hardcoded in code). **Status** = whether dynamic merge-tag support is Implemented (✅) or not (❌).

| # | Nudge ID | Notification type | Channel | Current merge tags | Mode | Status | Triggered from |
|---|---|---|---|---|---|---|---|
| 1 | `notify_via_email` | Payment succeeded → merchant, by email | Email | DB-configured (v2). v1: `Client_request_ID, submerchant_name, Remittance_ID, Transaction_ID, customer_name, payment_amount, Payment_date` | Dynamic (v2) / Static (v1) | ✅ Done (v2) | `nodejs_gps/src/modules/billing-info/v2/billinginfo.service.js:1433` |
| 2 | `notify_via_email_customer` | Payment succeeded → customer, by email | Email | DB-configured (v2). v1: `Client_request_ID, Remittance_ID, Transaction_ID, customer_name, payment_amount, Payment_date` | Dynamic (v2) / Static (v1) | ✅ Done (v2) | `nodejs_gps/src/modules/billing-info/v2/billinginfo.service.js:1462` |
| 3 | `notify_via_sms` | Payment succeeded → merchant, by SMS | SMS | DB-configured (v2). v1: `submerchant_name, customer_name, payment_amount` | Dynamic (v2) / Static (v1) | ✅ Done (v2) | `nodejs_gps/src/modules/billing-info/v2/billinginfo.service.js:1496` |
| 4 | `notify_via_sms_customer` | Payment succeeded → customer, by SMS | SMS | DB-configured (v2). v1: `customer_name, payment_amount` | Dynamic (v2) / Static (v1) | ✅ Done (v2) | `nodejs_gps/src/modules/billing-info/v2/billinginfo.service.js:1530` |
| 5 | `notify_via_email_ind` | Payment succeeded → merchant (India) | Email | `submerchant_name, Transaction_ID, customer_name, payment_amount, Payment_date` | Static | ❌ | `nodejs_gps/src/modules/billing-info/billinginfo.service.ind.js:421` |
| 6 | `notify_via_email_customer_ind` | Payment succeeded → customer (India) | Email | `Transaction_ID, customer_name, payment_amount, Payment_date` | Static | ❌ | `nodejs_gps/src/modules/billing-info/billinginfo.service.ind.js:448` |
| 7 | `notify_via_sms_ind` | Payment succeeded → merchant (India) | SMS | `submerchant_name, Transaction_ID, customer_name, payment_amount, Payment_date` | Static | ❌ | `nodejs_gps/src/modules/billing-info/billinginfo.service.ind.js:475` |
| 8 | `notify_via_sms_customer_ind` | Payment succeeded → customer (India) | SMS | `Transaction_ID, customer_name, payment_amount, Payment_date` | Static | ❌ | `nodejs_gps/src/modules/billing-info/billinginfo.service.ind.js:502` |
| 9 | `notify_payment_failure` | Payment failed → customer | Email | DB-configured; runtime provides `Payment_amount, last_four_digits_of_card` (+ request fields) | **Dynamic** | ✅ Done | `nodejs_gps/src/services/http.js:573` |
| 10 | `notify_payment_reversal` | Payment reversed / refunded → customer | Email | `customer_name, Refund_amount, Payment_amount, Refund_days, currency` | Static | ❌ **To do** | `nodejs_gps/src/services/http.js:518` |
| 11 | `payment_link_request` | "Here's your payment link" → customer | Email | `requested, amountUnit, requestedAmount, paymenturl, customer, payment_for, Customer_Name, customer_number` (+ `customField3–5`) | Static | ❌ **To do** | `nodejs_gps/src/modules/billing-info/billinginfo.service.js:1204`, `:1294`; India `nodejs_gps/src/modules/billing-info/billinginfo.service.ind.js:561` |
| 12 | `add_wallet_id` | Card/bank added to wallet | Email | `customerName, walletType, walletNumber, walletHolderName` | Static | ❌ | `nodejs_gps/src/modules/billing-info/v2/billinginfo.service.js:858, 986, 6021` |
| 13 | `update_wallet_id` | Saved card/bank details updated | Email | `customerName, walletType, walletNumber, walletHolderName, walletHolderUpdatedName` | Static | ❌ | `nodejs_gps/src/modules/billing-info/billinginfo.controller.js:1232` |
| 14 | `delete_wallet_id` | Saved card/bank removed | Email | `customerName, walletType, walletNumber, walletHolderName` | Static | ❌ | `nodejs_gps/src/modules/billing-info/v2/billinginfo.service.js:4329, 6268` |
| 15 | `save_wallet_id` | Wallet saved confirmation (Freeman flow) | Email | large static set (Freeman flow) | Static | ❌ | `nodejs_gps/src/modules/freeman/freeman.service.js:360, 368` |
| 16 | `notify_wallet_failure` | Google/Apple Pay payment failed | Email | `customer_name, Payment_amount, wallet` | Static | ❌ | `nodejs_gps/src/services/utillipay-gps.js:5507` |
| 17 | `phone_otp` | One-time code by SMS (login / verify) | SMS | `sms_otp` | Static | ❌ | `nodejs_gps/src/modules/users/user.repository.js:175`; `nodejs_gps/src/modules/users/user.service.js:1477, 1602`; `nodejs_gps/src/services/http.js:371` |
| 18 | `email_otp` | One-time code by email (login / verify) | Email | `otp, type` | Static | ❌ | `nodejs_gps/src/modules/users/user.service.js:1033, 1089, 1707, 1851, 1939`; `nodejs_gps/src/services/http.js:456` |
| 19 | `email_verify` | "Verify your email" link | Email | `name, url_new` | Static | ❌ | `nodejs_gps/src/modules/users/user.service.js:1326`; `nodejs_gps/src/modules/user-roles/user-roles.services.js:856` |
| 20 | `forget_password` | Password reset link | Email | `name, URL` | Static | ❌ | `nodejs_gps/src/modules/users/user.service.js:2035`; `nodejs_gps/src/modules/admin/admin.service.js:2486` |
| 21 | `wrong_password_email_nudge_id` | "Wrong password attempt" alert | Email | `CustomerName, PlatformName, CurrentDate` | Static | ❌ | `nodejs_gps/src/modules/users/user.service.js:2414` |
| 22 | `share_to_owner` | Share an agreement to the owner | Email | `user_name, owner_name` | Static | ❌ | `nodejs_gps/src/modules/aggrement/aggrement.service.js:97` |
| 23 | `notify_on_kyc_submit` | "KYC submitted" confirmation | Email | `Submerchant_name` | Static | ❌ | `nodejs_gps/src/services/http.js:611` |
| 24 | `notify_on_live_mode` | "You're now in live mode" | Email | `Submerchant_name` | Static | ❌ | `nodejs_gps/src/services/http.js:649` |
| 25 | `notify_to_admin` | Internal alert to admin (DB problem) | Email | (none) | Static | ❌ | `nodejs_gps/src/services/http.js:676` |
| 26 | `notify_to_csr` | New support ticket → notify CSR | Email | `Ticket_Id, Priority, Type, Subject, Email` | Static | ❌ | `nodejs_gps/src/modules/support/support.service.js:51` |
| 27 | `notify_to_assignee` | Ticket assigned → notify assignee | Email | `Ticket_Id, Priority, Type, Subject` | Static | ❌ | `nodejs_gps/src/modules/support/support.service.js:167` |
| 28 | `notify_to_customer_on_resolve` | Ticket resolved → notify customer | Email | `ticketId, resolutionNotes, customerEmail` | Static | ❌ | `nodejs_gps/src/modules/support/support.service.js:227` |

*The 4 non-notification fields on the same tab:* `nudge_token` (login token), `nudge_api_key`, `nudge_client_id`, `nudge_web` (these power client-side web push, not templated emails/SMS).

*Separate service:* the `gps-node-jobs` app also sends some emails (merchant cron reminders, card-expiry alerts) using its own sender (`gps-node-jobs/src/services/nudge.js`) — these are outside the dynamic merge-tag mechanism.
---

## 9. Where each notification is triggered from (grouped)

"Triggered from" = the real business event and the code that fires the notification. The per-notification file:line addresses are in the inventory table (section 8); this section adds the **callers** (who fires each one).

| Notification | Triggered when… | Where in code (function @ full path:line) |
|---|---|---|
| Payment success (email/SMS, merchant & customer) | A payment completes successfully | `notifyOnSuccessPayment` (v2) `nodejs_gps/src/modules/billing-info/v2/billinginfo.service.js:1278`; legacy `nodejs_gps/src/modules/billing-info/billinginfo.service.js:1792`; India `nodejs_gps/src/modules/billing-info/billinginfo.service.ind.js:393` |
| **Payment failure** | A payment attempt is declined / errors out | `notifyOnFailedpayment` `nodejs_gps/src/services/http.js:540`, called from `nodejs_gps/src/modules/billing-info/billinginfo.service.js:1654, 1770, 2154, 5432`; `nodejs_gps/src/modules/billing-info/v2/billinginfo.service.js:610, 731, 1137, 1256, 1863, 3166`; SDK `nodejs_gps/src/modules/sdk/v1/payment.service.js:254` |
| **Payment reversal / refund** | A refund or reversal is processed | `notifyOnPaymentRevarsal` `nodejs_gps/src/services/http.js:485`, called from ~18 spots: SDK `nodejs_gps/src/modules/sdk/v1/reversal.service.js:119`, SDK `nodejs_gps/src/modules/sdk/v1/refund.service.js:193`, `nodejs_gps/src/modules/billing-info/billinginfo.service.js:2011, 2795, 6051`, `nodejs_gps/src/modules/billing-info/v2/billinginfo.service.js:1635, 1734, 2084`, India `nodejs_gps/src/modules/billing-info/billinginfo.service.ind.js` (11 spots), v3 `nodejs_gps/src/modules/billing-info/v3/billinginfo.service.js:343` |
| **Payment link request** | A merchant shares a payment link with a customer | `notifyOnSharePaymentLink` `nodejs_gps/src/modules/billing-info/billinginfo.service.js:1204`, `sendPayLLink` `nodejs_gps/src/modules/billing-info/billinginfo.service.js:1294`, `sendUserPaymentLinkV2` `nodejs_gps/src/modules/billing-info/billinginfo.service.js:1032` (bulk), India `nodejs_gps/src/modules/billing-info/billinginfo.service.ind.js:561` |
| Wallet added | A card/bank is saved to a wallet | `saveCardDetails`/`saveBankDetails`/`sendAddWalletNudgeNotification` `nodejs_gps/src/modules/billing-info/v2/billinginfo.service.js:858, 986, 6021`; controller `nodejs_gps/src/modules/billing-info/v2/billinginfo.controller.js:225` |
| Wallet updated | Saved card/bank name is changed | `updateSavedCardACH` `nodejs_gps/src/modules/billing-info/billinginfo.controller.js:1232` |
| Wallet deleted | A saved token is removed | `nodejs_gps/src/modules/billing-info/v2/billinginfo.service.js:4329, 6268` |
| Wallet payment failed (Google/Apple Pay) | A wallet payment fails | `sendFailedPaymentNotification` `nodejs_gps/src/services/utillipay-gps.js:5507` |
| Phone OTP | Login two-factor, phone verification, share-pay OTP | `nodejs_gps/src/modules/users/user.repository.js:175`; `nodejs_gps/src/modules/users/user.service.js:1477, 1602`; `nodejs_gps/src/services/http.js:371` |
| Email OTP | Login two-factor, email/transaction verification | `nodejs_gps/src/modules/users/user.service.js:1033, 1089, 1707, 1851, 1939`; `nodejs_gps/src/services/http.js:456` |
| Email verification link | New user / sub-user registers | `nodejs_gps/src/modules/users/user.service.js:1326`; `nodejs_gps/src/modules/user-roles/user-roles.services.js:856` |
| Forgot password | User (or admin) requests a reset | `nodejs_gps/src/modules/users/user.service.js:2035`; `nodejs_gps/src/modules/admin/admin.service.js:2486` |
| Wrong-password alert | A wrong-password login attempt | `nodejs_gps/src/modules/users/user.service.js:2414` |
| Share agreement to owner | An agreement is shared | `nodejs_gps/src/modules/aggrement/aggrement.service.js:97` (route `POST /v1/aggrement/share`) |
| KYC submitted | Merchant finishes KYC | `sendMailKycSubmit` `nodejs_gps/src/services/http.js:611` |
| Switched to live mode | Merchant goes live | `sendMailLiveMode` `nodejs_gps/src/services/http.js:649` |
| Admin DB-problem alert | Database connection error at startup | `infoAdmin` `nodejs_gps/src/services/http.js:676` |
| Support: ticket created | A new support ticket is opened | `createTicket` `nodejs_gps/src/modules/support/support.service.js:51` |
| Support: ticket assigned | A ticket is picked/assigned | `pickTicket` `nodejs_gps/src/modules/support/support.service.js:167` |
| Support: ticket resolved | A ticket is marked resolved | `updateTicketResolution` `nodejs_gps/src/modules/support/support.service.js:227` |
---

## 10. One-line summary

Nudge configuration is a per-merchant settings table that maps each notification to a Nudge template and (for the flexible ones) to an editable list of blanks. **Five payment notifications are already flexible/dynamic — including payment failure.** The remaining job is to make **payment reversal** and **payment link request** flexible too, copying the pattern that payment-success already uses, with a safety fallback so nothing breaks for merchants who haven't configured their blanks yet.
