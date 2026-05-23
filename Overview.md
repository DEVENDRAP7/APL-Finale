# Venue Intelligence System (VIS) — Context for Claude

You are continuing work on a multi-agent venue intelligence system (VIS) for live events (stadiums, arenas, festivals). The previous session built and deployed v1; this folder is the starting point for the finale iteration where features will be improvised on top of the existing code.

---

## What this project is

A real-time crowd / safety / experience platform for a 72,000-seat venue. It ingests live sensor + camera data, decides who routes where, fires alerts, and chats with attendees. Decisions are split into two paths:

- **Rule Engine (~80% of decisions)** — fast, deterministic, no AI calls. Handles crowd density routing, queue overflow, parking alerts, standard thresholds.
- **AI Agents (~20% of decisions)** — for novel/judgement-call situations. Three agents:
  1. **AttendeeExperienceAgent** — WebSocket chat with fans (wayfinding, food, accessibility).
  2. **EmergencyResponseAgent** — handles medical / security / fire / crowd-crush incidents. Generates PA announcements.
  3. **VisionAnomalyAgent** — interprets camera frames when CV pipeline flags `anomaly_score > 0.7`.

Branding in the UI is **Gemini** (`gemini-2.5-pro` for Emergency, `gemini-2.5-flash` for Attendee + Vision). The actual code runs in **mock mode** — no real API calls. Agents return pre-written realistic responses. This lets the app run for free, no API key needed.

---

## Tech stack

- **Backend:** FastAPI + uvicorn (Python 3.13), async event bus, pydantic models, WebSockets for streaming chat.
- **Frontend:** Single-file vanilla HTML/CSS/JS (`public/index.html`) — no build step. Simulates live data client-side: KPI cards, 16 colour-coded zone cards, crowd-management decisions (Rule Engine vs AI Agent split), event log ticking every ~3s, cycling scenarios (Normal → Surge → Queue Peak → Post-Game).
- **Architecture page:** `public/architecture.html` — 6-layer system diagram with hover tooltips showing file paths and component descriptions.

---

## File structure

```
agents/
  base_agent.py            # mock-only invoke() returning pre-written AgentResponse
  attendee/agent.py        # chat() + chat_stream() — yields mock words for WebSocket
  emergency/agent.py       # handle_incident(), handle_sensor_alert()
  vision_anomaly/agent.py  # analyse() returns mock JSON
  */tools.py               # per-agent tool definitions + HANDLERS dict
api/
  main.py                  # FastAPI app, lifespan, mounts public/ as static
  routes/                  # events, attendee (WebSocket), ops
config/settings.py         # pydantic-settings, reads GEMINI_API_KEY (defaults to "mock")
core/
  client.py                # get_client() returns None, is_mock_mode() returns True
  event_bus.py             # asyncio.Queue pub/sub
  state_store.py           # in-memory VenueState
data/models.py             # pydantic models: SensorEvent, ZoneState, AgentResponse, etc.
rules/engine.py            # deterministic decisions (80% path)
scripts/simulate_realtime.py # generates sensor events
public/
  index.html               # main dashboard (live event log lives here)
  architecture.html        # interactive system diagram
main.py                    # thin entry point — imports api.main:app
requirements.txt           # NO anthropic / google SDK — pure FastAPI stack
```

---

## How to run locally

```bash
pip install -r requirements.txt
uvicorn api.main:app --reload --port 8000
# open http://localhost:8000
```

The frontend at `/` is fully simulated client-side, so it animates even with the backend off (just serve `public/index.html` directly).

The `/ops/status` endpoint returns live zone occupancy + queue depths from the rule engine + simulator.

---

## Important behaviour notes (don't re-discover these)

1. **The frontend simulates everything in JS.** KPIs, zones, decisions, event log all tick from `setInterval` loops inside `public/index.html`. Don't try to wire them to the backend unless asked — they're standalone by design.
2. **Mock mode is the default.** `is_mock_mode()` in `core/client.py` returns `True` unconditionally. Agents never call out. If you add a real Gemini integration, gate it behind a `GEMINI_API_KEY != "mock"` check — don't rip out mock mode.
3. **No external SDK in `requirements.txt`.** The original Anthropic SDK was removed during rebrand. If you add Gemini SDK (`google-generativeai`), keep mock mode as the fallback path.
4. **The `public/index.html` JS is fragile.** A previous bug had orphaned `drawCameras()` code with unclosed braces that broke the whole event log. If you edit the JS section, validate with a browser refresh — type checks won't catch this.
5. **Emojis matter — UTF-8 only.** Earlier a PowerShell scrub mangled all emojis into mojibake (â€" instead of —). Always read/write HTML files with explicit UTF-8 encoding (Python `open(..., encoding="utf-8")`, not raw `Get-Content`).
6. **`GEMINI_API_KEY=mock`** is the documented env var. `config/settings.py` reads `gemini_api_key`. Don't reintroduce `ANTHROPIC_API_KEY`.

---

## Previous deployment (v1, reference only)

The earlier iteration was deployed at:
- Frontend: https://devendrap7.github.io/APL (GitHub Pages)
- Backend + frontend: https://vis-backend-85834854394.us-central1.run.app (Google Cloud Run, project `project-d75ed264-63ea-4e45-ad9`, service `vis-backend`)

This new finale folder has NO `.git`, NO Dockerfile, NO Firebase/Pages config — start fresh if redeployment is needed.

---

## What's next (per the user)

The user is at a finale event, same problem statement, planning to improvise features on top of this baseline. Wait for them to describe the new requirements before scaffolding new code. When they ask for changes, prefer editing existing files over adding new modules — the codebase is small and intentionally flat.
