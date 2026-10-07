# Fitness AI v2.7

> 🇷🇸 Srpska verzija: [README.sr.md](./README.sr.md)

A private, chat-first fitness tracker built as a **PWA**. You type things like
`96.4kg this morning, ate 300g chicken, 4 toast and yogurt` or `bench 80 4x8 incline 60 3x10`,
and the app parses the message, saves it to a cloud database and updates your dashboard.
A built-in AI coach (Serbian-language) answers questions using your own data as context.

It works on phone and desktop against the same data, with no accounts: a private **sync key** acts as the access key.

---

## Features

- **Chat-first logging** – one input box for food, workouts, body weight and questions.
- **Hybrid parsing** – an LLM extracts structured JSON; a deterministic, catalog-based parser acts as a safety net so simple inputs never get saved as 0 kcal.
- **Food catalog** – built-in catalog of common foods (including local ones such as burek and pljeskavica) with aliases, per-100g and per-unit macros. Add your own foods in the Profile tab.
- **Nutrition tab** – calorie and protein rings, carbs/fat, remaining targets, quick-add buttons, edit/delete meals.
- **Training tab** – daily workouts, per-exercise progress (best weight / best volume), weight progress.
- **Profile tab** – body weight log and chart, profile and goals, 7/30-day analytics, AI progress analysis (14 days), custom foods, backup export, AI provider switch, progress photos.
- **AI coach with context** – the prompt includes your profile, today's totals, 7- and 30-day analytics and saved memories, plus safety rules (no diagnoses, no extreme dieting, refer to a professional for dangerous symptoms).
- **Multi-provider AI with fallback** – Groq, Gemini or OpenAI, with an automatic fallback chain ending in a local `mock` provider, so the app keeps working when quotas run out.
- **PWA** – installable on iOS/Android/desktop, standalone display mode, service worker for the app shell.
- **Multi-device sync** – same backend + same sync key = same data everywhere.

---

## Architecture

```text
┌────────────────────┐        ┌───────────────────────┐        ┌──────────────────────────┐
│  PWA (React + Vite)│  HTTPS │  Express API (Render) │        │ Supabase                 │
│  localStorage:     │ ─────▶ │  x-sync-key auth      │ ─────▶ │  PostgreSQL (11 tables)  │
│  sync key          │        │  owner_hash scoping   │        │  Storage (private bucket)│
└────────────────────┘        └──────────┬────────────┘        └──────────────────────────┘
                                         │
                                         ▼
                              Groq / Gemini / OpenAI  ──fallback──▶  local mock parser
```

### How a chat message is processed

1. The client decides with a heuristic (`looksLikeLogInput`) whether the text is a **log entry** or a **question**.
2. **Log entry** → `POST /api/ai/log`
   - builds coach context, runs the deterministic parser, asks the LLM for JSON,
   - merges both results (the catalog parser wins for simple known foods),
   - saves weight / meal / workout / memory and returns a reply plus the fresh state.
3. **Question** → `POST /api/ai/chat` with the last 10 messages as history and the full coach system prompt.
4. Every request goes through the provider chain `primary → fallback → mock`.

Typing `reset` (or a sentence such as "obriši dnevnik") wipes meals, workouts, weight and notes for the selected day.

### Data isolation

There are no user accounts. `owner_hash = sha256(sync key)` is stored on every row and every query filters by it.
When `SYNC_KEY` is set on the server, requests without the matching `x-sync-key` header are rejected with `401`.

---

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | React 19, Vite 6, lucide-react, plain CSS, hand-made SVG charts |
| PWA | `manifest.webmanifest`, `sw.js` (cache-first app shell) |
| Backend | Node 22, Express 4, multer, cors, dotenv |
| Database | Supabase PostgreSQL (`@supabase/supabase-js`, service-role key on the server only) |
| File storage | Supabase Storage, private bucket `progress-photos` (signed URLs) |
| AI | Groq (default), Gemini, OpenAI via the `openai` SDK / REST, local mock |
| Hosting | Render (Web Service for the API, Static Site for the client) |

---

## Project structure

```text
fitness-ai/
├─ package.json            # root scripts (install:all, dev:server, dev:client, build:client)
├─ render.yaml             # Render blueprint for the backend
├─ supabase/
│  └─ schema.sql           # all tables and indexes
├─ server/
│  ├─ index.js             # the whole API
│  ├─ package.json
│  └─ .env                 # copy from .env.example
└─ client/
   ├─ index.html
   ├─ vite.config.js       # Vite + @vitejs/plugin-react
   ├─ package.json
   ├─ public/
   │  ├─ manifest.webmanifest
   │  ├─ sw.js
   │  └─ icons/            # icon-192.png, icon-512.png
   └─ src/
      ├─ main.jsx          # React entry + service worker registration
      ├─ App.jsx           # all screens and components
      └─ styles.css
```

---

## Getting started (local)

### 1. Supabase

1. Create a free project at [supabase.com](https://supabase.com).
2. **SQL Editor** → paste and run `supabase/schema.sql`.
3. **Storage** → create a **private** bucket named `progress-photos`.
4. **Project Settings → API** → copy the *Project URL* and the `service_role` key.

> ⚠️ The `service_role` key belongs **only** in the server environment. Never put it in client code or in a public repo.

### 2. Server

```bash
cd server
cp .env.example .env     # then fill in the values
npm install
npm run dev              # http://localhost:3001
```

Check `http://localhost:3001/api/health` – you should get `{"ok":true,"database":"supabase",...}`.

### 3. Client

```bash
cd client
npm install
npm run dev -- --host 0.0.0.0 --port 5173
```

On port `5173` the app automatically talks to `http://localhost:3001/api`. Open `http://localhost:5173`.

### Shortcuts from the repo root

```bash
npm run install:all
npm run dev:server
npm run dev:client
npm run build:client
```

---

## Environment variables (server)

| Variable | Description | Example / default |
|---|---|---|
| `PORT` | HTTP port | `3001` (Render: `10000`) |
| `CLIENT_URL` | Allowed origin(s), comma-separated | `http://localhost:5173` |
| `CORS_ALLOW_ALL` | Relax CORS | `true` |
| `SYNC_KEY` | Private access key shared by your devices | long random string |
| `DATABASE_PROVIDER` | Only `supabase` is implemented | `supabase` |
| `SUPABASE_URL` | Supabase project URL | `https://xxxx.supabase.co` |
| `SUPABASE_SERVICE_ROLE_KEY` | Service-role key (server only) | – |
| `SUPABASE_STORAGE_BUCKET` | Bucket for progress photos | `progress-photos` |
| `AI_PROVIDER` | `groq` \| `gemini` \| `openai` \| `mock` | `groq` |
| `AI_FALLBACK_PROVIDER` | Used when the primary fails | `mock` |
| `GROQ_API_KEY`, `GROQ_BASE_URL` | Groq credentials / endpoint | – |
| `GROQ_TEXT_MODEL`, `GROQ_VISION_MODEL` | Groq models | `llama-3.1-8b-instant`, `meta-llama/llama-4-scout-17b-16e-instruct` |
| `GEMINI_API_KEY`, `GEMINI_BASE_URL` | Gemini credentials / endpoint | – |
| `GEMINI_TEXT_MODEL`, `GEMINI_VISION_MODEL` | Gemini models | `gemini-2.5-flash` |
| `OPENAI_API_KEY` | OpenAI credentials | – |
| `OPENAI_TEXT_MODEL`, `OPENAI_VISION_MODEL` | OpenAI models | `gpt-4.1` |

Client (`client/.env.local` or build env):

| Variable | Description |
|---|---|
| `VITE_API_URL` | Full API URL, e.g. `https://your-backend.onrender.com/api` |

AI provider and model choices can also be changed at runtime from the Profile tab (stored per owner in `app_settings`); API keys always stay in the server environment.

---

## Deployment (Render + Supabase)

### Backend – Render Web Service

```text
Root Directory: server
Build Command:  npm install
Start Command:  npm start
Plan:           Free
```

Add all the environment variables from the table above in the Render dashboard (secrets such as `SUPABASE_SERVICE_ROLE_KEY`, `SYNC_KEY` and the AI keys must be set there, not committed). `render.yaml` provides the non-secret defaults.

### Frontend – Render Static Site (or Vercel)

```text
Root Directory:    client
Build Command:     npm install && npm run build
Publish Directory: dist
Env:               VITE_API_URL=https://YOUR-BACKEND.onrender.com/api
```

### First launch on every device

Open the app once with your key in the URL – it is stored in `localStorage` and removed from the address bar:

```text
https://YOUR-FRONTEND.onrender.com/?syncKey=YOUR_SYNC_KEY
```

The Profile tab has a **Copy device link** button that builds this URL for you. On iPhone: Safari → Share → **Add to Home Screen**.

> **Free-tier notes:** Render free services sleep after inactivity (first request can take 30–60 s). Supabase free projects pause after a period of inactivity.

---

## API reference

All `/api/*` routes except `/api/health` require the `x-sync-key` header (or `?syncKey=`) when `SYNC_KEY` is configured.

| Method | Route | Purpose |
|---|---|---|
| GET | `/api/health` | Liveness + provider info |
| GET | `/api/state?date=YYYY-MM-DD` | Everything the UI needs for one day (profile, meals, workouts, weights, photos, analytics, exercise progress, food catalog, AI settings) |
| PUT | `/api/profile` | Update profile and goals |
| POST | `/api/weight` | Upsert weight for a date |
| POST | `/api/meal/manual` | Add a meal from structured items |
| PUT / DELETE | `/api/meal/:id` | Edit / delete a meal |
| PUT / DELETE | `/api/workout/:id` | Edit / delete a workout |
| POST | `/api/ai/log` | Parse free text and save food / workout / weight / memory |
| POST | `/api/ai/chat` | Coach conversation with context |
| POST | `/api/ai/progress` | AI analysis of the last N days |
| GET | `/api/analytics?days=N` | Averages, totals, weight change, exercise progress |
| GET / POST | `/api/foods` | List / upsert catalog foods |
| DELETE | `/api/foods/:id` | Delete a catalog food |
| GET / PUT | `/api/settings` | AI provider and model settings |
| POST | `/api/photos/:id/analyze` | Photo comment (currently a placeholder) |
| GET | `/api/export` | Full JSON export of the owner's data |
| POST | `/api/import` | Disabled – returns `501` |
| GET | `/uploads/:filename` | Redirects to a 10-minute signed Storage URL |

---

## Database

Defined in `supabase/schema.sql`:

`profile`, `weights`, `meals`, `meal_items`, `workouts`, `workout_exercises`, `progress_photos`, `ai_notes`, `ai_memory`, `food_catalog`, `app_settings` – every table carries `owner_hash`, and child tables cascade on delete.

---

## Security notes

- The sync key is both identity and password: use a long random value and don't share links that contain it.
- The server uses the Supabase **service-role** key, which bypasses Row Level Security. Isolation is enforced in application code (`owner_hash` filters). Consider also enabling RLS (with no policies) on all tables so they are never reachable through Supabase's public REST API.
- Don't commit `.env` files. Rotate keys immediately if they leak.
- AI endpoints cost quota/money: anyone with your sync key can use them.

---

## Known limitations

- **Progress photo upload** – the UI posts to `/api/photos`, but the server does not implement that route yet (only listing/signed URLs and the analyze placeholder exist).
- **Photo AI analysis** returns a static placeholder; vision models are configured but not wired in.
- **Import** is intentionally disabled in the cloud version; use Export as a backup.
- The coach and parsers are tuned for **Serbian** input and responses.
- Chat history is kept in memory on the client and is lost on refresh (messages are stored in `ai_notes` but not reloaded).

## Roadmap ideas

- Implement `POST /api/photos` (multer → Supabase Storage) and real vision analysis.
- Local-timezone date handling and per-day-logged averages.
- Network-first service worker strategy with versioned updates.
- Rate limiting, constant-time key comparison, and RLS on all tables.
- Replace `window.prompt` editors with proper forms; reload chat history.

---

*Private project. Fitness and nutrition output is informational and is not medical advice.*
