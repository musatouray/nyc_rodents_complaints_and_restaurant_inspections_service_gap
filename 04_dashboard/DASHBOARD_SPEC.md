# Dashboard Spec — NYC Service Gap Index

Build target: a **Databricks AI/BI dashboard** over the Gold layer with a working ZIP-code
filter, so anyone can type their ZIP and get something true (the ZIP-Code Challenge, 20% of
the score). All datasets read `nyc_rats.03_gold.*`.

> In Databricks SQL, back-tick the schema because it starts with a digit:
> `` nyc_rats.`03_gold`.zip_service_gap_gold ``.

---

## 1. Datasets (Dashboard → Data tab)

Create these named datasets. `:zip` is a **dashboard parameter** (String, default empty).

**`ds_zip_detail`** — the selected ZIP's row (drives the KPI tiles):
```sql
SELECT *
FROM nyc_rats.`03_gold`.zip_service_gap_gold
WHERE zip = :zip;
```

**`ds_all_zips`** — every ZIP (drives the citywide map/bar and the ranking table):
```sql
SELECT *
FROM nyc_rats.`03_gold`.zip_service_gap_gold;
```

**`ds_trend`** — monthly trend for the selected ZIP:
```sql
SELECT month, complaints, confirmed_complaints, rodent_restaurants
FROM nyc_rats.`03_gold`.zip_monthly_trend_gold
WHERE zip = :zip
ORDER BY month;
```

**`ds_cuisines`** — story hook:
```sql
SELECT cuisine_description, restaurant_count, rodent_rate
FROM nyc_rats.`03_gold`.top_rodent_cuisines_gold
ORDER BY rodent_rate DESC
LIMIT 12;
```

---

## 2. The ZIP filter

- Add a **Filter (single value)** widget bound to parameter `:zip`, field
  `zip` from `ds_all_zips`. Label it **"Type your ZIP code"**. Put it top-left, full width.
- Set a sensible default (e.g. a High-reliability Queens ZIP like `11419`) so the dashboard
  is never blank on open.
- Because `ds_zip_detail` / `ds_trend` filter on `:zip`, every tile below reacts to it.

---

## 3. Layout (top to bottom)

### Row 0 — title + filter
- Text tile: **"Is My Block Cursed? — NYC Service Gap Index"** with a one-line subtitle:
  *"When someone calls about rats, does the city show up? Type a ZIP to find out."*
- The ZIP filter widget.

### Row 1 — KPI counters (from `ds_zip_detail`)
Five counter tiles for the selected ZIP:
| Tile | Field |
|---|---|
| Service Gap Index | `service_gap_index` |
| Restaurants | `restaurant_count` |
| Rodent evidence rate | `rodent_evidence_rate` (format %) |
| Confirmed complaints | `confirmed_complaints` |
| Median days to close | `median_days_to_close` |

Add a **text/label tile** bound to `gap_label` and `reliability` so the headline reads in
plain English, e.g. *"High confirmed evidence, low reporting — reliability: High."*

### Row 2 — citywide context (from `ds_all_zips`)
- **Map** (preferred): counties/ZIP not built in, so use a **bar chart** — top 15 ZIPs by
  `service_gap_index` where `reliability = 'High'`, colored by `service_gap_index`
  (diverging: red high, blue low). Highlight/annotate the selected ZIP if possible.
- **Ranking table:** `zip, borough, service_gap_index, rodent_evidence_rate,
  complaint_total, gap_label`, sorted by `gap_rank`, filtered to `reliability = 'High'`.

### Row 3 — trend + story
- **Line chart** (`ds_trend`): `complaints` and `rodent_restaurants` by `month` for the
  selected ZIP.
- **Bar chart** (`ds_cuisines`): `rodent_rate` by `cuisine_description`.

### Row 4 — "what this means" (static text tile)
```
The Service Gap Index compares two things the city already records: rodent evidence found by
restaurant inspectors, and rat complaints filed by residents to 311. A high positive score
means inspectors keep finding rodents but residents rarely call — the city can see a problem
its residents aren't reporting. A raw complaint count is NOT this: it mostly measures who
calls 311 (civic trust, tenure, language, time), so ranking ZIPs by complaint volume sends
help to neighborhoods already getting attention.
```

---

## 4. Low-data / empty-ZIP behavior (this wins or loses the ZIP-Code Challenge)

Every ZIP returns a row, but low-data ones must not fake an extreme.
- The KPI/label tiles already show `reliability`. When it is `Low`, the `gap_label` reads
  **"Insufficient data to rank"** and `service_gap_index`/`gap_rank` are null — the counters
  still show real raw counts (`restaurant_count`, `complaint_total`).
- Add a conditional **text tile** for the honest message on sparse ZIPs. Dataset:
```sql
SELECT reliability, restaurant_count, complaint_total,
       CASE WHEN reliability = 'Low'
            THEN 'Not enough data to rank this ZIP. Showing raw counts only.'
            ELSE '' END AS note
FROM nyc_rats.`03_gold`.zip_service_gap_gold
WHERE zip = :zip;
```
- **Test before demo:** `11430` (airport, Low), a normal residential ZIP (High), and a
  non-NYC ZIP like `90210` (no rows → "no data for this ZIP").

---

## 5. Polish checklist
- [ ] Default ZIP set so the dashboard is never blank on load.
- [ ] Percent formatting on rate fields; 0–1 decimals on days.
- [ ] Diverging color scale on the index (positive = concern).
- [ ] Selected ZIP echoed in the title or a label tile.
- [ ] Three screenshots saved (three different ZIPs) for the submission.
