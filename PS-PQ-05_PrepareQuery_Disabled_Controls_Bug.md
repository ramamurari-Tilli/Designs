# Prepare Query — Having and Execute never recover after Select is emptied

**Test case:** PS-PQ-05
**Module:** Payment Analytics → Payment Stats → Prepare Query
**Priority:** High
**Environment:** QA — `https://qagps.tillipay.com/portal/paymentstats`
**Gateways seen on:** BillPay, MerchantE
**Date observed:** 22 September 2026

---

## Summary

Emptying the **Select** field correctly disables **Group**, **Having** and **Execute**.

Putting the Select fields back does **not** bring them all back. Group becomes usable
again, but **Having stays greyed out and Execute stays disabled**. From that point the
panel cannot be used at all — there is no way to build or run a query without leaving
the page.

---

## Steps to reproduce

1. Sign in to the customer portal.
2. Go to **Payment Analytics → Payment Stats** from the sidebar.
3. Expand **Prepare Query**.
4. Note that Select and Group are pre-filled and **Execute** is enabled.
5. Remove every chip from the **Select** field, one by one, using the small x on each chip.
6. Observe Group, Having and Execute — all three are now disabled. *(This part is correct.)*
7. Add the Select fields back from the dropdown.
8. Observe Having and Execute.

---

## Expected result

Step 6 of the written case: *"Add one Select field back and verify Group becomes usable again."*

The panel should return to a working state. Once Select has fields again, Group, Having
and Execute should all be usable, exactly as they were when the panel was opened.

---

## Actual result

| Control | After emptying Select | After adding Select fields back |
|---|---|---|
| Group | disabled | **usable again** — correct |
| Having (3 dropdowns) | disabled | **still disabled** — wrong |
| And/Or | disabled | **still disabled** — wrong |
| Execute | disabled | **still disabled** — wrong |

Only Group recovers. Having and Execute stay locked, so the query panel is dead until
the page is reloaded or the user navigates away and back.

---

## Evidence

**[Image #14]** — MerchantE (`ACH_MERCHANT_E`), 15:50.
Select holds all four fields — Request Type, Channel, transaction Amount, Fee Amount —
and Group holds the same four. Despite the panel being fully populated, all three Having
dropdowns and the And/Or dropdown are greyed out, and the **Execute** button is greyed
out while Cancel remains active. This is the state left behind after Select was emptied
and refilled.

**[Image #15]** — BillPay, 16:19.
Select has been emptied and shows only the "select options" placeholder. The Group chips
— Funding Method, Funding Category, Channel, Displayed Fee Amount — are greyed, the
Having dropdowns are greyed, and **Execute** is greyed. This is the intermediate state,
which is the correct behaviour for an empty Select.

In both screenshots the date range and the results table below are unaffected, so the
page itself is healthy — only the query controls are stuck.

---

## Impact

The Prepare Query panel is the only way a merchant can group or filter their payment
statistics. Once this state is reached the feature is unusable for the rest of the
visit, and nothing on screen explains why the buttons are dead or how to get them back.

Clearing a field and changing your mind is ordinary behaviour, so this is easy to hit by
accident.

**Workaround:** reload the page, or navigate away and back through the sidebar. The panel
opens fresh and works normally.

---

## Notes for the developer

- Group recovering while Having and Execute do not suggests the enable/disable rules for
  the three controls are evaluated separately, and only Group's is re-checked when Select
  changes.
- Having and Execute appear to recover only when **Group** itself is edited, not when
  Select is. That matches PS-PQ-06, where emptying Group disables Having and Execute.
- Seen on both BillPay and MerchantE, so it is not specific to one gateway's field set.

---

## Automation

This is covered by the automated suite, which records the partial recovery rather than
treating it as a flake:

- **Test:** `Payment Stats locks the query controls when Select is emptied @regression @PS-PQ-05`
- **File:** `tests/customer/payment-analytics/payment-analytics.spec.ts`

The test asserts that Group recovers, and separately asserts that Having is still
disabled. When this bug is fixed that last assertion will start failing, which is the
signal to update it.

---

## Related

| Case | Relevance |
|---|---|
| PS-PQ-06 | Emptying **Group** disables Having and Execute — the same controls, different trigger |
| PS-PQ-16 | MerchantE's Prepare Query panel is inert — separate defect on the same panel |
