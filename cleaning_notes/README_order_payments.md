# Order payments table: cleaning notes

**Source:** `olist_order_payments_dataset.csv` (Olist Brazilian E-Commerce Public Dataset)
**Output:** `cleaned/order_payments_clean.csv`
**Shape:** 103,886 rows (one per payment) × 6 columns. 5 raw columns, 1 flag column added.
**Principle:** no rows dropped and no values invented. Problem rows are flagged, not altered.

---

## 1. What the raw table looks like

| Column | Meaning |
|---|---|
| `order_id` | The order being paid. Joins to `orders`. Repeats when an order has several payments |
| `payment_sequential` | Position of the payment within its order (1, 2, 3, ...) |
| `payment_type` | Method: credit card, boleto, voucher, debit card, or not defined |
| `payment_installments` | Number of installments chosen by the customer |
| `payment_value` | Amount of this payment, in Brazilian reais |

**The key is a pair.** One order can be paid in several payments, so `order_id` alone is not unique. A row is identified by `order_id` plus `payment_sequential`.

**Payment methods**

| Type | Rows |
|---|---|
| credit_card | 76,795 |
| boleto | 19,784 |
| voucher | 5,775 |
| debit_card | 1,529 |
| not_defined | 3 |

---

## 2. Cleaning steps

1. **Inspected structure and nulls.** No missing values and no type problems in any of the 5 columns.
2. **Checked the key.** The pair (`order_id`, `payment_sequential`) has no duplicates, so no two rows can be identical and a full-row duplicate check is unnecessary. A check on `order_id` plus `payment_type` finds 2,200 repeats. That is expected and is not an error: an order can be paid twice with the same method (for example two vouchers).
3. **Profiled the numeric columns** and looked at every zero-value and zero-installment row (see section 4).
4. **Verified join integrity with `orders`** (see section 3).
5. **Flagged problem rows** with `has_data_issue` (see section 4).
6. **Reconciled payments against item values** per order (see section 5).
7. **Saved and verified.** The file was reloaded and re-checked: shape (103886, 6), no key duplicates, 11 flagged rows.

---

## 3. Join with `orders`

| Check | Result |
|---|---|
| Payment rows whose `order_id` exists in `orders` | 103,886 of 103,886 (no orphan payments) |
| Orders that have at least one payment | 99,440 of 99,441 |
| Orders with **no** payment | 1 (status `delivered`) |

The one order without a payment is a delivered order. It is most likely a gap in the source data and cannot be fixed here. It does have items, so it appears in sales figures but not in payment figures.

---

## 4. Data-quality findings

| Finding | Rows | Treatment |
|---|---|---|
| Zero-value payments | 9 | Flagged |
| ...of which `voucher` | 6 | Harmless: adds nothing to any total |
| ...of which `not_defined` | 3 | All on **canceled** orders: the customer was never charged |
| Credit-card payments with 0 installments | 2 | Flagged. Impossible value, but the payments are real (58.69 and 129.94) |
| **Total flagged** | **11** | `has_data_issue` = True. The two groups do not overlap |

The 2 zero-installment rows were not set to 1, because that would be inventing a value. They do not affect any money total.

**Numeric profile**

| Metric | `payment_installments` | `payment_value` |
|---|---|---|
| Mean | 2.85 | 154.10 |
| Median | 1 | 100.00 |
| 25th / 75th percentile | 1 / 4 | 56.79 / 171.84 |
| Min / Max | 0 / 24 | 0.00 / 13,664.08 |

The mean payment is well above the median, so a few large payments pull the average up. Report medians next to means.

---

## 5. Reconciliation with `order_items`

Payments and item values were compared per order (both tables have several rows per order, so each was summed by `order_id` first).

| Measure | Result |
|---|---|
| Orders present in both tables | 98,665 |
| Orders where paid and item totals match to within 1 cent | 98,362 (99.7%) |
| Orders where they differ | 303 (0.3%) |
| Smallest / largest gap (paid minus items) | -51.62 / +182.81 |
| Orders with payments but **no items** | 775 (the item-less orders from the order items notes) |
| Money recorded on those 775 orders | 162,591.95 |
| Orders with items but **no payment** | 1 |

**The 303 mismatches by payment type:** credit card 275, boleto 14, credit card + voucher 7, debit card 7. Credit cards make up about 74% of payment rows but 91% of the mismatches. That is consistent with installment interest (customers paying slightly more than the item total), but the table has no interest column, so this is not proven. The boleto and debit-card cases cannot involve installments, so they are probably small discounts, rounding or recording differences.

**Totals tie out.** Total `payment_value` is 16,008,872.12 and total `item_total` in `order_items` is 15,843,553.24, a difference of 165,318.88. Of that, 162,591.95 is money recorded on orders with no items. The remaining 2,726.93 is the net of the 303 small gaps, less the item value of the one unpaid order.

---

## 6. Decisions and rationale

| Decision | Reason |
|---|---|
| Keep all 103,886 rows | Every row is a real payment record with a matching order |
| Flag rather than drop or fix the 11 odd rows | Keeps the source information, makes the issue size visible, and avoids inventing values |
| Keep the 303 mismatched orders unchanged | The gaps are small and likely legitimate (interest, discounts). Nothing here can be corrected |
| Keep payments on item-less orders | They are real records of money recorded. Exclude them from revenue by using items, not payments |

---

## 7. Using this table downstream

- **Revenue and sales:** use `price` or `item_total` from `order_items`. Do **not** add up all `payment_value` and call it revenue: that includes 162,591.95 on orders with no items.
- **Payment-method, installment and voucher analysis:** use this table.
- **Join to `orders`** on `order_id`. It is one-to-many: an order with two payments appears twice. Joining `order_items` and `order_payments` on `order_id` directly multiplies rows (items times payments), which overstates both. Sum each table by `order_id` first, then join, as in the reconciliation.
- **Count orders** with `COUNT(DISTINCT order_id)`, not `COUNT(*)`, which counts payments.
- **Excluding flagged rows:** filter `has_data_issue = False` if a metric should ignore the 11 odd rows. For installment averages this is worth doing.
- **Reconciliation figure:** `SUM(payment_value)` in SQL Server must equal 16,008,872.12 after loading.
