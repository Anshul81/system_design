# Chapter 30: Design Uber / Ride Sharing

> **Phase 6 — Classic Design Problems** · Estimated Time: 4 hours · Difficulty: 🔴 Hard · Type: 🏗️ DESIGN

[← Previous: Instagram](./chapter-29-instagram-photo-sharing.md) · [Next: Dropbox / Google Drive →](./chapter-31-dropbox-google-drive.md)

---

## Why This Problem Matters

Uber introduces geospatial indexing, real-time location tracking, and matching algorithms — concepts not tested by social media designs. The core challenge is efficiently finding nearby drivers and handling real-time location updates from millions of drivers simultaneously.

---

## Step 1: Requirements (5 min)

### Functional Requirements

| Feature | Description |
|---------|-------------|
| **Request a ride** | Rider specifies pickup + destination |
| **Match with driver** | System finds nearest available driver |
| **Real-time tracking** | Live location during ride |
| **ETA estimation** | Estimated time of arrival |
| **Trip history** | Past rides with details |
| **Payments** | Automatic fare calculation and charging |

### Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| **DAU** | 100M riders, 5M drivers |
| **Active drivers** | 2M online at peak |
| **Rides/day** | 20M |
| **Location updates** | Every 3 seconds from active drivers |
| **Match latency** | < 5 seconds |
| **Availability** | 99.99% |

---

## Step 2: Estimation (3 min)

```
LOCATION UPDATES:
  2M active drivers × update every 3 sec = 667K updates/sec
  Peak: ~1M location updates/sec

RIDE REQUESTS:
  20M rides/day ÷ 86,400 ≈ 230 requests/sec
  Peak: ~500/sec

STORAGE:
  Location updates: 1M/sec × 100 bytes = 100 MB/sec → 8.6 TB/day
  (Historical data for analytics, not real-time queries)
  
  Trip data: 20M trips/day × 2KB = 40 GB/day
```

---

## Step 3: High-Level Design (10 min)

### Architecture

```
┌──────────────┐  GPS every 3s  ┌──────────────────┐
│ Driver App    │══════════════▶│ Location Service   │
│               │   WebSocket   │ (real-time driver  │
└──────────────┘               │  positions)        │
                                └────────┬───────────┘
                                         │
┌──────────────┐  ride request  ┌────────▼───────────┐
│ Rider App     │──────────────▶│ Matching Service    │
│               │◀──────────────│ (find nearest       │
│               │  match found  │  available driver)  │
└──────────────┘               └────────┬───────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    ▼                    ▼                    ▼
             ┌──────────┐        ┌──────────┐        ┌──────────┐
             │ Trip Svc  │        │ Payment  │        │ ETA Svc  │
             │           │        │ Svc      │        │          │
             └──────────┘        └──────────┘        └──────────┘
```

### API Design

```
POST /api/v1/rides/request
  Request:  { "rider_id": "u_123",
              "pickup": { "lat": 37.7749, "lng": -122.4194 },
              "destination": { "lat": 37.3382, "lng": -121.8863 },
              "ride_type": "UberX" }
  Response: { "ride_id": "ride_456",
              "status": "matching",
              "estimated_wait": "3 min" }

POST /api/v1/rides/{ride_id}/accept       (driver)
POST /api/v1/rides/{ride_id}/start        (driver: pickup done)
POST /api/v1/rides/{ride_id}/complete      (driver: arrived at destination)
POST /api/v1/rides/{ride_id}/cancel       (rider or driver)

GET /api/v1/rides/{ride_id}
  Response: { "ride_id": "ride_456",
              "status": "in_progress",
              "driver": { "id": "d_789", "name": "Alex", "rating": 4.8,
                          "vehicle": "Toyota Camry, ABC 1234",
                          "location": { "lat": 37.775, "lng": -122.418 } },
              "eta": { "pickup": "2 min", "destination": "25 min" },
              "fare_estimate": { "min": 18.50, "max": 24.00 } }

PUT /api/v1/drivers/{driver_id}/location  (high-frequency, via WebSocket)
  Request:  { "lat": 37.7752, "lng": -122.4183, "heading": 180,
              "speed": 35, "timestamp": 1695600000 }

GET /api/v1/drivers/{driver_id}/status
  Response: { "status": "available" | "on_trip" | "offline" }
```

---

## Step 4: Detailed Design (20 min)

### Deep Dive 1: Geospatial Indexing (The Core Challenge)

```
HOW TO FIND NEARBY DRIVERS EFFICIENTLY?

Option 1: Brute Force
  For each ride request, scan ALL 2M active drivers
  and compute distance to each. WAY TOO SLOW.

Option 2: GeoHash Grid (Recommended) ✅
  Divide the world into grid cells using geohash encoding.
  Each cell has a fixed-size string ID.

  ┌──────┬──────┬──────┐
  │ 9q8y │ 9q8z │ 9q9p │   Geohash precision 4:
  │      │      │      │   Each cell ≈ 20km × 20km
  ├──────┼──────┼──────┤
  │ 9q8v │ 9q8w │ 9q8x │   Precision 6:
  │  🚗  │  🚗  │      │   Each cell ≈ 600m × 600m
  │  🚗  │      │      │
  ├──────┼──────┼──────┤   Driver at (37.7749, -122.4194)
  │ 9q8s │ 9q8t │ 9q8u │   → GeoHash: "9q8yyk"
  │      │  📍  │      │
  │      │(rider)│      │   Find nearby drivers:
  └──────┴──────┴──────┘   Query Redis for drivers in
                            "9q8t" + neighboring cells

  Redis storage:
    Key: geohash:9q8v → Set of driver_ids
    Key: geohash:9q8w → Set of driver_ids

  When driver moves:
    1. Compute new geohash
    2. If changed: SREM from old cell, SADD to new cell
    3. Update driver's current location

  Finding nearby drivers:
    1. Compute rider's geohash
    2. Get drivers in that cell + 8 neighboring cells
    3. Calculate exact distance, sort by nearest
    4. Time: O(drivers_in_9_cells) — typically 10-50 drivers


Option 3: QuadTree
  Recursively divide space into 4 quadrants.
  Subdivide only when a cell has too many drivers.

  ┌─────────────────┬─────────────────┐
  │                  │                  │
  │                  │    ┌─────┬─────┐│
  │                  │    │ 🚗  │     ││
  │                  │    │ 🚗  │ 🚗  ││
  │                  │    ├─────┼─────┤│
  │                  │    │     │ 🚗  ││
  │                  │    │     │     ││
  │                  │    └─────┴─────┘│
  ├─────────────────┼─────────────────┤
  │    🚗           │                  │
  │                  │                  │
  │                  │                  │
  └─────────────────┴─────────────────┘

  Pro: Adapts to density (more cells in busy areas)
  Con: Harder to update with moving objects (need to rebuild)
  → Better for static POI (restaurants), less ideal for moving drivers


RECOMMENDATION FOR INTERVIEWS:
  GeoHash + Redis is the simplest, most explainable approach.
  Mention QuadTree as an alternative and explain the trade-off.
```

### Deep Dive 2: Ride Matching Algorithm

```
When rider requests a ride:

1. Find rider's geohash cell
2. Query Redis for drivers in cell + 8 neighboring cells
3. Filter: only available drivers (status = 'available')
4. Sort by:
   - Distance (nearest first)
   - ETA (considering traffic, not just straight-line distance)
   - Driver rating
   - Driver acceptance rate
5. Send ride offer to best matching driver
6. Driver has 15 seconds to accept
7. If declined → offer to next driver
8. If accepted → create trip, notify rider

Matching timeout: 30 seconds total
If no driver found → expand search radius (query more geohash cells)

DETAILED MATCHING FLOW:

┌────────┐  ride request  ┌──────────────┐
│ Rider   │──────────────▶│ Matching Svc  │
│ App     │               │               │
└────────┘               │ 1. Get nearby │
                          │    drivers    │
                          │    (GeoHash)  │
                          │               │
                          │ 2. Filter     │
                          │    available  │
                          │               │
                          │ 3. Rank by    │
                          │    distance,  │
                          │    ETA, rating│
                          │               │
                          │ 4. Send offer │──▶ 🚗 Driver A
                          │    to top     │    (15s timeout)
                          │    driver     │
                          │               │    ❌ Declined
                          │ 5. Try next   │──▶ 🚗 Driver B
                          │               │    (15s timeout)
                          │               │    ✅ Accepted!
                          └──────┬───────┘
                                 │
                          ┌──────▼───────┐
                          │ Create Trip   │
                          │ Notify Rider  │
                          │ Start Tracking│
                          └──────────────┘

SURGE PRICING:
  When demand > supply in a geohash region:
    demand_count = ride_requests_last_5min in region
    supply_count = available_drivers in region
    surge_multiplier = demand_count / supply_count
    
    If surge > 1.5x → apply surge pricing
    Surge incentivizes more drivers to come to the area.
```

### Deep Dive 3: Real-time Location Tracking During Ride

```
During active ride:

Driver App ── GPS every 3s ──▶ Location Service ──▶ Redis (current pos)
                                       │
                                       ├──▶ Rider App (live map update via WebSocket)
                                       │
                                       └──▶ Kafka (location history for analytics)

Location stream goes to:
  1. Redis (real-time: current driver position)
     Key: driver:d_789:location → { lat, lng, heading, speed, ts }
     TTL: 30 seconds (auto-cleanup if driver goes offline)
     
  2. WebSocket push to Rider (live tracking on map)
     Rider's app connected via WebSocket to the Location Service
     Every 3 seconds: push new driver position to rider's map
     
  3. Kafka → Cassandra (historical for ETA model training, analytics)
     All location points stored for route analytics
     Used to train ML models for better ETA prediction

HANDLING 1M LOCATION UPDATES/SEC:
  Each update: ~100 bytes (lat, lng, heading, speed, ts, driver_id)
  Redis: handles 100K ops/sec per instance → need 10 Redis instances
  Shard by: hash(driver_id) % 10
  
  WebSocket servers: each handles 50K connections
  2M active drivers + riders on trips → need ~80 WS servers

ETA CALCULATION:
  Not straight-line distance! Use real road network.
  
  Simple approach:
    Pre-compute routing table for major city zones
    ETA = route_distance / average_speed_for_time_of_day
    
  Better approach (what Uber does):
    ML model trained on historical trip data
    Features: distance, time of day, traffic, weather, events
    Output: predicted ETA with confidence interval
```

### Deep Dive 4: Trip State Machine & Payment

```
TRIP STATE MACHINE:

  ┌──────────┐    driver    ┌───────────┐    driver    ┌──────────┐
  │ MATCHING  │───accepts──▶│ ACCEPTED   │───arrives──▶│ ARRIVED  │
  └─────┬────┘             └───────────┘             └────┬─────┘
        │                                                   │
        │ timeout/                                    driver starts
        │ no driver                                   ride
        ▼                                                   │
  ┌──────────┐                                        ┌────▼─────┐
  │ CANCELLED │                                        │IN_PROGRESS│
  └──────────┘                                        └────┬─────┘
                                                           │
                                                     driver completes
                                                           │
                                                     ┌────▼─────┐
                                                     │ COMPLETED │
                                                     └────┬─────┘
                                                           │
                                                     calculate fare
                                                     charge payment
                                                           │
                                                     ┌────▼─────┐
                                                     │  PAID     │
                                                     └──────────┘

FARE CALCULATION:
  fare = base_fare 
       + (distance_km × per_km_rate) 
       + (duration_min × per_min_rate) 
       + surge_multiplier
       - promotions/discounts
  
  Minimum fare: $5.00
  Fare calculated from actual GPS route (not straight line)

PAYMENT FLOW:
  1. Trip completes → calculate fare
  2. Payment Service → charge rider's saved payment method
  3. If charge fails → retry 3× → flag for manual resolution
  4. If charge succeeds → hold funds
  5. Settlement: pay driver (fare - Uber commission) in weekly batch
```

---

## Complete Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                     UBER ARCHITECTURE                                │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────┐  GPS/WS  ┌─────────────────────────────────────────┐ │
│  │Driver App │═════════▶│  WebSocket Gateway (80+ servers)        │ │
│  └──────────┘          └──────────────┬──────────────────────────┘ │
│                                        │                            │
│  ┌──────────┐  HTTPS   ┌──────────────▼──────────────────────────┐ │
│  │Rider App  │════════▶│  API Gateway (auth, rate limit, routing) │ │
│  └──────────┘          └──────────────┬──────────────────────────┘ │
│                                        │                            │
│       ┌────────────────────────────────┼──────────────────┐        │
│       ▼              ▼                 ▼                  ▼        │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │Location   │  │Matching   │  │Trip Service   │  │ETA Service   │  │
│  │Service    │  │Service    │  │(state machine │  │(ML model)    │  │
│  │(1M upd/s) │  │(geohash + │  │ + history)    │  │              │  │
│  └─────┬────┘  │ ranking)  │  └───────┬──────┘  └──────────────┘  │
│        │       └──────────┘          │                             │
│  ┌─────▼────────────┐          ┌─────▼──────┐                     │
│  │Redis (10 shards)  │          │Trip DB     │                     │
│  │• driver locations │          │(MySQL)     │                     │
│  │• geohash indices  │          └────────────┘                     │
│  └──────────────────┘                                              │
│                                                                      │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌──────────────┐    │
│  │Payment Svc│  │Kafka       │  │Cassandra   │  │Notification  │    │
│  │(Stripe)   │  │(events)    │  │(location   │  │Service       │    │
│  └───────────┘  └───────────┘  │ history)   │  │(APNs/FCM)   │    │
│                                 └───────────┘  └──────────────┘    │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Key Trade-offs

| Decision | Choice | Alternative | Why |
|----------|--------|-------------|-----|
| Geospatial index | GeoHash + Redis | QuadTree, PostGIS | Fast lookups, easy updates for moving objects, simple to implement. QuadTree better for static POI. |
| Driver locations | Redis (in-memory) | PostgreSQL + PostGIS | Sub-ms reads for 1M updates/sec. Ephemeral — no need to persist real-time positions. |
| Location history | Kafka → Cassandra | Write to MySQL | Write-heavy (1M/sec), time-series. Used for analytics and ML, not real-time queries. |
| Driver-rider transport | WebSocket | HTTP polling, SSE | Bidirectional real-time needed for live tracking and ride offers. |
| ETA calculation | ML model on historical data | Straight-line distance / speed | Real roads, traffic, and time-of-day matter. ML gives 90%+ accuracy vs 50% for straight-line. |
| Matching | Greedy nearest-first | Global optimization | Greedy is fast (< 5s). Global optimization is theoretically better but too slow for real-time. |

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | How do you efficiently find nearby drivers? | GeoHash: divide world into grid cells. Store driver → cell mapping in Redis. Query cell + 8 neighbors for nearby drivers. O(drivers_in_9_cells). |
| 2 | How often do drivers send location updates? | Every 3-5 seconds via WebSocket to the Location Service. Stored in Redis for real-time, Kafka → Cassandra for history. |
| 3 | GeoHash vs QuadTree for moving objects? | GeoHash is better for moving objects (drivers) — simple update (SREM + SADD in Redis). QuadTree is better for static points (POI, restaurants). |
| 4 | How does ride matching work? | Find nearby available drivers via GeoHash. Rank by distance, ETA, rating, acceptance rate. Send ride offer to best match. 15s to accept. If declined, try next. |
| 5 | Where are real-time driver locations stored? | Redis (in-memory), sharded by driver_id. Key: driver:{id}:location → {lat, lng, heading, speed}. Updated every 3 seconds. TTL: 30s. |
| 6 | How does the rider see real-time driver tracking? | Driver → Location Service → Redis → WebSocket push to Rider App. Updates every 3 seconds on rider's map. |
| 7 | How is ETA calculated? | ML model trained on historical trip data. Features: distance, time of day, traffic, weather. Not straight-line distance — uses real road network. |
| 8 | How does surge pricing work? | demand_in_region / supply_in_region = surge multiplier. If demand > 1.5× supply → surge pricing activates. Incentivizes more drivers to the area. |
| 9 | How does the fare get calculated? | base_fare + (distance × per_km_rate) + (time × per_min_rate) × surge_multiplier - discounts. Calculated from actual GPS route, not straight line. |
| 10 | How do you handle 1M location updates/sec? | Redis sharded by driver_id across 10+ instances. Each Redis handles ~100K ops/sec. WebSocket gateway with 80+ servers for persistent connections. |

---

[← Previous: Instagram](./chapter-29-instagram-photo-sharing.md) · [Next: Dropbox / Google Drive →](./chapter-31-dropbox-google-drive.md)
