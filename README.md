# MB Sense

MB Sense is an AI-powered companion app for electric vehicle drivers. It reads your calendar, predicts how much battery each day's driving will use, and plans when and where to charge so the car is ready before you need it. It was built for the MBTMY Vibathon 2026 (AI Defined Vehicle) around a Mercedes-Benz EQS 450+.

## Features

- **Charging planner**: Gemini turns upcoming calendar events, vehicle state and weather into a charging plan with free time windows, target charge and station suggestions. A local fallback plan is used when no API key is set.
- **Calendar**: week and month views, drag-to-create events, and an AI-managed charging event that is kept in sync with your schedule.
- **Map**: MapLibre map with MapTiler tiles, OSRM driving routes, turn-by-turn steps, location search, today's and tomorrow's calendar destinations, and live charging stations from Open Charge Map.
- **AI assistant**: a chatbot that answers schedule questions and creates or edits events, plus a voice assistant streamed through the Gemini Live API.
- **Vehicle**: battery and range status, trip efficiency chart and ambient lighting settings.
- **Accounts**: sign in, register and a persisted per-user profile and app state.

## Architecture

```
Browser ── React + Vite SPA (frontend/)
   │  /api/*  (HTTP)     /live  (WebSocket)
   ▼
nginx (frontend container) ──► Express API (backend/server.ts, port 8000)
                                   ├─ Google Gemini (@google/genai)
                                   ├─ Open Charge Map
                                   ├─ OSRM routing
                                   └─ users.json (Docker volume)
```

- **Frontend**: React 19, TypeScript, Vite 6, Tailwind CSS 4, Zustand, React Router, MapLibre GL, Recharts, Motion.
- **Backend**: Node 22, Express 4, `ws`, `@google/genai`, bundled with esbuild.
- **Deployment**: Docker Compose with two services. The frontend is served by nginx, which proxies `/api` and `/live` to the backend.

## Project structure

| Path | Contents |
|---|---|
| `frontend/` | React app: `src/pages`, `src/components`, `src/lib` (charging agents and planner, chat scheduling), `src/constants` (vehicle and route data), `src/store` (Zustand stores) |
| `backend/` | Express API and WebSocket server in `server.ts` |
| `deploy/nginx/` | Host nginx config for `mbsense.online` |
| `docker-compose.yml` | Backend and frontend services |
| `.env.example` | Environment variables used by Docker Compose |

## Getting started

### Requirements

- Node.js 22 and npm (for local development)
- Docker with Compose (for the containerised setup)
- API keys: Gemini, MapTiler, Open Charge Map and Google Maps Platform. The app starts without them, but the matching features fall back to local data or are disabled.

### Run with Docker

```bash
cp .env.example .env      # then fill in your keys
docker compose up --build
```

- App: http://localhost:8080
- API health check: http://localhost:8000/api/health

Both ports are bound to `127.0.0.1`. User data is stored in the `mbsense_user_data` volume. Stop with `docker compose down`.

### Run locally

```bash
# Terminal 1: API on http://localhost:8000
cd backend
cp .env.example .env      # then fill in your keys
npm install
npm run dev

# Terminal 2: app on http://localhost:5173
cd frontend
npm install
npm run dev
```

The Vite dev server proxies `/api` and `/live` to `VITE_API_PROXY_TARGET` (default `http://localhost:8000`). If `VITE_MAPTILER_API_KEY` is not set for the frontend, Vite reads it from `backend/.env`.

### Scripts

| Package | Script | What it does |
|---|---|---|
| frontend | `npm run dev` | Vite dev server |
| frontend | `npm run build` | Type-check and build to `dist/` |
| frontend | `npm run lint` | Type-check only |
| frontend | `npm run preview` | Serve the production build |
| backend | `npm run dev` | Run `server.ts` with tsx |
| backend | `npm run build` | Type-check and bundle to `dist/server.cjs` |
| backend | `npm start` | Run the bundled server |
| backend | `npm run lint` | Type-check only |

## Environment variables

| Variable | Used by | Purpose |
|---|---|---|
| `GEMINI_API_KEY` | backend | Gemini models for chat, charging planning and the voice assistant |
| `OPEN_CHARGE_MAP_API_KEY` (or `OCM_API_KEY`) | backend | Live charging stations |
| `VITE_MAPTILER_API_KEY` | frontend build, backend | Map tiles and geocoding (backend also accepts `MAPTILER_API_KEY`) |
| `GOOGLE_MAPS_PLATFORM_KEY`, `VITE_GOOGLE_MAPS_PLATFORM_KEY`, `VITE_GOOGLE_MAPS_KEY` | frontend build | Google Maps key for the calendar page; any one of them works |
| `VITE_API_PROXY_TARGET` | frontend | Backend URL for the dev server proxy |
| `PORT` | backend | API port, default `8000` |
| `OSRM_BASE_URL` | backend | Routing server, default `https://router.project-osrm.org` |
| `MBSENSE_DATA_DIR`, `MBSENSE_DB_PATH` | backend | Where `users.json` is stored, default `./data/users.json` |
| `DISABLE_HMR` | frontend | Set to `true` to turn off hot reload |

`APP_URL` appears in `.env.example` but is not read by the code yet.

## API

All endpoints are served by `backend/server.ts`.

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/health` | Health check |
| GET | `/api/public-config` | Public client config (MapTiler key) |
| POST | `/api/auth/login`, `/api/auth/register` | Sign in and create an account |
| PUT | `/api/user-state` | Save a user's app state |
| POST | `/api/chat` | Chatbot replies |
| POST | `/api/charging/plan` | Gemini charging plan |
| POST | `/api/charging/predict` | Charging decision for the next trips |
| GET | `/api/charging/openchargemap` | Nearby charging stations |
| GET | `/api/routing/driving` | Driving route between two coordinates |
| POST | `/api/trip-efficiency` | Trip efficiency insights |
| WS | `/live` | Gemini Live voice assistant |

The server also exposes advisory endpoints that the current UI does not call: `/api/analyze-schedule`, `/api/optimize-route`, `/api/smart-routing-advisor`, `/api/weather-advisory`, `/api/trip-cost-estimator`, `/api/transit-rideshare-compare`, `/api/ev-charging-stations`, `/api/ambient-lighting-advisory`, `/api/ai-route-planner` and `/api/destination-parking-predictions`.

## Deployment

`deploy/nginx/mbsense.online.conf` is the host nginx config used in production. It forwards `mbsense.online` to the frontend container on `127.0.0.1:8080`, including WebSocket upgrades for `/live`. Add HTTPS on the host (for example with Certbot).

## Limitations and roadmap

This is a hackathon prototype. Before production use it needs:

- **PostgreSQL** in place of the `users.json` file store.
- **Redis** for caching AI responses, sessions and rate limiting.
- **Secure authentication**: passwords are currently stored in plain text and there are no session tokens. Hashed passwords, token-based sessions and HTTPS-only cookies are needed.

## Team

Michael, William, Maximillion and Vitto.
