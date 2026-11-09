# Walmart Hurricane Analytics — DS 2002 Midterm (Team Data Demons)

**Team:** Esther Man & Ayman Haque · DS 2002 · Fall 2026

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/aymanr8-dot/DS-2002-Walmart-Hurricane-Project_Data-Demons-/blob/main/notebooks/team_notebook.ipynb)

We clean five messy Walmart data files, join them to real historical weather (Open-Meteo API) and to the National
Hurricane Center's official track for **Hurricane Helene**, and answer five questions about how Florida stores' demand
moved before the storm. Helene made landfall on **September 26, 2024** (11:10 pm EDT, Big Bend near Perry, FL).
The transactions cover **September 1–30, 2024** across **12 Florida stores**.

---

## How to run

1. Click **Open in Colab** above (or open `notebooks/team_notebook.ipynb` in Colab or Jupyter).
2. **Runtime → Run all.** The notebook runs start to finish with no setup:
   - Data files load from the local `data/` folder if present, otherwise straight from this repo on GitHub.
   - The weather and geocoding calls need internet access; no API key is required.
   - If an API is temporarily down, the notebook retries with backoff and then falls back to a cache of a previous
     successful call (weather) or to the HURDAT2 rows copied into the notebook (hurricane track), and says so.
3. The last code cell is a self-check: it prints `PASS`/`FAIL` for every requirement and asserts they all pass.

Python packages used: `pandas`, `numpy`, `requests`, `matplotlib`, `sqlite3` (standard library). All are preinstalled in Colab.

## Repository layout

```
├── README.md                     this file
├── CHARTER.md                    team charter
├── data/
│   ├── walmart_transactions.csv        25,562 POS sales rows
│   ├── store_locations.csv             14 rows -> 12 stores after cleaning
│   ├── product_catalog.csv             45 SKU records
│   ├── distribution_centers (1).csv    5 distribution centers
│   └── inventory_and_sales (1).db      SQLite: daily_sales_summary (3,600 rows), inventory_levels (4,320 rows)
└── notebooks/
    ├── team_notebook.ipynb       ← the final deliverable
    ├── 2026-10-23 — Midterm Checkpoint — Lab.ipynb
    ├── 2026_10_30_midterm_progress_lab.ipynb
    └── scratch/                  working notebooks
```

## Notebook structure

| Section | What it does |
|---|---|
| 0 | Setup and analysis constants (landfall date, 7-day surge window, baseline) |
| 1 | Cleaning pipeline: one function per problem, chained, with asserts |
| 2 | Raw vs. clean profile, SKU mapping table, decision log, daily volume chart |
| 3 | Weather API (Open-Meteo archive): retries, `source` column, GMT → store-local time, checked join |
| 4 | **Q1** Demand surge |
| 5 | **Q2** SKU consolidation (the Pop-Tarts effect) |
| 6 | **Q3** Weather correlation and lag |
| 7 | **Q4** Store-level geography (distance to the official NHC track) and DC capacity |
| 8 | **Q5** Signal vs. artifact: board games (schemas first, then SQL on the SQLite database) |
| 9 | Recommendations to leadership |
| 10 | Full decision log and final self-check |
| 11 | Reflection (five prompts) |

## Data cleaning

Every problem from the brief is handled by its own function in one pipeline, with validation asserts, and logged in one
shared **decision log** (`File · Issue · Decision · Rows affected · Reason`) that covers every file we touched:

| Problem | What we did |
|---|---|
| Duplicate transactions | 30 exact duplicates dropped |
| Missing timestamps | 884 rows flagged, not dropped or imputed; excluded only from by-date analysis (checked: no category bias) |
| Inconsistent date formats | 5 formats parsed explicitly to datetime |
| Fragmented SKUs | 45 SKUs → 20 products through a mapping table (`SKU_MAP`), left join, `validate="many_to_one"`; units and revenue asserted unchanged |
| Inconsistent categories | 37 spellings → 10 categories |
| Negative quantities | 130 returns kept and flagged, excluded from demand |
| Prices stored as strings | `$` stripped, converted to float |
| Inconsistent store IDs | 60 spellings → 12 `FL-###` IDs, the same rule in the transactions and the store file; matches the SQLite tables |
| Missing values | Tallahassee coordinates geocoded; DC coordinates geocoded; blank DC cost left missing; NULL `units_on_order` left NULL |

Reference-table joins use `validate="many_to_one"`, and the sales × weather join is built on a full store × day grid
with `validate="one_to_one"`, asserting 360 rows, no nulls and every unit preserved.

## Questions and answers

**Q1 — Demand surge.** Surge window = the brief's **7 days before landfall (Sept 19–25)**; baseline = **Sept 1–14**
(two full weeks, flat, CV 2.9%, ends before demand broke out on Sept 16). Total demand rose **+145%**. Ranked by
**relative** lift: Breakfast (Pop-Tarts) +410%, Emergency +182%, Electronics +152%, Alcohol +126%, Health +124%.
Ranked by **absolute** increase, Emergency leads (+650 units/day), and Grocery moves from #8 to #3.

**Q2 — The Pop-Tarts effect.** Split across four SKUs, Pop-Tarts looks like four ordinary items (ranks 3, 5, 6, 7) and
bottled water looks like the top seller. Consolidated through the mapping table, Pop-Tarts is the **#1 product**
(377 units/day) with the highest lift of all 20 products (+410%, about double the next). Consolidation adds no sales:
fragmented and consolidated totals are identical.

**Q3 — Weather correlation.** Same-day weather barely explains sales (r ≈ ±0.2). Weather **two days later** explains
them well (max gust r = +0.71, minimum pressure r = −0.76): customers buy on the forecast, not the weather. Weather is
pulled for all 12 stores in one loop and converted from GMT to each store's local day (Pensacola is on Central time).

**Q4 — Geography.** Distances use the **official NHC best track** (HURDAT2), not just store locations. Every store
surged by about the same amount (+122% to +171%); distance to the track barely relates to lift (r = 0.21,
Spearman −0.08). Miami, farthest from the track, had the biggest lift. Demand peaked at 3.6× normal against ~2× DC
surge capacity, so shipping has to start when sales first rise, about 10 days out.

**Q5 — Board games.** Four of five tests say the spike is a real signal, not a bundle artifact: an independent table
in the SQLite database shows it, all three Monopoly SKUs rose, price and basket size were flat while buyers rose 2.6×,
and it appeared in all 12 stores. Inventory is inconclusive. Recommendation: ship board games sized to ~2.6× normal
(4× at the peak), after Emergency and Pop-Tarts.

## Sources

- **Project data:** the five files in `data/`, and the *Walmart Case*.
- **Weather:** Open-Meteo Historical Weather API (archive), https://archive-api.open-meteo.com/v1/archive — hourly
  precipitation, wind speed, wind gusts and sea-level pressure, requested in GMT for each store's coordinates.
- **Geocoding:** Open-Meteo Geocoding API, https://geocoding-api.open-meteo.com/v1/search — coordinates missing from
  the store and DC files.
- **Hurricane track:** NOAA National Hurricane Center, HURDAT2 Atlantic best-track database,
  https://www.nhc.noaa.gov/data/hurdat/ (Helene = AL092024), and the NHC Tropical Cyclone Report for Helene,
  https://www.nhc.noaa.gov/data/tcr/AL092024_Helene.pdf.

## Deliverables

- [x] Colab notebook that runs start to finish (`notebooks/team_notebook.ipynb`)
- [x] Five analytical questions, each with code, a visualization and a written interpretation
- [x] At least six labeled plots (the notebook has 14; the list is at the end of section 10)
- [x] Working weather API call with source documentation (section 3)
- [x] Reflection on all five prompts, with each team member contributing (section 11)
- [x] README and team project repo
- [x] Six-minute presentation with 2 minutes of Q&A
