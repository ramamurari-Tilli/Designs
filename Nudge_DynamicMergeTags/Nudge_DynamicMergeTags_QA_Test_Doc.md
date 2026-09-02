# Nudge Dynamic Merge Tags — QA Test Doc

Verifies that Payment Link, Payment Reversal and Payment Failure notification merge tags are **dynamic** — edited in Admin, reflected in the email, no code change. **Nudge ID** = the number (e.g. `25140`); keep it the **same on the merchant page and the For-Developers default page**.

## How to use

- Each test case: **Steps → Expected → Screenshot**.
- Core check: **change a tag in Admin → trigger the notification → if the email changes, it is dynamic.**
- "Before" = original tags/email; "After" = changed tags/email.

## Prerequisites

- Environment: **dev / QA** running this build; Admin portal access.
- A merchant that can **actually take payments**, and access to the **customer inbox** to verify emails.
- Each notification has a **valid Nudge ID (template)** set.

> **If no email arrives:** check the **Nudge ID (template number)** is set and is a **valid template** on the Nudge side — and is the **same on the merchant page and the For-Developers default page**. A blank/invalid template = no email is sent.

---

# SECTION A — Payment Link (per-merchant)

Where: Admin → Merchants → merchant → **Update Merchant Details → Nudge Configuration**; trigger from the merchant portal **Share Payment**.
Tags: `requestedAmount → amount`, `tilliPayPaymentUrl → uri`, `payment_for → payment_for`, `Customer_Name → customer_name`, `customer_number → customer_number`..............

### TC-NDT-A1 — Configure merge tags
- Steps: Admin → Merchants → merchant → Update Merchant Details → **Nudge Configuration** → tag icon on **Payment Link Request** → add/view tags → Save.
- Expected: tags saved (icon green).

![](<Screenshot 2026-09-01 152242.png>)

### TC-NDT-A2 — Email uses the tags
- Steps: Merchant portal → **Share Payment** → share link.
- Expected: email received — *"Dear murari with 123 … for 39 for foos"*.

![](<Screenshot 2026-09-01 152339.png>)

### TC-NDT-A3 — Prove dynamic
- Steps: Edit → swap `Customer_Name` / `customer_number` → Save → share again(change the tag values).
- Expected: email changes — *"Dear 123 with murari …"*.

![](<Screenshot 2026-09-01 152616.png>)

![](<Screenshot 2026-09-01 152610.png>)

---

# SECTION B — Payment Reversal (per-merchant)

Where: Admin → Merchants → merchant → **Update Merchant Details → Nudge Configuration**; trigger from the merchant portal **Transactions → refund/reversal**.
Tags: `customer_name → customer_name`, `Refund_amount → amount`, `Payment_amount → amount`,........

### TC-NDT-B1 — Configure merge tags
- Steps: Admin → Merchants → merchant → Update Merchant Details → **Nudge Configuration** → tag icon on **Notify payment reversal** → add/view tags → Save.
- Expected: tags saved (icon green).

![](<Screenshot 2026-09-01 195318.png>)

### TC-NDT-B2 — Email uses the tags
- Steps: Merchant portal → **Transactions** → process a refund/reversal.
- Expected: email received — *"Dear rama murari, you refund amount of 12 for payment amount 12"*.

![](<Screenshot 2026-09-01 195436.png>)

### TC-NDT-B3 — Prove dynamic
- Steps: Edit → change tag values → Save → process the reversal again.
- Expected: email changes — *"Dear 12, you refund amount of rama murari …"*.

![](<Screenshot 2026-09-01 195242.png>)

![](<Screenshot 2026-09-01 195227.png>)

---

# SECTION C — Payment Link (default page — For Developers)

Where: Admin → **For Developers → Third Party → Nudge (Nav bar)** → US; trigger from the merchant portal **Share Payment**.
Same tags as Section A; used when the merchant has no per-merchant tags. Same Nudge ID as the merchant page.

### TC-NDT-C1 — Configure merge tags (default)
- Steps: Admin → **For Developers → Third Party → Nudge (Nav bar)** → US → tag icon on **Payment Link Request** → add/view tags → Save.
- Expected: tags saved (icon green).

![](<Screenshot 2026-09-01 234148.png>)

### TC-NDT-C2 — Email uses the tags (default)
- Steps: Merchant portal → **Share Payment** → share link.
- Expected: email received — *"Dear murari with 123 … for 39 for foos"*.

![](<Screenshot 2026-09-01 234206.png>)

### TC-NDT-C3 — Prove dynamic (default)
- Steps: Edit → change tag values → Save → share again.
- Expected: email changes — *"Dear 123 with 123 … pay through 39"*.

![](<Screenshot 2026-09-01 234058.png>)

![](<Screenshot 2026-09-01 233858.png>)

---

# SECTION D — Payment Reversal (default page — For Developers)

Where: Admin → **For Developers → Third Party → Nudge (Nav bar)** → US; trigger from the merchant portal **Transactions → refund/reversal**.
Same tags as Section B. Same Nudge ID as the merchant page.

### TC-NDT-D1 — Configure merge tags (default)
- Steps: Admin → **For Developers → Third Party → Nudge (Nav bar)** → US → tag icon on **Notify payment reversal** → add/view tags → Save.
- Expected: tags saved (icon green).

![](<Screenshot 2026-09-02 145638.png>)

### TC-NDT-D2 — Email uses the tags (default)
- Steps: Merchant portal → **Transactions** → process a refund/reversal.
- Expected: email received — *"Dear murari, you refund amount of 39 for payment amount 39"*.

![](<Screenshot 2026-09-02 145605.png>)

### TC-NDT-D3 — Prove dynamic (default)
- Steps: Edit → change tag values → Save → process the reversal again.
- Expected: email changes — *"Dear 123, you refund amount of rama …"*.

![](<Screenshot 2026-09-02 145447.png>)

![](<Screenshot 2026-09-02 145429.png>)

---

# SECTION E — Payment Failure (already dynamic-tested in qa)

Where: Admin → Nudge Config → **Notify payment failure**; trigger by a declined/failed payment (from the payment page).
Tags: `customer_name → customerName`, `Payment_amount → amount`, `last_four_digits_of_card → last_four_digits_of_card`........

### TC-NDT-E1 — Configure merge tags
- Steps: Admin → Nudge Config → **Notify payment failure** → tag icon → add/view tags → Save.
- Expected: tags saved (icon green).

![](<Screenshot 2026-09-02 162019.png>)

### TC-NDT-E2 — Email uses the tags
- Steps: Make a payment that is **declined/fails**(from payment page).
- Expected: email received — *"Dear rama, your payment for 123 made with card ending with 5432 has been failed"*.

![](<Screenshot 2026-09-02 162004.png>)

### TC-NDT-E3 — Prove dynamic
- Steps: Edit → swap tag values → Save → trigger another failed payment(from payment page).
- Expected: email changes — *"Dear 123, your payment for rama …"*.

![](<Screenshot 2026-09-02 162153.png>)

![](<Screenshot 2026-09-02 162135.png>)

---

## Sign-off

- [ ] A — Payment Link (per-merchant)
- [ ] B — Payment Reversal (per-merchant)
- [ ] C — Payment Link (default page)
- [ ] D — Payment Reversal (default page)
- [ ] E — Payment Failure
