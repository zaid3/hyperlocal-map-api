# 🗺️ Hyperlocal Map API

> **An open-source geospatial REST API for discovering nearby business offers in real time — built with Supabase PostGIS and Node.js**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Built with Supabase](https://img.shields.io/badge/Database-Supabase%20PostGIS-green)](https://supabase.com)
[![Part of LocalPulse](https://img.shields.io/badge/Part%20of-LocalPulse%20AI-orange)](https://github.com/zaid3/local-ai-commerce)

---

## 🤔 Why I Built This

Most location APIs are built for finding *places*. I needed something different — an API that finds *live, time-sensitive offers* near a point, filters out expired ones automatically, and returns results fast enough for a real-time map.

PostGIS inside Supabase turned out to be the perfect foundation. The geospatial queries are incredibly fast, and the real-time subscriptions mean the map updates the moment a new offer is posted.

---

## ✨ What It Does

- 📍 **Radius search** — find all active offers within X metres of a point
- ⏰ **Auto-expiry** — expired offers never appear in results
- 🗂️ **Category filtering** — food, retail, services, beauty, and more
- 📊 **Analytics** — track views per offer, clicks, engagement
- 🔄 **Real-time** — Supabase subscriptions push new offers instantly
- 🌍 **Geocoding** — convert postcode or address to coordinates
- 📱 **Mobile-optimised** — responses designed for low-bandwidth use

---

## 🚀 API Endpoints

### Get Nearby Offers
```
GET /api/offers/nearby
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `lat` | float | ✅ | Latitude |
| `lng` | float | ✅ | Longitude |
| `radius` | integer | ❌ | Radius in metres (default: 1000) |
| `category` | string | ❌ | Filter by category |
| `limit` | integer | ❌ | Max results (default: 20) |

**Example Request:**
```bash
GET /api/offers/nearby?lat=51.5074&lng=-0.1278&radius=500&category=food
```

**Example Response:**
```json
{
  "offers": [
    {
      "id": "abc-123",
      "title": "Fresh Sourdough 20% Off",
      "description": "All sourdough loaves discounted today until 6pm",
      "business_name": "The Bread Basket",
      "distance_metres": 187,
      "location": {
        "lat": 51.5081,
        "lng": -0.1265
      },
      "address": "14 Baker Street, London W1U 3BW",
      "category": "food",
      "expires_at": "2024-01-15T18:00:00Z",
      "views": 34,
      "created_at": "2024-01-15T09:00:00Z"
    }
  ],
  "total": 1,
  "radius_metres": 500,
  "centre": {
    "lat": 51.5074,
    "lng": -0.1278
  }
}
```

---

### Get Single Offer
```
GET /api/offers/:id
```

---

### Track Offer View
```
POST /api/offers/:id/view
```

---

### Geocode Address
```
GET /api/geocode?address=14+Baker+Street+London
```

**Response:**
```json
{
  "address": "14 Baker Street, London W1U 3BW",
  "lat": 51.5081,
  "lng": -0.1265,
  "confidence": 0.95
}
```

---

## 🗄️ Database Schema

The API is powered by a PostGIS-enabled Supabase database. The core query uses a spatial index for sub-10ms lookups even at scale.

```sql
-- Core geospatial query
SELECT
  o.id,
  o.title,
  o.description,
  b.name AS business_name,
  ST_Distance(
    o.location::geography,
    ST_Point($lng, $lat)::geography
  ) AS distance_metres,
  ST_AsGeoJSON(o.location) AS geojson,
  o.address,
  o.category,
  o.expires_at,
  o.views
FROM offers o
JOIN businesses b ON o.business_id = b.id
WHERE
  o.is_active = true
  AND o.expires_at > NOW()
  AND ST_DWithin(
    o.location::geography,
    ST_Point($lng, $lat)::geography,
    $radius_metres
  )
ORDER BY distance_metres ASC
LIMIT $limit;
```

---

## ⚡ Performance

| Query Type | Avg Response Time |
|-----------|-----------------|
| Nearby offers (500m radius, London) | ~8ms |
| Nearby offers (2km radius, London) | ~14ms |
| Geocoding (UK address) | ~120ms |
| Single offer lookup | ~4ms |

Benchmarked on a 2GB VPS with 500 active offers in the database.

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|-----------|
| Runtime | Node.js + Express |
| Database | Supabase (PostgreSQL + PostGIS) |
| Geocoding | OpenStreetMap Nominatim |
| Real-time | Supabase Realtime |
| Hosting | Coolify (self-hosted VPS) |

---

## 🚀 Setup

### 1. Clone the repo

```bash
git clone https://github.com/zaid3/hyperlocal-map-api
cd hyperlocal-map-api
npm install
```

### 2. Set environment variables

```bash
cp .env.example .env
```

```env
SUPABASE_URL=your_supabase_url
SUPABASE_SERVICE_KEY=your_service_key
PORT=3001
DEFAULT_RADIUS_METRES=1000
MAX_RADIUS_METRES=10000
```

### 3. Run database migrations

```bash
npm run db:migrate
```

This enables the PostGIS extension and creates the spatial index:

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE INDEX offers_location_idx ON offers USING GIST (location);
```

### 4. Start the server

```bash
npm start
```

API will be available at `http://localhost:3001`

---

## 📁 Project Structure

```
hyperlocal-map-api/
├── src/
│   ├── routes/
│   │   ├── offers.js        # Offer endpoints
│   │   ├── geocode.js       # Geocoding endpoint
│   │   └── categories.js    # Category list endpoint
│   ├── db/
│   │   ├── supabase.js      # Database client
│   │   └── queries.js       # PostGIS spatial queries
│   ├── middleware/
│   │   ├── validate.js      # Request validation
│   │   └── rateLimit.js     # Rate limiting
│   └── utils/
│       └── geocoder.js      # Address to coordinates
├── migrations/
│   └── 001_enable_postgis.sql
├── tests/
│   └── offers.test.js
├── .env.example
└── README.md
```

---

## 🗺️ Roadmap

- [x] Radius-based offer search
- [x] Category filtering
- [x] Auto-expiry filtering
- [x] View tracking
- [x] Geocoding endpoint
- [ ] Bounding box queries (for map viewport)
- [ ] Offer clustering for zoom levels
- [ ] Route-based search (offers along a walking route)
- [ ] Historical offer analytics
- [ ] Webhook support for real-time map updates

---

## 🤝 Part of the LocalPulse AI Platform

This API powers the map in [LocalPulse AI](https://github.com/zaid3/local-ai-commerce).

Other components:
- 🗺️ [local-ai-commerce](https://github.com/zaid3/local-ai-commerce) — Main platform & live map
- 📱 [whatsapp-ai-agent](https://github.com/zaid3/whatsapp-ai-agent) — WhatsApp offer submission
- 📞 [voice-calling-agent](https://github.com/zaid3/voice-calling-agent) — AI phone agent

---

## 👤 Author

**A Haque**
- AI/ML Engineer | MSc Data Science
- Member, Royal Statistical Society
- 🐙 GitHub: [@zaid3](https://github.com/zaid3)
- 📧 Email: [ahaque@atomicmail.io](mailto:ahaque@atomicmail.io)
- ✍️ Writing: [hackernoon.com/u/ahaque](https://hackernoon.com/u/ahaque)

---

## 📄 Licence

MIT — free to use, modify, and deploy.
