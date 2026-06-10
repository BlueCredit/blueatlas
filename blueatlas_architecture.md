# Blue Atlas — Backend & Database Architecture Plan

**From:** Static CSV/HTML stack on GitHub Pages  
**To:** API-backed platform with persistent database, server-side aggregation, and a fast React frontend

---

## Why the current stack is slow

Every page load on the current Blue Atlas stack does all of this in the browser:

1. Fetches 3–5 raw CSV files over HTTP (wateruse.csv is 251 KB, wateroffsets.csv is 93 KB)
2. Parses all rows with PapaParse into JS arrays
3. Filters, groups, and aggregates entirely in JS
4. Passes the results into Chart.js for rendering

On a fast connection this takes 1–3 seconds. On mobile or slower connections it can hang for 5–10 seconds. There is no caching, no pre-aggregation, and no way to serve only the data a particular chart actually needs. As the dataset grows (you currently have 2,312 wateruse rows and 597 offset rows — but you have 411 Virginia facilities with zero data yet to fill in), this will only get worse.

---

## Recommended Stack

| Layer | Technology | Why |
|---|---|---|
| **Database** | PostgreSQL 16 | Spatial support via PostGIS, excellent JSON, mature, free |
| **API** | FastAPI (Python) | You already have Python in `.venv`; fast async, auto-generates OpenAPI docs |
| **ORM / migrations** | SQLAlchemy + Alembic | Schema version control, easy DB migrations as data evolves |
| **Frontend** | React + Vite | Replace the 4 single-page HTMLs with a proper SPA; keep Chart.js and Leaflet |
| **Hosting** | Render.com or Railway.app | Free tier is fine to start; $7/mo for always-on; no DevOps knowledge required |
| **Cache** | Redis (optional, later) | Pre-compute expensive aggregations; add when needed |

PostgreSQL is the right call here over alternatives:
- **SQLite** — great for local dev but no concurrent reads, no PostGIS, no JSON aggregations
- **MySQL** — fine but PostGIS support is weaker, and Postgres is better for analytical queries (GROUP BY, window functions)
- **Supabase** — managed Postgres with a free tier and a built-in REST API; excellent choice if you want zero DevOps. Use this if you don't want to run your own server.

---

## Database Schema

```sql
-- Core lookup tables

CREATE TABLE companies (
  id          SERIAL PRIMARY KEY,
  name        TEXT NOT NULL UNIQUE,
  slug        TEXT NOT NULL UNIQUE  -- e.g. 'amazon', 'meta'
);

CREATE TABLE utilities (
  id          SERIAL PRIMARY KEY,
  name        TEXT NOT NULL UNIQUE,
  state       CHAR(2),
  huc8_basin  TEXT   -- e.g. 'Middle Potomac-Catoctin'
);

CREATE TABLE sites (
  id           SERIAL PRIMARY KEY,
  company_id   INT REFERENCES companies(id),
  utility_id   INT REFERENCES utilities(id),
  address      TEXT,
  lat          NUMERIC(9,6),
  lng          NUMERIC(9,6),
  status       TEXT DEFAULT 'confirmed',  -- 'confirmed', 'unconfirmed'
  source       TEXT,
  note         TEXT
);

-- Water use billing records (site-specific)
CREATE TABLE water_use (
  id          BIGSERIAL PRIMARY KEY,
  site_id     INT REFERENCES sites(id),
  company_id  INT REFERENCES companies(id),
  utility_id  INT REFERENCES utilities(id),
  period      DATE NOT NULL,   -- normalized to first of month
  gallons     BIGINT,
  source      TEXT,
  created_at  TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX ON water_use (company_id, period);
CREATE INDEX ON water_use (utility_id, period);

-- Utility-level aggregates (Loudoun Water — no site breakdown)
CREATE TABLE water_use_aggregate (
  id          SERIAL PRIMARY KEY,
  utility_id  INT REFERENCES utilities(id),
  type        TEXT NOT NULL,   -- 'potable', 'reclaimed'
  period      DATE NOT NULL,
  gallons     BIGINT,
  source      TEXT,
  note        TEXT
);
CREATE INDEX ON water_use_aggregate (utility_id, period, type);

-- Offset / replenishment projects
CREATE TABLE water_offsets (
  id            SERIAL PRIMARY KEY,
  company_id    INT REFERENCES companies(id),
  project_name  TEXT,
  partner       TEXT,
  watershed     TEXT,
  category      TEXT,    -- 'Recycled Water', 'Watershed Replenishment', 'Water Quality'
  period        DATE,
  gallons       BIGINT,
  lat           NUMERIC(9,6),
  lng           NUMERIC(9,6),
  source        TEXT
);
CREATE INDEX ON water_offsets (company_id, period);

-- Corporate water reports / PDF links
CREATE TABLE water_reports (
  id          SERIAL PRIMARY KEY,
  company_id  INT REFERENCES companies(id),
  report_url  TEXT,
  year        INT,
  added_at    TIMESTAMPTZ DEFAULT now()
);
```

**Key design decisions:**
- `water_use` and `water_use_aggregate` are kept separate — this was the hard-won lesson from the Loudoun rebuild. Never conflate site-specific billing with utility-level aggregates.
- All `period` columns store as `DATE` normalized to the first of month (e.g. `2023-01-01`). This makes GROUP BY year/month trivial in SQL and eliminates the Excel date parsing issues you've been hitting.
- `sites` holds all 417 Loudoun DC inventory entries; the `status` field distinguishes confirmed vs. unconfirmed locations.

---

## API Design (FastAPI)

The API replaces every `fetch("wateruse.csv")` call in the frontend. Each endpoint returns pre-aggregated JSON — no CSV parsing in the browser.

```
GET /api/v1/water-use/summary
  ?company=amazon&year=2024&utility=loudoun-water
  → { total_gallons, by_month: [...], by_company: [...] }

GET /api/v1/water-use/by-site
  ?company=amazon&year=2024
  → [{ site_id, address, lat, lng, gallons, period }]

GET /api/v1/water-use/aggregate
  ?utility=loudoun-water&type=potable&year=2024
  → { utility, type, by_month: [{ month, gallons }], total }

GET /api/v1/offsets/summary
  ?company=amazon
  → { total_gallons, by_category, by_watershed, by_year }

GET /api/v1/offsets/map
  → [{ lat, lng, company, gallons, category, period }]

GET /api/v1/sites
  ?utility=loudoun-water&company=amazon
  → [{ id, address, lat, lng, company, utility, status }]

GET /api/v1/stats
  → { total_companies, total_sites, total_gallons_tracked, coverage_pct }
```

Every endpoint that powers a chart or counter returns a pre-aggregated shape. The frontend renders; it does not compute.

---

## Project Structure

```
blueatlas/
├── backend/
│   ├── main.py              # FastAPI app entry point
│   ├── database.py          # SQLAlchemy engine + session
│   ├── models.py            # ORM table definitions
│   ├── routers/
│   │   ├── water_use.py
│   │   ├── offsets.py
│   │   └── sites.py
│   ├── migrations/          # Alembic migration files
│   │   └── versions/
│   ├── scripts/
│   │   └── seed.py          # One-time CSV → DB import
│   ├── requirements.txt
│   └── alembic.ini
│
└── frontend/
    ├── src/
    │   ├── App.jsx
    │   ├── pages/
    │   │   ├── Home.jsx
    │   │   ├── Usage.jsx
    │   │   ├── Offsets.jsx
    │   │   └── Map.jsx
    │   ├── components/
    │   │   ├── WaterUseChart.jsx
    │   │   ├── OffsetChart.jsx
    │   │   ├── AtlasMap.jsx
    │   │   └── StatCounter.jsx
    │   └── api/
    │       └── client.js    # fetch wrappers for all API endpoints
    ├── index.html
    ├── vite.config.js
    └── package.json
```

---

## Step-by-Step Migration

### Step 1 — Install PostgreSQL locally (30 min)

**Mac:**
```bash
brew install postgresql@16
brew services start postgresql@16
createdb blueatlas
```

**Windows:** Download the installer from postgresql.org — accept all defaults, note the password you set for the `postgres` user.

**Linux:**
```bash
sudo apt install postgresql-16
sudo systemctl start postgresql
sudo -u postgres createdb blueatlas
```

---

### Step 2 — Set up the Python backend (20 min)

```bash
mkdir blueatlas/backend && cd blueatlas/backend

python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate

pip install fastapi uvicorn sqlalchemy psycopg2-binary alembic pandas python-dotenv
```

Create `backend/.env`:
```
DATABASE_URL=postgresql://postgres:yourpassword@localhost:5432/blueatlas
```

Create `backend/database.py`:
```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, DeclarativeBase
import os
from dotenv import load_dotenv

load_dotenv()
engine = create_engine(os.environ["DATABASE_URL"])
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

class Base(DeclarativeBase):
    pass

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

Run Alembic to create the schema:
```bash
alembic init migrations
# edit alembic.ini: sqlalchemy.url = %(DATABASE_URL)s
alembic revision --autogenerate -m "initial schema"
alembic upgrade head
```

---

### Step 3 — Seed the database from your CSVs (30 min)

Create `backend/scripts/seed.py` — run once:

```python
import pandas as pd
from sqlalchemy.orm import Session
from database import engine
from models import Company, Utility, Site, WaterUse, WaterUseAggregate, WaterOffset

def seed():
    with Session(engine) as db:
        # 1. Insert companies
        wu = pd.read_csv("../../wateruse.csv")
        companies = {name: Company(name=name, slug=name.lower().replace(" ", "-"))
                     for name in wu["Company"].unique()}
        db.add_all(companies.values())
        db.flush()

        # 2. Insert utilities
        utils = {name: Utility(name=name, state="VA")
                 for name in wu["Utility"].unique()}
        db.add_all(utils.values())
        db.flush()

        # 3. Insert sites + water use rows
        for _, row in wu.iterrows():
            # ... upsert site, insert water_use record
            pass

        # 4. Seed loudoun aggregates
        loud = pd.read_csv("../../wateruse_loudoun.csv")
        recl = pd.read_csv("../../waterreclaimed.csv")
        # ... insert into water_use_aggregate

        # 5. Seed offsets
        wo = pd.read_csv("../../wateroffsets.csv")
        # ... insert into water_offsets

        db.commit()
        print("Seeded successfully.")

if __name__ == "__main__":
    seed()
```

The full seed script is mechanical; each CSV column maps directly to a DB column per the schema above. Provide the seed file to your AI editor (Cursor, etc.) and it will fill in the loops in minutes.

---

### Step 4 — Build the API endpoints (1–2 hrs)

Minimal working example for `backend/main.py`:

```python
from fastapi import FastAPI, Depends
from sqlalchemy.orm import Session
from sqlalchemy import func, extract
from database import get_db
from models import WaterUse, Company, Utility

app = FastAPI(title="Blue Atlas API")

@app.get("/api/v1/water-use/summary")
def water_use_summary(
    company: str = None,
    year: int = None,
    db: Session = Depends(get_db)
):
    q = db.query(
        extract("year", WaterUse.period).label("year"),
        extract("month", WaterUse.period).label("month"),
        func.sum(WaterUse.gallons).label("gallons")
    )
    if company:
        q = q.join(Company).filter(Company.slug == company)
    if year:
        q = q.filter(extract("year", WaterUse.period) == year)
    rows = q.group_by("year", "month").order_by("year", "month").all()
    return [{"year": int(r.year), "month": int(r.month), "gallons": r.gallons}
            for r in rows]
```

Run the dev server:
```bash
uvicorn main:app --reload --port 8000
# OpenAPI docs auto-available at http://localhost:8000/docs
```

---

### Step 5 — Replace CSV fetches in the frontend

Replace this pattern (current):
```javascript
const res = await fetch("wateruse.csv");
const text = await res.text();
const rows = Papa.parse(text, { header: true }).data;
const totals = rows.reduce((acc, r) => { /* ... */ }, {});
```

With this (new):
```javascript
// frontend/src/api/client.js
const BASE = import.meta.env.VITE_API_URL || "http://localhost:8000";

export async function getWaterUseSummary({ company, year } = {}) {
  const params = new URLSearchParams({ ...(company && { company }), ...(year && { year }) });
  const res = await fetch(`${BASE}/api/v1/water-use/summary?${params}`);
  return res.json();
}
```

Then in a React component:
```jsx
import { useEffect, useState } from "react";
import { getWaterUseSummary } from "../api/client";

export function WaterUseChart({ company }) {
  const [data, setData] = useState([]);
  useEffect(() => {
    getWaterUseSummary({ company }).then(setData);
  }, [company]);
  // render Chart.js chart with data
}
```

The key difference: the browser receives 1–5 KB of pre-aggregated JSON instead of 251 KB of raw CSV it has to parse and aggregate itself.

---

### Step 6 — Deploy

**Recommended free/cheap path:** [Render.com](https://render.com)

1. Push `backend/` to a GitHub repo
2. Create a new **Web Service** on Render, point it at the repo
3. Set `DATABASE_URL` as an environment variable (Render provides a free managed Postgres)
4. Build command: `pip install -r requirements.txt`
5. Start command: `uvicorn main:app --host 0.0.0.0 --port $PORT`

For the frontend:
1. `npm run build` in the `frontend/` folder produces a `dist/` folder
2. Deploy `dist/` to Netlify or Cloudflare Pages (both free) — drag and drop the folder
3. Set `VITE_API_URL=https://your-backend.onrender.com` in Netlify's environment variables

**Alternative — Supabase (zero-DevOps option):**  
Supabase gives you a hosted Postgres instance with a REST API auto-generated from your schema — you may not need FastAPI at all. The tradeoff is less control over aggregation logic; complex GROUP BY queries will need database Views.

---

## What This Gets You

| Current | After migration |
|---|---|
| 251 KB CSV downloaded every page load | 1–5 KB JSON per chart, on demand |
| All aggregation in browser JS | Aggregation in PostgreSQL (sub-10ms for your data volume) |
| Re-parses same data 4× across pages | API response caching headers, optional Redis |
| Adding a new facility = editing CSV manually | INSERT into DB, reflected instantly |
| No search or filtering | Full SQL filter support on any field |
| No ingest pipeline | `seed.py` + future FOIA data goes straight to DB |
| 5–10 second page load on mobile | Sub-1 second first meaningful paint |

---

## Rough Timeline

| Phase | Time estimate |
|---|---|
| Local Postgres + schema | 1 hour |
| Seed script (CSV → DB) | 2–3 hours |
| FastAPI with 5 core endpoints | 3–4 hours |
| React frontend (port 4 HTML pages) | 4–8 hours |
| Deploy to Render + Netlify | 1 hour |
| **Total** | **~2 focused days** |

The seed script and API endpoints are highly repetitive — any AI-assisted coding tool (Cursor, GitHub Copilot) will cut these times in half.

---

## Optional Next Steps (post-migration)

- **Materialized views** for the heaviest aggregations (annual totals by company) — refreshed nightly via a cron
- **PostGIS** extension to store site geometries properly and do spatial queries (sites within HUC-8 basin, distance to nearest utility boundary)
- **Auth layer** (Supabase Auth or Clerk) if you want a private admin interface for data entry
- **Webhook + FOIA intake pipeline** — a simple POST endpoint that accepts new billing rows from a Google Sheet or uploaded CSV, bypassing manual seed runs
- **Redis caching** on the `/stats` and `/summary` endpoints — these are called on every page load and rarely change
