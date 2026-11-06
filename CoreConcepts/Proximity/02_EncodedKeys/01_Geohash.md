# Comprehensive Guide to Geohash and Spatial Indexing in System Design

This document covers the fundamentals of Geohashes, how index calculations and neighbor searches work, the architectural trade-offs between PostGIS and Geohash, and how to evaluate them for a massive-scale application like Yelp.

## Table of Contents
- [1. What is a Geohash?](#1-what-is-a-geohash)
  - [Key Characteristics](#key-characteristics)
- [2. How Geohash is Calculated (With an Example)](#2-how-geohash-is-calculated-with-an-example)
  - [Step-by-Step Algorithm](#step-by-step-algorithm)
  - [Calculation Walkthrough](#calculation-walkthrough)
- [3. How a Geohash Index is Created in a Table](#3-how-a-geohash-index-is-created-in-a-table)
  - [Table Schema & Index Design](#table-schema--index-design)
  - [Querying Nearby Locations (The Boundary Problem & 9-Cell Lookup)](#querying-nearby-locations-the-boundary-problem--9-cell-lookup)
- [4. PostGIS vs. Geohash at Scale](#4-postgis-vs-geohash-at-scale)
  - [PostGIS (The Relational / Exact Approach)](#postgis-the-relational--exact-approach)
  - [Geohash / Grid Indexing (The Distributed Approach)](#geohash--grid-indexing-the-distributed-approach)
- [5. Architectural Decision for High Scale (Yelp-Scale Example)](#5-architectural-decision-for-high-scale-yelp-scale-example)
  - [Why Geohash Wins at This Scale](#why-geohash-wins-at-this-scale)

---

## 1. What is a Geohash?

A **Geohash** is a public-domain geocoding system that encodes a geographic location (latitude and longitude) into a short string of letters and digits. Invented by Gustavo Niemeyer in 2008, it flattens a two-dimensional spatial coordinate into a one-dimensional string by mapping Earth's surface onto a hierarchical grid using a space-filling curve (specifically, a **Z-order curve** or Morton curve).

### Key Characteristics

* **Hierarchical Spatial Indexing:** The longer the Geohash string, the more precise the location. If two locations share a common prefix (e.g., `dr5ru` and `dr5rz`), they are geographically close to each other.

* **Base32 Encoding:** Geohashes use a 32-character alphabet (`0-9`, `b-h`, `j-k`, `m-n`, `p-z`). The letters `a`, `i`, `l`, and `o` are intentionally excluded to avoid visual confusion with numbers (`1` and `0`).

* **Grid Precision Matrix:** Each character added to a Geohash divides the Earth's surface into smaller rectangular grid cells.

| Geohash Length | Cell Width $\times$ Height (Approximate) | Area Coverage | 
| ----- | ----- | ----- | 
| **1 char** | $5,000\text{ km} \times 5,000\text{ km}$ | Global quadrants | 
| **3 chars** | $156\text{ km} \times 156\text{ km}$ | Regional / Country subdivision | 
| **5 chars** | $4.9\text{ km} \times 4.9\text{ km}$ | City district / Neighborhood | 
| **7 chars** | $152\text{ m} \times 152\text{ m}$ | City block / Street segment | 
| **9 chars** | $4.7\text{ m} \times 1.2\text{ m}$ | Building entrance / Exact property | 

## 2. How Geohash is Calculated (With an Example)

The calculation process involves **binary space partitioning** of latitude and longitude ranges, **interleaving** the resulting bits, and **encoding** them into Base32.

### Step-by-Step Algorithm

1. **Initialize Ranges:**

   * Latitude range: $[-90, 90]$

   * Longitude range: $[-180, 180]$

2. **Iterative Binary Division:**

   * Take the midpoint of the current range. If the coordinate is greater than or equal to the midpoint, assign a bit of `1`; otherwise, assign `0`.

   * Update the range boundaries accordingly and repeat for the desired precision.

3. **Bit Interleaving:**

   * Interleave the longitude bits and latitude bits (typically starting with longitude: `Lon_bit1, Lat_bit1, Lon_bit2, Lat_bit2...`).

4. **Base32 Conversion:**

   * Group the interleaved bits into chunks of 5 bits and map each chunk to its corresponding Base32 character.

### Calculation Walkthrough

Let's calculate the Geohash for a location with coordinates:

* **Latitude:** $42.605$

* **Longitude:** $-5.603$

#### Step A: Longitude Partitioning (Target: $-5.603$, Range: $[-180, 180]$)

* Range $[-180, 180]$, midpoint $0$. Since $-5.603 < 0 \rightarrow$ bit `0`. New range: $[-180, 0]$

* Range $[-180, 0]$, midpoint $-90$. Since $-5.603 > -90 \rightarrow$ bit `1`. New range: $[-90, 0]$

* Range $[-90, 0]$, midpoint $-45$. Since $-5.603 > -45 \rightarrow$ bit `1`. New range: $[-45, 0]$

#### Step B: Latitude Partitioning (Target: $42.605$, Range: $[-90, 90]$)

* Range $[-90, 90]$, midpoint $0$. Since $42.605 > 0 \rightarrow$ bit `1`. New range: $[0, 90]$

* Range $[0, 90]$, midpoint $45$. Since $42.605 < 45 \rightarrow$ bit `0`. New range: $[0, 45]$

* Range $[0, 45]$, midpoint $22.5$. Since $42.605 > 22.5 \rightarrow$ bit `1`. New range: $[22.5, 45]$

#### Step C: Interleaving & Encoding

By interleaving the longitude bits (even positions) and latitude bits (odd positions) and grouping them into 5-bit sets, we get:

* First 5 bits: `11011` $\rightarrow$ Base32 character `e`

* Next 5 bits: `11110` $\rightarrow$ Base32 character `z`

* Next 5 bits: `10011` $\rightarrow$ Base32 character `s`

* Resulting Geohash: **`ezs42`**

## 3. How a Geohash Index is Created in a Table

Because a Geohash converts a 2D coordinate into a standard string, you do **not** need complex spatial extensions (like PostGIS) to index and query locations. You can index a Geohash using a standard **B-Tree index**.

### Table Schema & Index Design

```sql
CREATE TABLE venues (
    venue_id INT PRIMARY KEY,
    venue_name VARCHAR(100),
    latitude DECIMAL(9,6),
    longitude DECIMAL(9,6),
    geohash VARCHAR(12) NOT NULL
);

CREATE INDEX idx_venues_geohash ON venues(geohash);
```

### Querying Nearby Locations (The Boundary Problem & 9-Cell Lookup)

A strict prefix query (`LIKE 'ezs42%'`) only finds points inside that exact grid cell. If a user is near the edge of cell `ezs42`, venues in adjacent cells will be missed.

To solve this, production systems compute the **8 surrounding neighbors** plus the center cell ($O(1)$ time complexity using directional adjacency lookup tables) and query all 9 ranges:

```sql
SELECT venue_id, venue_name, latitude, longitude
FROM venues
WHERE geohash LIKE 'ezs42%'
   OR geohash LIKE 'ezs43%'
   OR geohash LIKE 'ezs48%'
   -- ... (up to 9 conditions covering the neighborhood)
```

## 4. PostGIS vs. Geohash at Scale

When facing a system design interview, choosing between PostGIS and Geohash depends strictly on architectural scale and distribution requirements.

### PostGIS (The Relational / Exact Approach)

* **Pros:** Zero boundary issues (`ST_DWithin`), rich geometric polygon support, simpler application-layer logic.

* **Cons:** Hard to scale horizontally across distributed nodes for massive write/read throughput.

### Geohash / Grid Indexing (The Distributed Approach)

* **Pros:** Infinite horizontal scalability via sharding, lightweight B-Tree indexes, highly performant string lookups.

* **Cons:** Application-layer complexity (9-cell neighborhood querying) and two-step filtering (bounding box query followed by exact Haversine distance filtering).

## 5. Architectural Decision for High Scale (Yelp-Scale Example)

Consider the following high-scale metrics:

* **Businesses:** 200M | **DAU:** 100M

* **Search QPS:** 5,800 avg / **17,400 peak**

* **Read-to-Write Ratio:** \~360 : 1

### Why Geohash Wins at This Scale

1. **Massive Fan-Out Reduction (20× Improvement):**

   * *Doc-ID Sharded Search:* Requires broadcasting every search query to all 25 shards $\rightarrow$ **435,000 shard-queries/sec** internally.

   * *Geo-Partitioned Search:* Routes queries only to relevant regional shards $\rightarrow$ **21,000 shard-queries/sec** (\~20× less internal load).

2. **In-Memory Geo Routing:** The raw geo data for 200M businesses is only **6.4 GB**, fitting easily into RAM on an API gateway or Redis caching layer to instantly map coordinates to the correct shard.

3. **Advanced Tip:** In an actual interview, mentioning **Uber's H3 Hexagonal Index** instead of rectangular Geohash earns bonus points, as hexagons eliminate spatial distortion and boundary artifacts while preserving distributed grid indexing benefits.