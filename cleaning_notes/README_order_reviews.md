# Order reviews table: cleaning notes

**Source:** `olist_order_reviews_dataset.csv` (Olist Brazilian E-Commerce Public Dataset)
**Output:** `cleaned/order_reviews_clean.csv`
**Shape:** 99,224 rows (one per review) × 9 columns. 7 raw columns, 2 engineered columns added.
**Principle:** no rows dropped and no values invented. Empty comments stay empty.

---

## 1. What the raw table looks like

| Column | Meaning |
|---|---|
| `review_id` | Identifier of the review. **Not unique** (see section 3) |
| `order_id` | The order being reviewed. Joins to `orders` |
| `review_score` | Rating from 1 (worst) to 5 (best) |
| `review_comment_title` | Optional short title written by the customer |
| `review_comment_message` | Optional free-text comment, mostly in Portuguese |
| `review_creation_date` | Date the satisfaction survey was sent |
| `review_answer_timestamp` | When the customer answered the survey |

**Score distribution**

| Score | Reviews | Share |
|---|---|---|
| 1 | 11,424 | 11.5% |
| 2 | 3,151 | 3.2% |
| 3 | 8,179 | 8.2% |
| 4 | 19,142 | 19.3% |
| 5 | 57,328 | 57.8% |

Average score is about 4.09. Scores 4 and 5 make up 77.1% of reviews, and scores 1 and 2 make up 14.7%.

---

## 2. Cleaning steps

1. **Inspected structure and nulls.** No missing values in IDs, score or dates. The two comment columns are mostly empty (see section 3).
2. **Converted both date columns from text to datetime** (`datetime64[us]`).
3. **Checked the key.** `review_id` alone is not unique. The pair (`review_id`, `order_id`) is unique, so no two rows are identical and a full-row duplicate check is unnecessary.
4. **Checked date logic.** No review has an answer time earlier than its creation date.
5. **Verified join integrity with `orders`** (see section 3).
6. **Added two helper columns** (see section 5).
7. **Saved and verified.** The file was reloaded with the dates parsed: shape (99224, 9), no key duplicates, same score distribution.

---

## 3. Data-quality findings

### Empty comments (normal, not an error)

| Column | Empty | Filled |
|---|---|---|
| `review_comment_title` | 87,656 (88.3%) | 11,568 |
| `review_comment_message` | 58,247 (58.7%) | 40,977 |

Customers often give a score without writing anything. These were left empty, not filled. The empty cells load into SQL Server as `NULL`.

### Keys and repeats

| Finding | Value |
|---|---|
| Rows | 99,224 |
| Distinct `review_id` | 98,410 (814 rows repeat a `review_id` that appears on another order) |
| Duplicates on (`review_id`, `order_id`) | 0 |
| Orders that have a review | 98,673 |
| Orders with **more than one** review | 547 |

A `review_id` that repeats sits on a different order, which suggests one review covering several orders. Orders with several reviews look like repeat submissions or revised reviews. In the sample inspected, one pair was 41 seconds apart with the same 5-star score (a likely double submit), and another moved from 5 to 4 on a later review.

### Join with `orders`

| Check | Result |
|---|---|
| Reviews whose `order_id` exists in `orders` | 99,224 of 99,224 (no orphan reviews) |
| Orders that have at least one review | 98,673 of 99,441 |
| Orders with **no** review | 768 |

Orders with no review by status:

| Status | Orders |
|---|---|
| delivered | 646 |
| shipped | 75 |
| canceled | 20 |
| unavailable | 14 |
| processing | 6 |
| invoiced | 5 |
| created | 2 |

About 99.3% of delivered orders have a review. The 646 delivered orders without one are customers who did not respond. A missing review is **not** a score of zero. Those orders have no row in this table and show as blank after a join.

### Dates

No answer timestamp is earlier than its creation date (0 rows). In the rows inspected, `review_creation_date` has a time of 00:00:00, so it behaves as a date only.

---

## 4. Decisions and rationale

| Decision | Reason |
|---|---|
| Keep all 99,224 rows, including repeats | Every row is a real review. The repeats can be handled with a flag without losing information |
| Key is the pair `review_id` + `order_id` | `review_id` alone repeats across orders |
| Leave empty comments empty | Filling them would invent content. Missing text is meaningful: the customer wrote nothing |
| Do not translate or edit comment text | Out of scope for cleaning, and edits could change meaning |
| Flag the latest review per order instead of deleting older ones | Keeps the history and still allows one score per order |

---

## 5. Engineered columns

| Column | Type | Definition |
|---|---|---|
| `is_latest_review` | boolean | True for the review with the newest `review_answer_timestamp` within each order. 98,673 rows are True, one per reviewed order |
| `has_comment` | boolean | True when `review_comment_message` is not empty. 40,977 rows are True |

If two reviews of one order have exactly the same timestamp, one is chosen arbitrarily as the latest.

---

## 6. Using this table downstream

- **One score per order:** filter `is_latest_review = True`. Without it, the 547 multi-review orders are counted more than once when joined to `orders`.
- **Average rating:** average `review_score` over `is_latest_review = True` rows. Averaging all rows weights repeat reviewers twice.
- **Count reviews:** `COUNT(*)` counts rows. Count orders with `COUNT(DISTINCT order_id)`. Because `review_id` repeats, `COUNT(DISTINCT review_id)` gives 98,410, not 99,224.
- **Review coverage:** compare orders with a review (98,673) to delivered orders. Do not treat missing reviews as zero.
- **Comment analysis:** filter `has_comment = True`. Text is in Portuguese.
- **Join to `orders`** on `order_id`. Reading the CSV back into pandas, pass `parse_dates=['review_creation_date', 'review_answer_timestamp']`. In SQL Server use `DATETIME2` for both and `NVARCHAR(MAX)` for the comment columns.
