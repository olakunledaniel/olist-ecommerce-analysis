# Customers table: cleaning notes

**Source:** `olist_customers_dataset.csv` (Olist Brazilian E-Commerce Public Dataset)
**Output:** `cleaned/customers_clean.csv`
**Shape:** 99,441 rows (one per order) × 5 columns. No columns added or removed.
**Principle:** no rows dropped and no values invented. The only changes are format fixes (zip codes and city casing).

---

## 1. What the raw table looks like

| Column | Meaning |
|---|---|
| `customer_id` | Order-level customer key. A new value is issued for every order. Joins to `orders` |
| `customer_unique_id` | Identifies the actual person. Use this for repeat-customer analysis |
| `customer_zip_code_prefix` | First 5 digits of the customer's zip code |
| `customer_city` | Customer's city |
| `customer_state` | Two-letter Brazilian state code |

---

## 2. Cleaning steps

1. **Inspected structure and nulls.** No missing values in any column.
2. **Checked the keys.** `customer_id` is unique across all 99,441 rows, so there are no duplicate rows and a full-row duplicate check is unnecessary.
3. **Verified join integrity with `orders`.** Every `customer_id` in `orders` exists in `customers` (99,441 of 99,441), and every customer has an order. The join is one-to-one with no orphans in either direction.
4. **Restored zip code leading zeros.** The raw zip prefixes were read as numbers, so 23,995 of the 99,441 values lost their leading zero (for example `09790` appeared as `9790`). The column was converted to text and padded to 5 characters. All 99,441 values are now length 5.
5. **Checked states and cities.** 27 distinct states, matching Brazil's 26 states plus the Federal District. 4,119 distinct cities, with no leading or trailing spaces. The only city containing characters other than letters, spaces, hyphens or apostrophes is `quilometro 14 do mutum`, a real locality name, so it was kept.
6. **Standardised city casing.** Cities were lowercase and accent-free in the raw file. They were title-cased (`sao paulo` becomes `Sao Paulo`) so dashboards display cleanly. The distinct city count stayed at 4,119, so no spellings merged.
7. **Saved and verified.** The cleaned file was written to CSV, reloaded with the zip column read as text, and re-checked: shape (99441, 5), all zips length 5, 4,119 cities.

---

## 3. Data-quality findings

| Finding | Result |
|---|---|
| Missing values | None |
| Duplicate `customer_id` | None |
| Orphan keys between `customers` and `orders` | None, in either direction |
| Zip codes missing their leading zero | 23,995 rows (fixed) |
| Stray whitespace in city names | 0 rows |
| Distinct states | 27 |
| Distinct cities | 4,119 |

---

## 4. Key finding: orders versus people

| Measure | Value |
|---|---|
| Rows (`customer_id`, one per order) | 99,441 |
| Distinct people (`customer_unique_id`) | 96,096 |
| Extra orders from repeat buyers | 3,345 |
| Repeat customers (more than one order) | 2,997 |
| Repeat rate | 3.1% of people |
| Largest number of orders from one person | 17 |

`customer_id` is generated per order, so counting its distinct values counts orders, not customers. Count `customer_unique_id` for the number of customers. About 97% of customers bought only once, so a repeat-purchase measure will look low. That reflects the data, not an error.

The repeat count above includes every order regardless of status. If you restrict to delivered orders in SQL, the rate will come out slightly lower, so state which definition the dashboard uses.

---

## 5. Decisions and rationale

| Decision | Reason |
|---|---|
| Keep all 99,441 rows | Every row links to exactly one order. Removing any would break joins |
| Zip codes stored as text, not numbers | They are identifiers, not quantities. As numbers they lose leading zeros and would fail to join to the geolocation table |
| Title-case city names | Display only. Done once in pandas so SQL and Power BI show the same values |
| Keep the 17-order buyer | A real, plausible account. Use medians where one buyer could distort an average |
| Keep `quilometro 14 do mutum` | A genuine place name, not a data error |

**Known cosmetic side effect of title-casing:** small connecting words are capitalised too (`Quilometro 14 Do Mutum`). This has no effect on analysis.

---

## 6. Using this table downstream

- **Always read the zip column as text.** When loading `customers_clean.csv` into pandas, pass `dtype={'customer_zip_code_prefix': str}`. Otherwise the leading zeros are lost again. In SQL Server, use `VARCHAR(5)` for this column.
- **Join to `orders`** on `customer_id`.
- **Customer-level metrics** (repeat purchase, lifetime value, customers per state) must group by `customer_unique_id`, not `customer_id`.
- **Join to `geolocation`** on the zip prefix only after the geolocation zip column has been padded the same way.
