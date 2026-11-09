# Geolocation table: cleaning notes

**Source:** `olist_geolocation_dataset.csv` (Olist Brazilian E-Commerce Public Dataset)
**Output:** `cleaned/geolocation_clean.csv`
**Shape:** 19,015 rows (one per zip prefix) × 6 columns. The raw file had 1,000,163 rows (many points per zip) and 5 columns.
**Principle:** the raw table was **reshaped**, not just cleaned. Exact repeats were dropped and the many points per zip were collapsed to one row, so the table can join to customers and sellers without multiplying rows. No coordinate was invented: zips with no valid point stay empty.

---

## 1. What the raw table looks like

| Column | Meaning |
|---|---|
| `geolocation_zip_code_prefix` | First 5 digits of a zip code. Joins to the customer and seller zip columns |
| `geolocation_lat`, `geolocation_lng` | Coordinates of one geocoded point |
| `geolocation_city` | City name |
| `geolocation_state` | Two-letter state code |

A zip prefix covers an area, and the raw file holds **many points for each zip**, so the zip is not a key. The raw file has no missing values, and the zip column read as text with 5 characters throughout.

---

## 2. Cleaning steps

1. **Inspected structure and nulls.** 1,000,163 rows, no missing values.
2. **Checked for exact duplicates.** 261,831 rows (26.2%) were identical to an earlier row in every column. They were dropped, leaving **738,332** rows (see section 3).
3. **Profiled coordinates** and found points outside Brazil (see section 3). These were flagged for the build and excluded from the averages.
4. **Cleaned city text.** Odd apostrophes replaced, accents and garbled characters removed.
5. **Checked how consistent each zip is** (see section 3): 19,015 distinct zips, a median of 23 points per zip.
6. **Built one row per zip** (see section 4).
7. **Checked coverage of customers and sellers** (see section 5).
8. **Saved and verified.** The file was reloaded with the zip column as text: shape (19015, 6), unique zips, all length 5, 4 empty coordinates, latitude -33.69 to 4.48, longitude -72.91 to -32.41.

---

## 3. Data-quality findings

| Finding | Value |
|---|---|
| Rows in raw file | 1,000,163 |
| Exact duplicate rows | 261,831 (26.2%) |
| Rows after dropping them | 738,332 |
| Distinct zip prefixes | 19,015 |
| Points per zip (after de-duplication) | Median 23, mean 38.8, smallest 1, largest 779 |
| Points outside Brazil | 27 |
| Zips with more than one state | 8 |
| Zips with more than one city name | 549 |

**Exact repeats.** The same point appears several times, probably because several addresses were geocoded to the same spot. They carry no extra information, so dropping them loses nothing. This is the one table where rows were dropped.

**Points outside Brazil.** Latitude in the raw file ran from -36.6 to +45.1 and longitude from -101.5 to +121.1, but Brazil lies between about -34 and +5.5 latitude and -74 and -28 longitude. 27 points fall outside. They carry Brazilian state labels, so the coordinates are wrong (for example latitude +28 on a São Paulo zip, or a point near Buenos Aires). They look like sign flips and swapped values. They cannot be repaired without inventing numbers, so they were excluded from the averages.

**Zips with more than one state.** In each of the 8 cases one state holds nearly all the points and one stray point sits in another state (for example zip `21550`: 144 RJ points and 1 AC point; `79750`: 151 MS and 1 RS). The stray points are bad geocodes.

**City names.** The raw file had 8,011 distinct names, against 4,119 in customers and 589 in sellers. The causes:

| Cause | Example |
|---|---|
| Accents | `são paulo`, `jundiaí` |
| Garbled accents | `saƒo paulo` |
| No accent but a different spelling | `saopaulo`, `embu-guaçu` versus `embu guaçu` |
| A state code used as the city | `sp` |

After removing accents there are 5,967 names. The rest are variants that mostly appear once inside a zip, next to a dominant correct name (for example zip `01307`: 100 `sao paulo` and 1 `sao bernardo do campo`).

---

## 4. How the one-row-per-zip table was built

| Column | Rule |
|---|---|
| `geolocation_zip_code_prefix` | Key, text, 5 characters |
| `geolocation_city` | Most common city name among the zip's points |
| `geolocation_state` | Most common state among the zip's points |
| `geolocation_lat`, `geolocation_lng` | **Median** of the zip's valid points that lie in the zip's winning state. Points outside Brazil are excluded |
| `points_count` | Number of de-duplicated raw points behind the zip |

The **median** is used, not the mean, so a few stray points cannot pull the location away. Restricting to the winning state also removes the stray-state points.

**Result.** 19,015 rows, each zip once, 5,771 distinct cities. Latitude runs from -33.69 to 4.48 and longitude from -72.91 to -32.41, all inside Brazil.

**Four zips have no coordinates.** All of their points failed the checks, so they keep their city and state with empty coordinates:

| Zip | City | State |
|---|---|---|
| 18243 | bom retiro da esperanca | SP |
| 78131 | varzea grande | MT |
| 83252 | ilha dos valadares | PR |
| 95130 | santa lucia do piai | RS |

The city text in this file is lowercase and accent-free (it was not title-cased). It is a lookup table: display names should come from the customers and sellers tables.

---

## 5. Coverage of customers and sellers

| Check | Result |
|---|---|
| Customer rows whose zip has no coordinates | 279 of 99,441 (0.28%), across 158 distinct zips |
| Seller rows whose zip has no coordinates | 7 of 3,095 (0.23%) |

A zip counts as having no coordinates when it is missing from this table or is one of the 4 empty zips. A map or distance analysis covers about 99.7% of customers. The remaining rows appear blank after a join, not at a wrong location.

---

## 6. Decisions and rationale

| Decision | Reason |
|---|---|
| Drop 261,831 exact repeats | Identical in every column, so no information is lost |
| Collapse to one row per zip | The raw table has many points per zip. Joining it as it is multiplies every customer or seller row by up to 779 |
| Median instead of mean for coordinates | Resistant to the few stray points |
| Use only points in the winning state | Removes bad geocodes that sit in another state |
| Exclude the 27 outside-Brazil points | Wrong coordinates cannot be repaired without inventing values |
| Leave 4 zips empty | Better than guessing a location |
| Take the most common city per zip | Fixes the one-off spelling variants without hand-editing 5,000+ names |
| Remove accents | The customers and sellers tables are accent-free, so names stay consistent |
| Drop the `coord_outside_brazil` flag from the final table | It was only needed to build the averages. The final table has no outside-Brazil points |

---

## 7. Using this table downstream

- **Join to customers or sellers** on the zip prefix with a `LEFT JOIN`. This table has exactly one row per zip, so the join does not multiply customer or seller rows. Many customers can share one zip, but each still gets one matching row.
- **Always read the zip column as text.** In pandas pass `dtype={'geolocation_zip_code_prefix': str}`. In SQL Server use `VARCHAR(5)`.
- **Coordinates** are a zip-level centre, not a street address. Use them for maps by region and rough distances, not exact delivery locations.
- **Empty coordinates** load as `NULL`. Exclude them from maps and distance averages.
- **Distance between a customer and a seller** can be computed from the two zips' coordinates. This is approximate.
- **Do not use this table's city** for reporting. Use the city from customers or sellers, which came from the order data.
- **Reconciliation figures** after loading to SQL Server: 19,015 rows, 4 rows with `NULL` latitude.
