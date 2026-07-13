# Thermal Refuge Equity — Curico & Valdivia

**[→ Explore the interactive map](https://geoforestdata.github.io/ThermalEquity_CuricoValdivia/)**

Who lives in the parts of a city with the least relative access to thermal relief — and is that changing over time?

This project builds a hexagon-level **Cooling Index** from open satellite data and cross-references it with modeled population to measure thermal equity in two Chilean cities, Curico and Valdivia, comparing 2016 against 2026.

**This is a worked example, not a two-city-only tool.** The method — H3 grid, NDVI/LST from Landsat, population from WorldPop, all standardized within-city — only needs a city's built-up boundary and a bounding box to run anywhere Landsat and WorldPop have coverage. Curico and Valdivia were chosen to demonstrate the method end-to-end with a real, interpretable contrast; see [Applying this to another city](#applying-this-to-another-city) for what to change and what to check before trusting the output elsewhere.

## Problem

Urban heat exposure is not evenly distributed within a city. Vegetation cover and surface temperature vary block by block, and population is not spread evenly across that variation. Standard urban heat island studies often stop at mapping temperature; this project asks a narrower, more actionable question: **what share of a city's population lives in the quarter of the city with the worst relative cooling conditions, and is that share growing or shrinking?**

Two cities with different baseline climates are compared, which raises a design problem: a direct temperature comparison between a warmer and a cooler city says more about climate than about urban equity. The project addresses this by standardizing every metric **within each city-year** — the unit of comparison is a hexagon's position relative to its own city, not an absolute value comparable across cities.

## Methodology

**Spatial unit:** H3 hexagonal grid, resolution 9 (~174 m edge, locally ~0.088–0.092 km² per cell — H3 is not strictly equal-area, and cell area varies with latitude).

**City extent:** GHSL Urban Centre Database (R2024A), fixed built-up boundary, epoch 2025. Cities are selected by point-in-polygon containment against a known city-center coordinate, not by matching a name field (avoids encoding/naming-convention issues across dataset releases).

**Cooling Index**, per hexagon, per city-year:

```
Cooling Index = z(NDVI) − z(LST)
```

NDVI and Land Surface Temperature (Landsat 8/9, Collection 2 Level 2, `ST_B10`, austral-summer composite Jan–Mar) are each standardized (z-score) within their `(city, year)` group, then combined. Positive = relatively cool and vegetated for that city that year. Negative = relatively hot and bare.

**Population:** WorldPop (constrained settlement model), aggregated to each hexagon as an area-weighted sum.

**Equity metric:** hexagons are split into quartiles of Cooling Index within each city-year. The metric reported is the percentage of that city-year's population living in quartile 1 — the worst 25% of hexagons. By construction, an even population distribution across thermal conditions would put ~25% of the population in that quartile; deviation above 25% indicates concentration in thermally disadvantaged areas.

Full method-level decisions and how city candidates are validated before committing to them are logged in [`docs/DEVLOG.md`](docs/DEVLOG.md).

## Data

| Source | Use | Notes |
|---|---|---|
| [GHSL Urban Centre Database R2024A](https://human-settlement.emergency.copernicus.eu/ghs_ucdb_2024.php) | City boundary | Fixed delineation, epoch 2025 |
| Landsat 8/9 Collection 2 Level 2 | NDVI, Land Surface Temperature | Via Google Earth Engine |
| [WorldPop (sat-io community catalog)](https://gee-community-catalog.org/projects/worldpop/) | Population per hexagon | Constrained model, 2015–2030 annual; not the official `WorldPop/GP/100m/pop` GEE catalog, which stops at 2021 |
| [Uber H3](https://h3geo.org/) | Spatial indexing grid | Resolution 9 |

## Installation

```bash
git clone https://github.com/geoforestdata/ThermalEquity_CuricoValdivia.git
cd ThermalEquity_CuricoValdivia
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Requires a [Google Earth Engine](https://earthengine.google.com/) account with API access authenticated (`earthengine authenticate` or the `ee.Initialize()` flow in the notebook).

Download the GHSL UCDB regional package ("Latin America and the Caribbean") from the [JRC download portal](https://human-settlement.emergency.copernicus.eu/download.php) and place it under `data/ghsl/`. Not bundled in this repo — the file is several hundred MB.

## Usage

Run `notebooks/analysis.ipynb` top to bottom. It regenerates every output in `outputs/qgis_ready/` from source data:

1. H3 grid per city
2. NDVI/LST composites (exported as GeoTIFF)
3. Zonal aggregation + Cooling Index
4. Population per hexagon
5. Equity table (`thermal_equity.csv`)

Outputs are not versioned (`outputs/` is gitignored) — they're fully reproducible from the notebook and source data, not committed artifacts.

## Results

| City | 2016 | 2026 | Δ |
|---|---|---|---|
| Curico | 26.2% | 28.2% | +1.9 |
| Valdivia | 42.4% | 42.0% | −0.5 |

Curico sits close to the 25% baseline expected under an even distribution, drifting slightly upward. Valdivia holds close to double that share, stable across the decade — visually confirmed against satellite imagery as a contiguous cluster over the dense urban core, not a statistical artifact driven by a handful of high-population hexagons.

This does **not** claim Valdivia is thermally worse than Curico in absolute terms — the within-city standardization specifically rules that comparison out. It claims that exposure to relative thermal disadvantage is far more unevenly distributed among Valdivia's own residents than among Curico's.

## Applying this to another city

The pipeline is parameterized by two things per city: a center-point coordinate (for UCDB boundary lookup) and a country code (for the WorldPop filter). To add a city:

1. **Add a centroid** to `CITY_CENTROIDS` in the notebook — any point inside the city's built-up area works, it's only used for polygon containment.
2. **Check GHSL UCDB coverage.** The UCDB regional package used here is Latin America & Caribbean; a city outside that region needs the corresponding regional download from the [JRC portal](https://human-settlement.emergency.copernicus.eu/download.php).
3. **Screen Landsat thermal coverage before running the full pipeline.** This is the step that isn't optional. Good NDVI coverage does not imply good LST coverage — the surface-temperature retrieval algorithm has its own, stricter internal masking and can fail systematically in specific terrain, independent of cloud cover, scene revisit frequency, or seasonal window length. Before committing to a city, run:

   ```python
   aoi = ee.Geometry.Point([lon, lat]).buffer(5000)
   collection = ee.ImageCollection("LANDSAT/LC08/C02/T1_L2").filterBounds(aoi).filterDate(start, end)
   valid_fraction = collection.select("ST_B10").map(lambda img: img.mask()).max() \
       .reduceRegion(ee.Reducer.mean(), aoi, scale=30, maxPixels=1e9).get("ST_B10").getInfo()
   ```

   Below ~0.9, expect a meaningful share of hexagons to drop out of the LST-dependent metrics — decide up front whether that's acceptable for the city in question rather than discovering it mid-pipeline.
4. **Update `country_code`** in the WorldPop filter if outside Chile.
5. **Everything else — H3 generation, NDVI/LST extraction, zonal aggregation, z-scoring, quartile equity metric — is city-agnostic** and runs unchanged.

## Limitations

- **WorldPop 2026 is a model projection**, not a census measurement — it interpolates/extrapolates from circa-2010/2020 census rounds, adjusted to UN World Population Prospects 2024 estimates. 2016 sits within the census-supported range; 2026 does not carry the same certainty despite the output format looking identical.
- **Population source is a community-maintained GEE catalog** (`sat-io`, curated by Samapriya Roy), not Google's official WorldPop ingestion, which has no post-2021 epoch.
- **Constrained population model** — totals run systematically ~15% below unconstrained sources (e.g. GHSL) for the same footprint, because it excludes non-built-up land. Doesn't affect the equity metric (relative distribution, not absolute totals), but any total population figure quoted from this project should carry that caveat.
- **Landsat thermal-band coverage is not guaranteed uniform across cities.** Any city considered for this method should be screened for ST coverage before committing (see "Applying this to another city" above) — good vegetation-band coverage does not guarantee good thermal-band coverage in every location.
- **The Cooling Index measures relative position, not absolute comfort.** It cannot be used to claim one city is objectively hotter or more livable than another.

## Future improvements

- Extend to cities with complex terrain (river deltas, hill-heavy topography) using an alternative thermal source with different failure modes than Landsat ST — MODIS LST (daily revisit, coarser resolution) or ECOSTRESS.
- Extend the equity metric with an income or deprivation covariate to test whether thermal disadvantage correlates with socioeconomic disadvantage, rather than only reporting the spatial/demographic pattern on its own.
- Sensitivity check on quartile-based equity metric against tercile or quintile cuts, to confirm the finding isn't an artifact of the specific 25% threshold.

## Citation

If you use this repository, workflow, or derived products in academic work, please cite:

> Vega-Escobar, A. (2026). *Thermal Refuge Equity Mapping for Curico and Valdivia using Google Earth Engine* (Version 1.0.0). GitHub. https://github.com/geoforestdata/ThermalEquity_CuricoValdivia

## Data and Software Attribution

This project uses the following open data and software:

- Google Earth Engine
- USGS Landsat 8/9 Collection 2 Level 2
- GHSL Urban Centre Database R2024A (JRC, European Commission)
- WorldPop population estimates (`sat-io` community catalog, curated by Samapriya Roy)
- CartoDB basemaps (Positron / Voyager)
- Uber H3 (hexagonal hierarchical spatial index)
- Python ecosystem: Earth Engine API, geemap, GeoPandas, Rasterio, rasterstats, Folium, Matplotlib

Please cite these data sources according to their respective licenses and citation guidelines when appropriate.

## License

Code: MIT. Data sources retain their own licenses (GHSL — CC BY 4.0; WorldPop — CC BY 4.0; Landsat — public domain, USGS).
