# Chapter 35: Design Google Maps / Proximity Service

> **Phase 7 — Advanced Design Problems** · Estimated Time: 3 hours · Difficulty: 🔴 Hard · Type: 🏗️ DESIGN

[← Previous: Ticket Booking](./chapter-34-ticket-booking.md) · [Next: E-Commerce →](./chapter-36-ecommerce.md)

---

## Why This Problem Matters

Tests geospatial data structures (QuadTree, GeoHash, R-tree) and map rendering at scale. The "find nearby restaurants" variation is especially common.

---

## Requirements

| Requirement | Target |
|-------------|--------|
| **POI (Points of Interest)** | 200M businesses/locations |
| **Search QPS** | 100K "nearby" queries/sec |
| **Latency** | < 200ms for proximity search |
| **Radius** | Configurable (500m to 50km) |

---

## Core Design: Proximity Search

```
"Find restaurants within 2km of my location"

OPTION 1: GeoHash (Recommended for interviews)
  Encode location → string: (37.77, -122.41) → "9q8yy"
  Query: Get all POIs with geohash prefix "9q8yy" + neighbors

  Redis:
    GEOADD places (lng) (lat) "restaurant_123"
    GEORADIUS places (lng) (lat) 2 km ASC COUNT 20

OPTION 2: QuadTree (In-memory spatial index)
  Good for: static POI data (doesn't change frequently)
  200M POIs in a QuadTree fits in ~5 GB RAM
  Rebuild nightly from database

  ┌─────────────────────────────────────────┐
  │            World (root node)             │
  │  ┌──────────┬──────────┐               │
  │  │ NW       │ NE       │               │
  │  │ ┌──┬──┐ │          │               │
  │  │ │NW│NE│ │          │  Keep splitting│
  │  │ │──┼──│ │          │  until each    │
  │  │ │SW│SE│ │          │  leaf has      │
  │  │ └──┴──┘ │          │  ≤ 100 POIs   │
  │  ├──────────┼──────────┤               │
  │  │ SW       │ SE       │               │
  │  └──────────┴──────────┘               │
  └─────────────────────────────────────────┘
```

### Architecture

```
┌────────┐    ┌──────────┐    ┌──────────────────┐
│ Client  │───▶│ LB / API  │───▶│ Location Service  │
│ (app)   │    │ Gateway   │    │                   │
└────────┘    └──────────┘    │ 1. Get user coords │
                               │ 2. Query nearby    │
                               │    POIs (Redis     │
                               │    GEORADIUS or    │
                               │    QuadTree)       │
                               │ 3. Fetch POI       │
                               │    details from DB │
                               └─────────┬─────────┘
                                         │
                              ┌──────────┼──────────┐
                              ▼          ▼          ▼
                        ┌─────────┐ ┌────────┐ ┌────────┐
                        │ Redis    │ │ POI DB │ │ Map    │
                        │ (GeoIdx) │ │(MySQL) │ │ Tiles  │
                        └─────────┘ └────────┘ │ (S3+CDN│
                                                └────────┘
```

### Map Tile Rendering

```
Maps are served as pre-rendered IMAGE TILES:

Zoom level 0: 1 tile (whole world)
Zoom level 1: 4 tiles
Zoom level 2: 16 tiles
...
Zoom level 20: ~1 trillion tiles (street level)

Each tile: 256×256 px PNG
Pre-rendered and stored in S3, served via CDN.

When user pans/zooms: client requests specific tiles by (zoom, x, y)
  GET /tiles/{zoom}/{x}/{y}.png → CDN serves cached tile

Example:
  Zoom 15, tile (16384, 10923) → San Francisco downtown
  
Pre-rendering billions of tiles is a massive batch job.
Only render tiles that contain data (skip ocean tiles).
```

---

## Key Trade-offs

| Decision | Choice | Why |
|----------|--------|-----|
| Geospatial index | Redis GEORADIUS or QuadTree | Redis: simple, built-in. QuadTree: better for complex queries with filtering. |
| Map rendering | Pre-rendered tiles (S3+CDN) | Real-time rendering is too expensive. Tiles are cacheable and CDN-friendly. |
| POI updates | Async (CDC → rebuild index periodically) | POIs don't change frequently. Eventual consistency is fine. |

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | How do you find nearby restaurants efficiently? | GeoHash (encode location → string prefix matching) or Redis GEORADIUS. Alternatively, in-memory QuadTree for static POI data. |
| 2 | How are map tiles served? | Pre-rendered as 256×256 pixel images, stored in S3, served via CDN. Client requests tiles by (zoom, x, y) coordinates. |
| 3 | GeoHash vs QuadTree? | GeoHash: simpler, works with Redis. QuadTree: adapts to density, better for complex spatial queries with filtering. |
| 4 | How many tiles at zoom level 20? | ~1 trillion. But most are ocean/empty and not rendered. Only tiles with data are stored. |

---

[← Previous: Ticket Booking](./chapter-34-ticket-booking.md) · [Next: E-Commerce →](./chapter-36-ecommerce.md)
