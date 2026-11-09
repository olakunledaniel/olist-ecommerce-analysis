# Order items table: cleaning notes

**Source:** `olist_order_items_dataset.csv` (Olist Brazilian E-Commerce Public Dataset)
**Output:** `cleaned/order_items_clean.csv`
**Shape:** 112,650 rows (one per item sold) × 8 columns. 7 raw columns, 1 engineered column added.
**Principle:** no rows dropped and no values invented. One type conversion and one derived column.

---

## 1. What the raw table looks like

| Column | Meaning |
|---|---|
| `order_id` | The order the item belongs to. Joins to `orders`. Repeats when an order has several items |
| `order_item_id` | Position of the item within its order (1, 2, 3, ...) |
| `product_id` | The product sold. Joins to `products` |
| `seller_id` | The seller who sold it. Joins to `sellers` |
| `shipping_limit_date` | Deadline for the seller to hand the item to the carrier |
| `price` | Item price in Brazilian reais |
| `freight_value` | Shipping cost charged for this item, in Brazilian reais |

**The key is a pair.** One row is one item, so `order_id` alone is not unique. A row is identified by `order_id` plus `order_item_id`.

---

## 2. Cleaning steps

1. **Inspected structure and nulls.** No missing values in any of the 7 columns.
2. **Converted `shipping_limit_date` from text to datetime** (`datetime64[us]`).
3. **Checked the key.** The pair (`order_id`, `order_item_id`) has no duplicates, so no two rows can be identical and a full-row duplicate check is unnecessary. Item numbers also run 1 to n with no gaps inside every order.
4. **Verified join integrity with `orders`** (see section 3).
5. **Profiled the money columns** (see section 4). No zero or negative prices.
6. **Added `item_total`** (price plus freight).
7. **Saved and verified.** The file was written to CSV and reloaded with the date column parsed: shape (112650, 8), no key duplicates, date column is datetime.

---

## 3. Data-quality findings

### Size and shape of orders

| Measure | Value |
|---|---|
| Item rows | 112,650 |
| Orders that have items | 98,666 |
| Items per order (average) | 1.14 |
| Most items in one order | 21 |
| Items beyond the first in multi-item orders | 13,984 |

### Join with `orders`

| Check | Result |
|---|---|
| Item rows whose `order_id` exists in `orders` | 112,650 of 112,650 (no orphan items) |
| Orders that have at least one item | 98,666 of 99,441 |
| Orders with **no** items | 775 |

The 775 orders without items by status:

| Status | Orders |
|---|---|
| unavailable | 603 |
| canceled | 164 |
| created | 5 |
| invoiced | 2 |
| shipped | 1 |

Orders that were unavailable or canceled never produced an item, so they have no revenue. The invoiced and shipped orders without items (3 in total) are slightly odd but too few to affect any result.

**Orders count versus orders-with-revenue.** `orders` has 99,441 rows, but only 98,666 have items. A dashboard titled "orders" should state which one it counts.

---

## 4. Money columns

| Metric | `price` | `freight_value` |
|---|---|---|
| Count | 112,650 | 112,650 |
| Mean | 120.65 | 19.99 |
| Median | 74.99 | 16.26 |
| 25th / 75th percentile | 39.90 / 134.90 | 13.08 / 21.15 |
| Min / Max | 0.85 / 6,735.00 | 0.00 / 409.68 |

- **No zero or negative prices.** The cheapest item costs 0.85.
- **Mean is well above median** for price, so a few expensive items pull the average up. Report medians next to means.
- **383 rows have free shipping** (`freight_value` = 0). These are genuine and were kept.
- **4,124 rows (3.7%) have freight greater than price.** This is normal for cheap items where shipping costs more than the product. Kept, not flagged.
- **The highest price, 6,735.00,** is a single row (item 1 of a one-item order). It looks like a genuine high-value purchase and was kept.

---

## 5. Engineered column

| Column | Type | Definition |
|---|---|---|
| `item_total` | float | `price + freight_value`: what the customer paid for this item including shipping |

Total `item_total` across all rows: **15,843,553.24**. Use this as a reconciliation figure. After loading to SQL Server, `SUM(item_total)` must match it exactly.

**Which column for which question.** Use `price` for product revenue and sales performance. Use `item_total` for what customers paid. Use `freight_value` for logistics cost. Say in each chart which one it shows.

---

## 6. Decisions and rationale

| Decision | Reason |
|---|---|
| Keep all 112,650 rows | Every row is a real item with a matching order |
| Key is the pair `order_id` + `order_item_id` | One order can contain many items |
| Keep free-shipping and freight-above-price rows | Real commercial cases, not data errors |
| Keep the 6,735 item | Plausible single purchase. Use medians to limit its influence |
| Do not drop the 775 item-less orders from `orders` | They carry cancellation and availability information. They simply produce no revenue |

---

## 7. Using this table downstream

- **Join to `orders`** on `order_id`, but remember it is one-to-many: joining orders to items repeats each order once per item. Never sum an order-level column (such as a payment total) after this join without grouping first, or it will be counted once per item.
- **Count orders** with `COUNT(DISTINCT order_id)`, not `COUNT(*)`, which counts items.
- **Join to `products` and `sellers`** on `product_id` and `seller_id`.
- When reading the CSV back into pandas, pass `parse_dates=['shipping_limit_date']`. In SQL Server, load that column as `DATETIME2`.
- The total value of this table includes freight, and it will not necessarily equal the payment total in `order_payments`. Compare them once payments are cleaned.
