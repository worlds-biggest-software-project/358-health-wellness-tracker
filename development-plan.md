# Health & Wellness Tracker — Phased Development Plan

> Project: 358-health-wellness-tracker · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and `data-model-suggestion-1.md` (Entity-Centric Normalized Relational) into a concrete, phased build. The product is an open-source, self-hostable, AI-native health tracker that unifies **nutrition, exercise, sleep, symptoms, and biometrics** and reasons across them with an LLM to surface personalised, evidence-grounded insights.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | **Python 3.12** | The core value is LLM-driven cross-domain reasoning and correlation analytics over time-series health data. Python has the richest ecosystem for statistics (`scipy`, `pandas`), LLM SDKs, and data normalisation, and matches the dominant open-source health stack (Wger is Django; Open Wearables is Python-friendly). |
| API framework | **FastAPI** | Async-native (needed for concurrent wearable API fan-out), produces an **OpenAPI 3.1** document automatically (required by `standards.md`), and integrates Pydantic v2 for typed request/response validation that mirrors the typed columns of Data Model 1. |
| Data validation | **Pydantic v2** | Single source of truth for request/response schemas, config, and FHIR mapping models. CHECK constraints in the DB are mirrored as Pydantic validators. |
| Database | **PostgreSQL 16** | Data Model 1 relies on `NUMERIC` precision, `CHECK` constraints, `TEXT[]` arrays, GIN full-text indexes, `JSONB` for audit diffs, and **declarative range partitioning** by date for indefinitely growing health logs. SQLite cannot express these. Postgres also gives `pgcrypto`/`gen_random_uuid()` used throughout the schema. |
| ORM / DB access | **SQLAlchemy 2.0 (async) + Alembic** | Async ORM matches FastAPI; Alembic handles migrations and the creation of time-range partitions. Raw SQL is used for the correlation queries that benefit from `JOIN LATERAL`. |
| Migrations | **Alembic** | Versioned, reversible migrations; partition management scripts live alongside. |
| Task queue | **Celery + Redis** | Wearable sync (OAuth-authenticated polling of Oura/Fitbit/WHOOP/Withings), WHOOP webhook processing, AI insight generation, and PDF export are long-running/async. Redis doubles as the result backend and rate-limit store. |
| Scheduler | **Celery Beat** | Periodic wearable sync, nightly insight generation, and adaptive-goal recomputation. |
| LLM access | **Provider-abstracted client** (Anthropic Claude default; OpenAI optional; Ollama for on-device/self-hosted) | `research.md` and the README mandate "on-device processing preferred for sensitive data." An abstraction layer lets self-hosters run a local model (Ollama) while SaaS deployments use a hosted model. |
| Correlation engine | **pandas + scipy.stats** | Statistical pre-computation (Pearson/Spearman correlations, lag analysis) runs deterministically before the LLM narrates findings — grounding insights in real evidence and reducing token cost/hallucination. |
| Food data | **USDA FoodData Central API** (public domain) + **Open Food Facts API** (ODbL, barcode lookups) | `standards.md` mandates openly licensed sources; no proprietary scraping. |
| Photo recognition | **LLM vision (multimodal)** behind the same provider abstraction | Avoids a bespoke CV model; reuses the LLM client. Confidence stored in `nutrition_entries.ai_confidence`. |
| Auth | **OAuth 2.0 Authorization Code + PKCE (RFC 7636)**, **OpenID Connect**, JWT sessions | `standards.md` mandates PKCE for all wearable flows and OIDC for user identity; HEART profile principles applied to clinician sharing. |
| Frontend | **Next.js 15 (React, TypeScript) + Tailwind + shadcn/ui + Recharts** | A dashboard-heavy, chart-heavy SPA-style web UI (unified daily dashboard, 30/90/365-day trends). Recharts covers longitudinal trend charts. Mobile-native apps are out of scope for the reference implementation; the web app is responsive and offline-capable via a service worker. |
| Charts | **Recharts** | Declarative React charts for the longitudinal trend views. |
| PDF export | **WeasyPrint** | HTML/CSS → PDF for the clinician-ready medical appointment summary; easy to template and style. |
| FHIR mapping | **`fhir.resources` (Pydantic FHIR R5)** | Maps biometric/sleep/exercise entries to FHIR `Observation` resources for the export and future SMART-on-FHIR integration. |
| MCP server | **`mcp` Python SDK** | Exposes health data as MCP tools so any MCP-compatible assistant can query it (per `standards.md` MCP section). |
| Containerisation | **Docker + docker-compose** | Self-hosted deployment is a primary mode (README). Compose orchestrates api, worker, beat, postgres, redis, web. |
| Testing | **pytest + pytest-asyncio + httpx + testcontainers** | `testcontainers` spins up real Postgres for integration tests; `respx`/`vcr.py` mock external HTTP (wearable + food APIs). Frontend uses **Vitest + Playwright**. |
| Code quality | **ruff (lint+format), mypy (strict), pre-commit** | Standard modern Python toolchain; mypy enforces the typed contracts. |
| Package manager | **uv** | Fast, reproducible Python dependency management with lockfile. |
| Secrets / config | **pydantic-settings + .env**; OAuth tokens encrypted at rest (AES-256 via `pgcrypto` or app-layer Fernet) | HIPAA-aligned: tokens are never stored in plaintext; `oauth_token_ref` references an encrypted secret store. |

### Project Structure

```
health-wellness-tracker/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── .env.example
├── README.md
├── openapi.json                       # exported OpenAPI 3.1 contract (CI artifact)
├── migrations/
│   ├── env.py
│   └── versions/                      # alembic revisions + partition DDL
├── src/
│   └── hwt/
│       ├── main.py                    # FastAPI app factory, router mounting
│       ├── config.py                  # pydantic-settings Settings
│       ├── db/
│       │   ├── session.py             # async engine + session factory
│       │   ├── base.py                # declarative base
│       │   ├── models/                # SQLAlchemy models (1 module per dimension)
│       │   │   ├── user.py
│       │   │   ├── wearable.py
│       │   │   ├── food.py
│       │   │   ├── nutrition.py
│       │   │   ├── exercise.py
│       │   │   ├── sleep.py
│       │   │   ├── symptom.py
│       │   │   ├── biometric.py
│       │   │   ├── goal.py
│       │   │   ├── insight.py
│       │   │   ├── share.py
│       │   │   └── audit.py
│       │   └── partitions.py          # helpers to create monthly range partitions
│       ├── schemas/                   # Pydantic request/response models
│       ├── api/
│       │   ├── deps.py                # auth, db session, current_user deps
│       │   ├── routes/                # one router per resource
│       │   │   ├── auth.py
│       │   │   ├── nutrition.py
│       │   │   ├── exercise.py
│       │   │   ├── sleep.py
│       │   │   ├── symptoms.py
│       │   │   ├── biometrics.py
│       │   │   ├── foods.py
│       │   │   ├── goals.py
│       │   │   ├── dashboard.py
│       │   │   ├── trends.py
│       │   │   ├── insights.py
│       │   │   ├── wearables.py
│       │   │   ├── export.py
│       │   │   └── shares.py
│       ├── services/                  # business logic (dimension-agnostic)
│       │   ├── nutrition_service.py
│       │   ├── normalization.py       # unit/timestamp harmonisation
│       │   ├── dashboard_service.py
│       │   ├── trends_service.py
│       │   ├── goals_service.py
│       │   └── export_service.py
│       ├── integrations/
│       │   ├── food/
│       │   │   ├── usda.py
│       │   │   └── openfoodfacts.py
│       │   ├── wearables/
│       │   │   ├── base.py            # WearableProvider protocol
│       │   │   ├── oura.py
│       │   │   ├── fitbit.py
│       │   │   ├── whoop.py
│       │   │   ├── withings.py
│       │   │   └── healthkit_ingest.py  # accepts client-pushed HealthKit/HealthConnect payloads
│       │   └── fhir/
│       │       └── mapper.py
│       ├── ai/
│       │   ├── llm_client.py          # provider abstraction
│       │   ├── correlation.py         # scipy/pandas pre-computation
│       │   ├── prompts.py             # prompt templates
│       │   ├── insight_engine.py      # orchestrates correlation -> LLM -> insight
│       │   └── query.py               # natural-language query over health record
│       ├── security/
│       │   ├── crypto.py              # token encryption (Fernet/AES-256)
│       │   ├── jwt.py
│       │   ├── oauth_pkce.py
│       │   └── audit.py               # audit_log writer
│       ├── workers/
│       │   ├── celery_app.py
│       │   ├── sync_tasks.py
│       │   ├── insight_tasks.py
│       │   └── export_tasks.py
│       └── mcp/
│           └── server.py              # MCP tool definitions
├── tests/
│   ├── conftest.py                    # testcontainers Postgres, async client fixtures
│   ├── fixtures/                      # sample USDA/OFF/Oura/WHOOP payloads, sample diaries
│   ├── unit/
│   ├── integration/
│   └── e2e/
└── web/                               # Next.js frontend
    ├── package.json
    ├── app/
    ├── components/
    ├── lib/api-client.ts              # generated from openapi.json
    └── tests/
```

The structure groups by **concern** (models, schemas, routes, services, integrations, ai, security). Each phase adds modules without restructuring; the per-dimension model/route layout means a new health dimension is a new file set, never a rewrite.

---

## Phase 1: Foundation — Project Skeleton, Config, DB, Auth

### Purpose
Establish the runnable backbone: a FastAPI app, Postgres with the core schema, migrations, containerisation, and user authentication. After this phase the system can register users, authenticate them, and serve a health-checked, OpenAPI-documented API — the platform every other phase plugs into.

### Tasks

#### 1.1 — Project scaffolding and tooling

**What**: Create the repository skeleton with `uv`, FastAPI app factory, config, linting, and Docker.

**Design**:
- `pyproject.toml` declares dependencies (fastapi, uvicorn, sqlalchemy[asyncio], asyncpg, alembic, pydantic, pydantic-settings, celery, redis, httpx, scipy, pandas, weasyprint, fhir.resources, mcp, pyjwt, cryptography) and dev deps (pytest, pytest-asyncio, ruff, mypy, testcontainers, respx).
- `src/hwt/config.py`:
  ```python
  class Settings(BaseSettings):
      database_url: str
      redis_url: str = "redis://localhost:6379/0"
      jwt_secret: str
      jwt_algorithm: str = "HS256"
      access_token_ttl_min: int = 30
      refresh_token_ttl_days: int = 30
      token_encryption_key: str            # base64 Fernet key for wearable tokens
      llm_provider: Literal["anthropic","openai","ollama"] = "anthropic"
      llm_model: str = "claude-opus-4-8"
      llm_api_key: str | None = None
      usda_api_key: str | None = None
      data_region: str = "us"
      model_config = SettingsConfigDict(env_file=".env", env_prefix="HWT_")
  ```
- `src/hwt/main.py` exposes `create_app()` that mounts routers, adds CORS, exception handlers, and a `GET /health` endpoint returning `{"status":"ok","version":<str>}`.
- `Dockerfile` (multi-stage, uv install → slim runtime). `docker-compose.yml` services: `api`, `postgres:16`, `redis:7`.
- `.env.example` lists every `HWT_*` variable.

**Testing**:
- `Unit: Settings loads from env → required fields populated, defaults applied`
- `Unit: Settings missing HWT_DATABASE_URL → ValidationError naming the field`
- `Integration: GET /health → 200, {"status":"ok"}`
- `Integration: docker compose up → api container healthy, /health reachable`
- `CI: ruff check, ruff format --check, mypy --strict all pass`

#### 1.2 — Core data model: users, foods, audit + migration tooling

**What**: Implement the `users`, `foods`, and `audit_log` tables from Data Model 1 with Alembic migrations and partition helpers.

**Design**:
- SQLAlchemy models mirror the DDL in `data-model-suggestion-1.md` exactly (column types, CHECK constraints as `CheckConstraint`, GIN index on `foods.name`).
- `audit_log` and all future entry tables are **range-partitioned by date** (`logged_date` or `created_at`). `src/hwt/db/partitions.py`:
  ```python
  async def ensure_month_partition(conn, table: str, year: int, month: int) -> None:
      """CREATE TABLE IF NOT EXISTS <table>_yYYYYmMM PARTITION OF <table>
         FOR VALUES FROM ('YYYY-MM-01') TO (next month)."""
  ```
- First Alembic revision creates `users`, `foods`, `audit_log` (partitioned parent + current+next month partitions) and the `pgcrypto` extension.
- `security/audit.py`:
  ```python
  async def record_audit(session, *, user_id, actor_type, action,
                         entity_type, entity_id, changes=None,
                         ip=None, user_agent=None) -> None: ...
  ```

**Testing**:
- `Integration (real Postgres via testcontainers): run alembic upgrade head → tables + partitions exist; downgrade base → clean`
- `Unit: foods CHECK on source rejects 'bogus_source' (IntegrityError)`
- `Integration: ensure_month_partition twice → idempotent, no error`
- `Integration: record_audit writes a row routed to the correct monthly partition`

#### 1.3 — User authentication (OIDC + JWT + PKCE-ready)

**What**: Email/password and OIDC registration/login issuing JWT access + rotating refresh tokens.

**Design**:
- Endpoints:
  - `POST /auth/register` → body `{email, password, display_name, timezone?, units_system?}` → 201 `{user, access_token, refresh_token}`
  - `POST /auth/login` → `{email, password}` → 200 token pair
  - `POST /auth/refresh` → `{refresh_token}` → new pair (old refresh invalidated — rotation per RFC 9700)
  - `GET /auth/me` → current user profile
  - OIDC: `GET /auth/oidc/start` (PKCE code_challenge), `GET /auth/oidc/callback`
- Passwords hashed with `argon2`. Refresh tokens stored hashed in Redis with TTL; rotation invalidates the prior token.
- `api/deps.py::current_user` decodes JWT, loads user, attaches to request.
- Registration sets `gdpr_consent_at = now()` and `gdpr_data_region = settings.data_region`.

**Testing**:
- `Integration: register new email → 201, user persisted, gdpr_consent_at set`
- `Integration: register duplicate email → 409`
- `Integration: login wrong password → 401, generic message (no user enumeration)`
- `Integration: refresh with rotated (already used) token → 401`
- `Unit: argon2 hash verifies correct password, rejects wrong`
- `Integration: protected route without token → 401; with valid token → 200`

### Definition of Done
All tasks implemented; alembic up/down clean on real Postgres; auth flow e2e passes; ruff/mypy green; docker compose boots; `/health` and `/auth/*` appear in `openapi.json`.

---

## Phase 2: Manual Logging Across All Dimensions

### Purpose
Deliver the core data-capture loop. Implement the five health-dimension entry tables and CRUD APIs so a user can manually log nutrition, exercise, sleep, symptoms, and biometrics. This is the substrate the dashboard, trends, and AI engine all read from — nothing AI-related works without data in these tables.

### Tasks

#### 2.1 — Dimension models and migrations

**What**: Implement `nutrition_entries`, `exercise_entries`, `sleep_entries`, `symptom_entries`, `biometric_entries`, `recipes`, `recipe_ingredients` per Data Model 1.

**Design**:
- All five entry tables are `PARTITION BY RANGE (logged_date)`; Alembic revision creates parents + current/next month partitions; a Celery Beat task pre-creates next-month partitions.
- Pydantic create/read schemas mirror each table. Example:
  ```python
  class SymptomCreate(BaseModel):
      symptom_type: Literal["energy","mood","headache","digestive","pain",
                            "nausea","brain_fog","anxiety","insomnia","skin",
                            "respiratory","other"]
      severity: conint(ge=1, le=10)
      logged_date: date
      logged_at: datetime | None = None
      body_location: str | None = None
      duration_minutes: int | None = None
      notes: str | None = None
      tags: list[str] = []
  ```
- `nutrition_entries` carries denormalised portion-adjusted macros so a deleted/edited `foods` row never corrupts history.

**Testing**:
- `Integration: alembic upgrade creates all 7 tables + partitions`
- `Unit: SymptomCreate severity=0 → ValidationError; severity=11 → ValidationError`
- `Unit: ExerciseCreate exercise_type='moonwalk' → ValidationError (not in enum)`
- `Integration: insert nutrition entry with logged_date in next month → routed to correct partition`

#### 2.2 — CRUD routes for each dimension

**What**: REST CRUD for nutrition, exercise, sleep, symptom, biometric entries.

**Design**:
- Per dimension: `POST /{dim}`, `GET /{dim}?from=&to=&type=`, `GET /{dim}/{id}`, `PATCH /{dim}/{id}`, `DELETE /{dim}/{id}`. All scoped to `current_user`; writes call `record_audit`.
- List endpoints accept `from`/`to` dates (default: last 7 days), paginate, and filter by type.
- Generic service helper enforces ownership (404 if entry belongs to another user — no leakage).

**Testing**:
- `Integration: POST sleep entry → 201, GET returns it`
- `Integration: GET another user's entry by id → 404`
- `Integration: PATCH symptom severity → audit_log row written with changes_json diff`
- `Integration: list with from/to → only in-range entries returned`
- `Integration (mocked clock): default range = last 7 days`

#### 2.3 — Data normalisation service

**What**: Harmonise units and timestamps across manual entries (and later, wearable imports).

**Design**:
- `services/normalization.py`:
  ```python
  def to_metric(value: float, unit: str) -> tuple[float, str]: ...   # lb->kg, mi->km, F->C
  def normalize_timestamp(ts: datetime, user_tz: str) -> datetime:   # -> UTC, tz-aware
  def derive_logged_date(ts: datetime, user_tz: str) -> date:        # local calendar date
  ```
- All entries persist UTC timestamps; `logged_date` is the user's **local** calendar date (critical for "today's dashboard" correctness across timezones).
- Imperial-input users have values converted on write; storage is always metric.

**Testing**:
- `Unit: to_metric(150,'lb') → (68.04,'kg')`
- `Unit: normalize_timestamp at 23:30 America/New_York → correct UTC instant`
- `Unit: derive_logged_date for 00:30 local → local date, not UTC date`
- `Unit: round-trip metric→imperial→metric within tolerance`

### Definition of Done
All five dimensions fully CRUD-able; ownership enforced; normalisation unit-tested; audit rows written on every mutation; all endpoints in `openapi.json`.

---

## Phase 3: Nutrition Intelligence — Food Database, Search, Barcode, Photo AI

### Purpose
Nutrition is the highest-friction, highest-value logging path (per `features.md`, manual logging friction is the top abandonment cause). This phase connects the open food databases and adds three low-friction logging methods — search, barcode, and AI photo recognition — making nutrition logging fast and accurate without proprietary data.

### Tasks

#### 3.1 — USDA + Open Food Facts integration and food cache

**What**: Search foods via USDA FoodData Central and resolve barcodes via Open Food Facts, caching results into the `foods` table.

**Design**:
- `integrations/food/usda.py`:
  ```python
  async def search(query: str, page: int = 1) -> list[FoodHit]: ...  # GET /v1/foods/search
  async def get_by_fdc_id(fdc_id: int) -> FoodDetail: ...
  ```
- `integrations/food/openfoodfacts.py`:
  ```python
  async def get_by_barcode(barcode: str) -> FoodDetail | None: ...   # GET /api/v2/product/{barcode}
  ```
- Mapper normalises both sources into the `foods` column set (calories, macros, key micronutrients), tagging `source` = `usda_fdc` / `open_food_facts`. First lookup caches into `foods`; subsequent lookups hit the local cache (and GIN-indexed name search).
- Endpoints: `GET /foods/search?q=`, `GET /foods/barcode/{code}`, `POST /foods` (user_created), `GET /foods/{id}`.

**Testing**:
- `Integration (respx-mocked USDA): search 'banana' → hits mapped to Food schema, cached in foods`
- `Integration (respx-mocked OFF): barcode found → Food returned, source=open_food_facts`
- `Integration (mocked OFF 404): unknown barcode → 404`
- `Integration: second search for same term → served from cache, no outbound HTTP (respx asserts no call)`
- `Unit: USDA payload with missing micronutrient → null columns, no crash`

#### 3.2 — Recipe builder

**What**: Compose recipes from foods and log a recipe as a single nutrition entry.

**Design**:
- `POST /recipes` `{name, servings, ingredients:[{food_id, quantity, serving_size_g}]}` computes totals and stores `recipes` + `recipe_ingredients`.
- `POST /nutrition/from-recipe` `{recipe_id, servings, meal_type, logged_date}` creates a `nutrition_entries` row with `log_method='recipe'` and portion-scaled macros.

**Testing**:
- `Integration: create recipe with 3 ingredients → totals = sum of scaled ingredient macros`
- `Integration: log 0.5 servings of recipe → nutrition entry macros halved`
- `Unit: recipe with food_id not owned/not existing → 422`

#### 3.3 — AI photo meal recognition

**What**: Estimate foods and portions from a meal photo using LLM vision.

**Design**:
- `POST /nutrition/photo` (multipart image) → worker task → LLM vision call → returns candidate items with estimated grams and macros and an `ai_confidence` (0–1). User confirms/edits before persistence.
- Prompt template (`ai/prompts.py::FOOD_PHOTO_PROMPT`): instructs the model to return strict JSON: `{"items":[{"name","estimated_grams","calories_kcal","protein_g","fat_g","carbs_g","confidence"}]}`. Response validated against a Pydantic model; low-confidence (<0.4) items flagged.
- Persisted entries use `log_method='photo_ai'`, store `photo_url` and `ai_confidence`. Created foods get `source='ai_estimated'`, `is_verified=false`.

**Testing**:
- `Integration (mocked LLM): photo → parsed items mapped to draft entries`
- `Unit: LLM returns malformed JSON → ValidationError surfaced as 502 with retry, no DB write`
- `Unit: confidence 0.3 → item flagged low_confidence=true`
- `E2E (mocked LLM): upload photo → confirm draft → nutrition entry persisted with photo_ai method`

### Definition of Done
Search/barcode/photo all functional with mocked externals; food cache populated and reused; recipes compute correct totals; photo AI produces validated, user-confirmable drafts; openapi updated.

---

## Phase 4: Daily Dashboard & Longitudinal Trends

### Purpose
Make the captured data visible and useful. A unified daily dashboard (the product's home screen) and interactive 30/90/365-day trend charts turn raw logs into the at-a-glance and longitudinal views that every incumbent provides — and that the AI engine's narratives will reference.

### Tasks

#### 4.1 — Dashboard aggregation API

**What**: Single endpoint returning today's state across all dimensions.

**Design**:
- `GET /dashboard?date=YYYY-MM-DD` (default: user-local today) → runs the parallel per-dimension queries from Data Model 1's "Daily dashboard" example, concurrently via `asyncio.gather`.
  ```python
  class DashboardResponse(BaseModel):
      date: date
      nutrition: NutritionDaySummary   # totals + per-meal breakdown + goal progress
      sleep: SleepSummary | None
      exercise: list[ExerciseSummary]
      symptoms: list[SymptomSummary]
      biometrics: list[BiometricSummary]
      goal_progress: list[GoalProgress] # filled once Phase 6 lands; empty list before
  ```

**Testing**:
- `Integration: dashboard for date with entries in all dimensions → aggregated correctly`
- `Integration: dashboard for empty date → zeroed nutrition, null sleep, empty lists, 200`
- `Integration: nutrition totals = SUM across meals (matches per-meal breakdown)`
- `Performance: dashboard query set completes < 200ms on 1-year seeded dataset`

#### 4.2 — Trends API

**What**: Time-bucketed series for any metric over 30/90/365 days.

**Design**:
- `GET /trends/{dimension}/{metric}?window=30|90|365&bucket=day|week|month` → `{points:[{date, value}], stats:{min,max,avg,trend_slope}}`.
- Supported series: nutrition (calories, protein, fat, carbs, fiber, sugar), sleep (total_minutes, sleep_score, deep_minutes, avg_hrv_ms), exercise (duration, calories_burned, sessions), symptoms (count, avg_severity by type), biometrics (any metric_type).
- Aggregation in SQL (`date_trunc` + partition pruning); `trend_slope` from linear regression in `trends_service`.

**Testing**:
- `Integration: 90-day weekly calorie trend → 13 weekly buckets, correct sums`
- `Integration: window with gaps → missing buckets present with value=null/0 per metric semantics`
- `Unit: trend_slope on increasing series > 0; flat series ≈ 0`
- `Integration: invalid metric for dimension → 400`

#### 4.3 — Web frontend: dashboard + trends

**What**: Next.js dashboard and trend pages consuming the APIs.

**Design**:
- `web/lib/api-client.ts` generated from `openapi.json`. Dashboard page: per-dimension cards (nutrition ring, sleep summary, exercise list, symptom chips, biometric tiles). Trends page: Recharts line/area charts with 30/90/365 toggles and metric selector.
- Responsive layout; service worker caches the shell and last-fetched dashboard for offline read (offline-first per README).

**Testing**:
- `Vitest: dashboard card renders nutrition totals from mocked response`
- `Vitest: trend chart renders N points for N-bucket response`
- `Playwright (mocked API): login → dashboard loads cards → switch trend window 30→90 refetches`
- `Playwright: offline → cached dashboard still renders`

### Definition of Done
Dashboard and trends APIs correct and performant on seeded data; frontend renders both with live API; offline read works; charts respond to window/metric changes.

---

## Phase 5: AI Insight Engine (Core Value Proposition)

### Purpose
This is the product's heart and its sole AI-native differentiator: reasoning **across** nutrition, sleep, exercise, and symptoms together. A deterministic statistical layer detects candidate cross-dimension patterns; the LLM then narrates grounded, evidence-linked insights. After this phase the user receives the natural-language correlation insights no incumbent offers.

### Tasks

#### 5.1 — Correlation pre-computation engine

**What**: Deterministically detect candidate cross-dimension correlations over a date range.

**Design**:
- `ai/correlation.py` loads a per-user daily feature matrix (pandas) by joining dimensions on `logged_date` (e.g. prior-day dinner carbs/sugar/calories → today's sleep_score/deep_minutes; daily steps → next-day energy symptom severity).
  ```python
  @dataclass
  class CorrelationFinding:
      dimension_a: str; metric_a: str
      dimension_b: str; metric_b: str
      lag_days: int                  # e.g. dinner(t-1) vs sleep(t)
      coefficient: float             # Spearman rho
      p_value: float
      n: int
      evidence_entry_ids: list[UUID]
  def find_correlations(df: pd.DataFrame, *, min_n=14, max_p=0.05) -> list[CorrelationFinding]: ...
  ```
- Tests a fixed library of clinically plausible metric pairs with lags 0–2 days; keeps only `n>=min_n` and `p<=max_p`, ranked by `|coefficient|`. This grounds the LLM and bounds token cost.

**Testing**:
- `Unit: synthetic data with strong negative dinner-sugar↔deep-sleep relation → finding with rho<-0.6, p<0.05`
- `Unit: random noise → no findings above threshold`
- `Unit: n<min_n → pair skipped`
- `Unit: evidence_entry_ids point to the contributing entries`

#### 5.2 — LLM client abstraction

**What**: Provider-agnostic chat/vision client (Anthropic/OpenAI/Ollama).

**Design**:
- `ai/llm_client.py`:
  ```python
  class LLMClient(Protocol):
      async def complete_json(self, *, system: str, user: str,
                              schema: type[BaseModel], images: list[bytes] = []) -> BaseModel: ...
  def get_client(settings) -> LLMClient: ...  # factory by settings.llm_provider
  ```
- Enforces JSON-schema-constrained output (tool/JSON mode), retries on invalid JSON, records `tokens_used`. Ollama path enables fully self-hosted/on-device operation per the privacy requirement.

**Testing**:
- `Unit (mocked transport): complete_json returns validated model`
- `Unit: invalid JSON once then valid → retried, succeeds`
- `Unit: invalid JSON exhausting retries → raises LLMFormatError`
- `Unit: factory selects Ollama client when provider=ollama`

#### 5.3 — Insight generation orchestrator

**What**: Turn correlation findings into persisted `ai_insights`.

**Design**:
- `ai/insight_engine.py::generate_insights(user_id, window)`:
  1. Build feature matrix → `find_correlations`.
  2. For top-K findings, call LLM with `INSIGHT_PROMPT` to produce title + plain-language explanation + a non-prescriptive suggestion.
  3. Persist `ai_insights` rows: `insight_type='cross_dimension_correlation'`, `dimensions=[a,b]`, `confidence=|coefficient|`, `evidence_refs=evidence_entry_ids`, `llm_model`, `tokens_used`.
- `INSIGHT_PROMPT` (`ai/prompts.py`): supplies the statistical finding + sample data; instructs the model to explain the pattern, cite the date range, **avoid medical advice**, and append the standard wellness disclaimer (clinical boundary management per `research.md`).
- `GET /insights?unread=true`, `POST /insights/{id}/read`, `POST /insights/{id}/dismiss`.

**Testing**:
- `Integration (mocked LLM + seeded correlated data): generate_insights → insight persisted with correct dimensions, evidence_refs, confidence`
- `Integration: no significant correlations → no insights created`
- `Unit: prompt assembly includes finding stats and disclaimer instruction`
- `Integration: mark read → idx_insights_unread excludes it`

#### 5.4 — Natural-language health query

**What**: Answer free-text questions over the user's longitudinal record.

**Design**:
- `POST /insights/query {question, window_days?}` → `ai/query.py` retrieves a structured summary (dashboard + relevant trends + recent symptoms) for the window, passes it as grounding context to the LLM, returns a cited answer persisted as `insight_type='query_response'`.
- Strictly grounded: the prompt forbids facts not present in the supplied data and requires the wellness disclaimer.

**Testing**:
- `Integration (mocked LLM): "how was my sleep this month?" → answer references supplied sleep stats`
- `Unit: question with no relevant data → answer states insufficient data, no fabrication`
- `Integration: query_response persisted and retrievable in feed`

### Definition of Done
Correlation engine validated on synthetic data; LLM abstraction supports all three providers with JSON enforcement; insights generated, grounded, disclaimed, and persisted with evidence; NL query grounded to user data; all LLM tests mocked (no live calls in CI).

---

## Phase 6: Goals & Adaptive Coaching

### Purpose
Add configurable daily goals and adaptive targets that tune to recent activity and recovery — moving from static tracking to the adaptive coaching that differentiates AI-native trackers. Goal progress backfills the dashboard's `goal_progress` field.

### Tasks

#### 6.1 — Goals CRUD + progress

**What**: Configure goals and compute progress.

**Design**:
- `goals` table per Data Model 1. `POST/GET/PATCH/DELETE /goals`. `GET /goals/progress?date=` computes actual-vs-target per active goal by querying the relevant dimension; injected into `DashboardResponse.goal_progress`.
  ```python
  class GoalProgress(BaseModel):
      goal_id: UUID; goal_type: str; target_value: float
      actual_value: float; unit: str; pct: float; met: bool
  ```

**Testing**:
- `Integration: calorie goal 2000, logged 1500 → pct=75, met=false`
- `Integration: steps goal met → met=true`
- `Integration: weekly goal aggregates over the week, not the day`

#### 6.2 — Adaptive goal recomputation

**What**: For `is_adaptive=true` goals, adjust targets from recent context.

**Design**:
- Celery Beat nightly task `recompute_adaptive_goals`: for adaptive calorie/macro goals, adjust target from trailing 7-day activity (exercise calories burned) and recovery (sleep_score / avg_hrv trend) using a documented, bounded heuristic (±15% clamp). Writes a new `goals.target_value` and an `ai_insights` row `insight_type='adaptive_goal_adjustment'` explaining the change.

**Testing**:
- `Unit: high recent training load → calorie target increased, within ±15% clamp`
- `Unit: poor sleep trend → recovery-protective adjustment applied`
- `Integration: recompute writes new target + adjustment insight`
- `Unit: non-adaptive goals untouched`

### Definition of Done
Goals CRUD complete; progress correct for daily/weekly/monthly frequencies and surfaced on dashboard; adaptive recomputation bounded, scheduled, and explained via insight.

---

## Phase 7: Wearable & Platform Integration

### Purpose
Reduce manual logging by ingesting biometric, sleep, and exercise data from wearables and the mobile health platforms. This is an integration phase built on the stable entry tables and normalisation service, connecting external OAuth-protected APIs and client-pushed HealthKit/Health Connect payloads.

### Tasks

#### 7.1 — OAuth (PKCE) connection management + token encryption

**What**: Connect/disconnect wearable accounts via OAuth 2.0 Authorization Code + PKCE.

**Design**:
- `wearable_connections` table per Data Model 1. Endpoints: `GET /wearables/{provider}/connect` (PKCE challenge, returns provider auth URL), `GET /wearables/{provider}/callback` (exchanges code, encrypts tokens via `security/crypto.py` Fernet/AES-256, stores reference in `oauth_token_ref`, never plaintext), `DELETE /wearables/{provider}` (revoke + mark `revoked`), `GET /wearables`.
- Refresh-token rotation per RFC 9700; failures set `status='error'` + `error_message`.

**Testing**:
- `Integration (mocked provider): callback exchanges code → connection active, token stored encrypted (plaintext absent from DB)`
- `Unit: crypto round-trips token; tampered ciphertext → InvalidToken`
- `Integration: PKCE verifier mismatch → 400, no connection created`
- `Integration: disconnect → status=revoked, token reference cleared`

#### 7.2 — Provider adapters + sync (Oura, Fitbit, WHOOP, Withings)

**What**: Pull sleep/exercise/biometric data and map into entry tables, deduplicated.

**Design**:
- `integrations/wearables/base.py` defines:
  ```python
  class WearableProvider(Protocol):
      provider: str
      async def fetch_sleep(self, conn, since: date) -> list[SleepCreate]: ...
      async def fetch_workouts(self, conn, since: date) -> list[ExerciseCreate]: ...
      async def fetch_biometrics(self, conn, since: date) -> list[BiometricCreate]: ...
  ```
- Each adapter maps provider JSON to the dimension schemas, setting `source=<provider>` and `source_id` (provider record id). Sync upserts on `(source, source_id)` for idempotent dedup (matches `idx_sleep_source` etc.).
- Celery Beat `sync_wearables` runs per active connection on a schedule; WHOOP webhook endpoint `POST /wearables/whoop/webhook` (signature-verified) triggers targeted sync.

**Testing**:
- `Integration (fixture Oura payload): fetch_sleep → SleepCreate list with correct stage minutes`
- `Integration: re-sync same records → no duplicates (upsert on source_id)`
- `Integration (fixture WHOOP webhook, valid signature): → sync task enqueued, 200`
- `Integration (invalid signature): → 401, no task enqueued`
- `Unit: provider 401 during sync → connection status=error, error_message set`

#### 7.3 — HealthKit / Health Connect client ingestion

**What**: Accept batched health samples pushed from a mobile/web client.

**Design**:
- `POST /wearables/healthkit/ingest` accepts a normalised batch `{samples:[{type, value, unit, start, end, source}]}` (the client reads HealthKit/Health Connect and posts). Server normalises units/timestamps and writes to biometric/sleep/exercise tables with `source='apple_health'|'google_health_connect'`, deduped on a client-supplied `source_id`.

**Testing**:
- `Integration: ingest batch of steps + sleep → biometric + sleep entries created, units normalised to metric`
- `Integration: re-ingest same source_ids → idempotent`
- `Unit: unsupported sample type → skipped and reported in response, others succeed`

### Definition of Done
OAuth+PKCE connect/disconnect works with encrypted token storage; four provider adapters map and dedupe correctly against fixtures; scheduled + webhook sync functional; HealthKit/Health Connect ingestion idempotent; all external calls mocked in CI.

---

## Phase 8: Medical Export, Clinician Sharing & FHIR

### Purpose
Deliver the standout cross-product differentiator: a clinician-ready summary spanning months of multi-dimensional data, plus secure scoped sharing — capabilities no incumbent provides. FHIR mapping makes the data clinically interoperable.

### Tasks

#### 8.1 — FHIR mapping

**What**: Map biometric/sleep/exercise entries to FHIR R5 `Observation` resources.

**Design**:
- `integrations/fhir/mapper.py` converts entries to `fhir.resources` `Observation` objects using `biometric_entries.fhir_observation_code` (LOINC where known). `GET /export/fhir?from=&to=` returns a FHIR `Bundle`.

**Testing**:
- `Unit: weight biometric → Observation with correct LOINC code, UCUM unit 'kg'`
- `Unit: sleep entry → Observation components for stage minutes`
- `Integration: GET /export/fhir → valid Bundle validating against fhir.resources schema`

#### 8.2 — PDF medical appointment summary

**What**: Generate a formatted PDF over a user-defined period.

**Design**:
- `POST /export/pdf {from, to, dimensions[]}` → worker renders an HTML template (WeasyPrint) covering: profile header, per-dimension summary stats + trend sparklines, symptom frequency table, notable AI insights, and a prominent **"wellness data, not medical advice"** disclaimer. Returns a download URL.

**Testing**:
- `Integration (mocked render): 90-day export → PDF produced, contains section per requested dimension`
- `Unit: disclaimer present in rendered HTML`
- `Integration: empty period → PDF generated stating no data, 200`

#### 8.3 — Clinician sharing (HEART-aligned)

**What**: Token-based, read-only, dimension-and-date-scoped access for clinicians.

**Design**:
- `clinician_shares` table. `POST /shares {clinician_email, dimensions[], date_range, expires_at}` mints a random `access_token`. `GET /shared/{token}` returns read-only data scoped to the share's dimensions/date range (minimum-necessary access per HEART). Expired/inactive tokens → 403; every access writes `audit_log` (`actor_type='clinician'`) and updates `last_accessed_at`.

**Testing**:
- `Integration: create share for ['sleep','symptoms'] → /shared/{token} returns only those dimensions`
- `Integration: expired token → 403`
- `Integration: revoked share → 403`
- `Integration: clinician access → audit row + last_accessed_at updated`

### Definition of Done
FHIR Bundle validates; PDF includes all requested dimensions + disclaimer; sharing enforces scope/expiry; all clinician access audited.

---

## Phase 9: Proactive Insights, MCP Server & Hardening

### Purpose
Close the loop with proactive nudges, expose the data to external AI assistants via MCP, and complete the security/privacy posture (GDPR erasure, audit completeness, MASVS-aligned hardening) required to responsibly handle health data.

### Tasks

#### 9.1 — Proactive nudges & periodic summaries

**What**: Scheduled detection of actionable patterns delivered as insights.

**Design**:
- Celery Beat tasks: nightly `proactive_nudges` (e.g. accumulating sleep debt, rising symptom frequency, goal slippage → `insight_type='proactive_nudge'`) and weekly/monthly `periodic_summary` (`weekly_summary`/`monthly_summary`). Deduped so the same nudge isn't repeated within a cooldown window.

**Testing**:
- `Unit: 3 nights <6h sleep → sleep-debt nudge created`
- `Unit: nudge within cooldown of identical prior → suppressed`
- `Integration: weekly task → one weekly_summary insight per active user`

#### 9.2 — MCP server

**What**: Expose health data as MCP tools for external assistants.

**Design**:
- `mcp/server.py` registers tools: `get_dashboard(date)`, `get_trend(dimension, metric, window)`, `query_health(question)`, `list_insights()`. Auth via a per-user MCP token; gated by `users.mcp_enabled`. Read-only.

**Testing**:
- `Integration: MCP get_dashboard tool → same payload as REST dashboard`
- `Unit: mcp_enabled=false → tools unavailable/denied`
- `Integration: query_health tool reuses grounded NL query path`

#### 9.3 — Privacy, GDPR erasure & security hardening

**What**: Complete regulatory and security controls.

**Design**:
- `DELETE /account` → GDPR right-to-erasure: cascades/anonymises all user data across partitioned tables; writes a final audit record; revokes wearable tokens.
- `GET /account/export` → full data export (JSON) for portability.
- Enforce TLS-only, AES-256 at rest for tokens, security headers, per-IP + per-user rate limiting (Redis), and verify audit coverage on every mutating path. Document MASVS-L2 alignment for the client.

**Testing**:
- `Integration: erasure → all user rows removed/anonymised across every table; final audit row exists`
- `Integration: data export → JSON contains every dimension`
- `Integration: rate limit exceeded → 429`
- `Integration: every mutating endpoint writes an audit row (parametrised sweep)`

### Definition of Done
Proactive nudges and summaries scheduled and deduped; MCP server functional and permission-gated; erasure/export complete and audited; rate limiting and security headers enforced; audit coverage verified across all mutations.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (auth, DB, config, Docker)        ─── required by everything
    │
Phase 2: Manual logging (5 dimensions + normalise)    ─── requires P1
    │
Phase 3: Nutrition intelligence (food DB, photo AI)   ─── requires P2
    │
Phase 4: Dashboard & trends                           ─── requires P2 (richer with P3)
    │
Phase 5: AI insight engine (CORE)                     ─── requires P2,P4
    ├── Phase 6: Goals & adaptive coaching             ─── requires P2,P5 (adaptive); parallel with P7,P8
    ├── Phase 7: Wearable & platform integration       ─── requires P2; parallel with P6,P8
    └── Phase 8: Medical export, sharing, FHIR         ─── requires P2,P4 (P5 enriches); parallel with P6,P7
         │
Phase 9: Proactive insights, MCP, hardening           ─── requires P5,P7,P8
```

**Parallelism opportunities:**
- After Phase 5, **Phases 6, 7, and 8 can be developed concurrently** — goals/coaching, wearable integration, and export/sharing have no inter-dependencies (all depend only on the entry tables and, for 8, the dashboard/trends).
- Within Phase 3, the food-database integration (3.1) and recipe builder (3.2) can proceed in parallel with photo AI (3.3).
- The Next.js frontend (4.3) can be built incrementally alongside any backend phase once its endpoints exist.

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before it is considered complete:

1. All tasks implemented.
2. All unit and integration tests pass (LLM and external HTTP mocked in CI; real-Postgres integration via testcontainers).
3. `ruff check` and `ruff format --check` pass.
4. `mypy --strict` passes.
5. `docker compose up` builds and boots all services healthy.
6. The phase's feature works end-to-end (verified by at least one e2e test where a user-facing flow exists).
7. New configuration options added to `config.py` and documented in `.env.example`.
8. New API endpoints appear in the regenerated `openapi.json` (committed as a CI artifact).
9. Database changes ship as a reversible Alembic migration; new partitioned tables include current + next-month partitions and a Beat task for future partition creation.
10. Every mutating endpoint writes an `audit_log` entry (HIPAA-aligned audit trail).
11. Any LLM-produced user-facing output includes the wellness (non-medical-advice) disclaimer.
