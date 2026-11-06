# Google's S2: A Comprehensive Guide
## Computational Geometry and Spatial Indexing on a Sphere

### Table of Contents
1. [Introduction](#introduction)
2. [How It Works (Core Concept)](#how-it-works-core-concept)
3. [Why Not Just Use Geohash?](#why-not-just-use-geohash)
4. [Pros and Cons](#pros-and-cons)
   - [Pros](#pros)
   - [Cons](#cons)
5. [When It Should Be Used](#when-it-should-be-used)
6. [Implementing S2 in a Database (e.g., PostgreSQL)](#implementing-s2-in-a-database-eg-postgresql)
7. [Which Databases Support S2 Well?](#which-databases-support-s2-well)
8. [Applications and Services That Use Google's S2](#applications-and-services-that-use-googles-s2)

---

### Introduction

**Google's S2** (named after the mathematical notation $S^2$ for the surface of a unit sphere) is an open-source computational geometry library developed by Google. Its primary purpose is to manipulate geometric shapes and index spatial data directly on a spherical surface (the Earth) rather than flattening it onto a 2D plane. 

---

### How It Works (Core Concept)

Instead of traditional latitude-longitude pairs or flat map projections (like Mercator, which distorts polar regions and breaks at the 180th meridian), S2 maps the Earth onto a **cube**, and then projects the cube's faces onto a sphere. 

The surface of each face is recursively divided into a quadtree hierarchy of **S2 cells** (up to 30 levels deep). These cells are laid out sequentially using a **Hilbert space-filling curve**. This converts a multi-dimensional spatial location (latitude and longitude) into a single 64-bit integer (`S2CellId`). 

---

### Why Not Just Use Geohash?

Geohash works beautifully for a single city, but it starts to hurt across the whole globe, because it treats latitude and longitude as a flat rectangle and the earth isn't flat. A degree of longitude is about 111 kilometers at the equator and shrinks toward zero at the poles, so one geohash cell is a fat square near the equator and a thin sliver up north. At planet scale you want cells of roughly equal area everywhere, and an ordering that respects distance on a sphere instead of a rectangle.

Google's S2 is the spherical cousin of geohash. It wraps the globe in a cube and projects onto the six faces, which lets it carve out cells that stay roughly the same size anywhere on earth, each with a 64-bit hierarchical ID you can truncate to get a parent cell. Because S2 actually understands the sphere, it handles cases a flat grid mangles, like a polygon that crosses the antimeridian (the 180-degree longitude line where coordinates wrap from +180 to -180). It's the cell system underneath MongoDB's 2dsphere index.

---

### Pros and Cons

#### Pros
* **No Projection Distortions or Singularities:** Because it works on a sphere, it handles the North and South poles seamlessly and avoids the massive scale distortions found near the poles in flat 2D maps.
* **Blazing-Fast Spatial Indexing:** By converting 2D coordinates into 1D Hilbert curve integer IDs (`uint64`), spatial range queries translate into simple 1D range lookups. This means they can be indexed efficiently in standard database B-Trees (like MySQL, PostgreSQL, or DynamoDB) without needing specialized GIS extensions.
* **Hierarchical Structure:** Cells are nested cleanly (children fit perfectly inside parent cells), allowing you to aggregate or expand geographic data dynamically at different zoom levels or resolutions.
* **Mathematical Robustness:** S2 uses exact geometric predicates and conservative error bounds to prevent floating-point rounding errors from crashing polygon intersections, point-in-polygon tests, or containment calculations.

#### Cons
* **Sphere vs. Ellipsoid Trade-off:** S2 models the Earth as a *sphere*, not an *ellipsoid* (like WGS84). While the Earth is much closer to a sphere than a flat plane (max distortion is ~0.56%), applications requiring hyper-precise sub-centimeter surveying, geodesy, or legal land-boundary mapping often require true ellipsoidal models (such as GeographicLib).
* **Steep Learning Curve:** Concepts like Hilbert curves, covering algorithms (`S2RegionCoverer`), and vector-based math rather than standard lat-long pairs can be challenging for developers unfamiliar with computational geometry.
* **Language Support Discrepancies:** The reference implementation is written in **C++**. While ports exist for Go, Java, and Python, the Java and Python ports historically lag behind the C++ version in feature completeness and ongoing performance optimizations.

---

### When It Should Be Used

* **Proximity and "Find Nearby" Services:** Ideal for finding things "within X miles of my current location" (e.g., drivers, restaurants, or stores) at massive global scales.
* **Distributed Spatial Databases:** When you need to shard or partition geographic data across a distributed cluster (like Bigtable, Cassandra, or DynamoDB) using simple, sortable 64-bit keys.
* **Large-Scale Polygon Containment:** When you need to quickly check if millions of GPS points fall inside complex geographic boundaries (such as checking if a delivery truck is inside a specific delivery zone or geofence).
* **Global Datasets Spanning Multiple Hemispheres:** Essential if your system cannot afford to break or distort at the international date line or the poles.

---

### Implementing S2 in a Database (e.g., PostgreSQL)

**Do you need Postgres (or a database like it) to use S2? No — not strictly.** An S2 cell ID is just a 64-bit integer, so any datastore that can store and index an integer works: BigQuery, Bigtable, Cassandra, Redis, Elasticsearch, and so on. The important difference from a grid system like H3: **S2 cell IDs are laid out on a Hilbert space-filling curve**, so numerically adjacent IDs *are* spatially adjacent. That's what makes S2 support true **range queries** (`BETWEEN`) on an ordinary B-tree, rather than only exact-membership lookups — the "Blazing-Fast Spatial Indexing" pro listed above.

#### 1. No equally mainstream native extension exists (worth being honest about)

Unlike H3, which has the well-established `h3-pg` extension, there isn't an equally standard, widely-adopted S2 extension for Postgres. The common real-world pattern is to compute S2 cell IDs and coverings in the **application/service layer** using Google's official S2 library (the C++ reference, with ports for Go — `golang/geo` — Java, and Python — `s2sphere`), then store just the resulting integer in Postgres. By contrast, **BigQuery GIS** (see Applications below) bakes S2 directly into SQL via functions like `S2_CELLIDFROMPOINT` — that's a case where the query engine itself ships S2 support, which Postgres doesn't natively have.

#### 2. Store the cell ID as a signed `BIGINT`

```sql
ALTER TABLE businesses ADD COLUMN s2_cell_id BIGINT;
```
S2 cell IDs are natively `uint64`. Postgres's `BIGINT` is signed 64-bit, so storing the raw bit pattern makes some valid cell IDs appear negative. That's fine — it's the same thing BigQuery GIS itself does (`INT64` return type) — as long as every comparison and sort consistently uses signed semantics rather than assuming an unsigned interpretation.

#### 3. Index it as a plain B-tree

```sql
CREATE INDEX idx_businesses_s2 ON businesses USING btree (s2_cell_id);
```

#### 4. Query using ranges, not an IN-list

Compute a region covering in the app layer (`S2RegionCoverer`), which returns a small set of cells — each expandable to a contiguous `[range_min, range_max]` on the Hilbert curve:
```sql
SELECT * FROM businesses
WHERE (s2_cell_id BETWEEN :range1_min AND :range1_max)
   OR (s2_cell_id BETWEEN :range2_min AND :range2_max)
   -- one OR per range in the covering, typically just a handful
;
```
Then apply exact Haversine/PostGIS distance filtering on the (small) result set — the same two-step "cheap filter, then exact math" pattern used by geohash and H3.

#### 5. Populating it

Done entirely in app code at write time (or as part of a CDC indexer step): `S2CellId::FromLatLng(S2LatLng::FromDegrees(lat, lng)).parent(level).id()`, then `UPDATE`/`INSERT` the resulting integer. There's no SQL-native function to lean on here the way `h3_lat_lng_to_cell(...)` works for H3 — the computation happens client-side, in whichever language's S2 binding the service already uses.

**When Postgres is the right call vs. not:** if the data already lives in Postgres, adding an `s2_cell_id` column and a B-tree index is cheap and gets you real range-query performance with no new infrastructure. At a scale where geo-serving needs to be decoupled from OLTP writes entirely, the same cell ID computation happens at index time and the result is stored as a filterable/sortable field in whatever dedicated search or serving layer is used instead — same idea, different system.

---

### Which Databases Support S2 Well?

**Native support — S2 is built directly into the engine, not bolted on:**

| Database | How |
|---|---|
| **MongoDB** (`2dsphere` index) | The server internally vendors Google's S2 library and uses S2 cells to index GeoJSON data — part of why `2dsphere` handles spherical geometry correctly, unlike the older, flat geohash-based legacy `2d` index. |
| **Google BigQuery GIS** | Exposes S2 functions directly in SQL — `S2_CELLIDFROMPOINT`, `ST_COVERINGCELLIDS`, etc. — so the query engine itself ships S2 support and you never touch the library directly. |

*(CockroachDB's spatial indexing is sometimes described as S2-derived, but the exact internal mechanism isn't independently confirmed here — worth verifying directly against their docs rather than taking it as given.)*

**Not native, but a very good fit — any datastore built around an ordered/sorted key space benefits from S2's Hilbert-curve property**, because a region covering becomes a handful of contiguous key ranges rather than requiring any geo-aware query logic at all:
* **Bigtable, Cassandra / ScyllaDB, DynamoDB** — row/partition key = S2 cell ID; a region query becomes a range scan.
* **Redis** — sorted sets (`ZRANGEBYSCORE`) over S2 cell IDs work the same way.
* **PostgreSQL** (and similar RDBMSs) — exactly the pattern in the section above: `BIGINT` + B-tree + `BETWEEN`.

**Poor fit / not where S2 adds value:**
* **Elasticsearch / OpenSearch** already has its own purpose-built `geo_point` (BKD-tree) index for the distance predicate itself — S2 is only useful there for *routing/partitioning* across shards, not as a replacement for the index.
* **PostGIS** is built around GiST/R-tree, a fundamentally different indexing strategy — S2 would be redundant rather than complementary.

---

### Applications and Services That Use Google's S2

* **Google Maps & Google Earth:** Uses S2 internally for indexing map features, routing, and spatial lookups.
* **Google BigQuery GIS:** Powers spatial indexing and geometric functions for massive data warehousing queries.
* **Pokémon GO:** Famously utilized S2 cells (specifically Level 10 and Level 17 cells) to manage game mechanics, spawn points, weather tracking, and dividing the real world into interactive gameplay areas.
* **Uber & Foursquare (and similar logistics/ride-sharing platforms):** Heavily inspired or directly utilized S2 (or similar discrete global grid systems derived from it) for driver-rider matching, surge pricing zones, and geofencing.
* **KnowWhereGraph:** Academic and semantic web projects that use S2 as a Discrete Global Grid System (DGGS) to link multi-source geospatial data.