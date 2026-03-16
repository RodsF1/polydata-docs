# PolyData API Documentation

> **The auction intelligence API.** Real-time sentiment, historical results, and predictive valuations for luxury auctions — cars, watches, art, and collectibles.

**Base URL:** `https://<project-ref>.supabase.co/functions/v1`

**Authentication:** All requests require an API key via the `x-api-key` header.

```bash
curl -H "x-api-key: pd_live_your_key_here" \
  "https://<ref>.supabase.co/functions/v1/polydata-sentiment?category=cars&limit=10"
```

---

## Quick Start

1. **Get an API key** at [polydata.polybets.fun](https://polydata.polybets.fun)
2. **Pick an endpoint:**
   - `/polydata-sentiment` — Crowd-sourced pricing signals
   - `/polydata-results` — Historical hammer prices & outcomes
   - `/polydata-valuations` — Price predictions *(Pro+)*
   - `/polydata-auctions` — Real-time listings across sources
3. **Start building.**

---

## Authentication

| Header | Description |
|--------|-------------|
| `x-api-key` | Your PolyData API key (`pd_live_...` or `pd_test_...`) |

Alternatively, pass as a query parameter: `?apikey=pd_live_...`

### Tiers

| Tier | Rate Limit | Valuations | Price |
|------|-----------|------------|-------|
| **Free** | 100/day, 10/min | ❌ | $0 |
| **Pro** | 5,000/day, 60/min | ✅ 20/day | $99/mo |
| **Enterprise** | 50,000/day, 300/min | ✅ 200/day | Contact us |

Rate limit headers are included in every response:
```
X-RateLimit-Limit-Day: 100
X-RateLimit-Remaining-Day: 87
X-RateLimit-Limit-Minute: 10
X-RateLimit-Remaining-Minute: 9
```

---

## Endpoints

### 1. Market Sentiment — `GET /polydata-sentiment`

Returns crowd-sourced market sentiment for auction prediction markets. This data is unique to PolyBets — real money wagered on auction outcomes.

#### Parameters

| Param | Type | Description |
|-------|------|-------------|
| `auction_id` | uuid | Single market lookup |
| `category` | string | `cars`, `watches`, `art`, `collectibles`, `properties` |
| `status` | string | `live`, `ended`, `upcoming` |
| `source` | string | `bat`, `phillips`, `heritage`, `liveauctioneers`, etc. |
| `include_snapshots` | bool | Include price snapshot time-series |
| `sort` | string | `volume`, `end_time`, `created` |
| `order` | string | `asc` or `desc` |
| `limit` | int | 1–100 (default 25) |
| `offset` | int | Pagination offset |

#### Response

```json
{
  "success": true,
  "data": {
    "auction_id": "abc-123",
    "title": "2005 Ferrari 575M Maranello 6-Speed",
    "category": "cars",
    "source": "bat",
    "status": "live",
    "bet_line": 125000,
    "final_price": null,
    "current_bid": 85000,
    "sentiment": {
      "over_pct": 62.5,
      "under_pct": 37.5,
      "total_volume": 480,
      "bet_count": 12,
      "over_volume": 300,
      "under_volume": 180,
      "over_bets": 7,
      "under_bets": 5
    },
    "auction_url": "https://bringatrailer.com/listing/...",
    "image_url": "https://...",
    "end_time": "2026-03-16T22:00:00Z"
  }
}
```

#### With Snapshots

Add `?include_snapshots=true` to get the full time-series of how sentiment shifted:

```json
{
  "data": {
    "sentiment": { "..." : "..." },
    "snapshots": [
      { "timestamp": "2026-03-14T10:00:00Z", "over_pct": 50, "under_pct": 50, "volume": 20 },
      { "timestamp": "2026-03-14T14:30:00Z", "over_pct": 55.2, "under_pct": 44.8, "volume": 120 },
      { "timestamp": "2026-03-15T09:15:00Z", "over_pct": 62.5, "under_pct": 37.5, "volume": 480 }
    ]
  }
}
```

---

### 2. Historical Results — `GET /polydata-results`

Returns resolved auction outcomes: hammer prices, sale rates, and how they compared to crowd predictions.

#### Parameters

| Param | Type | Description |
|-------|------|-------------|
| `auction_id` | uuid | Single result lookup |
| `category` | string | Filter by asset category |
| `source` | string | Filter by auction source |
| `sold` | bool | `true` = sold, `false` = no-sale |
| `min_price` / `max_price` | int | Price range filter |
| `make` | string | Filter by make/brand |
| `model` | string | Filter by model |
| `year_min` / `year_max` | int | Year range |
| `search` | string | Full-text search across title |
| `winning_side` | string | `over`, `under`, `tie` |
| `date_from` / `date_to` | string | Date range (ISO format) |
| `include_bets` | bool | Include anonymized bet distribution |
| `aggregate` | bool | Return aggregate statistics only |
| `sort` | string | `final_price`, `end_time`, `volume`, `bet_line` |
| `limit` / `offset` | int | Pagination |

#### Response

```json
{
  "success": true,
  "data": [
    {
      "auction_id": "def-456",
      "title": "1989 Porsche 911 Carrera 4",
      "category": "cars",
      "source": "bat",
      "bet_line": 65000,
      "final_price": 78500,
      "sold": true,
      "winning_side": "over",
      "price_vs_line_pct": 20.77,
      "end_time": "2026-03-10T22:00:00Z",
      "auction_url": "https://...",
      "bet_summary": {
        "total_volume": 650,
        "over_pct": 71.2,
        "under_pct": 28.8,
        "bet_count": 18
      }
    }
  ],
  "meta": { "total": 1240, "limit": 25, "offset": 0, "has_more": true }
}
```

#### Aggregate Mode

```
GET /polydata-results?aggregate=true&category=watches&date_from=2026-01-01
```

```json
{
  "data": {
    "total_auctions": 342,
    "sold_count": 289,
    "no_sale_count": 53,
    "sale_rate_pct": 84.5,
    "avg_price": 45200,
    "median_price": 32000,
    "min_price": 1200,
    "max_price": 892000,
    "over_wins": 156,
    "under_wins": 128,
    "ties": 5,
    "avg_price_vs_line_pct": 8.3
  }
}
```

---

### 3. Valuations — `POST /polydata-valuations` *(Pro+ only)*

Auction price predictions using category-specific bid dynamics modeling. This is the same engine that powers PolyBets' internal bet line generation.

#### Request Body

```json
{
  "title": "2005 Ferrari 575M Maranello Fiorano Handling Package 6-Speed",
  "category": "cars",
  "make": "Ferrari",
  "model": "575M Maranello",
  "year": 2005,
  "current_bid": 85000,
  "hours_remaining": 12,
  "reserve_status": "no_reserve",
  "estimate_low": 60000,
  "estimate_high": 80000
}
```

Only `title` is required. The more context you provide, the better the prediction.

#### Response

```json
{
  "success": true,
  "data": {
    "valuation": {
      "predicted_price": 125000,
      "low_estimate": 105000,
      "high_estimate": 155000,
      "confidence": "high",
      "reasoning": "Manual-trans 575Ms with the Fiorano package have sold between $100K-$160K on BaT in the past year. No-reserve + strong current bid suggests spirited bidding."
    },
    "market_context": {
      "category": "cars",
      "bid_dynamics_applied": true
    }
  },
  "meta": {
    "valuation_usage": "3/20",
    "latency_ms": 1840
  }
}
```

#### Categories Supported

- `cars` — BaT hockey-stick bid dynamics, reserve status modeling
- `watches` — Estimate multiple modeling, reference-level pricing, brand hierarchy
- `art` — Artist market trends, medium/size factors, provenance weighting
- `collectibles` — Condition grading, authentication premiums, cultural relevance

---

### 4. Live Auctions — `GET /polydata-auctions`

Real-time auction listings aggregated from 6+ sources, updated every 30 minutes.

#### Parameters

| Param | Type | Description |
|-------|------|-------------|
| `category` | string | Filter by asset category |
| `source` | string | `bat`, `cars_and_bids`, `phillips`, `heritage`, `liveauctioneers`, `christies` |
| `active` | bool | Active auctions only (default: `true`) |
| `min_bid` / `max_bid` | int | Bid range filter |
| `has_market` | bool | Whether a PolyBets market exists |
| `ending_within_hours` | int | Auctions ending within N hours |
| `search` | string | Search title text |
| `brand` | string | Filter by brand (watches, etc.) |
| `stats` | bool | Return aggregate stats only |
| `sort` | string | `end_time`, `bid`, `created` |
| `limit` / `offset` | int | Pagination (max 100) |

#### Response

```json
{
  "data": [
    {
      "id": 12345,
      "title": "2019 Porsche 911 Speedster",
      "source": "bat",
      "category": "cars",
      "url": "https://bringatrailer.com/listing/...",
      "current_bid": 285000,
      "currency": "USD",
      "end_time": 1710626400,
      "end_time_iso": "2026-03-17T02:00:00.000Z",
      "estimate_low": null,
      "estimate_high": null,
      "brand": null,
      "no_reserve": true,
      "thumbnail_url": "https://...",
      "has_polybets_market": true
    }
  ]
}
```

#### Stats Mode

```
GET /polydata-auctions?stats=true
```

```json
{
  "data": {
    "total_active": 847,
    "ending_within_24h": 52,
    "by_source": {
      "bat": 312,
      "liveauctioneers": 245,
      "heritage": 120,
      "phillips": 38,
      "christies": 82,
      "cars_and_bids": 50
    },
    "by_category": {
      "cars": 362,
      "watches": 185,
      "collectibles": 142,
      "art": 108,
      "properties": 50
    }
  }
}
```

---

## Data Sources

| Source | Categories | Update Frequency | Coverage |
|--------|-----------|-----------------|----------|
| Bring a Trailer | Cars | 30 min | All live auctions |
| Cars & Bids | Cars | 30 min | All live auctions |
| Phillips | Watches, Art | 30 min | Upcoming & live sales |
| Heritage Auctions | Collectibles, Art | 30 min | Featured auctions |
| LiveAuctioneers | All categories | 30 min | 5,000+ auction houses |
| Christie's | Watches, Art | 30 min | Major sales |

---

## Errors

All errors follow the same format:

```json
{
  "success": false,
  "error": {
    "code": "rate_limit_exceeded",
    "message": "Daily rate limit exceeded (100/day). Upgrade at https://polydata.polybets.fun/pricing"
  }
}
```

| Code | HTTP Status | Description |
|------|------------|-------------|
| `missing_api_key` | 401 | No API key provided |
| `invalid_api_key` | 401 | Key not found or malformed |
| `api_key_revoked` | 403 | Key has been deactivated |
| `feature_not_available` | 403 | Endpoint requires higher tier |
| `rate_limit_exceeded` | 429 | Daily or per-minute limit hit |
| `valuation_rate_limit_exceeded` | 429 | Valuation daily limit hit |
| `not_found` | 404 | Requested resource doesn't exist |
| `invalid_json` | 400 | Malformed request body |
| `missing_field` | 400 | Required field not provided |
| `method_not_allowed` | 405 | Wrong HTTP method |
| `internal_error` | 500 | Server-side failure |

---

## SDKs & Examples

### JavaScript / TypeScript

```typescript
const POLYDATA_BASE = 'https://<ref>.supabase.co/functions/v1'
const API_KEY = 'pd_live_your_key_here'

// Fetch live car auctions ending within 24 hours
const res = await fetch(
  `${POLYDATA_BASE}/polydata-auctions?category=cars&ending_within_hours=24&limit=10`,
  { headers: { 'x-api-key': API_KEY } }
)
const { data } = await res.json()

// Get valuation for a watch
const valuation = await fetch(`${POLYDATA_BASE}/polydata-valuations`, {
  method: 'POST',
  headers: { 'x-api-key': API_KEY, 'Content-Type': 'application/json' },
  body: JSON.stringify({
    title: 'Patek Philippe Nautilus 5711/1A-010 Blue Dial',
    category: 'watches',
    make: 'Patek Philippe',
    estimate_low: 120000,
    estimate_high: 180000,
  })
})
```

### Python

```python
import requests

BASE = "https://<ref>.supabase.co/functions/v1"
HEADERS = {"x-api-key": "pd_live_your_key_here"}

# Historical results for Ferraris sold over $100K
results = requests.get(f"{BASE}/polydata-results", headers=HEADERS, params={
    "category": "cars",
    "make": "Ferrari",
    "min_price": 100000,
    "sold": "true",
    "sort": "final_price",
    "order": "desc",
    "limit": 50,
}).json()

# Aggregate stats for watches
stats = requests.get(f"{BASE}/polydata-results", headers=HEADERS, params={
    "aggregate": "true",
    "category": "watches",
    "date_from": "2026-01-01",
}).json()
```

### cURL

```bash
# Market sentiment for a specific auction
curl -s -H "x-api-key: pd_live_xxx" \
  "https://<ref>.supabase.co/functions/v1/polydata-sentiment?auction_id=abc-123&include_snapshots=true"

# Live auction stats
curl -s -H "x-api-key: pd_live_xxx" \
  "https://<ref>.supabase.co/functions/v1/polydata-auctions?stats=true"
```

---

## Use Cases

- **Trading bots** — Use sentiment signals + valuations to identify mispriced auctions
- **Market research** — Aggregate historical results to track category trends
- **Appraisals** — Instant valuations for insurance, lending, or estate planning
- **Content** — Power auction blogs, newsletters, or dashboards with real-time data
- **Alerts** — Monitor ending auctions by category/price range/source

---

## Support

- **Email:** contactpolybets@gmail.com (ask for Rod) 
