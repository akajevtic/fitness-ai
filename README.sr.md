# Fitness AI v2.7

> 🇬🇧 English version: [README.md](./README.md)

Privatna fitness aplikacija (**PWA**) u kojoj sve ide kroz jedan chat. Napišeš
`96.4kg jutros, jeo sam 300g piletine, 4 tosta i jogurt` ili `bench 80 4x8 incline 60 3x10`,
a aplikacija prepozna unos, upiše ga u cloud bazu i osveži sve tabove.
Ugrađeni AI coach (na srpskom) odgovara na pitanja koristeći tvoje podatke kao kontekst.

Radi na telefonu i računaru nad istim podacima, bez naloga: privatni **sync key** služi kao ključ za pristup.

---

## Funkcionalnosti

- **Unos preko chata** – jedno polje za hranu, trening, kilažu i pitanja.
- **Hibridno parsiranje** – LLM izvlači strukturisani JSON, a deterministički parser sa katalogom namirnica je sigurnosna mreža da se jednostavni unosi ne sačuvaju kao 0 kcal.
- **Katalog namirnica** – ugrađen katalog čestih namirnica (uključujući domaće, npr. burek i pljeskavicu) sa alias-ima i makroima po 100g i po komadu. Svoje namirnice dodaješ u tabu Profil.
- **Tab Ishrana** – prstenovi za kalorije i protein, UH/masti, preostali ciljevi, brzi unos, izmena/brisanje obroka.
- **Tab Trening** – dnevni treninzi, progres po vežbama (najveća kilaža / najbolji volumen), napredak težine.
- **Tab Profil** – dnevna težina i graf, profil i ciljevi, analitika za 7/30 dana, AI analiza 14 dana, moje namirnice, export backup-a, izbor AI provajdera, slike napretka.
- **AI coach sa kontekstom** – prompt sadrži profil, današnje zbirove, analitiku 7 i 30 dana i zapamćene stvari, plus sigurnosna pravila (bez dijagnoza, bez ekstremnih dijeta, upućivanje stručnjaku kod opasnih simptoma).
- **Više AI provajdera sa fallback-om** – Groq, Gemini ili OpenAI, sa lancem rezerve koji se završava lokalnim `mock` provajderom, pa aplikacija radi i kad se potroše kvote.
- **PWA** – instalira se na iOS/Android/desktop, standalone prikaz, service worker za app shell.
- **Sinhronizacija između uređaja** – isti backend + isti sync key = isti podaci svuda.

---

## Arhitektura

```text
┌────────────────────┐        ┌───────────────────────┐        ┌──────────────────────────┐
│  PWA (React + Vite)│  HTTPS │  Express API (Render) │        │ Supabase                 │
│  localStorage:     │ ─────▶ │  x-sync-key provera   │ ─────▶ │  PostgreSQL (11 tabela)  │
│  sync key          │        │  owner_hash filtriranje│       │  Storage (privatni bucket)│
└────────────────────┘        └──────────┬────────────┘        └──────────────────────────┘
                                         │
                                         ▼
                              Groq / Gemini / OpenAI  ──rezerva──▶  lokalni mock parser
```

### Kako se obrađuje poruka u chatu

1. Klijent heuristikom (`looksLikeLogInput`) odlučuje da li je tekst **unos u dnevnik** ili **pitanje**.
2. **Unos** → `POST /api/ai/log`
   - gradi kontekst za coach-a, pokreće deterministički parser, traži JSON od LLM-a,
   - spaja oba rezultata (katalog pobeđuje za jednostavne poznate namirnice),
   - čuva težinu / obrok / trening / memoriju i vraća odgovor i osvežen state.
3. **Pitanje** → `POST /api/ai/chat` sa poslednjih 10 poruka kao istorijom i kompletnim system promptom.
4. Svaki zahtev prolazi kroz lanac `primarni → fallback → mock`.

Kucanje `reset` (ili rečenice poput „obriši dnevnik“) briše obroke, treninge, težinu i beleške za izabrani dan.

### Izolacija podataka

Nema korisničkih naloga. `owner_hash = sha256(sync key)` upisuje se u svaki red i svaki upit filtrira po njemu.
Kada je `SYNC_KEY` postavljen na serveru, zahtevi bez odgovarajućeg `x-sync-key` header-a dobijaju `401`.

---

## Tehnologije

| Sloj | Tehnologija |
|---|---|
| Frontend | React 19, Vite 6, lucide-react, običan CSS, ručno rađeni SVG grafici |
| PWA | `manifest.webmanifest`, `sw.js` (cache-first app shell) |
| Backend | Node 22, Express 4, multer, cors, dotenv |
| Baza | Supabase PostgreSQL (`@supabase/supabase-js`, service-role ključ samo na serveru) |
| Fajlovi | Supabase Storage, privatni bucket `progress-photos` (potpisani URL-ovi) |
| AI | Groq (podrazumevano), Gemini, OpenAI, lokalni mock |
| Hosting | Render (Web Service za API, Static Site za klijent) |

---

## Struktura projekta

```text
fitness-ai/
├─ package.json            # root skripte (install:all, dev:server, dev:client, build:client)
├─ render.yaml             # Render blueprint za backend
├─ supabase/
│  └─ schema.sql           # sve tabele i indeksi
├─ server/
│  ├─ index.js             # ceo API
│  ├─ package.json
│  └─ .env                 # kopija iz .env.example
└─ client/
   ├─ index.html
   ├─ vite.config.js       # Vite + @vitejs/plugin-react
   ├─ package.json
   ├─ public/
   │  ├─ manifest.webmanifest
   │  ├─ sw.js
   │  └─ icons/            # icon-192.png, icon-512.png
   └─ src/
      ├─ main.jsx          # React ulaz + registracija service worker-a
      ├─ App.jsx           # svi ekrani i komponente
      └─ styles.css
```

---

## Pokretanje lokalno

### 1. Supabase

1. Napravi besplatan projekat na [supabase.com](https://supabase.com).
2. **SQL Editor** → nalepi i pokreni `supabase/schema.sql`.
3. **Storage** → napravi **privatni** bucket `progress-photos`.
4. **Project Settings → API** → kopiraj *Project URL* i `service_role` ključ.

> ⚠️ `service_role` ključ ide **samo** u okruženje servera. Nikad u klijentski kod ili javni repozitorijum.

### 2. Server

```bash
cd server
cp .env.example .env     # pa popuni vrednosti
npm install
npm run dev              # http://localhost:3001
```

Proveri `http://localhost:3001/api/health` – treba da dobiješ `{"ok":true,"database":"supabase",...}`.

### 3. Klijent

```bash
cd client
npm install
npm run dev -- --host 0.0.0.0 --port 5173
```

Na portu `5173` aplikacija automatski koristi `http://localhost:3001/api`. Otvori `http://localhost:5173`.

### Prečice iz root foldera

```bash
npm run install:all
npm run dev:server
npm run dev:client
npm run build:client
```

---

## Promenljive okruženja (server)

| Promenljiva | Opis | Primer / podrazumevano |
|---|---|---|
| `PORT` | HTTP port | `3001` (Render: `10000`) |
| `CLIENT_URL` | Dozvoljeni origin(i), odvojeni zarezom | `http://localhost:5173` |
| `CORS_ALLOW_ALL` | Opušten CORS | `true` |
| `SYNC_KEY` | Privatni ključ koji dele tvoji uređaji | dugačak nasumičan string |
| `DATABASE_PROVIDER` | Implementiran je samo `supabase` | `supabase` |
| `SUPABASE_URL` | URL Supabase projekta | `https://xxxx.supabase.co` |
| `SUPABASE_SERVICE_ROLE_KEY` | Service-role ključ (samo server) | – |
| `SUPABASE_STORAGE_BUCKET` | Bucket za slike napretka | `progress-photos` |
| `AI_PROVIDER` | `groq` \| `gemini` \| `openai` \| `mock` | `groq` |
| `AI_FALLBACK_PROVIDER` | Koristi se kad primarni padne | `mock` |
| `GROQ_API_KEY`, `GROQ_BASE_URL` | Groq pristupni podaci / endpoint | – |
| `GROQ_TEXT_MODEL`, `GROQ_VISION_MODEL` | Groq modeli | `llama-3.1-8b-instant`, `meta-llama/llama-4-scout-17b-16e-instruct` |
| `GEMINI_API_KEY`, `GEMINI_BASE_URL` | Gemini pristupni podaci / endpoint | – |
| `GEMINI_TEXT_MODEL`, `GEMINI_VISION_MODEL` | Gemini modeli | `gemini-2.5-flash` |
| `OPENAI_API_KEY` | OpenAI ključ | – |
| `OPENAI_TEXT_MODEL`, `OPENAI_VISION_MODEL` | OpenAI modeli | `gpt-4.1` |

Klijent (`client/.env.local` ili build okruženje):

| Promenljiva | Opis |
|---|---|
| `VITE_API_URL` | Pun API URL, npr. `https://tvoj-backend.onrender.com/api` |

AI provajder i modele možeš menjati i u tabu Profil (čuva se po vlasniku u `app_settings`); API ključevi uvek ostaju u okruženju servera.

---

## Deploy (Render + Supabase)

### Backend – Render Web Service

```text
Root Directory: server
Build Command:  npm install
Start Command:  npm start
Plan:           Free
```

U Render dashboard-u dodaj sve promenljive iz tabele iznad (tajne poput `SUPABASE_SERVICE_ROLE_KEY`, `SYNC_KEY` i AI ključeva postavljaš tamo, ne commit-uješ ih). `render.yaml` sadrži samo podrazumevane vrednosti koje nisu tajne.

### Frontend – Render Static Site (ili Vercel)

```text
Root Directory:    client
Build Command:     npm install && npm run build
Publish Directory: dist
Env:               VITE_API_URL=https://TVOJ-BACKEND.onrender.com/api
```

### Prvo otvaranje na svakom uređaju

Otvori aplikaciju jednom sa ključem u URL-u – čuva se u `localStorage` i briše iz adresne trake:

```text
https://TVOJ-FRONTEND.onrender.com/?syncKey=TVOJ_SYNC_KEY
```

U tabu Profil postoji dugme **Kopiraj link za uređaj** koje sastavlja taj URL. Na iPhone-u: Safari → Share → **Add to Home Screen**.

> **Napomena za free plan:** Render free servisi „zaspe“ posle neaktivnosti (prvi zahtev može trajati 30–60 s). Supabase free projekti se pauziraju posle dužeg perioda neaktivnosti.

---

## API pregled

Sve `/api/*` rute osim `/api/health` zahtevaju `x-sync-key` header (ili `?syncKey=`) kada je `SYNC_KEY` podešen.

| Metod | Ruta | Namena |
|---|---|---|
| GET | `/api/health` | Provera rada + info o provajderu |
| GET | `/api/state?date=YYYY-MM-DD` | Sve što UI treba za jedan dan (profil, obroci, treninzi, težine, slike, analitika, progres vežbi, katalog, AI podešavanja) |
| PUT | `/api/profile` | Izmena profila i ciljeva |
| POST | `/api/weight` | Upis težine za datum |
| POST | `/api/meal/manual` | Dodavanje obroka iz strukturisanih stavki |
| PUT / DELETE | `/api/meal/:id` | Izmena / brisanje obroka |
| PUT / DELETE | `/api/workout/:id` | Izmena / brisanje treninga |
| POST | `/api/ai/log` | Parsiranje slobodnog teksta i čuvanje hrane / treninga / težine / memorije |
| POST | `/api/ai/chat` | Razgovor sa coach-em uz kontekst |
| POST | `/api/ai/progress` | AI analiza poslednjih N dana |
| GET | `/api/analytics?days=N` | Proseci, zbirovi, promena težine, progres vežbi |
| GET / POST | `/api/foods` | Lista / upis namirnica u katalog |
| DELETE | `/api/foods/:id` | Brisanje namirnice iz kataloga |
| GET / PUT | `/api/settings` | AI provajder i modeli |
| POST | `/api/photos/:id/analyze` | Komentar na sliku (trenutno placeholder) |
| GET | `/api/export` | Kompletan JSON export podataka vlasnika |
| POST | `/api/import` | Isključeno – vraća `501` |
| GET | `/uploads/:filename` | Preusmerava na potpisani Storage URL (važi 10 min) |

---

## Baza podataka

Definisana u `supabase/schema.sql`:

`profile`, `weights`, `meals`, `meal_items`, `workouts`, `workout_exercises`, `progress_photos`, `ai_notes`, `ai_memory`, `food_catalog`, `app_settings` – svaka tabela ima `owner_hash`, a zavisne tabele se brišu kaskadno.

---

## Bezbednost

- Sync key je istovremeno identitet i lozinka: koristi dugačku nasumičnu vrednost i ne deli linkove koji ga sadrže.
- Server koristi Supabase **service-role** ključ koji zaobilazi Row Level Security. Izolacija se sprovodi u kodu (`owner_hash` filteri). Razmisli i o uključivanju RLS-a (bez policy-ja) na svim tabelama, da ne budu dostupne preko javnog Supabase REST API-ja.
- Ne commit-uj `.env` fajlove. Ako ključ procuri, odmah ga zameni.
- AI endpointi troše kvotu/novac: svako ko ima tvoj sync key može da ih koristi.

---

## Poznata ograničenja

- **Upload slika napretka** – UI šalje na `/api/photos`, ali server tu rutu još nema (postoje samo potpisani URL-ovi i placeholder za analizu).
- **AI analiza slika** vraća statičan placeholder; vision modeli su podešeni ali nisu povezani.
- **Import** je namerno isključen u cloud verziji; koristi Export kao backup.
- Coach i parseri su podešeni za **srpski** unos i odgovore.
- Istorija chata živi u memoriji klijenta i nestaje pri osvežavanju (poruke se čuvaju u `ai_notes`, ali se ne učitavaju nazad).

## Ideje za dalje

- Implementirati `POST /api/photos` (multer → Supabase Storage) i pravu vision analizu.
- Rad sa datumima u lokalnoj vremenskoj zoni i proseci samo po danima sa unosom.
- Network-first strategija za service worker uz verzionisana ažuriranja.
- Rate limiting, poređenje ključa u konstantnom vremenu i RLS na svim tabelama.
- Zameniti `window.prompt` izmene pravim formama; učitavati istoriju chata.

---

*Privatni projekat. Fitness i nutritivni saveti su informativni i nisu medicinski savet.*
