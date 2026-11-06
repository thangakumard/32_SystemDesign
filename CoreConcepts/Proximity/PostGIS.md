# PostGIS

## What it is

PostGIS is an extension to PostgreSQL that adds **spatial data types, spatial indexes, and spatial functions** directly into the relational database. Without it, Postgres only knows how to compare numbers and strings; PostGIS teaches it geometry — points, lines, polygons — and the operations that matter on them: distance, containment, intersection, nearest-neighbor.

```sql
CREATE EXTENSION postgis;

CREATE TABLE drivers (
  driver_id  BIGINT PRIMARY KEY,
  location   GEOGRAPHY(POINT, 4326)   -- lon/lat, SRID 4326 = WGS84
);

CREATE INDEX drivers_location_gix
  ON drivers USING GIST (location);

-- "find the 20 nearest drivers to this pickup point"
SELECT driver_id
FROM drivers
ORDER BY location <-> ST_SetSRID(ST_MakePoint(-122.42, 37.77), 4326)::geography
LIMIT 20;
```

Two data types matter in practice:

| Type | Model | Use when |
|---|---|---|
| `geometry` | Flat Cartesian plane (planar math) | Small areas, local projected coordinates (e.g. a city's UTM zone), fastest math |
| `geography` | Curved earth surface (great-circle math) | Anything crossing large distances / spanning the globe — correct but costlier than `geometry` |

The index that makes any of this fast is **GiST** (Generalized Search Tree) — a tree that indexes bounding boxes, so "give me everything near X" doesn't require scanning every row and computing distance by hand. This is the spatial equivalent of a B-tree: without it, every proximity query is `O(n)`.

## Why not just store lat/lng as two floats and compute distance in application code?

You can, for small datasets. It breaks down because:
- **No index speeds up "nearest to me" or "inside this polygon."** A B-tree on `lat` and a separate one on `lng` doesn't compose — you can't intersect two 1-D range scans to get an efficient 2-D proximity search. You'd scan the whole table and compute Haversine distance per row.
- **Geometry operations are easy to get wrong by hand** — polygon containment, intersection, and great-circle distance all have edge cases (antimeridian crossing, poles, self-intersecting polygons) that a spatial library has already solved.
- PostGIS gives you both the index (GiST) and the correct math (`ST_DWithin`, `ST_Contains`, `ST_Intersects`, `ST_Distance`) in the same query the rest of your transactional data lives in — no separate system to keep in sync for simple cases.

## When to reach for it in a distributed system

PostGIS is a natural fit when the spatial query needs to be **transactionally consistent with other relational data**, and the dataset is *geographically bounded enough* to shard sensibly. Concretely:

- **Ride-hailing / delivery dispatch** — "find available drivers within 3km of this pickup point, who aren't already assigned, ordered by ETA" — this is a spatial join against live, mutable driver state that also needs transactional guarantees (don't double-assign a driver).
- **Geofencing** — "is this device's current location inside any of these 50,000 delivery zones?" is `ST_Contains`/`ST_Within` over polygons.
- **Store/location finders** — "nearest 10 branches to this address," combined with normal relational filters (open now, has inventory).
- **Asset tracking with territory logic** — fleet management, field service routing, where location data lives next to ownership/scheduling data that's already relational.

It's the *wrong* tool when:
- The dataset is write-heavy at huge scale and read patterns are simple key-lookups by geohash prefix (then a NoSQL store with a geohash-based partition key, e.g. DynamoDB + geohashing, is cheaper to scale horizontally).
- You need true global, horizontally-partitioned spatial search across billions of points with no single-node bottleneck (then a purpose-built geospatial index service — Elasticsearch/OpenSearch geo-queries, or Uber's H3 hex-grid approach — scales out more naturally than a single Postgres primary).

## How it's used in a distributed architecture

PostGIS itself is **not distributed** — it's a single-node Postgres extension. Scaling it out follows the same playbook as scaling Postgres for anything else, with a few spatial-specific wrinkles:

### 1. Read replicas for read-heavy proximity queries
Nearest-driver / nearest-store queries are overwhelmingly reads. Fan them out to read replicas; route only the write path (driver location pings, new store records) to the primary. GiST indexes replicate like any other index — no special handling needed, but replica lag means a replica might serve a "nearest driver" result that's a few hundred milliseconds stale, which is usually fine for dispatch but worth flagging in the design.

### 2. Geo-sharding by region
Because most spatial queries ("nearest to me") are naturally bounded to a region, you can **partition by geography** rather than by a hash: one shard (or one Postgres cluster) per city/metro/country. This:
- Keeps each shard's working set and index small enough to stay fast and memory-resident.
- Lets you route a query to exactly one shard if you know which region the query originates from (reverse-geocode the query point to a region ID, then route).
- Creates a cross-shard problem at the boundary: a search radius that straddles two regions needs to fan out to both shards and merge results — unavoidable edge case, usually handled by padding the shard boundary or querying neighboring shards when the point is within some margin of the edge.

### 3. Hot-path writes avoid the spatial index entirely
High-frequency location updates (a driver's GPS ping every few seconds) are usually **not** written straight into the PostGIS-indexed table on every tick — that would thrash the GiST index with constant updates. Common pattern:
- Write current location to a fast in-memory store (Redis with geospatial commands `GEOADD`/`GEOSEARCH`, or just a key-value cache) for the "where is this driver right now" hot path.
- Periodically (or on significant movement) persist to the PostGIS-backed table, which serves durable/historical/analytical spatial queries, not the millisecond-level dispatch loop.
- This two-tier split — fast ephemeral geo-index for live lookups, PostGIS for durable/transactional spatial queries — is the most common pattern in practice; see [[GeoIndexing]] for the Redis/geohash side of this tradeoff.

### 4. Precomputed geo buckets (geohash / S2 / H3) as a sharding and pre-filter key
Even with PostGIS doing the precise `ST_DWithin` math, systems often add a coarse **geohash, Google S2 cell ID, or Uber H3 hex ID** column, indexed with a plain B-tree, as a cheap pre-filter: narrow the candidate set to "same geohash prefix" first (fast, coarse), then run the precise PostGIS distance/containment check only on that narrowed set (slow, exact). This is the same two-phase pattern as a bounding-box check before an expensive polygon-intersection test.

### 5. Caching for geofence/polygon lookups
Polygon containment checks (`ST_Contains`) are heavier than point-distance checks. If the set of polygons (delivery zones, service areas) changes rarely, cache the polygon set in application memory or Redis and do containment checks in-process, falling back to PostGIS only for the authoritative/complex cases (overlapping zones, zone priority resolution).

## Key tradeoffs to call out in an interview

| Concern | PostGIS's answer | Where it falls short at extreme scale |
|---|---|---|
| Correctness of spatial math | Mature, well-tested (`ST_*` functions, both planar and geodetic) | N/A — this is PostGIS's strength |
| Index performance | GiST index on bounding boxes, `<->` KNN operator for nearest-neighbor | Single-node; doesn't horizontally shard spatial queries for you |
| Consistency with relational data | Full ACID transactions, same query can join spatial + non-spatial columns | N/A — this is the main reason to choose it over a separate geo service |
| Horizontal write scale | Scales like Postgres — read replicas, or manual geo-sharding | No native multi-shard spatial query planner; cross-shard queries are hand-rolled |
| Global, planet-scale point density | Works fine up to tens/low hundreds of millions of rows per shard with good indexing | Billions of points with uniform global query load favor H3/S2 + a horizontally-native store |

## Where this fits among this repo's designs

None of the current `HighLevelSystemDesign` docs are location-matching systems (ride-hailing, delivery dispatch, geofencing) yet, so there's no existing design in this repo that uses PostGIS today. If a ride-hailing/delivery/"nearby X" design is added, this doc is the natural reference for the driver/location-matching component — in particular the geo-sharding and hot-path-write split described above.
