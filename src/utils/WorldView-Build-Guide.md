# WorldView — OSINT Geospatial Command Center
### Build Guide: Fusing Public Data on a 4D Globe (100% Free Stack)

> Inspired by Bilawal Sidhu's *"The Intelligence Monopoly Is Over"*  
> Goal: Reconstruct real-world events by layering open-source intelligence feeds onto an interactive 3D globe with a scrubbable timeline.  
> **Cost: $0.00** — every component below is free, open source, or has a perpetual free tier.

---

## 💰 What Changed From v1 (Paid → Free)

| Component | ❌ Removed (Paid) | ✅ Replaced With (Free) |
|---|---|---|
| AI Agent | Anthropic Claude API ($) | **Ollama + Llama 3.1** (local, free forever) OR **Google Gemini API** (free tier) |
| Agent Trigger | Twilio WhatsApp ($) | **Telegram Bot API** (completely free, no expiry) |
| Maritime Data | MarineTraffic API ($) | **aisstream.io** (free WebSocket) |
| Globe Tiles | Cesium Ion (freemium) | **OpenStreetMap** tiles via Cesium (no token needed) |
| Hosting | Cloud servers ($) | **Run locally** OR Railway/Render free tier |

Everything else was already free and stays the same.

---

## 1. Project Overview

WorldView fuses six public data streams onto a 3D globe with 4D (time-aware) playback:

| Layer | Free Source | What It Reveals |
|---|---|---|
| Commercial Flights | OpenSky Network (free account) | Airspace clearing, route diversions |
| Satellite Constellations | Celestrak (no account needed) | Who was watching overhead |
| GPS Jamming | ADS-B nav confidence (derived) | Electronic warfare zones |
| Maritime Traffic | aisstream.io (free WebSocket) | Shipping panic, chokepoint closures |
| No-Fly Zones | aviationweather.gov NOTAM API (free) | Cascading airspace shutdowns |
| Strike / Event Coordinates | Open reporting + manual geolocation | Ground truth anchors |

---

## 2. Tech Stack (All Free)

### Frontend
- **CesiumJS** — 3D globe, terrain, 4D entity playback (open source)
- **React** (Vite) — UI shell, layer controls, timeline
- **Tailwind CSS** — Styling
- **Zustand** — Global state
- **date-fns** — Timestamp normalization

### Backend / Data Pipeline
- **Node.js + Express** — API proxy, data ingestion
- **PostgreSQL + PostGIS** — Spatial queries, storing snapshots
- **Redis** — Hot cache for live feed data
- **node-cron** — Simple job scheduling (replaces BullMQ — lighter, no queue dependency)

### AI Agent Layer (Free — Pick One)

**Option A: Ollama (Fully Local — Zero Cost Forever)**
- Run Llama 3.1 or Mistral locally on your machine
- No API key, no internet required for AI, no rate limits

**Option B: Google Gemini API (Free Cloud)**
- Free tier: 15 requests/min, 1,500 requests/day, 1M tokens/day
- More than enough for personal OSINT use

### Agent Trigger
- **Telegram Bot API** — 100% free, no trial credits, no expiry
  Send a message to your private bot → triggers data snapshot

### Satellite Tracking
- **satellite.js** — SGP4 orbital propagation (runs in browser, free)
- **Celestrak** — Free TLE catalog, no auth needed
- **Space-Track.org** — Free account for full NORAD catalog

---

## 3. Architecture

```
┌────────────────────────────────────────────────────┐
│                   User Browser                     │
│  ┌──────────┐  ┌───────────┐  ┌─────────────────┐ │
│  │ 3D Globe │  │ Timeline  │  │  Layer Controls  │ │
│  │ CesiumJS │  │  Scrubber │  │  (toggle each)   │ │
│  └──────────┘  └───────────┘  └─────────────────┘ │
└────────────────────────┬───────────────────────────┘
                         │ REST / WebSocket
┌────────────────────────▼───────────────────────────┐
│                  Backend API (Node/Express)         │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐ │
│  │  ADS-B   │  │   AIS    │  │  Satellite/TLE   │ │
│  │ Ingestor │  │ Ingestor │  │    Ingestor      │ │
│  └──────────┘  └──────────┘  └──────────────────┘ │
│  ┌────────────────────────────────────────────┐    │
│  │       node-cron (scheduled polling)        │    │
│  └────────────────────────────────────────────┘    │
└────────────────────────┬───────────────────────────┘
                         │
          ┌──────────────▼──────────────┐
          │  PostgreSQL + PostGIS + Redis│
          └─────────────────────────────┘
                         │
          ┌──────────────▼──────────────────────┐
          │  AI Agent: Ollama (local) OR Gemini  │
          │  Triggered via Telegram Bot (free)   │
          └──────────────────────────────────────┘
```

---

## 4. Data Sources & APIs (All Free)

### 4.1 ADS-B — Commercial Flights
```
Provider:   OpenSky Network
Endpoint:   https://opensky-network.org/api/states/all
Auth:       Free account (4,000 req/day; 400/day anonymous)
Polling:    Every 30 seconds
Fields:     icao24, callsign, lat, lon, altitude, velocity, heading, on_ground, position_source
```

```js
const res = await fetch(
  'https://opensky-network.org/api/states/all?lamin=20&lomin=40&lamax=40&lomax=65',
  { headers: { Authorization: 'Basic ' + btoa('user:pass') } }
);
const { states } = await res.json();
```

**GPS Jamming Detection (free, derived):**
`position_source = 0` → GPS lock. `position_source = 1` → MLAT (GPS degraded).
A cluster of value `1` in a region = active GPS jamming. No paid sensor needed.

---

### 4.2 Satellite Orbital Data (NORAD TLEs)
```
Provider:   Celestrak — free, no account needed
Endpoints:
  Analyst catalog:  https://celestrak.org/satcat/tle.php?GROUP=analyst&FORMAT=tle
  GPS satellites:   https://celestrak.org/satcat/tle.php?GROUP=gps-ops&FORMAT=tle
  Starlink:         https://celestrak.org/satcat/tle.php?GROUP=starlink&FORMAT=tle
Format:     Two-Line Element sets (TLE)
```

```js
// No API key required
const res = await fetch('https://celestrak.org/satcat/tle.php?GROUP=analyst&FORMAT=tle');
const tleText = await res.text(); // Parse into [name, line1, line2] triplets

import * as satellite from 'satellite.js';
const satrec = satellite.twoline2satrec(line1, line2);
const pv = satellite.propagate(satrec, new Date());
```

Key satellites to track by NORAD ID (search Space-Track.org for TLEs):
- KH-11 Keyhole (US recon), BARS-M (Russian recon), Gaofen (Chinese dual-use), Capella (SAR)

---

### 4.3 Maritime AIS Traffic — aisstream.io
```
Provider:   aisstream.io
Type:       Free WebSocket (real-time push — no polling needed)
Auth:       Free API key at https://aisstream.io — no credit card required
Fields:     MMSI, vessel name, lat, lon, heading, speed, vessel type, destination
```

```js
import WebSocket from 'ws';

const ws = new WebSocket('wss://stream.aisstream.io/v0/stream');

ws.on('open', () => {
  ws.send(JSON.stringify({
    APIKey: process.env.AISSTREAM_API_KEY,  // free
    BoundingBoxes: [[[22.0, 55.0], [27.0, 60.0]]],  // Strait of Hormuz
    FilterMessageTypes: ['PositionReport']
  }));
});

ws.on('message', (data) => {
  const msg = JSON.parse(data.toString());
  const { UserID, Latitude, Longitude, Sog } = msg.Message.PositionReport;
  // Store to DB
});
```

---

### 4.4 NOTAM / No-Fly Zones
```
Provider:   aviationweather.gov (US FAA — free, no key required)
Endpoint:   https://aviationweather.gov/api/data/notam?format=geojson&location=OIIX
            OIIX = Tehran FIR | OBBB = Bahrain | OKAC = Kuwait | OTBD = Qatar
Format:     GeoJSON polygons with time windows
```

```js
// No API key needed
const res = await fetch('https://aviationweather.gov/api/data/notam?format=geojson&location=OIIX');
const notams = await res.json();
// GeoJSON features with start/end times — render as Cesium polygons
```

---

### 4.5 Strike/Event Coordinates (Manual, Free)
1. Monitor open reporting: Reuters, BBC, Al Jazeera, Radio Free Europe
2. Cross-reference with official government statements
3. Geolocate using **Google Earth Web** (free) or **Overpass Turbo** (free)
4. Only graduate **high-confidence** events to the map
5. Store as GeoJSON with `time`, `confidence`, `source` fields

---

## 5. Free AI Agent

### Option A: Ollama — Fully Local, Free Forever

```bash
# Install Ollama (Mac/Linux/Windows — free)
curl -fsSL https://ollama.ai/install.sh | sh
ollama pull llama3.1   # ~4.7GB one-time download
```

```ts
// agent/ollamaAgent.ts — no API key, no cost
import ollama from 'ollama';

const TOOLS = [
  {
    type: 'function',
    function: {
      name: 'snapshot_all_feeds',
      description: 'Snapshot all live OSINT feeds and store to database',
      parameters: {
        type: 'object',
        properties: {
          label: { type: 'string', description: 'Event label e.g. "Iran strikes T+0"' }
        },
        required: ['label']
      }
    }
  },
  {
    type: 'function',
    function: {
      name: 'geotag_event',
      description: 'Pin an event on the globe',
      parameters: {
        type: 'object',
        properties: {
          name: { type: 'string' },
          lat: { type: 'number' },
          lon: { type: 'number' },
          confidence: { type: 'string', enum: ['low', 'medium', 'high'] }
        },
        required: ['name', 'lat', 'lon']
      }
    }
  }
];

export async function runAgent(userMessage: string) {
  const response = await ollama.chat({
    model: 'llama3.1',
    messages: [
      { role: 'system', content: 'You are a geospatial OSINT analyst. Use tools to snapshot and tag events.' },
      { role: 'user', content: userMessage }
    ],
    tools: TOOLS
  });

  if (response.message.tool_calls) {
    for (const call of response.message.tool_calls) {
      await dispatchTool(call.function.name, call.function.arguments);
    }
  }
  return response;
}
```

### Option B: Google Gemini API — Free Cloud AI

```
Free tier:  15 req/min, 1,500 req/day, 1M tokens/day
Get key:    https://aistudio.google.com/app/apikey (free Google account)
Model:      gemini-1.5-flash
```

```ts
// agent/geminiAgent.ts
import { GoogleGenerativeAI } from '@google/generative-ai';
const genai = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);

const model = genai.getGenerativeModel({
  model: 'gemini-1.5-flash',
  tools: [{ functionDeclarations: TOOL_DECLARATIONS }]
});

export async function runAgent(userMessage: string) {
  const chat = model.startChat();
  const result = await chat.sendMessage(userMessage);
  const calls = result.response.functionCalls();
  if (calls) {
    for (const call of calls) await dispatchTool(call.name, call.args);
  }
  return result.response.text();
}
```

---

## 6. Free Agent Trigger — Telegram Bot

Telegram Bot API is 100% free — no trial credits, no credit card, no expiry.

```bash
# Setup (one-time):
# 1. Message @BotFather on Telegram → /newbot → copy BOT_TOKEN
# 2. Message @userinfobot → copy your CHAT_ID
```

```ts
// agent/telegramTrigger.ts
import TelegramBot from 'node-telegram-bot-api';

const bot = new TelegramBot(process.env.TELEGRAM_BOT_TOKEN!, { polling: true });

bot.on('message', async (msg) => {
  if (String(msg.chat.id) !== process.env.MY_TELEGRAM_CHAT_ID) return; // only you

  bot.sendMessage(msg.chat.id, '⚡ Snapshotting all feeds...');
  await runAgent(msg.text || '');
  bot.sendMessage(msg.chat.id, '✅ Snapshot saved to database.');
});
```

Now you can send `"snapshot Iran strikes T+30min"` from Telegram and your agent fires — exactly like the WhatsApp trigger in the article, but free.

---

## 7. CesiumJS Globe — Free Tile Setup

```bash
npm create vite@latest worldview-web -- --template react-ts
cd worldview-web
npm install cesium resium zustand date-fns satellite.js
```

```tsx
// Globe.tsx — free OpenStreetMap tiles, no Cesium Ion token required
import { Viewer } from 'resium';
import * as Cesium from 'cesium';

const osmProvider = new Cesium.OpenStreetMapImageryProvider({
  url: 'https://tile.openstreetmap.org/'
});

export function Globe() {
  return (
    <Viewer
      full
      imageryProvider={osmProvider}
      terrainProvider={new Cesium.EllipsoidTerrainProvider()}
      timeline={true}
      animation={true}
      baseLayerPicker={false}
    />
  );
}
```

> **Optional:** Sign up for Cesium Ion free tier to get real terrain and better imagery (1M tile requests/month free — more than enough for personal use).

### 4D Playback
```ts
const property = new Cesium.SampledPositionProperty();
flightData.forEach(({ time, lat, lon, alt }) => {
  property.addSample(
    Cesium.JulianDate.fromIso8601(time),
    Cesium.Cartesian3.fromDegrees(lon, lat, alt)
  );
});
viewer.entities.add({
  position: property,
  point: { pixelSize: 5, color: Cesium.Color.CYAN },
  path: { show: true, width: 1, leadTime: 0, trailTime: 300 }
});
```

---

## 8. Satellite Propagation Engine

```ts
import * as satellite from 'satellite.js';

export function getSatellitePosition(tle1: string, tle2: string, date: Date) {
  const satrec = satellite.twoline2satrec(tle1, tle2);
  const pv = satellite.propagate(satrec, date);
  if (!pv.position || typeof pv.position === 'boolean') return null;

  const gmst = satellite.gstime(date);
  const geo = satellite.eciToGeodetic(pv.position, gmst);
  return {
    lat: satellite.radiansToDegrees(geo.latitude),
    lon: satellite.radiansToDegrees(geo.longitude),
    alt: geo.height * 1000
  };
}

export function buildOrbitalArc(tle1: string, tle2: string, startTime: Date, durationMinutes = 90) {
  const property = new Cesium.SampledPositionProperty();
  for (let i = 0; i <= durationMinutes; i += 0.5) {
    const t = new Date(startTime.getTime() + i * 60000);
    const pos = getSatellitePosition(tle1, tle2, t);
    if (pos) property.addSample(
      Cesium.JulianDate.fromDate(t),
      Cesium.Cartesian3.fromDegrees(pos.lon, pos.lat, pos.alt)
    );
  }
  return property;
}
```

---

## 9. GPS Jamming Heatmap

```ts
export function buildJammingGrid(flights: Flight[], gridSize = 0.5) {
  const cells: Record<string, { count: number; jammed: number }> = {};
  flights.forEach(f => {
    const key = `${Math.floor(f.lat / gridSize)},${Math.floor(f.lon / gridSize)}`;
    if (!cells[key]) cells[key] = { count: 0, jammed: 0 };
    cells[key].count++;
    if (f.positionSource !== 0) cells[key].jammed++;
  });
  return Object.entries(cells)
    .filter(([, v]) => v.count >= 3)
    .map(([key, v]) => {
      const [latIdx, lonIdx] = key.split(',').map(Number);
      return {
        lat: latIdx * gridSize + gridSize / 2,
        lon: lonIdx * gridSize + gridSize / 2,
        intensity: v.jammed / v.count
      };
    });
}
```

---

## 10. Database Schema (PostGIS)

```sql
CREATE TABLE flight_snapshots (
  id BIGSERIAL PRIMARY KEY,
  snapshot_label TEXT,
  icao24 CHAR(6),
  callsign TEXT,
  position GEOMETRY(Point, 4326),
  altitude_m REAL,
  velocity_ms REAL,
  heading REAL,
  position_source SMALLINT,
  on_ground BOOLEAN,
  captured_at TIMESTAMPTZ NOT NULL
);
CREATE INDEX ON flight_snapshots USING GIST(position);
CREATE INDEX ON flight_snapshots (captured_at);

CREATE TABLE satellite_passes (
  id BIGSERIAL PRIMARY KEY,
  norad_id INTEGER,
  name TEXT,
  position GEOMETRY(Point, 4326),
  altitude_km REAL,
  computed_at TIMESTAMPTZ NOT NULL
);

CREATE TABLE vessel_snapshots (
  id BIGSERIAL PRIMARY KEY,
  mmsi BIGINT,
  vessel_name TEXT,
  vessel_type SMALLINT,
  position GEOMETRY(Point, 4326),
  speed_kn REAL,
  heading SMALLINT,
  destination TEXT,
  captured_at TIMESTAMPTZ NOT NULL
);

CREATE TABLE events (
  id BIGSERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  event_type TEXT,
  position GEOMETRY(Point, 4326) NOT NULL,
  occurred_at TIMESTAMPTZ,
  confidence TEXT CHECK (confidence IN ('low', 'medium', 'high')),
  sources TEXT[],
  notes TEXT
);
```

---

## 11. Docker Compose (All Free)

```yaml
version: '3.9'
services:
  db:
    image: postgis/postgis:16-3.4
    environment:
      POSTGRES_DB: worldview
      POSTGRES_USER: worldview
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  api:
    build: ./apps/api
    environment:
      DATABASE_URL: postgresql://worldview:secret@db:5432/worldview
      REDIS_URL: redis://redis:6379
      OPENSKY_USER: ${OPENSKY_USER}
      OPENSKY_PASS: ${OPENSKY_PASS}
      AISSTREAM_API_KEY: ${AISSTREAM_API_KEY}
      GEMINI_API_KEY: ${GEMINI_API_KEY}        # OR leave blank if using Ollama
      TELEGRAM_BOT_TOKEN: ${TELEGRAM_BOT_TOKEN}
      MY_TELEGRAM_CHAT_ID: ${MY_TELEGRAM_CHAT_ID}
    ports:
      - "3001:3001"
    depends_on: [db, redis]

  web:
    build: ./apps/web
    ports:
      - "5173:5173"
    # No paid token needed — using free OSM tiles

  # Uncomment to run Ollama locally instead of Gemini
  # ollama:
  #   image: ollama/ollama
  #   ports: ["11434:11434"]
  #   volumes: [ollama_data:/root/.ollama]

volumes:
  pgdata:
```

---

## 12. Free Hosting Options

| Platform | Free Tier | Notes |
|---|---|---|
| **Local machine** | $0 forever | Best option — fastest, no limits |
| **Railway** | ~$5/month credit | Covers small full-stack app |
| **Render** | Free web service + Postgres | DB sleeps after inactivity |
| **Supabase** | Free Postgres + PostGIS | Database only |
| **Vercel** | Free | Frontend (React) only |

---

## 13. Build Phases

### Phase 1 — Globe Shell (Week 1)
- [ ] Vite + React + CesiumJS boilerplate
- [ ] Free OSM tile provider (no token needed)
- [ ] Basic 3D globe rendering
- [ ] Timeline scrubber UI
- [ ] Layer toggle panel

### Phase 2 — Live Flights (Week 1–2)
- [ ] OpenSky API (free account)
- [ ] Flight entity rendering
- [ ] node-cron polling (every 30s)
- [ ] GPS jamming heatmap

### Phase 3 — Satellites (Week 2)
- [ ] Celestrak TLE fetcher (no auth)
- [ ] satellite.js propagation in Web Worker
- [ ] Orbital arc rendering
- [ ] Filter by satellite type

### Phase 4 — Maritime + NOTAM (Week 3)
- [ ] aisstream.io WebSocket (free key)
- [ ] Vessel icons on globe
- [ ] NOTAM GeoJSON overlay
- [ ] Strait of Hormuz bounding box

### Phase 5 — AI Agent + Telegram (Week 3–4)
- [ ] Ollama setup OR Gemini free key
- [ ] `snapshot_all_feeds` tool
- [ ] `geotag_event` tool
- [ ] **Telegram bot** trigger
- [ ] Event sidebar in UI

### Phase 6 — Replay & Polish (Week 4)
- [ ] Full 4D replay from database
- [ ] Export KML/GeoJSON
- [ ] PointPrimitiveCollection for performance
- [ ] Share links for time ranges

---

## 14. Key Dependencies (All Free / Open Source)

```json
{
  "dependencies": {
    "cesium": "^1.115.0",
    "resium": "^1.18.0",
    "satellite.js": "^5.0.0",
    "react": "^18.3.0",
    "zustand": "^4.5.0",
    "date-fns": "^3.6.0",
    "ollama": "^0.5.0",
    "@google/generative-ai": "^0.15.0",
    "node-telegram-bot-api": "^0.64.0",
    "node-cron": "^3.0.0",
    "ws": "^8.18.0",
    "pg": "^8.12.0",
    "ioredis": "^5.3.0",
    "express": "^4.19.0"
  }
}
```

---

## 15. Free Account Checklist

- [ ] **OpenSky Network** — https://opensky-network.org (free account)
- [ ] **Celestrak** — https://celestrak.org (no account needed)
- [ ] **Space-Track.org** — https://space-track.org (free account)
- [ ] **aisstream.io** — https://aisstream.io (free API key, no credit card)
- [ ] **Google AI Studio** — https://aistudio.google.com (free Gemini key) *OR* install Ollama locally
- [ ] **Telegram** — Create bot via @BotFather (free)
- [ ] **Cesium Ion** — https://ion.cesium.com (optional free tier for terrain)

**Total monthly cost: $0**

---

## 16. Performance Notes

- Render 3,400+ flights with **Cesium.PointPrimitiveCollection** — much faster than individual entities
- Use **clustering** for maritime vessels at low zoom
- Run satellite.js propagation in a **Web Worker** — avoid blocking the render thread
- Cache TLEs in Redis with a 12-hour TTL
- For historical replay, serve **pre-computed snapshots** from Postgres — not live re-propagation

---

*"The intelligence monopoly is over. The question is what we do with that capability."*  
— Bilawal Sidhu, March 2026
