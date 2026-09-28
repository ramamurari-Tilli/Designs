# Payment Trends — defects

**Module:** Payment Analytics → Payment Trends
**Environment:** QA — `https://qagps.tillipay.com/portal/paymenttrends`
**Scope:** everything found while covering PT-LOAD, PT-RANGE, PT-LEG and PT-TIP
**Last updated:** 28 September 2026

| | Test case | Defect | Priority | Gateways |
|---|---|---|---|---|
| 1 | PT-LOAD-09 | The chart draws no month names at all | High | MerchantE, CommerceHub |
| 2 | PT-NEG-12 | The chart counts payments the drill-down cannot find | Critical | IPG |
| 3 | PT-NEG-13 | A summary card reads zero beside a real amount | High | IPG |
| 4 | PT-TIP-04 | The tooltip and the summary cards disagree | High | IPG |
| 5 | PT-NEG-01 | A failed load tells the merchant nothing | High | all gateways |
| 6 | PT-NEG-11 | The legend falls back to another gateway's layout | High | MerchantE, SnapPay, CommerceHub |

Defects 2, 3 and 4 are one fault seen from three angles. Fixing 2 should close all three.
Defects 5 and 6 both follow from the same failed call and are why two further test cases,
PT-LEG-02 on MerchantE and PT-TIP-02, cannot be run at all.

Only PT-LOAD-09 is automated so far, as an expected failure. The rest are recorded from
manual runs and are next in line for automation.

A closing section lists three things that look like defects and are not, so they do not get
raised again.

---

# 1. PT-LOAD-09 — the chart draws no month names at all

**Test case:** PT-LOAD-09
**Priority:** High
**Gateways seen on:** MerchantE and CommerceHub
**Date observed:** 24 and 25 September 2026

## Summary

The Payment Trends chart should always be labelled. Whatever the merchant's data, the
horizontal axis names the months the chart covers, so the merchant knows what period they
are looking at.

On two accounts it draws nothing. The heading renders, the value axis down the side renders,
both captions render, and the bottom of the chart is blank. There is no indication that
anything is missing — the chart simply has no months on it.

## Steps to reproduce

1. Sign in as the MerchantE merchant.
2. Go to **Payment Analytics → Payment Trends** from the sidebar.
3. Wait for the page to finish loading.
4. Read along the bottom edge of the chart, where the month names belong.
5. Repeat as the CommerceHub merchant.

## Expected result

Six month names are drawn along the bottom of the chart, oldest on the left through to the
current month on the right, exactly as they are on every other account.

## Actual result

No month names are drawn on either account. The rest of the chart frame renders normally.

## Evidence

Measured across all six accounts on 25 September, with the range left at its default of
Last 6 Months:

| Account | Payments in range | Month names drawn |
|---|---|---|
| IPG | 2,436 | 6 |
| BillPay | 2,090 | 6 |
| SnapPay | 224 | 6 |
| NuveiPaya | call failed | 6 |
| **CommerceHub** | **35** | **none** |
| **MerchantE** | **0** | **none** |

## What pins the cause

The obvious explanation — that an empty merchant cannot draw labels — is wrong, and it is
worth saying so plainly because the defect was first written up that way. CommerceHub has
35 payments in the range and still draws nothing. NuveiPaya's call fails outright and still
draws all six.

What the two broken accounts share is a **short response**. They returned 2 and 3 monthly
figures where the three working accounts returned 12.

The mechanism is in `PaymentTrends/index.js`. The month names are drawn from the chart's
data rows, not from the axis, and the call that supplies those rows sits behind a guard:

- line 527 — `if (response.data?.data.data?.length > 0)`
- line 567 — `setmonthStateData(responsedata)`, inside that guard

So when the response carries no usable rows, the chart's data is never set and the axis has
nothing to label. The six month names the page already knows about are computed separately
and are never used as a fallback.

That accounts for MerchantE, whose call fails. **It does not yet account for CommerceHub**,
which has data and still draws nothing, and that half should be confirmed before the fix is
designed.

## Impact

The merchant sees a chart they cannot read. Bars may be drawn with nothing to say which month
each belongs to, so the page is useless for the one thing it exists to do. It also looks like
a rendering fault rather than a data problem, so it is likely to be reported as a broken page.

## Notes for the developer

Drawing the axis from the month list the page already computes, rather than from the returned
rows, would make the chart label itself correctly whatever the response contains. That list
exists and is already correct — it is what the working accounts end up displaying.

## How to tell it is fixed

Open Payment Trends as MerchantE and as CommerceHub. Both should draw six month names ending
at the current month, with or without bars above them.

The automated test is `PT-LOAD-09` in `payment-analytics.spec.ts`. It runs on MerchantE and is
marked expected-fail, so it will turn red the day this is fixed — that red is the signal to
remove the mark.

---

# 2. PT-NEG-12 — the chart counts payments the drill-down cannot find

**Test case:** PT-NEG-12
**Priority:** Critical
**Gateways seen on:** IPG
**Date observed:** 14 September 2026

## Summary

Hovering a month on the chart shows how many payments it holds. Clicking that month opens a
drill-down that should list those same payments.

On IPG the two disagree completely. The chart reports hundreds of card payments for a month;
clicking it returns an empty table.

## Steps to reproduce

1. Sign in as the IPG merchant.
2. Go to **Payment Analytics → Payment Trends**.
3. Hover a month bar and read the Card count from the tooltip.
4. Click that same bar.
5. Scroll below the chart to the transaction table.
6. Count the rows, and read the record total in the table's footer.
7. Compare that total against the tooltip count from step 3.

## Expected result

The drill-down lists the payments the chart counted. The footer total matches the tooltip.

## Actual result

For May 2026 the tooltip read **Card (544)**. The drill-down returned **no rows** and a record
total of **0**.

## Evidence

Querying the transactions endpoint directly for the same merchant **with no filter at all**
also returned zero records. So this is not the month filter excluding rows — there is nothing
for that merchant to return.

The two sources genuinely disagree: the figures behind the chart and the figures behind the
transaction list are not describing the same data.

## Impact

Critical, because the merchant is shown a number they cannot act on. The chart says 544 card
payments were taken; clicking through to see them shows an empty table with no explanation.
Either the chart is overstating or the list is missing records, and until that is settled
neither figure on this page can be trusted for IPG.

This also drives two further defects — a summary card reading zero beside a real amount
(defect 3), and the tooltip disagreeing with the cards (defect 4). Both should close when this
one does.

## Notes for the developer

The question to settle first is which of the two is right — whether those 544 payments exist.
The chart's figures come from the periodic stats call; the drill-down's come from the
transactions call. Whichever is wrong, the fix belongs on that side rather than in the page.

## How to tell it is fixed

Hover any month on IPG, note the Card count, click it, and confirm the table's footer total
matches.

---

# 3. PT-NEG-13 — a summary card reads zero beside a real amount

**Test case:** PT-NEG-13
**Priority:** High
**Gateways seen on:** IPG
**Date observed:** 14 September 2026

## Summary

Clicking a month opens a row of summary cards, one per payment type. Each card carries a count
in brackets and an amount below it.

On IPG the Card card reads **Card (0)** with a real, non-zero amount beneath it. Zero payments
cannot add up to a positive amount, so the card contradicts itself.

## Steps to reproduce

1. Sign in as the IPG merchant.
2. Go to **Payment Analytics → Payment Trends**.
3. Click any month bar.
4. Read the count in brackets on the Card summary card.
5. Read the currency amount on that same card.

## Expected result

The count and the amount agree. A card showing an amount also shows how many payments made it
up.

## Actual result

The count reads 0 while the amount is non-zero.

On BillPay, whose drill-down does return rows, the same cards are consistent — Ach (43) and
Card (80), each beside a matching amount. So the fault is specific to the accounts where the
drill-down comes back empty.

## What pins the cause

The two halves of the card come from different places. The **amount** comes from the stats
response, which has the figures. The **count** is derived from the transaction list, which on
IPG returns nothing — so the count collapses to zero while the amount survives.

That is why this only appears where defect 2 appears.

## Impact

The merchant sees a self-contradictory figure. It is the most visible symptom of defect 2 and
the one most likely to be reported, since it is wrong on its face without needing anything to
be compared against.

## How to tell it is fixed

Click any month on IPG. Every summary card showing an amount should show a matching non-zero
count.

---

# 4. PT-TIP-04 — the tooltip and the summary cards disagree

**Test case:** PT-TIP-04
**Priority:** High
**Gateways seen on:** IPG
**Date observed:** 25 September 2026

## Summary

The tooltip on a month bar and the summary cards below the chart describe the same month, so
their counts should be identical. On IPG they are not.

## Steps to reproduce

1. Sign in as the IPG merchant.
2. Go to **Payment Analytics → Payment Trends**.
3. Hover a month bar and note every count in the tooltip.
4. Without moving to a different bar, click the one being hovered.
5. Read the counts in brackets on the summary cards.
6. Compare the two sets.

## Expected result

The counts are identical. They describe one month.

## Actual result

Measured by hovering and clicking the same bar on four accounts:

| Account | Tooltip | Summary cards | Agree |
|---|---|---|---|
| BillPay | Ach 43, Card 80 | Ach 43, Card 80 | yes |
| SnapPay | Token 19, Card 92 | Token 19, Card 92 | yes |
| Paymentus | Ach 29, Token 246, Card 63 | Ach 29, Token 246, Card 63 | yes |
| **IPG** | **Ach 0, Card 712** | **Ach 0, Card 0** | **no** |

## What pins the cause

This is defect 2 and defect 3 showing through, not a separate fault. The tooltip reads the
chart's figures; the cards read the transaction list. Where the drill-down returns rows the two
agree, on three accounts out of four.

It is recorded separately because it is the check that catches the problem without needing
anyone to open a network tab — hover, click, compare.

## Impact

Same as defect 3. Listed on its own so the comparison is not lost if the other two are closed
individually.

## How to tell it is fixed

The automated test is `PT-TIP-04` in `payment-analytics.spec.ts`. It runs on all four accounts,
with IPG marked expected-fail. When IPG turns red, the mismatch is gone and the mark should be
removed.

---

# 5. PT-NEG-01 — a failed load tells the merchant nothing

**Test case:** PT-NEG-01
**Priority:** High
**Gateways seen on:** all gateways
**Date observed:** 14 September 2026

## Summary

When the call behind Payment Trends fails, the page shows no error of any kind. The loading
indicator stops and an empty chart is left on screen.

A broken page and a genuinely quiet month look exactly the same.

## Steps to reproduce

1. Sign in as the MerchantE merchant, whose call currently fails on QA. On any other account,
   block `periodic_payment_stats_us` in the browser's network tools first.
2. Go to **Payment Analytics → Payment Trends**.
3. Watch the page as it loads.
4. Look for any error message, banner or notification.

## Expected result

The merchant is told the figures could not be loaded, so they know not to read the empty chart
as real.

## Actual result

Nothing is shown. No banner, no notification, no inline message. The chart renders empty.

Confirmed live rather than hypothetically — the call returned:

`HTTP 400 {"error_code":"400","message":"Merchant not exists"}`

and the page displayed none of it.

## What pins the cause

The failure is caught and then discarded. The handler the page calls,
`checkErroStatus` in `utils/commonFunctions.js:203`, acts on only three error codes:

- **167** — redirects to the no-users-assigned page
- **401** and **484** — clears storage and returns to sign-in

Anything else falls through the function and returns without doing anything. A 400 is not in
that list, so nothing happens at all — no message is raised and no state is changed.

## Impact

A merchant whose data fails to load is shown an empty chart and told it is their figures. They
may reasonably conclude they took no payments that period. Support cannot tell the two cases
apart from a screenshot either.

It also hides other faults during testing: a test that only checks the page renders will pass
against a broken load, which is exactly how defect 1 went unnoticed for several weeks.

## Notes for the developer

The pattern already exists elsewhere in the product — Payment Stats surfaces a message on the
same class of failure. Adding a default branch to `checkErroStatus`, or raising a message at
the call site, would cover every code it does not already handle rather than only the 400.

## How to tell it is fixed

Block the periodic stats call and open Payment Trends. A message should appear saying the
figures could not be loaded.

---

# 6. PT-NEG-11 — the legend falls back to another gateway's layout

**Test case:** PT-NEG-11
**Priority:** High
**Gateways seen on:** MerchantE, and any account whose gateway shows the Token series
**Date observed:** 14 September 2026

## Summary

Payment Trends draws a different set of series depending on the merchant's gateway. Some show
ACH and Card; others show Token and Card.

When the call fails, the page does not find out which gateway the merchant is on — and instead
of saying so, it silently draws the ACH layout. A Token-gateway merchant is shown a chart shaped
for a different kind of account, with nothing to indicate anything went wrong.

## Steps to reproduce

1. Sign in as the MerchantE merchant.
2. Confirm the gateway named in the portal header at the top right reads MerchantE.
3. Go to **Payment Analytics → Payment Trends** while the periodic stats call fails. On a
   healthy account, block `periodic_payment_stats_us` in the browser's network tools.
4. Read the chart legend.

## Expected result

The legend matches the merchant's real gateway — Token and Card for MerchantE. If the gateway
cannot be determined, the page says so rather than guessing.

## Actual result

The legend rendered **Ach** and **Card**: the layout belonging to a different group of
gateways, on an account whose header plainly reads MerchantE.

## What pins the cause

In `PaymentTrends/index.js`:

- line 357 — `const [gateWay, setGateWay] = useState("")`
- line 489 — `setGateWay(response?.data.data.payment_gateway)`

The gateway is only set once the call has succeeded. When it fails, the value stays as the
empty string it started as. Every later check is written as a comparison against a specific
gateway name — `gateWay === "SNAP_PAY"`, `gateWay === "MERCHANT_E"` and so on — so an empty
value fails all of them and the code falls through to the ACH and Card default.

The empty string is treated as a real answer rather than as "not known yet".

## Impact

The legend, the bars and the summary cards all key off the same unresolved value, so a failed
call does not just lose the data — it redraws the page as a different kind of account. The
merchant has no way to tell.

This is also why **PT-LEG-02 cannot be completed on MerchantE** and why **PT-TIP-02 cannot be
run at all**: both need a Token legend that this defect replaces.

## Notes for the developer

Holding the gateway as unknown until it is actually known, and rendering nothing rather than a
default while it is, would stop the page asserting something untrue. Pairing that with defect 5
would cover the case properly — say the load failed, and do not draw a chart shaped for the
wrong account in the meantime.

## How to tell it is fixed

Block the periodic stats call and open Payment Trends as MerchantE. The page should not show an
Ach and Card legend.

---

# Reviewed and closed — not defects

These three were raised or recorded as defects and have since been reviewed and closed. They
are listed so they are not raised again.

## PT-LOAD-10 — the right-hand value axis never renders

The chart declares two value axes but binds every series to the left one, so the right-hand
axis never appears. Its fixed 0-110 range and currency label are configuration that does
nothing.

**Closed on 25 September 2026.** It is unfinished setup rather than a fault, and the amounts
that axis would have carried are already one click away — clicking a month opens a panel
showing the count and the amount for each payment type.

## PT-LOAD-11 — the axis labels carry no ' k' or ' $' suffix

The label factory takes a symbol and never applies it, so every tick is drawn as a plain
number.

**Closed on 25 September 2026.** Applying the suffix would make the chart wrong. The left axis
counts payments, so a month with 20 payments would draw a tick reading `20 k` — twenty
thousand. The only real effect of leaving it is that a very high-volume merchant reads `50000`
rather than `50 k`: long, but correct.

## PT-NEG-09 — drill-down rows show an unexpected status

On BillPay the drill-down filters on one field while the Status column displays a different
one, so rows can legitimately show values such as `TIMEOUT_REFUND_FAILED` under a filter for
authorised payments.

**Not a defect.** The two fields are different by design. This is recorded because the rows
look wrong at a glance and have been raised before.
