<div align="center">

<img src="https://github.com/user-attachments/assets/7e3176b3-a580-44b6-a8a8-6e47134163a1" alt="Vendgram / S T P" width="12000"/>

# **S T P**
_A platform for clean, comparable commodity prices — query **standard** prices and **location‑specific** prices across Nigeria (state → city → market)._

**Service:** S T P API (AI) • v4.1.0

</div>

---

## TL;DR
- Ingest prices from CSV/PDF/Image or manual entry → **normalize units & currency** → validate → store.
- Users query:
  - **Standard price** (robust aggregates like median per window/location).
  - **Specific price** (latest per location with canonical unit/currency).
- API-first FastAPI service with SQLite by default; optional Gemini-powered extraction/budget tips.

---


# S T P — Data Flow & Model (README)

This document captures the **end‑to‑end data flow**, **request/response lifecycles**, and **data model** for the S T P Platform. It’s designed to be pasted directly into your project’s `README.md` on GitHub.

---

## Data Flow (End-to-End)

The system has two main paths: **User path** (read-only queries for prices/history/budgeting) and **Admin path** (data ingestion $\rightarrow$ normalization $\rightarrow$ storage $\rightarrow$ aggregation).

```mermaid
flowchart LR
  subgraph Frontend
    U[User UI<br/>frontend/user.html]
    A[Admin UI<br/>frontend/admin.html]
  end

  subgraph API
    R1[User Router<br/>backend/routers/user.py]
    R2[Admin Router<br/>backend/routers/admin.py]
    AI[AI Helpers<br/>backend/ai.py]
    ING[Ingest Service<br/>standardize_and_store<br/>backend/services/ingest.py]
  end

  subgraph Database
    L[Locations]
    S[StagingPrice]
    M[MainPrice]
    G[AggregateDaily]
  end

  %% User path (one edge per line)
  U -->|GET /commodities,/brands,/units,/locations| R1
  U -->|GET /prices history| R1
  U -->|POST /budget optional| R1
  R1 -->|READ| M
  R1 -->|READ trends| G
  R1 --> AI

  %% Admin path (split preview into two one-way edges)
  A -->|POST /admin/parse-csv| R2
  R2 --> AI
  A -->|POST /admin/parse-pdf| R2
  R2 --> AI
  A -->|POST /admin/parse-image| R2
  R2 --> AI
  A -->|Preview parsed rows| R2
  R2 -->|Preview parsed rows| A
  A -->|POST /admin/commit selected rows plus source| R2
  R2 --> ING
  ING -->|UPSERT| L
  ING -->|WRITE raw| S
  ING -->|WRITE normalized| M
  ING -->|UPDATE daily medians| G



```

### What happens

- **Admin “parse”**: CSV/PDF/Image is parsed (optionally via Gemini) → preview rows with flags (valid/outlier).
- **Admin “commit”**: Accepted rows go through `standardize_and_store()`:
  - currency/unit parsing & normalization (`backend/utils.py`)
  - location upsert (`backend/validators.py`)
  - write to **StagingPrice** (raw) and **MainPrice** (normalized NGN + base unit)
  - recompute **AggregateDaily** median for *commodity × location × day*
- **User queries** read from **MainPrice** and **AggregateDaily** to render lists and charts.

---

## Request/Response Lifecycle (Typical)

### User: Query prices for a commodity/location/date range

```mermaid
sequenceDiagram
  autonumber
  participant User as User UI
  participant API as FastAPI (user router)
  participant DB as DB (MainPrice, AggregateDaily)

  User->>API: GET /prices?commodity=rice&state=Lagos&range=P30D
  API->>DB: SELECT (MainPrice / AggregateDaily) with filters
  DB-->>API: rows (normalized NGN + unit, daily medians)
  API-->>User: JSON (list + stats for charts)
```

### Admin: Parse → Preview → Commit

```mermaid
sequenceDiagram
  autonumber
  participant Admin as Admin UI
  participant API as FastAPI (admin router)
  participant AI as backend/ai.py
  participant ING as services/ingest.py
  participant DB as DB (Locations, StagingPrice, MainPrice, AggregateDaily)

  Admin->>API: POST /admin/parse-csv (file)
  API->>AI: extract_rows_from_file()
  AI-->>API: parsed rows (commodity, price_text, unit_text, ...)
  API-->>Admin: preview (accepted/rejected/errors)

  Admin->>API: POST /admin/commit (rows, source)
  API->>ING: standardize_and_store(row, source)
  ING->>DB: UPSERT Location
  ING->>DB: INSERT StagingPrice (raw + flags)
  ING->>DB: INSERT MainPrice (normalized NGN + unit)
  ING->>DB: UPSERT AggregateDaily (median, n)
  API-->>Admin: commit results (ids, standardized payload)
```

---


## Normalization & Validation (where it happens)

- **Price & currency parsing**: `parse_price_and_currency()`  
- **Unit cleaning**: `clean_unit()` → `(unit_value, unit_name)`  
- **Outlier flagging**: `iqr_outlier()` vs historical staging values  
- **Location upsert**: `validate_or_upsert_location()`  
- **Commit logic**: `standardize_and_store()` writes **StagingPrice** and **MainPrice**, then recomputes **AggregateDaily** medians.

---

## Implementation References

- `backend/services/ingest.py`  
- `backend/utils.py`  
- `backend/validators.py`  
- `backend/routers/admin.py`, `backend/routers/user.py`



## CSV Template (Minimum)

```csv
commodity,brand,price_text,unit_text,unit_value,unit_name,currency,state,city,market,date_uploaded,note
rice,Mama Gold,"75000",bag,50,kg,NGN,Lagos,Ikeja,Onigbongbo,2025-09-29,Retail quote
sugar,,95000,kg,1,kg,NGN,Kaduna,Chikun,Barnawa,2025-09-28,Wholesaler
cement,Dangote,7200,bag,50,kg,NGN,Abuja,AMAC,Garki,2025-09-30,Dangote 50kg
groundnut oil,,12000,liter,1,liter,NGN,Oyo,Ibadan North,Bodija,2025-09-30,Branded
```

- `price_text` may contain symbols/commas — the parser extracts the number & currency.
- **Always include `currency` and proper `unit_text`/`unit_value`** to prevent “75,000 becomes 750” type errors.
- `date_uploaded` in `YYYY-MM-DD` (defaults to “today” if omitted).

---

## Admin Ingestion Flow

1. **Parse**: Upload CSV/PDF/Image for preview parsing (`/admin/parse-csv`, `/admin/parse-pdf`, `/admin/parse-image`).
2. **Review**: The API returns standardized rows (accepted/rejected/errors). Frontend shows a preview.
3. **Commit**: Post selected rows to `/admin/commit` with a `source` label to persist into `MainPrice` and update `AggregateDaily`.

Manual single-entry is also supported (see `schemas.ManualEntry` and the admin router).



## Tech Stack

<div id="tech-stack"></div>

### Core

<table>
<tr>
  <td align="center">
    <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/python/python-original.svg" height="36"><br>
    Python 3.11+
  </td>
  <td align="center">
    <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/fastapi/fastapi-original.svg" height="36"><br>
    FastAPI
  </td>
  <td align="center">
    <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/sqlalchemy/sqlalchemy-original.svg" height="36"><br>
    SQLModel/SQLAlchemy
  </td>
  <td align="center">
    <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/sqlite/sqlite-original.svg" height="36"><br>
    SQLite (dev)
  </td>
  <td align="center">
    <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/postgresql/postgresql-original.svg" height="36"><br>
    Postgres (prod)
  </td>
</tr>
<tr>
  <td align="center">
    <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/html5/html5-original.svg" height="36"><br>
    HTML
  </td>
  <td align="center">
    <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/javascript/javascript-original.svg" height="36"><br>
    Vanilla JS
  </td>
  <td align="center">
    <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/tailwindcss/tailwindcss-plain.svg" height="36"><br>
    TailwindCSS
  </td>
  <td align="center">
    <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/docker/docker-original.svg" height="36"><br>
    Docker
  </td>
  <td align="center">
    <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/githubactions/githubactions-original.svg" height="36"><br>
    GitHub Actions
  </td>
</tr>
</table>

## Typical stack :

**Backend / API**  
[![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=for-the-badge&logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/) 
[![Python](https://img.shields.io/badge/Python_3.11+-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://www.python.org/)

**DB / Warehouse**  
[![SQLite](https://img.shields.io/badge/SQLite-003B57?style=for-the-badge&logo=sqlite&logoColor=white)](https://www.sqlite.org/) 
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL_(prod)-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](https://www.postgresql.org/) 
![PostGIS](https://img.shields.io/badge/PostGIS-8BB7A3?style=for-the-badge)  

**Object Store**  
![Local FS (dev)](https://img.shields.io/badge/Local_FS-(dev)-6B7280?style=for-the-badge) 
[![MinIO](https://img.shields.io/badge/MinIO-C72E49?style=for-the-badge&logo=minio&logoColor=white)](https://min.io/) 
[![Amazon S3](https://img.shields.io/badge/Amazon_S3-569A31?style=for-the-badge&logo=amazons3&logoColor=white)](https://aws.amazon.com/s3/)

**Orchestration (optional)**  
[![Airflow](https://img.shields.io/badge/Airflow-017CEE?style=for-the-badge&logo=apache-airflow&logoColor=white)](https://airflow.apache.org/) 
[![Prefect](https://img.shields.io/badge/Prefect-1A2B6B?style=for-the-badge&logo=prefect&logoColor=white)](https://www.prefect.io/) 
[![Dagster](https://img.shields.io/badge/Dagster-1E4FFF?style=for-the-badge&logo=dagster&logoColor=white)](https://dagster.io/)

**Transforms (optional)**  
[![dbt](https://img.shields.io/badge/dbt-FF694B?style=for-the-badge&logo=dbt&logoColor=white)](https://www.getdbt.com/) 
[![Apache Spark](https://img.shields.io/badge/Apache_Spark-E25A1C?style=for-the-badge&logo=apachespark&logoColor=white)](https://spark.apache.org/) 
[![Polars](https://img.shields.io/badge/Polars-3643A8?style=for-the-badge&logo=polars&logoColor=white)](https://www.pola.rs/)

**Validation / Data Quality (optional)**  
[![Great Expectations](https://img.shields.io/badge/Great_Expectations-0A0A23?style=for-the-badge&logo=greatexpectations&logoColor=white)](https://greatexpectations.io/) 
[![Pandera](https://img.shields.io/badge/Pandera-3775A9?style=for-the-badge&logo=python&logoColor=white)](https://pandera.readthedocs.io/) 
[![Soda](https://img.shields.io/badge/Soda-24B47E?style=for-the-badge&logo=soda&logoColor=white)](https://docs.soda.io/)

**Frontend**  
[![HTML5](https://img.shields.io/badge/HTML5-E34F26?style=for-the-badge&logo=html5&logoColor=white)](https://developer.mozilla.org/docs/Web/HTML) 
[![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=for-the-badge&logo=javascript&logoColor=000000)](https://developer.mozilla.org/docs/Web/JavaScript) 
[![TailwindCSS](https://img.shields.io/badge/TailwindCSS-38BDF8?style=for-the-badge&logo=tailwindcss&logoColor=white)](https://tailwindcss.com/)

**Auth (optional)**  
![JWT](https://img.shields.io/badge/JWT-000000?style=for-the-badge&logo=jsonwebtokens&logoColor=white) 
![OAuth 2.0](https://img.shields.io/badge/OAuth_2.0-3D5AFE?style=for-the-badge) 
[![Supabase](https://img.shields.io/badge/Supabase-3FCF8E?style=for-the-badge&logo=supabase&logoColor=000000)](https://supabase.com/)

**Observability (optional)**  
[![OpenTelemetry](https://img.shields.io/badge/OpenTelemetry-683D87?style=for-the-badge&logo=opentelemetry&logoColor=white)](https://opentelemetry.io/) 
[![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=for-the-badge&logo=prometheus&logoColor=white)](https://prometheus.io/) 
[![Grafana](https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white)](https://grafana.com/)

**CI / CD**  
[![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?style=for-the-badge&logo=githubactions&logoColor=white)](https://github.com/features/actions)

**Containers**  
[![Docker](https://img.shields.io/badge/Docker-0db7ed?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/) 
![Docker Compose](https://img.shields.io/badge/Docker_Compose-384D54?style=for-the-badge&logo=docker&logoColor=white)


### Optional

- **AI Parsing**: Google Gemini (Vision/Text)
- **Charts**: Chart.js  
- **Validation**: Pandera / Great Expectations  
- **Orchestration**: Airflow / Prefect / Dagster  
- **Observability**: OpenTelemetry • Prometheus • Grafana  

> You can swap pieces freely; this repo keeps defaults minimal.

---


## User Features

- **Browse commodities**: `/commodities`, `/brands`, `/units`, `/locations`
- **Specific price**: latest normalized price for a commodity at a given location
- **Standard price**: robust aggregates (daily medians etc.) via `AggregateDaily`
- **Budgeting**: `/budget` — simple cart optimizer; may return a short Gemini tip if configured

Frontend (`user.html`) includes **Chart.js** for trends and filters for location/commodity.

---

## Running Notes

- First run creates SQLite tables and seeds locations (see `backend/database.py`).
- If you switch to Postgres, set `PRICING_DB_URL` accordingly and ensure drivers are installed.
- CORS is open (`*`) by default; tighten for production.

---

## Troubleshooting

- **Empty `requirements.txt`**: install the minimal stack using the command above.
- **Price scales look wrong**: confirm `unit_value`, `unit_name`, and `currency` are present in your CSV.
- **No rows after parse**: check file format/headers. The admin router contains header synonym mappings to auto-map common column names.
- **AI parsing slow/unavailable**: reduce `AI_MAX_CHARS`, increase timeout slightly, or disable Gemini usage.

---

## Roadmap (suggested)

- Historical FX table & per-day normalization
- Confidence scoring per source
- Geo-aware queries (nearest markets)
- Alerts on spikes/drops
- Role-based auth for admin routes

---

## License
MIT (see `LICENSE` if present).





