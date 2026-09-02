# Nudge Dynamic Merge Tags — Gateway Test Cases

Dev test cases proving Payment Link and Payment Reversal merge tags are dynamic across gateways. Each test case: configure tags → trigger → note email (before) → change tag values → trigger again (after). The after email differing — with no code change — proves it is dynamic.

---

# SECTION 1 — Payment Link (per-merchant)

Tags (Tag Name → Tag Value): requestedAmount → amount, tilliPayPaymentUrl → uri, payment_for → payment_for, Customer_Name → customer_name, customer_number → customer_number.

### TC-NDT-A1 — IPG

- Steps: Admin → Merchants → merchant → Update Merchant Details → Nudge Configuration → Payment Link Request tags → Save. Merchant portal → Share Payment → share link (before). Change tag values → Save → share again (after).
- Expected: The After email differs from the Before email (same data, changed mapping) → merge tags are dynamic.

**Config — before**

![](<Screenshot 2026-09-01 152242.png>)

**Email — before**

![](<Screenshot 2026-09-01 152339.png>)

**Config — after**

![](<Screenshot 2026-09-01 152616.png>)

**Email — after**

![](<Screenshot 2026-09-01 152610.png>)

### TC-NDT-A2 — SnapPay

- Steps: Admin → Merchants → merchant → Update Merchant Details → Nudge Configuration → Payment Link Request tags → Save. Merchant portal → Share Payment → share link (before). Change tag values → Save → share again (after).
- Expected: The After email differs from the Before email (same data, changed mapping) → merge tags are dynamic.

**Config — before**

![](<Screenshot 2026-09-01 154345.png>)

**Email — before**

![](<Screenshot 2026-09-01 154650.png>)

**Config — after**

![](<Screenshot 2026-09-01 155114.png>)

**Email — after**

![](<Screenshot 2026-09-01 155401.png>)

### TC-NDT-A3 — Paymentus

- Steps: Admin → Merchants → merchant → Update Merchant Details → Nudge Configuration → Payment Link Request tags → Save. Merchant portal → Share Payment → share link (before). Change tag values → Save → share again (after).
- Expected: The After email differs from the Before email (same data, changed mapping) → merge tags are dynamic.

**Config — before**

![](<Screenshot 2026-09-01 162420.png>)

**Email — before**

![](<Screenshot 2026-09-01 162448.png>)

**Config — after**

![](<Screenshot 2026-09-01 162620.png>)

**Email — after**

![](<Screenshot 2026-09-01 162634.png>)

### TC-NDT-A4 — MerchantE

- Steps: Admin → Merchants → merchant → Update Merchant Details → Nudge Configuration → Payment Link Request tags → Save. Merchant portal → Share Payment → share link (before). Change tag values → Save → share again (after).
- Expected: The After email differs from the Before email (same data, changed mapping) → merge tags are dynamic.

**Config — before**

![](<Screenshot 2026-09-01 174130.png>)

**Email — before**

![](<Screenshot 2026-09-01 174146.png>)

**Config — after**

![](<Screenshot 2026-09-01 174405.png>)

**Email — after**

![](<Screenshot 2026-09-01 174433.png>)

### TC-NDT-A5 — Commerce Hub

- Steps: Admin → Merchants → merchant → Update Merchant Details → Nudge Configuration → Payment Link Request tags → Save. Merchant portal → Share Payment → share link (before). Change tag values → Save → share again (after).
- Expected: The After email differs from the Before email (same data, changed mapping) → merge tags are dynamic.

**Config — before**

![](<Screenshot 2026-09-01 175245.png>)

**Email — before**

![](<Screenshot 2026-09-01 175311.png>)

**Config — after**

![](<Screenshot 2026-09-01 175416.png>)

**Email — after**

![](<Screenshot 2026-09-01 175504.png>)

### TC-NDT-A6 — BillPay

- Steps: Admin → Merchants → merchant → Update Merchant Details → Nudge Configuration → Payment Link Request tags → Save. Merchant portal → Share Payment → share link (before). Change tag values → Save → share again (after).
- Expected: The After email differs from the Before email (same data, changed mapping) → merge tags are dynamic.

**Config — before**

![](<Screenshot 2026-09-01 180343.png>)

**Email — before**

![](<Screenshot 2026-09-01 180426.png>)

**Config — after**

![](<Screenshot 2026-09-01 180543.png>)

**Email — after**

![](<Screenshot 2026-09-01 180609.png>)

### TC-NDT-A7 — NuveiPay

- Steps: Admin → Merchants → merchant → Update Merchant Details → Nudge Configuration → Payment Link Request tags → Save. Merchant portal → Share Payment → share link (before). Change tag values → Save → share again (after).
- Expected: The After email differs from the Before email (same data, changed mapping) → merge tags are dynamic.

**Config — before**

![](<Screenshot 2026-09-01 181044.png>)

**Email — before**

![](<Screenshot 2026-09-01 181205.png>)

**Config — after**

![](<Screenshot 2026-09-01 181324.png>)

**Email — after**

![](<Screenshot 2026-09-01 181418.png>)

---

# SECTION 2 — Payment Reversal (per-merchant)

Tags (Tag Name → Tag Value): customer_name → customer_name, Refund_amount → amount, Payment_amount → amount.

### TC-NDT-B1 — IPG

- Steps: Admin → Merchants → merchant → Update Merchant Details → Nudge Configuration → Notify payment reversal tags → Save. Merchant portal → Transactions → refund/reversal (before). Change tag values → Save → reverse again (after).
- Expected: The After email differs from the Before email (same data, changed mapping) → merge tags are dynamic.

**Config — before**

![](<Screenshot 2026-09-01 195318.png>)

**Email — before**

![](<Screenshot 2026-09-01 195436.png>)

**Config — after**

![](<Screenshot 2026-09-01 195242.png>)

**Email — after**

![](<Screenshot 2026-09-01 195227.png>)

### TC-NDT-B2 — NuveiPay

- Steps: Admin → Merchants → merchant → Update Merchant Details → Nudge Configuration → Notify payment reversal tags → Save. Merchant portal → Transactions → refund/reversal (before). Change tag values → Save → reverse again (after).
- Expected: The After email differs from the Before email (same data, changed mapping) → merge tags are dynamic.

**Config — before**

![](<Screenshot 2026-09-01 212048.png>)

**Email — before**

![](<Screenshot 2026-09-01 212110.png>)

**Config — after**

![](<Screenshot 2026-09-01 212026.png>)

**Email — after**

![](<Screenshot 2026-09-01 212011.png>)

---

# SECTION 3 — Payment Link (default page — For Developers)

Tags (Tag Name → Tag Value): requestedAmount → amount, tilliPayPaymentUrl → uri, payment_for → payment_for, Customer_Name → customer_name, customer_number → customer_number.

### TC-NDT-C1 — IPG

- Steps: Admin → For Developers → Third Party → Nudge (Nav bar) → US → Payment Link Request tags → Save. Merchant portal → Share Payment → share link (before). Change tag values → Save → share again (after).
- Expected: The After email differs from the Before email (same data, changed mapping) → merge tags are dynamic.

**Config — before**

![](<Screenshot 2026-09-01 234148.png>)

**Email — before**

![](<Screenshot 2026-09-01 234206.png>)

**Config — after**

![](<Screenshot 2026-09-01 234058.png>)

**Email — after**

![](<Screenshot 2026-09-01 233858.png>)

---

# SECTION 4 — Payment Reversal (default page — For Developers)

Tags (Tag Name → Tag Value): customer_name → customer_name, Refund_amount → amount, Payment_amount → amount.

### TC-NDT-D1 — IPG

- Steps: Admin → For Developers → Third Party → Nudge (Nav bar) → US → Notify payment reversal tags → Save. Merchant portal → Transactions → refund/reversal (before). Change tag values → Save → reverse again (after).
- Expected: The After email differs from the Before email (same data, changed mapping) → merge tags are dynamic.

**Config — before**

![](<Screenshot 2026-09-02 145638.png>)

**Email — before**

![](<Screenshot 2026-09-02 145605.png>)

**Config — after**

![](<Screenshot 2026-09-02 145447.png>)

**Email — after**

![](<Screenshot 2026-09-02 145429.png>)

---

# SECTION 5 — Payment Failure (already dynamic-Tested in qa)

Tags (Tag Name → Tag Value): customer_name → customerName, Payment_amount → amount, last_four_digits_of_card → last_four_digits_of_card.

### TC-NDT-E1 — IPG

- Steps: Admin → Nudge Config → Notify payment failure tags → Save. Make a declined/failed payment (before). Change tag values → Save → trigger another failed payment (after).
- Expected: The After email differs from the Before email (same data, changed mapping) → merge tags are dynamic.

**Config — before**

![](<Screenshot 2026-09-02 162019.png>)

**Email — before**

![](<Screenshot 2026-09-02 162004.png>)

**Config — after**

![](<Screenshot 2026-09-02 162153.png>)

**Email — after**

![](<Screenshot 2026-09-02 162135.png>)
