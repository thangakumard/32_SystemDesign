# Uber's H3: A Comprehensive Guide

## Hexagonal Hierarchical Spatial Index and Geospatial Grid System

### Table of Contents

1. [Introduction](#introduction)

2. [How It Works (Core Concept)](#how-it-works-core-concept)

3. [Pros and Cons](#pros-and-cons)

   * [Pros](#pros)

   * [Cons](#cons)

4. [When It Should Be Used](#when-it-should-be-used)

5. [Implementing H3 in a Database (e.g., PostgreSQL)](#implementing-h3-in-a-database-eg-postgresql)

6. [Applications and Services That Use Uber's H3](#applications-and-services-that-use-ubers-h3)

### Introduction

**Uber's H3** is an open-source, highly efficient **Hexagonal Hierarchical Spatial Index** designed to bin, index, and analyze geospatial data. Originally created by Uber in 2018, H3 partitions the surface of the Earth into a grid of hexagonal cells (with a small number of pentagons to allow closure over a sphere), providing a uniform way to measure distances and aggregate spatial phenomena.

### How It Works (Core Concept)

Unlike traditional rectangular grids (like latitude/longitude squares), which suffer from diagonal distance distortion (where the distance to a corner is greater than the distance to an edge), H3 uses **hexagons**. 

* **Uniform Adjacency:** Every neighbor of a hexagon is equidistant from its center, eliminating directional bias in spatial analysis and routing.
* **Hierarchical Resolution:** H3 supports 16 different resolution levels (from Level 0 covering massive regions down to Level 15 covering fine sub-meter areas). Each resolution divides a parent hexagon into smaller child hexagons cleanly.
* **Compact Indexing:** Every cell is encoded into a single 64-bit unsigned integer, making it trivial to store, sort, and query in standard databases.

### Pros and Cons

#### Pros

* **No Directional Distance Distortion:** Because hexagons are regular polygons where every adjacent neighbor shares an edge at an equal distance, calculations involving nearest neighbors, spread, and radial gradients are much more accurate than square grids.
* **Smooth Gradient Modeling:** Hexagons approximate circles better than squares do, making them ideal for modeling phenomena that spread uniformly outward (like signal strength, disease spread, or ride-sharing demand).
* **Multi-Resolution Hierarchy:** You can easily roll up data from fine resolutions to coarse resolutions (e.g., aggregating neighborhood ride requests into a city-wide overview) while maintaining mathematical consistency.
* **Robust Core Library:** Written in optimized C with official bindings for Python, Java, JavaScript, Go, and more, making it fast and easy to integrate into existing data pipelines.

#### Cons

* **Pentagonal Distortion:** To map a flat grid system seamlessly onto a 3D sphere, H3 *must* include exactly 12 pentagons per resolution level (a mathematical requirement of Euler's formula). While rare, calculations falling directly on or near these pentagonal cells can introduce minor geometric edge cases.
* **Inflexible Boundaries:** Like any discrete grid system, events occurring right across a cell boundary can be artificially separated, requiring spatial smoothing techniques for precise boundary analysis.
* **Learning Curve for Non-Spatial Engineers:** Understanding hierarchical indexing, resolution transitions, and compact integer parsing requires shifting away from traditional lat-long point mindset.

### When It Should Be Used

* **Ride-Sharing and Logistics Heatmaps:** Perfect for calculating dynamic pricing zones, driver supply vs. rider demand heatmaps, and dispatch regions.
* **Spatial Aggregation and Visualization:** When you need to bin millions of point events (like GPS pings, deliveries, or check-ins) into clean polygonal areas for rendering on web maps (such as Kepler.gl).
* **Machine Learning Spatial Features:** When building ML models that require spatial features (e.g., "how many orders happened within this 500-meter hexagonal bin in the last hour").
* **Fleet Management and Tracking:** Tracking vehicles across global grids with consistent distance boundaries.

### Implementing H3 in a Database (e.g., PostgreSQL)

**Do you need Postgres (or a database like it) to use H3? No — not strictly.** H3 at its core is just a library that converts `(lat, lng, resolution)` into a 64-bit integer cell ID. That integer is the only artifact H3 produces, so storing and querying it is a generic "store an integer, look it up" problem — it works with almost any datastore: DynamoDB (cell ID as partition key), Cassandra, Redis, Elasticsearch (`term` filter on a `long`/`keyword` field), BigQuery (native H3 UDFs), or even an in-memory hash map. Postgres is a popular choice mainly because of the **`h3-pg`** extension, which exposes native SQL functions for H3 instead of requiring every computation to happen in application code.

#### 1. Install the extension

```sql
-- https://github.com/zachasme/h3-pg
CREATE EXTENSION h3;
CREATE EXTENSION h3_postgis;  -- optional, if you also use PostGIS geometry types
```

#### 2. Store the cell ID as a column

```sql
ALTER TABLE businesses ADD COLUMN h3_index h3index;  -- h3-pg's native type, backed by a bigint

UPDATE businesses
SET h3_index = h3_lat_lng_to_cell(POINT(lng, lat), 9);  -- resolution 9 ≈ 0.1 km² hexagons
```
Keep it in sync going forward with a trigger, or as part of whatever pipeline already re-reads business state on change (e.g., a CDC indexer step).

#### 3. Index it

```sql
CREATE INDEX idx_businesses_h3 ON businesses USING btree (h3_index);
```
This is a plain **equality** index. Important nuance: unlike S2/geohash, **H3 cell IDs are not laid out on a space-filling curve**, so numerically adjacent integers aren't necessarily spatially adjacent. A `BETWEEN` range scan on `h3_index` doesn't give you "nearby" the way it can with S2 — you always query by **exact membership in a precomputed set of cells**, not a numeric range.

#### 4. The "find nearby" query pattern

```sql
-- App/service layer: pick a resolution matching the search radius, then expand outward
SELECT h3_grid_disk(h3_lat_lng_to_cell(POINT(:lng, :lat), 9), 2);
-- center cell + 2 rings of neighbors → an array of h3index values covering roughly the search radius

SELECT * FROM businesses
WHERE h3_index = ANY(:cell_array)   -- fast B-tree/hash lookup, one IN-list
  AND <other filters>;
-- then apply an exact Haversine/PostGIS distance filter on this small candidate set
-- to trim the hexagon-shaped overcoverage down to a precise circle
```
That two-step shape — a cheap cell-membership filter to shrink the candidate set, then exact distance math only on the survivors — is the same pattern geohash and S2 use: push the expensive geometry into a cheap, precomputed filter, and reserve exact math for a small final set.

#### 5. Useful `h3-pg` functions beyond the basics

* `h3_grid_ring(cell, k)` — just the k-th ring, not the filled disk (useful for "between 1 km and 2 km away" bands).
* `h3_polygon_to_cells(geometry, resolution)` — tile an arbitrary polygon (e.g., a delivery zone) into H3 cells.
* `h3_cell_to_parent(cell, coarser_resolution)` — roll up to a coarser hex for aggregation/heatmaps (the multi-resolution hierarchy pro above).
* Consider storing **multiple resolution columns** (`h3_res7`, `h3_res9`) if you need both fine-grained point lookups and coarse aggregation without recomputing on every query.

**When Postgres is the right call vs. not:** if the underlying data is already relational, adding an `h3_index` column plus a B-tree index is the lowest-friction option — no new infrastructure. At a scale where geo-serving needs to be decoupled from OLTP writes entirely, the same cell ID gets computed at index time and stored as a filterable field in whatever dedicated search/serving layer is used instead (e.g., Elasticsearch) — same idea, different system.

---

### Applications and Services That Use Uber's H3

* **Uber:** Used extensively across core marketplace systems, surge pricing calculations, supply-demand forecasting, and fraud detection.
* **Kepler.gl:** Uber's open-source geospatial visualization tool, which natively supports rendering massive H3 datasets out of the box.
* **Foursquare & Location Analytics Platforms:** Used for spatial indexing, venue aggregation, and foot traffic analysis.
* **Autonomous Vehicle Systems:** Employed by various AV companies for semantic environment mapping and routing grids.