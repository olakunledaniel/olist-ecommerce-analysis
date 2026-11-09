# Orders table: cleaning notes

**Source:** `olist_orders_dataset.csv` (Olist Brazilian E-Commerce Public Dataset)
**Output:** `cleaned/orders_clean.csv`
**Shape:** 99,441 rows (one per order). 8 raw columns, 6 engineered columns added.
**Principle:** no rows dropped and no values invented. Problem rows are flagged, not altered.

---

## 1. What the raw table looks like

| Column | Meaning |
|---|---|
| `order_id` | Unique order identifier |
| `customer_id` | Order-level customer key (joins to `customers`) |
| `order_status` | Lifecycle status of the order |
| `order_purchase_timestamp` | When the customer placed the order |
| `order_approved_at` | When payment was approved |
| `order_delivered_carrier_date` | When the seller handed the package to the carrier |
| `order_delivered_customer_date` | When the customer received the package |
| `order_estimated_delivery_date` | Delivery date promised to the customer at purchase |

Expected order of events: purchase, approval, carrier hand-off, customer delivery.

**Order status breakdown**

| Status | Orders |
|---|---|
| delivered | 96,478 |
| shipped | 1,107 |
| canceled | 625 |
| unavailable | 609 |
| invoiced | 314 |
| processing | 301 |
| created | 5 |
| approved | 2 |

---

## 2. Cleaning steps

1. **Converted data types.** The five timestamp columns were loaded as text and converted to `datetime64[us]`. Non-null counts were identical before and after, so no values were lost.
2. **Checked uniqueness.** `order_id` has 99,441 distinct values, equal to the row count, so there are no duplicate orders. A full-row duplicate check is therefore unnecessary.
3. **Analysed nulls in context.** Raw null counts: `order_approved_at` 160, `order_delivered_carrier_date` 1,783, `order_delivered_customer_date` 2,965. Most are legitimate (orders that were canceled or never delivered have no delivery dates). Only nulls that contradict the order status were treated as problems (see section 3).
4. **Checked timestamp sequence.** Counted rows where events happen out of order (see section 3).
5. **Engineered delivery features and data-quality flags** (see section 4).

---

## 3. Data-quality findings

### Missing dates that contradict the status

| Finding | Rows |
|---|---|
| Delivered orders with no `order_approved_at` | 14 |
| Delivered orders with no `order_delivered_carrier_date` | 2 |
| Delivered orders with no `order_delivered_customer_date` | 8 |
| Non-delivered orders that *do* have a customer delivery date (all `canceled`) | 6 |

### Timestamps out of sequence

| Check | Rows |
|---|---|
| Customer delivery before purchase | 0 |
| Approval before purchase | 0 |
| Carrier hand-off before purchase | 166 |
| Approval after carrier hand-off | 1,359 |
| Customer delivery before carrier hand-off | 23 |

All 166 carrier-before-purchase rows are also in the 1,359 approval-after-carrier group. Since no order is approved before it is purchased, a carrier date earlier than the purchase automatically means an approval later than the carrier date. The groups of 1,359 and 23 do not overlap. The combined count of out-of-sequence orders is therefore **1,382 (1.4%)**.

Carrier-before-purchase and delivery-before-carrier are impossible and almost certainly timestamp errors. Approval-after-hand-off is odd but could reflect how payment confirmation was recorded.

---

## 4. Engineered columns

| Column | Type | Definition | Notes |
|---|---|---|---|
| `delivery_time_days` | integer, nullable | Customer delivery date minus purchase timestamp, in whole days | Null when no delivery date. 96,476 non-null values |
| `delivery_delay_days` | integer, nullable | Customer delivery date minus estimated delivery date, in whole days | Positive means late, negative means early. 96,476 non-null values |
| `is_late` | boolean, nullable | `delivery_delay_days > 0` | **Null** (not False) when there is no delivery date. True 6,535, False 89,941, null 2,965 |
| `date_has_issue` | boolean | True for delivered orders missing a customer delivery date, or non-delivered orders that have one | 14 rows (8 + 6) |
| `has_sequence_issue` | boolean | True if approval came after carrier hand-off or delivery came before carrier hand-off | 1,382 rows |
| `has_data_issue` | boolean | `date_has_issue` OR `has_sequence_issue` | 1,396 rows. The two flags do not overlap |

**Why `is_late` is nullable.** In pandas, `NaN > 0` evaluates to `False`. A plain comparison would label every undelivered order as "not late" and understate late-delivery rates. The column is built so undelivered orders stay null and are excluded from averages.

**Whole days are truncated.** The day columns keep only the whole-day part of the difference. An order delivered on the estimated date at any time of day counts as 0 days, which is on time. Early deliveries round down (2.5 days early shows as -3), which does not affect the late/not-late logic.

---

## 5. Descriptive statistics (orders with a customer delivery date)

| Metric | `delivery_time_days` | `delivery_delay_days` |
|---|---|---|
| Count | 96,476 | 96,476 |
| Mean | 12.1 | -11.9 |
| Median | 10 | -12 |
| 25th / 75th percentile | 6 / 15 | -17 / -7 |
| Min / Max | 0 / 209 | -147 / 188 |

- Typical orders take about 10 days and arrive about 12 days **before** the estimated date. Olist's delivery estimates are padded.
- **Late rate:** 6,535 of 96,476 orders with a delivery date, which is **6.8%**. Counting undelivered orders as on time (6,535 of 99,441) gives 6.6% and is misleading.
- The extremes (209 days to deliver, 188 days late, 147 days early) are rare but real events. They were kept. Report medians or percentiles alongside means to limit their influence.

---

## 6. Decisions and rationale

| Decision | Reason |
|---|---|
| Keep all 99,441 rows, including canceled and undelivered orders | Dropping them would remove the ability to analyse cancellation and fulfilment rates. Filter by `order_status` in analysis instead |
| Do not impute missing dates | Invented timestamps would corrupt delivery metrics. Nulls are skipped naturally by SQL and Power BI aggregates |
| Flag problem rows instead of dropping or fixing them | Keeps the raw information intact and makes the issue size transparent and filterable |
| Keep extreme delivery times | They are plausible real events, not clear errors |

---

## 7. Using this table downstream

- Delivery-time and lateness metrics: filter to `order_status = 'delivered'`, and decide whether to exclude `has_data_issue = True` rows. The 14 date-issue rows are the ones with contradictory delivery dates.
- When reading the CSV back into pandas, pass the five timestamp columns to `parse_dates`, because CSV stores them as text.
- Empty cells in `is_late` and the day columns load into SQL Server as `NULL`.
