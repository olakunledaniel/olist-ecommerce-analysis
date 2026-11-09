# Products table: cleaning notes

**Sources:** `olist_products_dataset.csv` and `product_category_name_translation.csv` (Olist Brazilian E-Commerce Public Dataset)
**Output:** `cleaned/products_clean.csv`
**Shape:** 32,951 rows (one per product) × 11 columns. 9 raw columns (2 renamed), 2 added.
**Principle:** no rows dropped and no measurements invented. Problem rows are flagged, not altered.

---

## 1. What the raw table looks like

| Column | Meaning |
|---|---|
| `product_id` | Unique product identifier. Joins to `order_items` |
| `product_category_name` | Category, in Portuguese |
| `product_name_lenght` | Number of characters in the product name (misspelled in the source) |
| `product_description_lenght` | Number of characters in the product description (misspelled in the source) |
| `product_photos_qty` | Number of photos in the listing |
| `product_weight_g` | Weight in grams |
| `product_length_cm`, `product_height_cm`, `product_width_cm` | Package dimensions in centimetres |

The table holds no product names or descriptions, only their lengths.

---

## 2. Cleaning steps

1. **Inspected structure and nulls** (see section 3).
2. **Checked the key.** `product_id` is unique across all 32,951 rows, so there are no duplicate rows and a full-row duplicate check is unnecessary.
3. **Checked whether the missing values overlap.** All 610 products without a category are also missing name length, description length and photo count: one block of 610 products with no descriptive data.
4. **Profiled weight and size,** and looked at every zero-weight and missing-weight product.
5. **Checked category text.** No stray spaces and no capital letters. All values are lowercase underscore names.
6. **Checked the translation file** and found two categories without an English name (see section 4).
7. **Verified the link with `order_items`:** no item points to an unknown product, and every product has sold at least once.
8. **Renamed the two misspelled columns** to `product_name_length` and `product_description_length`.
9. **Converted counts and sizes to whole numbers** (`Int64`), keeping empty cells empty. All values were already whole, so nothing was rounded.
10. **Added English category names** (`product_category_name_english`) from the translation file, plus two translations added by hand. The merge was checked to leave the row count at 32,951.
11. **Added the flag `has_size_issue`.**
12. **Saved and verified.** The file was reloaded and checked: shape (32951, 11), unique `product_id`, 6 flagged rows, 610 `unknown` categories, no empty English names.

---

## 3. Data-quality findings

### Missing values

| Column group | Missing | Notes |
|---|---|---|
| Category, name length, description length, photo count | 610 each | The same 610 products in all four columns |
| Weight and the three dimensions | 2 each | The same 2 products. One of them is inside the 610 group |

### Weight and size

| Metric | Weight (g) | Length (cm) | Height (cm) | Width (cm) |
|---|---|---|---|---|
| Count | 32,949 | 32,949 | 32,949 | 32,949 |
| Mean | 2,276.5 | 30.8 | 16.9 | 23.2 |
| Median | 700 | 25 | 13 | 20 |
| 25th / 75th percentile | 300 / 1,900 | 18 / 38 | 8 / 21 | 15 / 30 |
| Min / Max | 0 / 40,425 | 7 / 105 | 2 / 105 | 6 / 118 |

- **No zero dimensions.** The smallest length, height and width are 7, 2 and 6 cm.
- **4 products have a weight of 0 g,** which cannot be real. All four are in `cama_mesa_banho` and share identical dimensions of 30 × 25 × 30 cm, one photo and a description of about 528 characters. This looks like one seller's near-identical listings where the weight was left blank and stored as 0.
- **The largest weight, 40,425 g (about 40 kg),** is plausible for furniture or appliances. Kept.
- The mean weight is far above the median, so a few heavy items pull the average up. Report medians next to means.

### Categories

- **73 distinct categories.** The largest are `cama_mesa_banho` (3,029 products), `esporte_lazer` (2,867), `moveis_decoracao` (2,657), `beleza_saude` (2,444) and `utilidades_domesticas` (2,335). The smallest are `cds_dvds_musicais` (1 product), `seguros_e_servicos` (2) and `pc_gamer` (3).
- **The translation file has 71 rows,** no nulls and unique keys, but two categories in `products` are not in it:

| Category | English name added by hand |
|---|---|
| `pc_gamer` | `pc_gamer` |
| `portateis_cozinha_e_preparadores_de_alimentos` | `portable_kitchen_and_food_preparers` |

These two translations were added by the analyst. They are translations of a label, not measured data, and are listed here so they can be checked.

### Link with `order_items`

| Check | Result |
|---|---|
| Items pointing to an unknown product | 0 |
| Products that never appear in an order | 0 |

---

## 4. Engineered and changed columns

| Column | Type | Definition |
|---|---|---|
| `product_category_name_english` | text | English category from the translation file. Products with no category are labelled `unknown` (610 rows) |
| `has_size_issue` | boolean | True when `product_weight_g` is missing (2 products) or equal to 0 (4 products). 6 rows are True |

The original Portuguese `product_category_name` keeps its empty cells. Only the English column uses the label `unknown`.

---

## 5. Decisions and rationale

| Decision | Reason |
|---|---|
| Keep all 32,951 products | Every product has sold, so removing any would break joins to sales |
| Keep the 610 products with no category, labelled `unknown` in the English column | They are real, sold products. A visible `unknown` group in charts is more honest than dropping them or inventing a category |
| Flag, not fix, the 6 weight problems | A guessed weight would be invented data. The flag lets analysis exclude them |
| Keep the 40 kg maximum | Plausible for large items |
| Translate the 2 missing categories by hand | Simple label translations, documented above |
| Rename the two misspelled columns | Avoids carrying a typo into SQL and Power BI |

---

## 6. Using this table downstream

- **Join to `order_items`** on `product_id`. It is one-to-many: one product appears in many items.
- **Category charts:** use `product_category_name_english`. Expect an `unknown` category with 610 products, and consider filtering it out of top-category rankings.
- **Weight-based analysis** (for example freight cost against weight): filter `has_size_issue = False`.
- **Counts and sizes** are `Int64`. In SQL Server they can be `INT` columns that allow NULL.
- **Sales by category** should be built from `order_items` (item `price`) joined to this table. The number of products in a category is not a measure of sales.
