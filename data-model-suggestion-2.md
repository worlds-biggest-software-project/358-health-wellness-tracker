# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Health & Wellness Tracker · Created: 2026-05-25

## Philosophy

Core operational entities — users, daily health logs, and food reference data — are relational tables with indexed columns for user-scoped queries and date-range filtering. Variable-structure data — wearable sync configurations, per-dimension health entries (nutrition meals, exercise sessions, sleep stages, symptom tags, biometric readings), AI insights, goals, and clinician sharing — lives in JSONB columns with GIN indexes.

Health trackers have a strongly user-centric, date-centric access pattern: the dominant UI operation is "load today's unified dashboard showing nutrition totals, exercise, sleep, symptoms, and biometrics." Embedding all dimensions into a single `daily_logs` row per user per day means the dashboard is assembled from one row read. The per-entry detail (individual meals, individual symptom occurrences) is navigable within the JSONB arrays.

The trade-off is that cross-dimension analytics requiring per-entry granularity (e.g., "which specific meal preceded poor sleep?") need JSONB path extraction. But for a consumer health app where data is always accessed within a single user context and the primary view is a daily summary, the schema simplicity and dashboard performance pay for themselves.

**Best for:** Teams building an MVP where rapid iteration on health dimensions, minimal schema migrations, and fast daily dashboard loading are priorities.

**Trade-offs:**
- Pro: 5 tables — simple schema, fast to deploy
- Pro: Daily dashboard is a single-row read per day
- Pro: New health dimensions, symptom types, and biometric metrics require no migration
- Pro: Wearable sync configs absorb provider-specific variation naturally
- Con: Per-meal or per-symptom analytics require JSONB path queries
- Con: Heavy logging days can produce oversized JSONB on daily_logs rows
- Con: No FK enforcement on food references within JSONB nutrition entries
- Con: Cross-dimension correlation requires JSONB extraction rather than JOINs

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEEE 11073 | Biometric field names in `daily_logs.biometrics_json` follow 11073 data types |
| HL7 FHIR R5 | `daily_logs` dimension structure maps to FHIR Observation categories |
| USDA FoodData Central | `foods.fdc_id` links to authoritative nutrition data |
| Open Food Facts | `foods.off_barcode` for global packaged food coverage |
| OpenAPI 3.1 | REST API documented in OpenAPI |
| OAuth 2.0 / OIDC | User auth; `users.wearable_connections_json` stores OAuth config refs |
| HEART Profile | Clinician sharing in `users.clinician_shares_json` |
| ISO/IEC 27001:2022 | Security controls for health data |
| ISO/IEC 27701:2025 | Privacy controls for GDPR-aligned processing |
| HIPAA Security Rule | Encryption and audit trail requirements |
| GDPR Article 9 | `users.gdpr_json` for consent tracking |
| OWASP MASVS 2.1 | Mobile security baseline |
| MCP | `users.mcp_json` for AI assistant integration |

---

## Users

```sql
CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               TEXT UNIQUE NOT NULL,
    display_name        TEXT NOT NULL,
    auth_provider       TEXT NOT NULL CHECK (auth_provider IN (
                            'email_password','apple','google','oidc'
                        )),
    timezone            TEXT NOT NULL DEFAULT 'America/New_York',
    locale              TEXT NOT NULL DEFAULT 'en-US',
    units_system        TEXT NOT NULL CHECK (units_system IN ('metric','imperial')) DEFAULT 'metric',
    profile_json        JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "date_of_birth": "1990-03-15", "sex": "female",
    --   "height_cm": 168.0, "weight_kg": 65.0,
    --   "activity_level": "moderately_active",
    --   "avatar_url": "https://..."
    -- }
    wearable_connections_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "provider": "oura",
    --   "provider_user_id": "...",
    --   "oauth_token_ref": "vault://...",
    --   "scopes": ["sleep","activity","heart_rate"],
    --   "last_sync_at": "2026-05-25T08:00:00Z",
    --   "sync_enabled": true, "status": "active"
    -- }, {
    --   "id": "uuid", "provider": "apple_health",
    --   "scopes": ["steps","sleep","heart_rate","nutrition"],
    --   "last_sync_at": "2026-05-25T07:45:00Z",
    --   "sync_enabled": true, "status": "active"
    -- }]
    goals_json          JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "goal_type": "calories", "target_value": 2200,
    --   "unit": "kcal", "frequency": "daily", "is_adaptive": true,
    --   "start_date": "2026-01-01", "is_active": true
    -- }, {
    --   "id": "uuid", "goal_type": "steps", "target_value": 10000,
    --   "unit": "steps", "frequency": "daily", "is_adaptive": false
    -- }, {
    --   "id": "uuid", "goal_type": "sleep_hours", "target_value": 8,
    --   "unit": "hours", "frequency": "daily"
    -- }]
    clinician_shares_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "clinician_email": "dr.smith@clinic.com",
    --   "clinician_name": "Dr. Smith",
    --   "access_token": "...",
    --   "dimensions": ["nutrition","sleep","symptoms","biometrics"],
    --   "date_range_start": "2026-01-01",
    --   "expires_at": "2026-07-01",
    --   "is_active": true, "last_accessed_at": "2026-05-20T14:00:00Z"
    -- }]
    gdpr_json           JSONB,
    -- {
    --   "consent_at": "2026-01-01T00:00:00Z",
    --   "data_region": "eu-west-1",
    --   "right_to_erasure_requested": false
    -- }
    mcp_json            JSONB,
    -- {"enabled": true, "tools": [...], "resources": [...]}
    settings_json       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "notifications_enabled": true,
    --   "insight_frequency": "daily",
    --   "photo_logging_default": true,
    --   "hipaa_aligned": false
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_users_wearables ON users USING GIN (wearable_connections_json);
```

---

## Foods (Reference Data)

```sql
CREATE TABLE foods (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                TEXT NOT NULL,
    brand               TEXT,
    barcode             TEXT,
    fdc_id              INTEGER,
    off_barcode         TEXT,
    source              TEXT NOT NULL CHECK (source IN (
                            'usda_fdc','open_food_facts','user_created','ai_estimated'
                        )),
    nutrients_json      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "serving_size_g": 150, "serving_description": "1 cup",
    --   "calories_kcal": 230, "protein_g": 8.5, "fat_g": 12.0,
    --   "carbs_g": 28.0, "fiber_g": 3.5, "sugar_g": 6.0,
    --   "sodium_mg": 450, "saturated_fat_g": 4.0,
    --   "cholesterol_mg": 30, "potassium_mg": 320,
    --   "calcium_mg": 120, "iron_mg": 2.1,
    --   "vitamin_a_mcg": 80, "vitamin_c_mg": 12,
    --   "vitamin_d_mcg": 2.5, "vitamin_b12_mcg": 1.2,
    --   "folate_mcg": 45, "magnesium_mg": 35,
    --   "zinc_mg": 1.8, "omega_3_g": 0.5, "omega_6_g": 2.1
    -- }
    is_verified         BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_foods_barcode ON foods (barcode) WHERE barcode IS NOT NULL;
CREATE INDEX idx_foods_fdc ON foods (fdc_id) WHERE fdc_id IS NOT NULL;
CREATE INDEX idx_foods_name ON foods USING GIN (to_tsvector('english', name));
CREATE INDEX idx_foods_nutrients ON foods USING GIN (nutrients_json);
```

---

## Daily Logs

```sql
CREATE TABLE daily_logs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    log_date            DATE NOT NULL,
    nutrition_json      JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "meal_type": "breakfast",
    --   "food_id": "uuid", "food_name": "Greek Yogurt with Berries",
    --   "quantity": 1, "serving_size_g": 200,
    --   "calories_kcal": 180, "protein_g": 15, "fat_g": 5, "carbs_g": 22,
    --   "fiber_g": 3, "sugar_g": 12,
    --   "log_method": "photo_ai", "photo_url": "s3://...",
    --   "ai_confidence": 0.87, "logged_at": "2026-05-25T07:30:00Z"
    -- }, {
    --   "id": "uuid", "meal_type": "lunch",
    --   "food_id": "uuid", "food_name": "Grilled Chicken Salad",
    --   "quantity": 1, "serving_size_g": 350,
    --   "calories_kcal": 420, "protein_g": 35, "fat_g": 18, "carbs_g": 30,
    --   "log_method": "text_search", "logged_at": "2026-05-25T12:15:00Z"
    -- }]
    nutrition_summary_json JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "total_calories": 1850, "total_protein_g": 95,
    --   "total_fat_g": 72, "total_carbs_g": 210,
    --   "total_fiber_g": 28, "total_sugar_g": 45,
    --   "meal_count": 4, "water_ml": 2400
    -- }
    exercise_json       JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "exercise_type": "running",
    --   "started_at": "2026-05-25T06:00:00Z",
    --   "duration_minutes": 35, "distance_km": 5.2,
    --   "calories_burned": 380, "avg_heart_rate_bpm": 145,
    --   "max_heart_rate_bpm": 172, "intensity": "vigorous",
    --   "source": "apple_health", "source_id": "...",
    --   "strain_score": 12.5
    -- }]
    sleep_json          JSONB,
    -- {
    --   "bedtime": "2026-05-24T22:30:00Z",
    --   "wake_time": "2026-05-25T06:15:00Z",
    --   "total_minutes": 465, "rem_minutes": 95,
    --   "deep_minutes": 82, "light_minutes": 258,
    --   "awake_minutes": 30, "sleep_score": 78,
    --   "latency_minutes": 12, "efficiency_pct": 93.5,
    --   "avg_heart_rate_bpm": 52, "avg_hrv_ms": 45.2,
    --   "avg_respiratory_rpm": 14.5, "skin_temp_deviation": 0.3,
    --   "spo2_avg_pct": 96.8,
    --   "source": "oura", "source_id": "..."
    -- }
    symptoms_json       JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "symptom_type": "energy",
    --   "severity": 7, "notes": "Feeling good after morning run",
    --   "tags": ["post_exercise","morning"],
    --   "logged_at": "2026-05-25T07:00:00Z"
    -- }, {
    --   "id": "uuid", "symptom_type": "digestive",
    --   "severity": 3, "body_location": "stomach",
    --   "notes": "Mild bloating after lunch",
    --   "tags": ["post_meal","bloating"],
    --   "logged_at": "2026-05-25T13:00:00Z"
    -- }]
    biometrics_json     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "weight_kg": 72.5, "body_fat_pct": 18.2,
    --   "resting_heart_rate_bpm": 58, "hrv_ms": 48.5,
    --   "blood_pressure": {"systolic": 118, "diastolic": 76},
    --   "blood_glucose_mgdl": 95,
    --   "spo2_pct": 97, "steps": 8542,
    --   "active_minutes": 65,
    --   "sources": {"weight": "withings", "heart_rate": "oura", "steps": "apple_health"}
    -- }
    mood_score           INTEGER CHECK (mood_score BETWEEN 1 AND 10),
    energy_score         INTEGER CHECK (energy_score BETWEEN 1 AND 10),
    goal_progress_json  JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "calories": {"target": 2200, "actual": 1850, "pct": 84},
    --   "protein": {"target": 120, "actual": 95, "pct": 79},
    --   "steps": {"target": 10000, "actual": 8542, "pct": 85},
    --   "sleep_hours": {"target": 8, "actual": 7.75, "pct": 97}
    -- }
    insights_json       JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "type": "cross_dimension_correlation",
    --   "title": "Sleep improved on high-protein days",
    --   "body": "Over the past 2 weeks, your sleep score averaged 82 on days where protein exceeded 100g, vs 68 on lower-protein days.",
    --   "dimensions": ["nutrition","sleep"],
    --   "confidence": 0.78, "is_read": false,
    --   "created_at": "2026-05-25T09:00:00Z"
    -- }]
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, log_date)
) PARTITION BY RANGE (log_date);

CREATE INDEX idx_daily_user_date ON daily_logs (user_id, log_date);
CREATE INDEX idx_daily_nutrition ON daily_logs USING GIN (nutrition_json);
CREATE INDEX idx_daily_symptoms ON daily_logs USING GIN (symptoms_json);
CREATE INDEX idx_daily_biometrics ON daily_logs USING GIN (biometrics_json);
```

---

## AI Conversations

```sql
CREATE TABLE ai_conversations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    query               TEXT NOT NULL,
    response            TEXT NOT NULL,
    dimensions_queried  TEXT[] NOT NULL DEFAULT '{}',
    date_range_start    DATE,
    date_range_end      DATE,
    context_json        JSONB,
    -- {
    --   "entries_referenced": 45,
    --   "dimensions": ["nutrition","sleep","symptoms"],
    --   "llm_model": "claude-sonnet-4-6",
    --   "tokens_input": 3200, "tokens_output": 850,
    --   "cost_cents": 2
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_ai_conv_user ON ai_conversations (user_id, created_at DESC);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    actor_type          TEXT NOT NULL CHECK (actor_type IN (
                            'user','system','ai','clinician','wearable_sync'
                        )),
    action              TEXT NOT NULL,
    entity_type         TEXT NOT NULL,
    entity_id           UUID NOT NULL,
    changes_json        JSONB,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_user ON audit_log (user_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
```

---

## Example Queries

### Daily dashboard — single row read

```sql
SELECT log_date, nutrition_summary_json, exercise_json,
       sleep_json, symptoms_json, biometrics_json,
       mood_score, energy_score, goal_progress_json, insights_json
FROM daily_logs
WHERE user_id = 'user-uuid' AND log_date = CURRENT_DATE;
```

### Weekly nutrition trend

```sql
SELECT log_date,
       (nutrition_summary_json->>'total_calories')::INTEGER AS calories,
       (nutrition_summary_json->>'total_protein_g')::NUMERIC AS protein,
       (nutrition_summary_json->>'total_fat_g')::NUMERIC AS fat,
       (nutrition_summary_json->>'total_carbs_g')::NUMERIC AS carbs
FROM daily_logs
WHERE user_id = 'user-uuid'
  AND log_date >= CURRENT_DATE - 7
ORDER BY log_date;
```

### Sleep quality vs. dinner macros over 30 days

```sql
SELECT d.log_date,
       (d.sleep_json->>'sleep_score')::INTEGER AS sleep_score,
       (d.sleep_json->>'deep_minutes')::INTEGER AS deep_minutes,
       (prev.nutrition_summary_json->>'total_carbs_g')::NUMERIC AS prev_day_carbs,
       (prev.nutrition_summary_json->>'total_sugar_g')::NUMERIC AS prev_day_sugar
FROM daily_logs d
LEFT JOIN daily_logs prev ON prev.user_id = d.user_id
    AND prev.log_date = d.log_date - 1
WHERE d.user_id = 'user-uuid'
  AND d.log_date >= CURRENT_DATE - 30
  AND d.sleep_json IS NOT NULL
ORDER BY d.log_date;
```

### Symptom frequency from JSONB

```sql
SELECT s->>'symptom_type' AS symptom_type,
       COUNT(*) AS occurrences,
       AVG((s->>'severity')::INTEGER) AS avg_severity
FROM daily_logs d,
     jsonb_array_elements(d.symptoms_json) AS s
WHERE d.user_id = 'user-uuid'
  AND d.log_date >= CURRENT_DATE - 90
GROUP BY s->>'symptom_type'
ORDER BY occurrences DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Users | 1 | users (embeds profile, wearable connections, goals, clinician shares, GDPR, MCP, settings) |
| Foods | 1 | foods (reference data with embedded nutrients) |
| Health Logs | 1 | daily_logs (partitioned; embeds nutrition, exercise, sleep, symptoms, biometrics, insights, goals) |
| AI | 1 | ai_conversations (partitioned; query/response history) |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **5** | |

---

## Key Design Decisions

1. **`daily_logs` as a single row per user per day** — the dashboard is the primary view; embedding all health dimensions into one row means the dashboard is a single-row read. The `UNIQUE (user_id, log_date)` constraint ensures exactly one row per day.

2. **`nutrition_summary_json` alongside `nutrition_json`** — the per-meal detail array stores individual entries, but a pre-computed summary avoids re-aggregating JSONB arrays for the dashboard's macro totals.

3. **`nutrients_json` on foods** — micronutrient profiles vary widely (84 nutrients in Cronometer vs. 25 in Lose It!); JSONB accommodates whatever granularity the food source provides without fixed columns for every possible nutrient.

4. **`wearable_connections_json` on users** — a user typically has 1-3 wearable connections; embedding them with their sync status and OAuth references avoids a separate table for a small, user-scoped dataset.

5. **`goals_json` on users** — goals are user-level configuration (5-10 per user); JSONB accommodates adding new goal types (hydration, mindfulness minutes) without migration.

6. **`symptoms_json` as an array on daily_logs** — symptom entries are timestamped events within a day; embedding them in the daily log keeps all health dimensions co-located. Tags and free-text notes accommodate the variable nature of symptom journalling.

7. **`biometrics_json` as a map on daily_logs** — biometric readings (weight, heart rate, steps, etc.) are one-per-day values from various sources; a JSONB map keyed by metric type with a `sources` sub-map tracks provenance without a separate table.

8. **`ai_conversations` as a separate table** — AI query/response pairs grow independently of daily logs and benefit from their own partition scheme. They don't need to be co-located with the daily dashboard.

9. **`mood_score` and `energy_score` as top-level columns** — these are the most frequently queried subjective metrics (trend charts, correlation analysis) and benefit from being typed INTEGER columns rather than buried in JSONB.

10. **5 tables** — health trackers have a strongly user-centric, date-centric data model; embedding variable health dimensions into the daily log row minimises joins for the dominant dashboard and trend query operations.
