# Sellers table: cleaning notes

**Source:** `olist_sellers_dataset.csv` (Olist Brazilian E-Commerce Public Dataset)
**Output:** `cleaned/sellers_clean.csv`
**Shape:** 3,095 rows (one per seller) × 5 columns. 4 raw columns, 1 flag column added.
**Principle:** no rows dropped and no values invented. City text was repaired only where the data itself supports the fix. Every changed row is flagged.

---

## 1. What the raw table looks like

| Column | Meaning |
|---|---|
| `seller_id` | Unique seller identifier. Joins to `order_items` |
| `seller_zip_code_prefix` | First 5 digits of the seller's zip code |
| `seller_city` | Seller's city, as typed by the seller |
| `seller_state` | Two-letter Brazilian state code |

---

## 2. Cleaning steps

1. **Inspected structure and nulls.** No missing values in any column.
2. **Checked the key.** `seller_id` is unique across all 3,095 rows, so there are no duplicate rows and a full-row duplicate check is unnecessary.
3. **Checked zip codes.** Stored as text, all 3,095 values have 5 characters (none had lost a leading zero). A pad to 5 characters is kept in the script as a safeguard.
4. **Checked states.** 23 distinct states, no stray spaces.
5. **Searched cities for odd characters** with the pattern `[^a-z \-']` (anything other than lowercase letters, spaces, hyphens and apostrophes). 24 rows matched (see section 3).
6. **Repaired city text** (see section 3): odd apostrophes, accents, and extra text after `/`, `\`, `,` or `(`.
7. **Fixed values that were not city names** (see section 3) and flagged every changed row with `city_was_fixed`.
8. **Title-cased the cities** (`sao paulo` becomes `Sao Paulo`).
9. **Checked the link with `order_items`** (see section 4).
10. **Saved and verified.** The file was reloaded with the zip column read as text: shape (3095, 5), unique `seller_id`, all zips length 5, 589 distinct cities, 29 flagged rows.

---

## 3. Data-quality findings: city names

Raw cities are free text typed by sellers, so they contain mistakes. The raw file had 611 distinct cities. After cleaning there are 589, because spelling variants of the same city merged.

| Problem | Rows | Example | Fix |
|---|---|---|---|
| City followed by state or country text, or with an accent or odd apostrophe | 20 | `auriflama/sp`, `novo hamburgo, rio grande do sul, brasil`, `arraial d'ajuda (porto seguro)`, `são paulo`, `santa barbara d´oeste` | Kept the text before the first `/`, `\`, `,` or `(`. Accents removed, apostrophe replaced with `'` |
| Abbreviation `sbc` | 2 | `sbc`, `sbc/sp` | Replaced by `sao bernardo do campo` (manual mapping, see below) |
| State typed as city (`sp`) | 5 | `sp`, `sp / sp` | Looked up by zip, otherwise `unknown` |
| Zip code typed as city | 1 | `04482255` | Looked up by zip |
| Email address typed as city | 1 | `vendas@creditparts.com.br` | Looked up by zip |

The pattern check found 24 rows. The plain `sp` and `sbc` entries contain only normal letters, so the pattern could not catch them. They were found by listing the cities that were really state codes or abbreviations. Together there are **29 changed rows**.

**Resolving the 9 non-city rows**

| Zip | Raw value | Result | Method |
|---|---|---|---|
| 22790 | `04482255` | `Rio De Janeiro` | Four other sellers with the same zip use this city |
| 87025 | email address | `Maringa` | Three other sellers with the same zip use this city |
| 05141 | `sp` | `Pirituba` | One other seller with the same zip uses this name. It is a district of São Paulo city |
| 09726, 09861 | `sbc` | `Sao Bernardo Do Campo` | Manual mapping: SBC is the standard abbreviation, and both zips fall in that city's range. No other seller shares these zips, so the data cannot confirm it |
| 03363, 12903, 04776, 16021 | `sp` | `Unknown` | No other seller shares the zip, so no city can be derived without guessing |

These rows were **not** all in São Paulo city. The `sp` zips point to different cities, so a blanket replacement would have been wrong.

Items to check:
- `sbc` is the one value not derived from the data. It is a manual mapping, like the two category translations in the products table.
- `Pirituba` is a district, not a city. The rule used was "take the city the data gives". If `Sao Paulo` is preferred, change this one value.

---

## 4. Link with `order_items`

| Check | Result |
|---|---|
| Items pointing to an unknown seller | 0 |
| Sellers that never appear in an order | 0 |

Every seller has sold at least once, so no seller can be dropped without breaking sales joins.

---

## 5. Engineered column

| Column | Type | Definition |
|---|---|---|
| `city_was_fixed` | boolean | True when `seller_city` differs from the raw value or was replaced. 29 rows are True |

---

## 6. Decisions and rationale

| Decision | Reason |
|---|---|
| Keep all 3,095 sellers | Every seller has sold. Removing any would break joins to sales |
| Cut city text at `/`, `\`, `,` and `(` | In every case the city came first and the rest was state or country text |
| Look up the city by zip from other sellers | The data supplies the answer, so nothing is invented |
| Use `Unknown` when the zip has no other seller | A visible `Unknown` is more honest than a guess |
| Flag changed rows instead of hiding the change | Anyone can see or exclude the 29 repaired rows |
| Title-case cities | Display only. Done once in pandas so SQL and Power BI show the same values |

**Known cosmetic side effects of title-casing:** letters after an apostrophe and small connecting words are capitalised (`Santa Barbara D'Oeste`, `Sao Bernardo Do Campo`). This does not affect analysis. The value `Unknown` is capitalised too.

---

## 7. Using this table downstream

- **Always read the zip column as text.** In pandas pass `dtype={'seller_zip_code_prefix': str}`. In SQL Server use `VARCHAR(5)`.
- **Join to `order_items`** on `seller_id`. It is one-to-many: a seller appears in many items.
- **Seller sales** must be built from `order_items` (item `price`) joined to this table. This table holds no sales figures.
- **Sellers by city:** about 589 distinct cities with a few very large ones. Expect São Paulo to dominate. Consider excluding `Unknown` (4 sellers) from city rankings.
- **Seller state:** 23 of Brazil's 27 states appear, so four states have no sellers.
- **Seller and customer cities** come from different free-text sources. Comparing them (for example, for freight by distance) should use the zip prefix and the geolocation table, not city names.
- **Join to `geolocation`** on the zip prefix only after the geolocation zip column has been padded the same way.
