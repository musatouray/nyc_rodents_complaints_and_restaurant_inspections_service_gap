# Genie Space — "Is My Block Cursed?"

This is the build + prep guide for the Genie space the judges will sit down at.
Genie answers well **only** because the Gold tables are clean and every column is
described. It reads your metadata; it does not read your mind. Treat this document as the
script for the Live Question Round (30% of the score).

---

## 1. Create the space

1. **Genie** (left nav) → **New** → New Genie space.
2. **Name:** `NYC Service Gap`.
3. **Tables** — add **only** the Gold tables (clean surface, described columns):
   - `nyc_rats.03_gold.zip_service_gap_gold`
   - `nyc_rats.03_gold.zip_monthly_trend_gold`
   - `nyc_rats.03_gold.top_rodent_cuisines_gold`
   - (optional) `nyc_rats.02_silver.restaurants_silver` — only if you want restaurant-name
     lookups. Do **not** add the raw bronze tables; they will make Genie miscount.
4. Pick the SQL warehouse your team shares.

> Run notebook `03_silver_to_gold` first. If you rebuild a Gold table, its column comments
> are wiped — re-run the `COMMENT` / `ALTER TABLE` cells or Genie quietly gets worse.

---

## 2. General instructions (paste into the space's *Instructions* box)

```
This space answers questions about NYC rodent complaints (311) and restaurant health
inspections, joined at the ZIP-code level.

Definitions and rules:
- A "restaurant" is a distinct camis. Never count rows to count restaurants; the violations
  data has one row per violation, so a restaurant with 8 violations is 8 rows. Use
  restaurant_count in zip_service_gap_gold, or COUNT(DISTINCT camis).
- A "rodent complaint" is a 311 request. A "confirmed sighting" is descriptor Rat Sighting,
  Signs of Rodents, or Mouse Sighting (confirmed_complaints). "Condition Attracting Rodents"
  is a risk condition, not a sighting.
- "Rodent evidence" in restaurants means violation codes 04K (rats), 04L (mice), or 08A
  (harborage). rodent_evidence_rate is the share of a ZIP's restaurants with such a code.
- The Service Gap Index = z(rodent_evidence_rate) - z(complaint_intensity). A positive index
  means restaurants show rodent evidence but residents file few 311 complaints: an
  under-reporting gap. Rank ZIPs only where reliability = 'High'.
- 311 volume measures who calls 311 (civic trust, tenure, language, time), not where rodents
  are. Do not call the ZIP with the most complaints the "worst rat ZIP". If asked for the
  worst rat ZIP, answer with rodent_evidence_rate or service_gap_index, and say why raw
  counts are misleading.
- The default table for ZIP questions is zip_service_gap_gold (one row per ZIP).
- For a ZIP with reliability = 'Low' (few restaurants or complaints, e.g. the airport ZIP
  11430), say there is not enough data to rank it and report the raw counts instead.
```

---

## 3. Curated example SQL (add each as a *sample query* / *instruction* with a title)

Adding these teaches Genie your table shapes and trains it on the phrasings judges use.
All names are fully qualified; the schema is backticked because it starts with a digit.

**Q — How many restaurants are in the data?**
```sql
SELECT COUNT(DISTINCT camis) AS restaurants
FROM nyc_rats.`02_silver`.restaurant_violations_silver;
-- 26,114. Not the 158k violation-row count.
```

**Q — Which ZIP has the worst service gap?**
```sql
SELECT zip, borough, service_gap_index, rodent_evidence_rate, complaint_total, gap_label
FROM nyc_rats.`03_gold`.zip_service_gap_gold
WHERE reliability = 'High'
ORDER BY service_gap_index DESC
LIMIT 5;
```

**Q — Tell me about ZIP 11373 (any ZIP the judge types).**
```sql
SELECT zip, borough, reliability, gap_rank, service_gap_index, gap_label,
       restaurant_count, rodent_evidence_rate,
       complaint_total, confirmed_complaints, closure_rate, median_days_to_close
FROM nyc_rats.`03_gold`.zip_service_gap_gold
WHERE zip = '11373';
```

**Q — Which ZIPs have the most rat complaints, and does that match rodent evidence?**
```sql
SELECT zip, borough, complaint_total, rodent_evidence_rate, service_gap_index
FROM nyc_rats.`03_gold`.zip_service_gap_gold
WHERE reliability = 'High'
ORDER BY complaint_total DESC
LIMIT 10;
-- Note for the demo: this list is NOT the high-gap list. Raw volume tracks who calls 311.
```

**Q — How fast does the city close rodent complaints in a borough?**
```sql
SELECT borough,
       ROUND(AVG(closure_rate), 3)        AS avg_closure_rate,
       ROUND(AVG(median_days_to_close), 1) AS avg_median_days_to_close
FROM nyc_rats.`03_gold`.zip_service_gap_gold
WHERE reliability = 'High'
GROUP BY borough
ORDER BY avg_median_days_to_close DESC;
```

**Q — Which cuisines have the highest rodent-evidence rate?**
```sql
SELECT cuisine_description, restaurant_count, rodent_rate
FROM nyc_rats.`03_gold`.top_rodent_cuisines_gold
ORDER BY rodent_rate DESC
LIMIT 10;
```

**Q — Trend for a ZIP over time.**
```sql
SELECT month, complaints, confirmed_complaints, rodent_restaurants
FROM nyc_rats.`03_gold`.zip_monthly_trend_gold
WHERE zip = '11373'
ORDER BY month;
```

---

## 4. Live Question Round prep

**Three you get in advance** — pre-run them, screenshot the answers, and confirm Genie
reproduces them:
1. How many restaurants are in the inspections data? → **26,114**.
2. Which ZIP has the worst service gap? → top `service_gap_index`, `reliability = 'High'`
   (expect a Queens ZIP such as 11419 / 11429 / 11355).
3. How many rodent complaints were filed in ZIP _____ and how quickly were they closed?

**Two you will not** — you cannot touch the keyboard, so rehearse *shape*, not exact text.
Likely curveballs and the column that answers them:
- "Where should the city send inspectors first?" → high `service_gap_index` + `High`
  reliability (evidence the city sees, residents not reporting).
- "Is the ZIP with the most complaints actually the worst?" → compare `complaint_total`
  ranking vs `service_gap_index` ranking; they differ. This is the trap answer.
- Borough rollups, per-cuisine, month-over-month trend → covered by the marts above.

**If Genie guesses wrong**, the fix is almost always a weak/missing **column comment**, not
the prompt. Improve the description in notebook `02`/`03` and re-run the comment cells.

---

## 5. Edge cases to test before the demo
- **Airport ZIP 11430 (JFK):** restaurants exist, ~no residential complaints →
  `reliability = 'Low'`. Genie should say "not enough data to rank" and give raw counts,
  not a fake extreme.
- **A ZIP not in NYC / typo (e.g. 90210):** returns no rows. Confirm Genie says it has no
  data for that ZIP rather than inventing an answer.
- **A sparse residential ZIP:** should also come back `Low` and behave gracefully.
