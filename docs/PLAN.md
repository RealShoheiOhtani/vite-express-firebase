# GardenPulse — Smart Community Garden Monitor

## Project Plan

> **One-liner:** An ESP32-powered garden monitoring system that streams soil and weather data through Firebase, uses Gemini AI for plant care recommendations, and displays everything on a live web dashboard — built for KACB's community gardens and after-school environmental program.

---

## Sponsors & Context

- **Primary sponsor/user:** [Keep Alachua County Beautiful (KACB)](https://www.kacb.org/)
  - Runs 15+ community gardens across Gainesville (partners with GROW HUB)
  - Operates an after-school environmental program teaching kids ecology
  - See `docs/sponsors.md` for full sponsor analysis
- **Secondary use case (stretch):** Mill Creek Farm — pasture monitoring for elderly horses (same hardware, different thresholds)
- **Hackathon:** MLH-affiliated, Gemini/Google is a sponsor

---

## Architecture

```
[ESP32 + Sensors]
       │
       │  HTTP POST (JSON over WiFi)
       │  Interval: every 30-60 seconds
       ▼
[Firebase Cloud Function]  ────►  [Gemini API]
  "POST /api/readings"              │
       │                            │ Returns plain-English
       │ Writes to Firestore        │ garden care recommendation
       ▼                            ▼
[Firestore Database]         [Recommendation stored
       │                      alongside the reading]
       │
       │  Realtime subscription (onSnapshot)
       ▼
[Frontend — React on Vercel]
  └── Live dashboard + Gemini advice + historical charts
```

### Key Design Decisions

- **ESP32 never talks to Firebase directly.** It POSTs JSON to a Cloud Function URL. This keeps the ESP code dead simple and avoids Firestore auth complexity on the microcontroller.
- **Gemini is called server-side** in the Cloud Function, not from the frontend. This protects the API key and lets us control prompt engineering in one place.
- **Firestore realtime listeners** push updates to the frontend — no polling, no refresh needed.
- **All Google stack** (Firebase + Gemini) for MLH sponsor prize eligibility.

---

## Tech Stack

| Layer | Technology | Why |
|---|---|---|
| **Hardware** | ESP32 + DHT22 (temp/humidity) + soil moisture sensor + LDR (light) | Already owned by team |
| **Backend** | Firebase Cloud Functions (Node.js) | Serverless, free tier, handles ESP POST + Gemini call |
| **Database** | Firebase Firestore | Realtime listeners, NoSQL document model, free tier |
| **AI** | Google Gemini API | Plant care recommendations from sensor data, sponsor prize |
| **Frontend** | React (Vite) | Fast to scaffold, team familiarity |
| **Frontend hosting** | Vercel or Firebase Hosting | Free, auto-deploy from GitHub |
| **Language** | TypeScript (backend + frontend) | Type safety, shared interfaces |

---

## Data Model

### Firestore Collections

#### `gardens` collection
```
{
  id: "fifth-avenue",
  name: "Fifth Avenue Community Garden",
  location: "Gainesville, FL",
  description: "KACB community garden on 5th Ave",
  created_at: Timestamp
}
```

#### `readings` collection
```
{
  id: auto-generated,
  garden_id: "fifth-avenue",
  moisture: 42.0,          // percentage (0-100)
  temperature: 88.5,       // degrees Fahrenheit
  humidity: 73.0,          // percentage (0-100)
  sunlight: 920.0,         // lux or raw analog value
  health_status: "warning", // "good" | "warning" | "critical" — computed server-side
  recommendation: "Soil moisture is dropping while temperature remains high. Water deeply at root level this evening after sun intensity drops. Current conditions risk wilting in shallow-rooted vegetables.",
  created_at: Timestamp
}
```

### Health Status Logic (computed in Cloud Function before Gemini call)

```
CRITICAL if:
  - moisture < 20%
  - temperature > 100°F
  - temperature < 32°F

WARNING if:
  - moisture < 35%
  - temperature > 90°F
  - humidity > 85% AND temperature > 85°F (heat stress index)
  - sunlight < 200 lux for 4+ hours during daytime

GOOD: everything else
```

These thresholds are the fallback. Gemini provides the nuanced, contextual advice on top.

---

## API Endpoints (Cloud Functions)

### `POST /api/readings`
**Called by:** ESP32
**Body:**
```json
{
  "garden_id": "fifth-avenue",
  "moisture": 42.0,
  "temperature": 88.5,
  "humidity": 73.0,
  "sunlight": 920.0
}
```
**What it does:**
1. Validates incoming JSON
2. Computes `health_status` from thresholds
3. Calls Gemini API with sensor data + context prompt
4. Stores reading + recommendation in Firestore
5. Returns `201 Created` with the stored document

### `GET /api/readings?garden_id=fifth-avenue&limit=1`
**Called by:** Frontend
**Returns:** Latest reading(s) for a garden

### `GET /api/readings/history?garden_id=fifth-avenue&hours=24`
**Called by:** Frontend
**Returns:** All readings for the past N hours (for charts)

### `GET /api/gardens`
**Called by:** Frontend
**Returns:** List of all registered gardens

---

## Gemini Integration

### Prompt Template (called in Cloud Function)

```
You are a community garden advisor for Keep Alachua County Beautiful in Gainesville, Florida.

Given these real-time sensor readings from a community garden plot:
- Soil Moisture: {moisture}%
- Temperature: {temperature}°F
- Humidity: {humidity}%
- Sunlight: {sunlight} lux
- Current date/time: {timestamp}
- Health status: {health_status}

Provide a 2-3 sentence actionable recommendation for a community gardener.
Be specific about what to do and when.
Consider Gainesville's subtropical climate and the current season.
Use plain language — the gardener may be a beginner or a student in an after-school program.
Do not repeat the sensor values back. Focus on what action to take.
```

### Response handling
- Store the recommendation text in Firestore alongside the reading
- If Gemini API fails or is slow, fall back to a hardcoded recommendation based on `health_status`:
  - `"critical"` → `"Your garden needs immediate attention. Check soil moisture and water if dry."`
  - `"warning"` → `"Conditions are not ideal. Monitor your garden closely today."`
  - `"good"` → `"Your garden is looking healthy! No action needed right now."`

---

## Frontend Pages

### 1. Dashboard (main page)
- Garden selector dropdown (if multiple gardens)
- **Current readings** — four cards showing moisture, temp, humidity, sunlight with color-coded status (green/yellow/red)
- **Gemini recommendation** — prominent text box showing latest AI advice
- **Health status badge** — large green/yellow/red indicator
- **24-hour trend charts** — line charts for each sensor value over time

### 2. Garden Overview (stretch)
- Card grid of all registered gardens
- Each card shows garden name, health status, last reading timestamp
- Click through to individual garden dashboard

### 3. History (stretch)
- Date range picker
- Exportable data (CSV) for KACB grant reports

---

## ESP32 Code Outline

```
loop():
  1. Read moisture from soil sensor (analog pin)
  2. Read temperature + humidity from DHT22
  3. Read sunlight from LDR (analog pin)
  4. Package as JSON
  5. HTTP POST to Cloud Function URL
  6. Sleep for INTERVAL (30-60 seconds)
  7. Repeat
```

**Key considerations:**
- Use WiFi credentials from `config.h` (not hardcoded in main)
- Handle WiFi reconnection gracefully
- LED blink on successful POST for demo visibility
- Serial print readings for debugging

---

## Team Assignments

| Person | Responsibility | Key Deliverables |
|---|---|---|
| **CPE #1 — Hardware** | ESP32 + sensors | Working sensor reads, HTTP POST to Cloud Function, reliable WiFi connection |
| **CPE #2 — Backend** | Firebase Cloud Functions + Firestore + Gemini | POST endpoint, GET endpoints, Gemini prompt, threshold logic, Firestore schema |
| **CPE #3 — Frontend** | React dashboard + Firestore realtime | Dashboard UI, live data cards, charts, recommendation display |
| **CPE #4 — Integration & Polish** | Glue everything together | End-to-end testing, demo script, Gemini prompt tuning, stretch features, presentation |

---

## Implementation Order

### Phase 1 — Skeleton (first few hours)
- [ ] Initialize Firebase project (Firestore + Cloud Functions)
- [ ] Create `POST /api/readings` Cloud Function that writes to Firestore (no Gemini yet, just store data)
- [ ] ESP32 reads sensors and POSTs JSON to the Cloud Function URL
- [ ] React app scaffolded with Vite, reads from Firestore with `onSnapshot`
- [ ] **Milestone:** ESP sends data → shows up on dashboard in real time

### Phase 2 — Core Features
- [ ] Add threshold-based `health_status` computation in Cloud Function
- [ ] Integrate Gemini API call in Cloud Function, store recommendation
- [ ] Frontend displays health status badges (green/yellow/red) + Gemini recommendation
- [ ] Add 24-hour history chart (recharts or chart.js)
- [ ] `GET /api/readings/history` endpoint for chart data
- [ ] **Milestone:** Full pipeline working — ESP → Firebase → Gemini → Dashboard with advice

### Phase 3 — Polish & Demo
- [ ] Style the dashboard (clean, presentable for judges)
- [ ] Add garden selector (support multiple gardens/plots)
- [ ] Error handling: Gemini fallback, ESP reconnection, loading states
- [ ] Prepare demo script: show live sensor changes → dashboard updates → Gemini reacts
- [ ] **Milestone:** Demo-ready, presentation-ready

### Phase 4 — Stretch Goals (if time permits)
- [ ] Garden overview page (all gardens at a glance)
- [ ] CSV export of historical data (for KACB grant reports)
- [ ] Alerts via email/SMS when status goes critical (Firebase Cloud Messaging or Twilio)
- [ ] Mill Creek Farm mode — same platform, pasture monitoring thresholds + horse-specific Gemini prompt
- [ ] Mobile-responsive design

---

## Project Structure

```
ESPbackendAPI/
├── PLAN.md                  # This file
├── docs/
│   └── sponsors.md          # Sponsor research and analysis
├── esp32/
│   ├── src/
│   │   └── main.cpp         # Arduino/PlatformIO main loop
│   └── config.h             # WiFi creds, API URL, intervals
├── functions/
│   ├── src/
│   │   ├── index.ts         # Cloud Function entry point
│   │   ├── routes/
│   │   │   └── readings.ts  # POST and GET handlers
│   │   ├── services/
│   │   │   ├── gemini.ts    # Gemini API call + prompt template
│   │   │   └── health.ts    # Threshold logic for health_status
│   │   └── types/
│   │       └── reading.ts   # TypeScript interfaces
│   ├── package.json
│   └── tsconfig.json
├── frontend/
│   ├── src/
│   │   ├── App.tsx
│   │   ├── components/
│   │   │   ├── Dashboard.tsx      # Main dashboard layout
│   │   │   ├── SensorCard.tsx     # Individual sensor display
│   │   │   ├── HealthBadge.tsx    # Green/yellow/red status
│   │   │   ├── Recommendation.tsx # Gemini advice display
│   │   │   └── HistoryChart.tsx   # 24hr trend chart
│   │   ├── hooks/
│   │   │   └── useReadings.ts     # Firestore realtime hook
│   │   ├── lib/
│   │   │   └── firebase.ts        # Firebase config + init
│   │   └── types/
│   │       └── reading.ts         # Shared types
│   ├── package.json
│   ├── vite.config.ts
│   └── index.html
└── .gitignore
```

---

## Environment Variables / Secrets

```
# Cloud Functions (set via firebase functions:config:set)
GEMINI_API_KEY=<your-gemini-api-key>

# ESP32 (in config.h — DO NOT commit)
WIFI_SSID=<hackathon-wifi>
WIFI_PASSWORD=<hackathon-wifi-password>
CLOUD_FUNCTION_URL=https://<region>-<project>.cloudfunctions.net/api

# Frontend (in .env.local)
VITE_FIREBASE_API_KEY=<firebase-api-key>
VITE_FIREBASE_AUTH_DOMAIN=<project>.firebaseapp.com
VITE_FIREBASE_PROJECT_ID=<project-id>
```

---

## Demo Script (for judges)

1. Show the ESP32 with sensors connected — "This is deployed in a community garden"
2. Show the dashboard — live readings updating in real time
3. Cover the sunlight sensor or breathe on the humidity sensor — readings change on dashboard instantly
4. Trigger a warning/critical state — Gemini recommendation updates with specific advice
5. Show the history chart — "KACB can track conditions over time for grant reports"
6. Explain the sponsor connection — "KACB runs 15 community gardens and an after-school program. This gives them data-driven garden management and turns sensor data into a STEM teaching tool."
7. Mention Mill Creek Farm stretch — "Same platform can monitor horse pastures. One product, multiple nonprofits."

---

## Key Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Hackathon WiFi is unreliable | ESP32 buffers readings locally, retries on reconnection. For demo, use phone hotspot as backup. |
| Gemini API rate limits or latency | Hardcoded fallback recommendations based on health_status. Don't block the POST response on Gemini — store reading first, update recommendation async. |
| Firestore free tier limits | 50K reads/day, 20K writes/day — more than enough for a hackathon demo at 1 reading/minute. |
| Sensors give garbage data | Calibrate before the event. Add sanity checks in Cloud Function (reject moisture > 100, temp > 150, etc.) |
| Team member gets stuck | Phase 1 is designed so each person has an independent deliverable. Integration happens in Phase 2. |
