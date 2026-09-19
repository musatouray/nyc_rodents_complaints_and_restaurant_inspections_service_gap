# Is My Block Cursed? — NYC Service Gap Index

An NYC Open Data challenge (Queens College Databricks Hackathon). Two public records —
**311 rodent complaints** and **restaurant health inspections**, January 2025 to present —
have never been compared. This project compares them, ZIP by ZIP, to answer one question:

> **When a New Yorker calls for help about rats, does the city show up?**

The deliverable is a **Service Gap Index** for every NYC ZIP code, served through a **Genie
space** and an **interactive dashboard** where anyone can type their ZIP and learn how their
block is served.

---

## The finding (why the obvious analysis is wrong)

Ranking ZIPs by raw 311 complaint volume does **not** find the worst rat neighborhoods — it
finds the neighborhoods that *call 311 the most*. Complaint volume tracks civic trust,
tenure, language, and free time, not rodents. In this data:

- Correlation between raw complaint volume and restaurant rodent-evidence rate is **~0.12** —
  essentially none. The two records **do not agree**.
- The ZIPs with the most complaints (parts of Manhattan, the Bronx, brownstone Brooklyn) are
  **not** the ZIPs with the biggest service gap.
- The largest gaps cluster in **Queens** (e.g. 11419 Richmond Hill, 11429 Queens Village,
  11355 Flushing, 11432 Jamaica): inspectors keep finding rodents, but residents file
  comparatively few 311 calls.

**If you rank by complaints and call it a rat map, you have measured civic trust and
mislabeled it** — and you would send resources to neighborhoods already getting attention.

---

## The metric

For every ZIP, using only the two provided files:

- `rodent_evidence_rate` — share of the ZIP's **restaurants** (distinct `camis`) cited for a
  rodent-evidence violation (`04K` rats, `04L` mice, `08A` harborage). *Regulator-confirmed
  presence.*
- `complaint_intensity` — confirmed 311 rodent complaints per restaurant. *Resident demand
  signal.*
- **`service_gap_index = z(rodent_evidence_rate) − z(complaint_intensity)`** — standardized
  over reliable ZIPs. **Positive = evidence the city sees but residents aren't reporting.**
- `closure_rate` / `median_days_to_close` — does the city actually close the tickets?
- `reliability` — `High` only when a ZIP has ≥20 restaurants and ≥15 complaints. Low-data
  ZIPs (including the mostly-airport 11430) are reported honestly, never ranked.

Normalization is **per restaurant** on purpose: it is self-contained (no external data),
and it keeps both signals on the same denominator. Swapping in Census population per ZIP is
a documented stretch — see the plan.

---

## Architecture (medallion)

Databricks Free Edition · Unity Catalog · catalog **`nyc_rats`**.

| Layer | Schema | Tables |
|---|---|---|
| Source | `source_data` | volume `raw_data` (the two raw CSVs) |
| Bronze | `01_bronze` | `rat_sightings_bronze`, `restaurant_inspections_bronze` |
| Silver | `02_silver` | `rodent_complaints_silver`, `restaurant_violations_silver`, `restaurants_silver` |
| Gold | `03_gold` | `zip_service_gap_gold`, `zip_monthly_trend_gold`, `top_rodent_cuisines_gold` |

Bronze = raw as-ingested. Silver = cleaned, typed, ZIP-standardized, **every table and
column described** (Genie reads that metadata). Gold = business-ready; Genie and the
dashboard read **only** Gold.

**Data lineage:** bronze stamps each row with `file_name`, `file_path`, and `ingested_at`.
Silver carries those through and adds `silver_loaded_at`; gold adds `gold_loaded_at` — so any
row can be traced back to its source file and the time it moved through each layer.

---

## Repository layout

```
01_raw_data/                     Source CSVs + the challenge brief (Is_My_Block_Cursed.md)
02_notebooks/
  00_config.ipynb                Catalog/schema config, run via %run ./00_config
  01_raw_data_to_bronze.ipynb    Volume CSVs -> bronze Delta tables
  02_bronze_to_silver.ipynb      Clean, standardize, describe (3 silver tables)
  03_silver_to_gold.ipynb        Service Gap Index + marts + sanity checks
  DeltaSharing.ipynb             Delta Sharing setup
03_genie/GENIE_SPACE.md          Genie space build + Live Question Round prep
04_dashboard/DASHBOARD_SPEC.md   Dashboard datasets, ZIP filter, tiles, low-data behavior
IMPLEMENTATION_PLAN.md           Full step-by-step build plan (optional, if committed)
```

---

## How to run

1. **Upload** `rat_sightings.csv` and `restaurant_inspections.csv` to the volume
   `nyc_rats.source_data.raw_data` (Catalog → Volumes).
2. Run the notebooks in order: **`01` → `02` → `03`**. (`00_config` is auto-run by each via
   `%run ./00_config`.) Notebook `01` prints the Block One answers, including the checkpoint:
   **26,114 restaurants** (`COUNT(DISTINCT camis)`, not the 158k row count).
3. Build the **Genie space** — follow [`03_genie/GENIE_SPACE.md`](03_genie/GENIE_SPACE.md).
4. Build the **dashboard** — follow [`04_dashboard/DASHBOARD_SPEC.md`](04_dashboard/DASHBOARD_SPEC.md).

> Re-running a `CREATE OR REPLACE TABLE` cell wipes that table's column comments — re-run the
> `COMMENT` / `ALTER TABLE` cells too, or Genie quietly gets worse.

---

## Data notes / decisions

- **One row per violation, not per restaurant.** Count restaurants with
  `COUNT(DISTINCT camis)`.
- **"Rodent complaint" = confirmed sighting** (Rat Sighting + Signs of Rodents + Mouse
  Sighting). "Condition Attracting Rodents" is kept but flagged as a risk condition, not a
  sighting.
- **Excluded:** 1,513 inspection rows with a blank ZIP (cannot join to a ZIP).
- **Biggest limitation:** 311 measures *who reports*, not where rats are. The index is built
  to expose that, not to hide it.

**If I worked for the city, I would** target inspection and outreach at ZIPs with high
confirmed rodent evidence but low complaint volume, **because our data shows** those
neighborhoods have rats the city can already see, but residents aren't reporting them.
