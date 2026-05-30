# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Health & Wellness Tracker · Created: 2026-05-25

## Philosophy

Every health dimension — nutrition, exercise, sleep, symptoms, biometrics — gets its own dedicated table with typed columns, foreign keys, and purpose-built indexes. This approach mirrors how clinical data systems (FHIR Observation resources, HL7 data types) separate observations by category while linking them through patient and encounter references. Each table enforces domain-specific constraints: nutrition entries carry macronutrient columns with CHECK constraints, exercise logs carry duration and intensity, sleep records carry stage breakdowns, and symptom entries carry structured severity scales.

The dominant query pattern for a health tracker is "load today's dashboard across all dimensions for a single user" — which means parallel queries across 4-5 tables filtered by user_id and date. Normalisation means each dimension can be independently indexed, partitioned, and optimised. Cross-dimension correlation queries (the AI insight engine's primary workload) benefit from joining clean, typed columns rather than extracting values from JSONB paths.

The trade-off is schema rigidity: adding a new health dimension (e.g., medication tracking, menstrual cycle) requires a new table and migration. But for a domain with well-established data types (USDA nutrition fields, ISO 11073 biometric types, sleep stage classifications), the schema is unlikely to change frequently, and the query performance and data integrity benefits outweigh migration cost.

**Best for:** Teams building a production-grade health platform where data integrity, clinical-grade accuracy, cross-dimension analytics, and FHIR/HealthKit interoperability are priorities.

**Trade-offs:**
- Pro: Full referential integrity across all health dimensions
- Pro: Typed columns enable precise validation (macro ranges, sleep stage percentages)
- Pro: Cross-dimension correlation queries use standard JOINs with indexed columns
- Pro: Each dimension can be independently partitioned by time
- Pro: Maps cleanly to FHIR Observation resource types
- Con: 15 tables — more complex schema to deploy and maintain
- Con: Adding new health dimensions requires migration
- Con: Dashboard loading requires parallel queries across multiple tables
- Con: Wearable data import must map to rigid column structures

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEEE 11073 | Biometric data types (heart rate, HRV, SpO₂, temperature) inform `biometrics` column types |
| HL7 FHIR R5 | Observation resource types guide per-dimension table design; `fhir_mapping` on biometrics |
| USDA FoodData Central | `foods` reference table sourced from FDC; `fdc_id` links to authoritative nutrition data |
| Open Food Facts | `off_barcode` on foods for global packaged food coverage |
| OpenAPI 3.1 | REST API documented in OpenAPI |
| OAuth 2.0 / OIDC | User authentication; wearable API authorisation (Oura, Fitbit, WHOOP, Garmin, Withings) |
| HEART Profile | OAuth 2.0 profile for healthcare contexts applied to clinician sharing |
| SMART on FHIR | Framework for EHR integration and medical records pull |
| RFC 7636 (PKCE) | Required for mobile OAuth flows with wearable APIs |
| ISO/IEC 27001:2022 | Security controls for health data (encryption, access control, audit logging) |
| ISO/IEC 27701:2025 | Privacy controls for GDPR-aligned health data processing |
| HIPAA Security Rule | AES-256 at rest, TLS 1.2+ in transit, 6-year audit log retention |
| GDPR Article 9 | Explicit consent for health data; right to erasure; DPIA |
| OWASP MASVS 2.1 | Mobile security baseline (L2 for health data) |
| MCP | AI assistant integration for natural-language health queries |

---

## Users

```sql
CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               TEXT UNIQUE NOT NULL,
    display_name        TEXT NOT NULL,
    avatar_url          TEXT,
    auth_provider       TEXT NOT NULL CHECK (auth_provider IN (
                            'email_password','apple','google','oidc'
                        )),
    timezone            TEXT NOT NULL DEFAULT 'America/New_York',
    locale              TEXT NOT NULL DEFAULT 'en-US',
    date_of_birth       DATE,
    sex                 TEXT CHECK (sex IN ('male','female','other','prefer_not_to_say')),
    height_cm           NUMERIC(5,1),
    weight_kg           NUMERIC(5,1),
    activity_level      TEXT CHECK (activity_level IN (
                            'sedentary','lightly_active','moderately_active',
                            'very_active','extremely_active'
                        )),
    units_system        TEXT NOT NULL CHECK (units_system IN ('metric','imperial')) DEFAULT 'metric',
    gdpr_consent_at     TIMESTAMPTZ,
    gdpr_data_region    TEXT,
    hipaa_aligned       BOOLEAN NOT NULL DEFAULT FALSE,
    mcp_enabled         BOOLEAN NOT NULL DEFAULT FALSE,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Wearable Connections

```sql
CREATE TABLE wearable_connections (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    provider            TEXT NOT NULL CHECK (provider IN (
                            'apple_health','google_health_connect',
                            'oura','fitbit','whoop','garmin','withings',
                            'polar','suunto'
                        )),
    provider_user_id    TEXT,
    oauth_token_ref     TEXT,
    scopes              TEXT[] NOT NULL DEFAULT '{}',
    last_sync_at        TIMESTAMPTZ,
    sync_enabled        BOOLEAN NOT NULL DEFAULT TRUE,
    status              TEXT NOT NULL CHECK (status IN (
                            'active','disconnected','error','revoked'
                        )) DEFAULT 'active',
    error_message       TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, provider)
);
CREATE INDEX idx_wearable_user ON wearable_connections (user_id);
CREATE INDEX idx_wearable_sync ON wearable_connections (last_sync_at)
    WHERE sync_enabled = TRUE;
```

---

## Foods (Reference Data)

```sql
CREATE TABLE foods (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                TEXT NOT NULL,
    brand               TEXT,
    fdc_id              INTEGER,
    off_barcode         TEXT,
    barcode             TEXT,
    serving_size_g      NUMERIC(8,2),
    serving_description TEXT,
    calories_kcal       NUMERIC(8,2),
    protein_g           NUMERIC(8,2),
    fat_g               NUMERIC(8,2),
    carbs_g             NUMERIC(8,2),
    fiber_g             NUMERIC(8,2),
    sugar_g             NUMERIC(8,2),
    sodium_mg           NUMERIC(8,2),
    saturated_fat_g     NUMERIC(8,2),
    cholesterol_mg      NUMERIC(8,2),
    potassium_mg        NUMERIC(8,2),
    calcium_mg          NUMERIC(8,2),
    iron_mg             NUMERIC(8,2),
    vitamin_a_mcg       NUMERIC(8,2),
    vitamin_c_mg        NUMERIC(8,2),
    vitamin_d_mcg       NUMERIC(8,2),
    vitamin_b12_mcg     NUMERIC(8,4),
    folate_mcg          NUMERIC(8,2),
    magnesium_mg        NUMERIC(8,2),
    zinc_mg             NUMERIC(8,2),
    omega_3_g           NUMERIC(8,4),
    omega_6_g           NUMERIC(8,4),
    source              TEXT NOT NULL CHECK (source IN (
                            'usda_fdc','open_food_facts','user_created','ai_estimated'
                        )),
    is_verified         BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_foods_barcode ON foods (barcode) WHERE barcode IS NOT NULL;
CREATE INDEX idx_foods_fdc ON foods (fdc_id) WHERE fdc_id IS NOT NULL;
CREATE INDEX idx_foods_name ON foods USING GIN (to_tsvector('english', name));
```

---

## Nutrition Entries

```sql
CREATE TABLE nutrition_entries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    food_id             UUID REFERENCES foods(id),
    meal_type           TEXT NOT NULL CHECK (meal_type IN (
                            'breakfast','lunch','dinner','snack','supplement'
                        )),
    logged_date         DATE NOT NULL,
    logged_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    quantity            NUMERIC(8,2) NOT NULL DEFAULT 1,
    serving_size_g      NUMERIC(8,2),
    calories_kcal       NUMERIC(8,2) NOT NULL,
    protein_g           NUMERIC(8,2),
    fat_g               NUMERIC(8,2),
    carbs_g             NUMERIC(8,2),
    fiber_g             NUMERIC(8,2),
    sugar_g             NUMERIC(8,2),
    sodium_mg           NUMERIC(8,2),
    log_method          TEXT NOT NULL CHECK (log_method IN (
                            'text_search','barcode_scan','photo_ai',
                            'voice','quick_add','recipe','manual'
                        )),
    photo_url           TEXT,
    ai_confidence       NUMERIC(4,3),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (logged_date);

CREATE INDEX idx_nutrition_user_date ON nutrition_entries (user_id, logged_date);
CREATE INDEX idx_nutrition_meal ON nutrition_entries (meal_type);
CREATE INDEX idx_nutrition_food ON nutrition_entries (food_id);
```

---

## Recipes

```sql
CREATE TABLE recipes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    name                TEXT NOT NULL,
    servings            INTEGER NOT NULL DEFAULT 1,
    prep_time_minutes   INTEGER,
    total_calories_kcal NUMERIC(8,2),
    total_protein_g     NUMERIC(8,2),
    total_fat_g         NUMERIC(8,2),
    total_carbs_g       NUMERIC(8,2),
    instructions        TEXT,
    source_url          TEXT,
    is_public           BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_recipes_user ON recipes (user_id);

CREATE TABLE recipe_ingredients (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recipe_id           UUID NOT NULL REFERENCES recipes(id) ON DELETE CASCADE,
    food_id             UUID NOT NULL REFERENCES foods(id),
    quantity            NUMERIC(8,2) NOT NULL,
    serving_size_g      NUMERIC(8,2),
    sort_order          INTEGER NOT NULL DEFAULT 0
);
CREATE INDEX idx_recipe_ing_recipe ON recipe_ingredients (recipe_id);
```

---

## Exercise Entries

```sql
CREATE TABLE exercise_entries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    exercise_type       TEXT NOT NULL CHECK (exercise_type IN (
                            'running','cycling','swimming','walking','hiking',
                            'strength_training','yoga','pilates','hiit',
                            'rowing','elliptical','stair_climbing','dancing',
                            'martial_arts','team_sport','stretching','other'
                        )),
    logged_date         DATE NOT NULL,
    started_at          TIMESTAMPTZ,
    duration_minutes    INTEGER NOT NULL,
    calories_burned     NUMERIC(8,2),
    distance_km         NUMERIC(8,3),
    avg_heart_rate_bpm  INTEGER,
    max_heart_rate_bpm  INTEGER,
    intensity           TEXT CHECK (intensity IN ('low','moderate','vigorous')),
    source              TEXT NOT NULL CHECK (source IN (
                            'manual','apple_health','google_health_connect',
                            'oura','fitbit','whoop','garmin','withings',
                            'strava','polar'
                        )),
    source_id           TEXT,
    strain_score        NUMERIC(4,1),
    vo2_max             NUMERIC(4,1),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (logged_date);

CREATE INDEX idx_exercise_user_date ON exercise_entries (user_id, logged_date);
CREATE INDEX idx_exercise_type ON exercise_entries (exercise_type);
CREATE INDEX idx_exercise_source ON exercise_entries (source, source_id);
```

---

## Sleep Entries

```sql
CREATE TABLE sleep_entries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    logged_date         DATE NOT NULL,
    bedtime             TIMESTAMPTZ NOT NULL,
    wake_time           TIMESTAMPTZ NOT NULL,
    total_minutes       INTEGER NOT NULL,
    rem_minutes         INTEGER,
    deep_minutes        INTEGER,
    light_minutes       INTEGER,
    awake_minutes       INTEGER,
    sleep_score         INTEGER CHECK (sleep_score BETWEEN 0 AND 100),
    latency_minutes     INTEGER,
    efficiency_pct      NUMERIC(5,2),
    avg_heart_rate_bpm  INTEGER,
    avg_hrv_ms          NUMERIC(6,2),
    avg_respiratory_rpm NUMERIC(4,1),
    skin_temp_deviation NUMERIC(4,2),
    spo2_avg_pct        NUMERIC(5,2),
    source              TEXT NOT NULL CHECK (source IN (
                            'manual','apple_health','google_health_connect',
                            'oura','fitbit','whoop','garmin','withings'
                        )),
    source_id           TEXT,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (logged_date);

CREATE INDEX idx_sleep_user_date ON sleep_entries (user_id, logged_date);
CREATE INDEX idx_sleep_source ON sleep_entries (source, source_id);
```

---

## Symptom Entries

```sql
CREATE TABLE symptom_entries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    logged_date         DATE NOT NULL,
    logged_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    symptom_type        TEXT NOT NULL CHECK (symptom_type IN (
                            'energy','mood','headache','digestive',
                            'pain','nausea','brain_fog','anxiety',
                            'insomnia','skin','respiratory','other'
                        )),
    severity            INTEGER NOT NULL CHECK (severity BETWEEN 1 AND 10),
    body_location       TEXT,
    duration_minutes    INTEGER,
    notes               TEXT,
    tags                TEXT[] NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (logged_date);

CREATE INDEX idx_symptom_user_date ON symptom_entries (user_id, logged_date);
CREATE INDEX idx_symptom_type ON symptom_entries (symptom_type);
CREATE INDEX idx_symptom_tags ON symptom_entries USING GIN (tags);
```

---

## Biometric Entries

```sql
CREATE TABLE biometric_entries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    logged_date         DATE NOT NULL,
    logged_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    metric_type         TEXT NOT NULL CHECK (metric_type IN (
                            'weight','body_fat_pct','bmi','blood_pressure_systolic',
                            'blood_pressure_diastolic','resting_heart_rate',
                            'hrv_ms','blood_glucose_mgdl','body_temperature_c',
                            'spo2_pct','respiratory_rate','waist_cm',
                            'steps','active_minutes','vo2_max'
                        )),
    value               NUMERIC(10,4) NOT NULL,
    unit                TEXT NOT NULL,
    source              TEXT NOT NULL CHECK (source IN (
                            'manual','apple_health','google_health_connect',
                            'oura','fitbit','whoop','garmin','withings',
                            'scale','blood_pressure_monitor','glucometer'
                        )),
    source_id           TEXT,
    fhir_observation_code TEXT,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (logged_date);

CREATE INDEX idx_biometric_user_date ON biometric_entries (user_id, logged_date);
CREATE INDEX idx_biometric_type ON biometric_entries (metric_type);
CREATE INDEX idx_biometric_source ON biometric_entries (source, source_id);
```

---

## Goals

```sql
CREATE TABLE goals (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    goal_type           TEXT NOT NULL CHECK (goal_type IN (
                            'calories','protein','fat','carbs','fiber',
                            'steps','active_minutes','exercise_sessions',
                            'sleep_hours','water_ml','weight_target',
                            'body_fat_target','custom'
                        )),
    target_value        NUMERIC(10,2) NOT NULL,
    unit                TEXT NOT NULL,
    frequency           TEXT NOT NULL CHECK (frequency IN (
                            'daily','weekly','monthly'
                        )) DEFAULT 'daily',
    is_adaptive         BOOLEAN NOT NULL DEFAULT FALSE,
    start_date          DATE NOT NULL,
    end_date            DATE,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_goals_user ON goals (user_id) WHERE is_active = TRUE;
```

---

## AI Insights

```sql
CREATE TABLE ai_insights (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    insight_type        TEXT NOT NULL CHECK (insight_type IN (
                            'cross_dimension_correlation','pattern_detected',
                            'proactive_nudge','adaptive_goal_adjustment',
                            'anomaly_detected','weekly_summary',
                            'monthly_summary','query_response'
                        )),
    title               TEXT NOT NULL,
    body                TEXT NOT NULL,
    dimensions          TEXT[] NOT NULL DEFAULT '{}',
    -- e.g. ['nutrition','sleep'] for a correlation between diet and sleep quality
    date_range_start    DATE,
    date_range_end      DATE,
    confidence          NUMERIC(4,3),
    evidence_refs       UUID[] NOT NULL DEFAULT '{}',
    llm_model           TEXT,
    tokens_used         INTEGER,
    is_read             BOOLEAN NOT NULL DEFAULT FALSE,
    is_dismissed        BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_insights_user ON ai_insights (user_id, created_at DESC);
CREATE INDEX idx_insights_unread ON ai_insights (user_id)
    WHERE is_read = FALSE AND is_dismissed = FALSE;
CREATE INDEX idx_insights_dimensions ON ai_insights USING GIN (dimensions);
```

---

## Clinician Shares

```sql
CREATE TABLE clinician_shares (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    clinician_email     TEXT NOT NULL,
    clinician_name      TEXT,
    access_token        TEXT UNIQUE NOT NULL,
    dimensions          TEXT[] NOT NULL DEFAULT '{}',
    -- which health dimensions are shared: ['nutrition','sleep','symptoms','biometrics']
    date_range_start    DATE,
    date_range_end      DATE,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    last_accessed_at    TIMESTAMPTZ,
    expires_at          TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_shares_user ON clinician_shares (user_id);
CREATE INDEX idx_shares_token ON clinician_shares (access_token) WHERE is_active = TRUE;
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
    ip_address          INET,
    user_agent          TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_user ON audit_log (user_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
```

---

## Example Queries

### Daily dashboard — all dimensions for today

```sql
-- Nutrition totals
SELECT meal_type,
       SUM(calories_kcal) AS calories,
       SUM(protein_g) AS protein,
       SUM(fat_g) AS fat,
       SUM(carbs_g) AS carbs
FROM nutrition_entries
WHERE user_id = 'user-uuid' AND logged_date = CURRENT_DATE
GROUP BY meal_type;

-- Last night's sleep
SELECT total_minutes, sleep_score, rem_minutes, deep_minutes,
       avg_hrv_ms, efficiency_pct
FROM sleep_entries
WHERE user_id = 'user-uuid' AND logged_date = CURRENT_DATE
ORDER BY created_at DESC LIMIT 1;

-- Today's exercise
SELECT exercise_type, duration_minutes, calories_burned, intensity
FROM exercise_entries
WHERE user_id = 'user-uuid' AND logged_date = CURRENT_DATE;

-- Today's symptoms
SELECT symptom_type, severity, notes, logged_at
FROM symptom_entries
WHERE user_id = 'user-uuid' AND logged_date = CURRENT_DATE
ORDER BY logged_at;
```

### Cross-dimension correlation — sleep quality vs. dinner macros

```sql
SELECT s.logged_date,
       s.sleep_score,
       s.deep_minutes,
       n.calories_kcal AS dinner_calories,
       n.carbs_g AS dinner_carbs,
       n.sugar_g AS dinner_sugar
FROM sleep_entries s
JOIN LATERAL (
    SELECT SUM(calories_kcal) AS calories_kcal,
           SUM(carbs_g) AS carbs_g,
           SUM(sugar_g) AS sugar_g
    FROM nutrition_entries
    WHERE user_id = s.user_id
      AND logged_date = s.logged_date - 1
      AND meal_type = 'dinner'
) n ON TRUE
WHERE s.user_id = 'user-uuid'
  AND s.logged_date >= CURRENT_DATE - 30
ORDER BY s.logged_date;
```

### Symptom frequency by type over 90 days

```sql
SELECT symptom_type,
       COUNT(*) AS occurrences,
       AVG(severity) AS avg_severity,
       MIN(logged_date) AS first_occurrence,
       MAX(logged_date) AS last_occurrence
FROM symptom_entries
WHERE user_id = 'user-uuid'
  AND logged_date >= CURRENT_DATE - 90
GROUP BY symptom_type
ORDER BY occurrences DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Users | 1 | users |
| Wearables | 1 | wearable_connections |
| Nutrition | 4 | foods, nutrition_entries (partitioned), recipes, recipe_ingredients |
| Exercise | 1 | exercise_entries (partitioned) |
| Sleep | 1 | sleep_entries (partitioned) |
| Symptoms | 1 | symptom_entries (partitioned) |
| Biometrics | 1 | biometric_entries (partitioned) |
| Goals | 1 | goals |
| AI | 1 | ai_insights (partitioned) |
| Sharing | 1 | clinician_shares |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **15** | |

---

## Key Design Decisions

1. **Separate table per health dimension** — nutrition, exercise, sleep, symptoms, and biometrics each get a dedicated table with typed columns. This mirrors FHIR's Observation resource type model and enables dimension-specific indexing, partitioning, and validation.

2. **`foods` as a reference table** — food nutritional data is normalised into a shared reference table sourced from USDA FoodData Central and Open Food Facts, avoiding duplicate nutrition data across entries while preserving per-entry portion-adjusted values.

3. **`log_method` on nutrition entries** — tracking how food was logged (text search, barcode, photo AI, voice) enables accuracy analysis and AI model improvement over time.

4. **All entry tables partitioned by date** — health data grows indefinitely; time-range partitioning enables efficient recent-period queries (dashboard, trends) while allowing archival of older partitions.

5. **`source` and `source_id` on wearable-synced tables** — exercise, sleep, and biometric entries track their origin (manual, apple_health, oura, etc.) with a provider-specific ID for deduplication during sync.

6. **`biometric_entries` as a generic metric table** — rather than separate tables for weight, blood pressure, heart rate, etc., a single table with `metric_type` and `value` accommodates the wide variety of biometric readings from different wearables, with `fhir_observation_code` for clinical interoperability.

7. **`ai_insights` as a dedicated table** — AI-generated correlations, nudges, and summaries are persisted as read/dismissible insight cards with references to the evidence entries and the dimensions involved, enabling the insight feed and notification system.

8. **`clinician_shares` with dimension-scoped access** — clinician sharing uses token-based read-only access scoped to specific health dimensions and date ranges, implementing the HEART profile's principle of minimum necessary access.

9. **`symptom_entries` with structured severity and free-text tags** — symptoms use a 1-10 severity scale and TEXT[] tags to balance structured querying (correlation engine) with the free-form nature of symptom journalling.

10. **GDPR and HIPAA fields on users** — `gdpr_consent_at`, `gdpr_data_region`, and `hipaa_aligned` support regulatory compliance without embedding complex consent structures in the MVP.
