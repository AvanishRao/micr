# Safe Exit Score – README

## Overview

This code calculates a **Safe Exit Score** for Australian residential markets, answering one core question:

> *“If I buy here, how easy is it to sell later without a painful discount?”*

It turns messy geospatial and transaction data into a single, investor-friendly metric at an area level.

---

## Data Inputs

The pipeline uses these core inputs:

1. **`transactions.parquet`**

   * Contains property-level transactions.
   * Key fields (explicit or auto-detected):

     * Unique property/link ID (e.g. `gnaf_pid`)
     * Sale price
     * Sale/contract/settlement date
     * Optional: `days_on_market`

2. **`gnaf_prop.parquet`**

   * G-NAF property reference table.
   * Used to attach **coordinates/geometry** to each transaction via `gnaf_pid`.

3. **`cadastre.gpkg`**

   * Polygon layer including an **area identifier** (e.g. `sa4`).
   * Used to assign each transaction to an area via spatial join.

---

## What the Code Does (Step-by-Step)

### 1. Schema Detection & Validation

* Inspects `transactions.parquet` to find:

  * Sale price column
  * Sale date column
* Validates presence of `gnaf_pid` and area codes in cadastre.
* Validates/constructs geometry in `gnaf_prop` from either:

  * Existing `geometry`, or
  * Numeric lon/lat columns.

### 2. Geospatial Linking

* Joins transactions to G-NAF using `gnaf_pid` to obtain point geometry.
* Spatially joins transaction points into `cadastre.gpkg` polygons to assign an **`area_id`** (e.g. SA4).
* Result: every usable transaction is tagged with an area.

### 3. Lookback Filtering

* Filters transactions to a configurable lookback window (e.g. **5 years**).
* Ensures the metric reflects recent, relevant market behaviour.

### 4. Component Metrics (by Area)

For each `area_id`, the code computes:

1. **Liquidity / Turnover**

   * Distinct properties (proxied by `gnaf_pid`) vs. number of sales in the lookback period.
   * Higher turnover ⇒ deeper buyer pool, easier exit.
   * Scaled to a **0–100 Liquidity Score**.

2. **Days on Market (optional)**

   * Median `days_on_market` where available.
   * Faster selling areas ⇒ higher **DOM Score**.
   * If missing, this component is down-weighted automatically.

3. **Price Stability / Downside Risk**

   * Builds annual median prices.
   * Calculates:

     * Volatility in year-on-year growth.
     * Maximum drawdown from peak prices.
   * Low volatility + shallow drawdowns ⇒ higher **Stability Score** (0–100).

### 5. Safe Exit Score

* Combines:

  * Liquidity Score
  * DOM Score (if available)
  * Stability Score
* Uses weighted average with dynamic re-weighting if some components are missing.
* Produces a **0–100 Safe Exit Score per area**:

  * **80–100**: Very strong exit market
  * **60–79**: Generally safe to exit
  * **40–59**: Higher risk – review carefully
  * **0–39**: High exit risk – expect difficulty/discounts

---

## Outputs

Typical final output table (by `area_id`):

* `num_properties_proxy`
* `num_sales`
* `turnover_rate`
* `liquidity_score`
* `median_dom` (if available)
* `dom_score`
* `price_volatility`
* `max_drawdown`
* `stability_score`
* `safe_exit_score`
* `safe_exit_label`

This can be exported (CSV/Parquet) or surfaced as:

* heatmaps (choropleths),
* rankings of safer vs riskier exit markets,
* an API feeding investor dashboards or listing pages.

---

## Key Assumptions

* Transactions represent real, arm’s-length sales with reliable price and date.
* `gnaf_pid` correctly links transactions to G-NAF coordinates.
* Cadastre polygons and area codes (e.g. SA4) are correct and stable.
* Distinct `gnaf_pid` counts reasonably proxy dwelling stock for turnover.

---

## Notes & Next Steps

The current implementation is an MVP focused on **clarity and defensibility**. With more time, it can be extended to:

* finer geographies (SA2, SA1, custom microburbs),
* property-level adjustments (main-road, noise, climate/insurance risk),
* better stock universes (official dwelling counts),
* hardened configs, validation, and automated production pipelines.




## Results Visualization - Safe Exit Score Analysis

This chart shows:
1. Score distribution across NSW suburbs
2. How liquidity, DOM, and stability combine
3. Risk profile scatter with transaction volumes
4. Real top/bottom performer suburbs

## Results Visualizations - Safe Exit Score Heatmap

It visually ranks local areas by how easy it is to sell there, turning complex liquidity and risk metrics into a simple, intuitive signal for investors.
