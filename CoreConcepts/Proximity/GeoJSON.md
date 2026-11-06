# GeoJSON

## What it is

GeoJSON is a plain-text, JSON-based **format for encoding geographic data** — points, lines, polygons, and collections of them, each optionally carrying arbitrary key/value properties. It's defined by [RFC 7946](https://datatracker.ietf.org/doc/html/rfc7946) and is to spatial data what JSON itself is to generic data: not a database, not an index, just an interchange format everything can read and write.

```json
{
  "type": "Feature",
  "geometry": {
    "type": "Point",
    "coordinates": [-122.4194, 37.7749]
  },
  "properties": {
    "driver_id": "d_8831",
    "status": "available"
  }
}
```

Coordinates are always `[longitude, latitude]` — easy to get backwards, since most people say "lat/lng" out loud but GeoJSON stores `lng` first. This single footgun causes more real bugs than anything else about the format.

### The geometry types

| Type | Shape | Typical use |
|---|---|---|
| `Point` | Single `[lng, lat]` | A driver, a device, a store location |
| `LineString` | Ordered list of points | A route, a road segment, a GPS trail |
| `Polygon` | Closed ring(s) of points (first ring = outer boundary, extra rings = holes) | A delivery zone, a geofence, a country boundary |
| `MultiPoint` / `MultiLineString` / `MultiPolygon` | Collections of the above | A city made of several disjoint polygons, a transit line with branches |
| `GeometryCollection` | Mixed geometries | Rare — usually a sign the data model should be split into separate features |

### The wrapper types

- **`Feature`** — one geometry + a `properties` bag. This is the unit you almost always actually work with (a point *plus* the driver ID, a polygon *plus* the zone name).
- **`FeatureCollection`** — an array of `Feature`s. This is what most APIs return: "give me all the delivery zones" → one `FeatureCollection`.

## Why it exists / what problem it solves

Before GeoJSON, every GIS tool had its own binary or XML format (Shapefile, KML, GML, WKT/WKB) — none of them played nicely with web APIs or JavaScript. GeoJSON solves the **interchange** problem specifically:

- It's valid JSON, so every HTTP client, every language's standard library, every browser can parse it with zero extra dependencies.
- Mapping libraries (Leaflet, Mapbox GL, Google Maps, OpenLayers) accept it natively — `map.addLayer(geojsonData)` just works.
- It's human-readable, which matters a lot for debugging ("why is this polygon drawn wrong") compared to a binary WKB blob.

What it explicitly does **not** solve: indexing, querying, or storage. GeoJSON is the wire format you hand *to* a spatial index/database or *from* one — it's not a substitute for [[PostGIS]], Elasticsearch geo-queries, or a geohash-based key-value layout. A database stores geometry internally in a far more compact binary form (e.g., Postgres/PostGIS stores it as WKB internally) and only serializes to GeoJSON at the API boundary.

## When it shows up in a distributed system

GeoJSON appears at **boundaries between services or between a service and a client**, not inside a storage engine's internals:

- **API responses for map rendering** — a delivery-zone service returns `FeatureCollection` of polygons; the mobile/web client feeds that straight into a map SDK to draw the zones. No translation layer needed because the client library speaks GeoJSON already.
- **Geofence/zone configuration payloads** — an ops team draws a new delivery zone in a map UI, the UI exports it as a `Polygon`, and that JSON blob is what gets POSTed to the backend, stored (as PostGIS geometry, or just as a JSON column), and later re-served to other services that need "is this point inside any zone."
- **Event payloads in a pub/sub pipeline** — a location-update event (`{"type":"Point","coordinates":[...]}` plus metadata) published to Kafka/Kinesis and consumed by multiple downstream services (fraud detection, ETA calculation, analytics) that all need to agree on one geometry encoding without each team inventing its own lat/lng JSON shape.
- **Interop between heterogeneous systems** — when a routing engine, a mapping vendor, a GIS import/export tool, and your own services all need to exchange geometry, GeoJSON is the lowest-common-denominator format none of them have to custom-build a parser for.
- **Logging/debugging spatial data** — because it's human-readable JSON, it's what you paste into a browser-based GeoJSON viewer to sanity-check "is this polygon actually where I think it is" during an incident.

It's the wrong tool for: **high-frequency, high-volume internal data transfer**. A GPS ping every second from a million devices, serialized as verbose JSON with full key names repeated on every message, wastes bandwidth and CPU on parsing compared to a compact binary encoding (protobuf with two floats, or a packed geohash string). Use GeoJSON at the edges where human-readability and library interop matter; use something tighter on the hot internal path.

## How it's used, concretely, in a distributed architecture

### 1. As the API contract, not the storage format
A typical flow:

```
Client (map UI) ──POST GeoJSON Polygon──▶ Zone Service
                                               │
                                               ▼
                                   Store as PostGIS geometry
                                   (ST_GeomFromGeoJSON(...))
                                               │
Other services ◀──query "is point P in any zone"──┘
      │
      ▼
Zone Service responds with GeoJSON FeatureCollection
(ST_AsGeoJSON(...)) for the matching zones
```

The database never *is* GeoJSON internally — PostGIS has `ST_GeomFromGeoJSON()` and `ST_AsGeoJSON()` precisely because GeoJSON is the boundary encoding, converted to/from the database's native geometry type at the edges.

### 2. Validating before trusting it
Because GeoJSON is user/client-authored JSON, a service accepting it needs to validate:
- Winding order and ring closure for polygons (a malformed polygon with an unclosed ring, or rings in the wrong order, silently breaks containment checks in some parsers).
- Coordinate order (`[lng, lat]`, not `[lat, lng]`) — the most common integration bug between two teams.
- Coordinate bounds (`lng` in [-180, 180], `lat` in [-90, 90]) and self-intersection, especially when geometry is user-drawn (an ops tool, a delivery-zone editor) rather than machine-generated.

### 3. Size management for large collections
A `FeatureCollection` of thousands of complex polygons (e.g., every delivery zone in a country, each with hundreds of vertices) can become a multi-megabyte payload. Common mitigations:
- **Simplify geometry** (Douglas-Peucker / `ST_Simplify`) to reduce vertex count for zoom levels where full precision isn't visible anyway.
- **Tile it** — serve vector tiles (Mapbox Vector Tile format) instead of one giant GeoJSON blob, so the client only fetches the geometry for the map area currently in view.
- **Paginate `FeatureCollection` by region/bounding box** rather than returning the whole collection in one response.

### 4. Event-driven pipelines standardizing on GeoJSON geometry
In a system with many consumers of location events (ETA service, fraud/anomaly detection, analytics warehouse, live map for support agents), publishing each event's geometry as a GeoJSON `Point` (even nested inside a larger protobuf/Avro envelope as a JSON string, or as a small sub-message mirroring GeoJSON's shape) keeps every consumer's deserialization logic identical and avoids N different ad-hoc `{lat, lng}` shapes across teams.

## Key tradeoffs to call out in an interview

| Concern | GeoJSON's answer | Where it falls short |
|---|---|---|
| Interop / human readability | Plain JSON, every map library and language supports it natively | Verbose compared to binary encodings |
| Storage efficiency | N/A — not meant for storage | Don't store raw GeoJSON as your primary record if you need spatial indexing; convert to a database's native geometry type |
| Query/index support | None built in | Pair with [[PostGIS]], Elasticsearch geo, or a geohash/H3 key for actual querying |
| High-frequency internal transport | Works, but costly at scale | Prefer protobuf/compact binary for internal hot paths; keep GeoJSON at external/API boundaries |
| Validity of client-authored geometry | Well-specified by RFC 7946 (winding order, ring closure, coordinate order) | The spec doesn't stop a buggy client from sending invalid geometry — validate server-side |

## Where this fits among this repo's designs

As with [[PostGIS]], none of the current `HighLevelSystemDesign` docs are location-matching systems (ride-hailing, delivery dispatch, geofencing) yet, so there's no existing design in this repo that uses GeoJSON today. If such a design is added, GeoJSON is the natural answer for "what format does the zone-management API and the map-rendering client speak," while PostGIS (or a geohash-based store) remains the answer for "how is it actually indexed and queried server-side."
