# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Health & Wellness Tracker · Created: 2026-05-25

## Philosophy

Every health action — logging a meal, recording a symptom, syncing a sleep session from a wearable, generating an AI insight — is captured as an immutable event in a single append-only event store. The current state of a user's health dashboard, trend analytics, and AI insight feed is derived by replaying or projecting events into purpose-built read models (CQRS pattern). The event store is the source of truth; read models are disposable and rebuildable.

Health data is inherently temporal: users ask "what changed in my health this month?", clinicians want "show me the progression over 90 days", and the AI correlation engine needs to reason over sequences of events across dimensions. An event-sourced architecture makes these temporal queries natural — every state transition is preserved with its timestamp, actor, and context. Corrections (editing a mislogged meal, re-syncing wearable data) are new events that supersede prior ones, preserving the full correction history for audit.

The trade-off is query complexity: the daily dashboard can't be answered by a direct SELECT — it requires a materialised read model. But for a health tracker where full audit trail, temporal reasoning, cross-dimension correlation, and GDPR right-to-erasure (via crypto-shredding) are core requirements, event sourcing provides capabilities that relational snapshots cannot replicate without significant additional infrastructure.

**Best for:** Teams building a health platform where full audit trail, AI-powered temporal reasoning, regulatory compliance (HIPAA audit, GDPR erasure), and the ability to retroactively recompute insights from historical data are priorities.

**Trade-offs:**
- Pro: Complete audit trail — every health data change is preserved with context
- Pro: Temporal queries are natural — "what was true on date X?" is a replay
- Pro: AI correlation engine can process event streams directly
- Pro: GDPR right-to-erasure via crypto-shredding without data loss
- Pro: Read models can be rebuilt when new health dimensions or analytics are added
- Con: Dashboard requires materialised read models, not direct queries
- Con: Event replay for long-history users can be slow without snapshots
- Con: Higher storage costs — events are never deleted
- Con: More complex application code (command validation, event projection)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0 | Event envelope format (ce_source, ce_type, ce_specversion, ce_time) |
| ISO/IEEE 11073 | Biometric event data follows 11073 data type naming |
| HL7 FHIR R5 | Event types map to FHIR Observation resource categories |
| USDA FoodData Central | Food references in nutrition events carry fdc_id |
| Open Food Facts | Barcode-logged foods reference off_barcode |
| OpenAPI 3.1 | REST API for commands and read model queries |
| OAuth 2.0 / OIDC | User auth and wearable API authorisation |
| HEART Profile | Clinician sharing events align with HEART access principles |
| ISO/IEC 27001:2022 | Event store encryption and access control |
| ISO/IEC 27701:2025 | Privacy-by-design via crypto-shredding for erasure |
| HIPAA Security Rule | Immutable event store satisfies 6-year audit retention |
| GDPR Article 9 | Explicit consent events; crypto-shredding for right to erasure |
| OWASP MASVS 2.1 | Mobile security baseline |
| MCP | AI assistant integration via event-derived read models |

---

## Event Store (Infrastructure)

```sql
CREATE TABLE event_store (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type         TEXT NOT NULL CHECK (stream_type IN (
                            'user','nutrition','exercise','sleep',
                            'symptom','biometric','insight','goal',
                            'wearable','sharing','config'
                        )),
    stream_id           UUID NOT NULL,
    sequence_num        BIGINT NOT NULL,
    event_type          TEXT NOT NULL,
    event_data          JSONB NOT NULL,
    metadata            JSONB NOT NULL DEFAULT '{}',
    ce_source           TEXT NOT NULL DEFAULT '/health-wellness-tracker',
    ce_specversion      TEXT NOT NULL DEFAULT '1.0',
    ce_type             TEXT NOT NULL,
    ce_time             TIMESTAMPTZ NOT NULL DEFAULT now(),
    actor_id            UUID,
    actor_type          TEXT NOT NULL CHECK (actor_type IN (
                            'user','system','ai','clinician','wearable_sync'
                        )),
    encryption_key_ref  TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_num)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_events_stream ON event_store (stream_id, sequence_num);
CREATE INDEX idx_events_type ON event_store (event_type, created_at);
CREATE INDEX idx_events_actor ON event_store (actor_id, created_at);
CREATE INDEX idx_events_ce_type ON event_store (ce_type, ce_time);
```

---

## Stream Snapshots (Infrastructure)

```sql
CREATE TABLE stream_snapshots (
    stream_id           UUID NOT NULL,
    sequence_num        BIGINT NOT NULL,
    snapshot_data       JSONB NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, sequence_num)
);
```

---

## Projection Checkpoints (Infrastructure)

```sql
CREATE TABLE projection_checkpoints (
    projection_name     TEXT PRIMARY KEY,
    last_event_id       UUID NOT NULL,
    last_sequence_num   BIGINT NOT NULL,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Types by Stream

### User Stream
- `user_registered` — profile, auth method, timezone
- `user_profile_updated` — changed fields (height, weight, activity level)
- `user_settings_changed` — notification prefs, units, privacy settings
- `user_gdpr_consent_given` — consent timestamp, data region
- `user_erasure_requested` — triggers crypto-shredding
- `user_deactivated`

### Nutrition Stream
- `meal_logged` — food_id, food_name, meal_type, quantity, serving_size, macros, log_method, photo_url, ai_confidence
- `meal_updated` — corrected entry with original reference
- `meal_deleted` — soft removal from projections
- `recipe_created` — name, servings, ingredients with food refs
- `recipe_updated`
- `daily_nutrition_summarised` — system-computed daily totals
- `water_logged` — amount_ml, logged_at

### Exercise Stream
- `exercise_logged` — type, duration, calories, distance, heart rate, intensity, source
- `exercise_synced` — wearable-originated exercise with source_id
- `exercise_updated` — manual correction
- `exercise_deleted`

### Sleep Stream
- `sleep_recorded` — bedtime, wake_time, stages, score, biometrics, source
- `sleep_synced` — wearable-originated sleep with source_id
- `sleep_updated` — manual correction or re-sync

### Symptom Stream
- `symptom_logged` — type, severity, body_location, duration, notes, tags
- `symptom_updated`
- `symptom_deleted`
- `mood_recorded` — mood_score, energy_score, notes

### Biometric Stream
- `biometric_recorded` — metric_type, value, unit, source, fhir_code
- `biometric_synced` — wearable-originated reading
- `weight_logged` — special case with body composition fields

### Insight Stream
- `correlation_detected` — dimensions, title, body, confidence, evidence_refs, date_range
- `proactive_nudge_generated` — nudge_type, message, trigger_conditions
- `anomaly_detected` — metric, expected_range, actual_value, severity
- `weekly_summary_generated` — cross-dimension summary
- `monthly_summary_generated`
- `insight_read` — user acknowledged
- `insight_dismissed` — user dismissed
- `health_query_asked` — natural-language query text
- `health_query_answered` — response, dimensions queried, tokens used

### Goal Stream
- `goal_created` — type, target_value, unit, frequency, is_adaptive
- `goal_updated` — changed target
- `goal_achieved` — daily/weekly goal met
- `goal_adaptively_adjusted` — AI-driven target change with rationale

### Wearable Stream
- `wearable_connected` — provider, scopes, oauth flow completed
- `wearable_sync_completed` — entries_synced, duration_ms
- `wearable_sync_failed` — error, retry_count
- `wearable_disconnected` — reason

### Sharing Stream
- `clinician_share_created` — clinician info, dimensions, date range, access token
- `clinician_share_accessed` — access event for audit
- `clinician_share_revoked`
- `medical_export_generated` — format, date range, dimensions included

---

## Read Model: Daily Dashboard

```sql
CREATE TABLE rm_daily_dashboard (
    user_id             UUID NOT NULL,
    log_date            DATE NOT NULL,
    nutrition_json      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "meals": [{
    --     "id": "uuid", "meal_type": "breakfast",
    --     "food_name": "Greek Yogurt", "calories_kcal": 180,
    --     "protein_g": 15, "carbs_g": 22, "fat_g": 5,
    --     "log_method": "photo_ai", "logged_at": "..."
    --   }],
    --   "totals": {
    --     "calories": 1850, "protein_g": 95, "fat_g": 72,
    --     "carbs_g": 210, "fiber_g": 28, "water_ml": 2400
    --   }
    -- }
    exercise_json       JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "type": "running", "duration_minutes": 35,
    --   "calories_burned": 380, "distance_km": 5.2,
    --   "source": "apple_health"
    -- }]
    sleep_json          JSONB,
    -- {
    --   "total_minutes": 465, "sleep_score": 78,
    --   "rem_minutes": 95, "deep_minutes": 82,
    --   "avg_hrv_ms": 45.2, "source": "oura"
    -- }
    symptoms_json       JSONB NOT NULL DEFAULT '[]',
    biometrics_json     JSONB NOT NULL DEFAULT '{}',
    mood_score          INTEGER,
    energy_score        INTEGER,
    goal_progress_json  JSONB NOT NULL DEFAULT '{}',
    insights_json       JSONB NOT NULL DEFAULT '[]',
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, log_date)
);
```

---

## Read Model: Trend Analytics

```sql
CREATE TABLE rm_trend_analytics (
    user_id             UUID NOT NULL,
    period_type         TEXT NOT NULL CHECK (period_type IN ('weekly','monthly')),
    period_start        DATE NOT NULL,
    nutrition_json      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "avg_daily_calories": 2050, "avg_protein_g": 98,
    --   "avg_carbs_g": 225, "avg_fat_g": 78,
    --   "logging_adherence_pct": 85,
    --   "top_foods": [{"name": "Chicken Breast", "frequency": 12}]
    -- }
    exercise_json       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "total_sessions": 5, "total_minutes": 210,
    --   "total_calories_burned": 1650,
    --   "avg_heart_rate": 138,
    --   "types": {"running": 3, "strength_training": 2}
    -- }
    sleep_json          JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "avg_total_minutes": 438, "avg_sleep_score": 74,
    --   "avg_deep_minutes": 75, "avg_rem_minutes": 88,
    --   "avg_hrv_ms": 42.5, "consistency_pct": 78
    -- }
    symptoms_json       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "total_entries": 8,
    --   "by_type": {"energy": 3, "digestive": 3, "headache": 2},
    --   "avg_severity": 4.2,
    --   "trend": "improving"
    -- }
    biometrics_json     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "weight_start_kg": 73.2, "weight_end_kg": 72.5,
    --   "weight_change_kg": -0.7,
    --   "avg_resting_hr": 57, "avg_hrv_ms": 44.8,
    --   "avg_steps": 8200
    -- }
    correlations_json   JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "dimension_a": "nutrition.sugar_g", "dimension_b": "sleep.sleep_score",
    --   "correlation": -0.42, "p_value": 0.03,
    --   "description": "Higher sugar intake correlates with lower sleep scores"
    -- }]
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, period_type, period_start)
);
```

---

## Read Model: Wearable Status

```sql
CREATE TABLE rm_wearable_status (
    user_id             UUID NOT NULL,
    provider            TEXT NOT NULL,
    status              TEXT NOT NULL,
    last_sync_at        TIMESTAMPTZ,
    entries_synced_total BIGINT NOT NULL DEFAULT 0,
    last_sync_entries   INTEGER,
    last_sync_duration_ms INTEGER,
    error_message       TEXT,
    scopes              TEXT[] NOT NULL DEFAULT '{}',
    connected_at        TIMESTAMPTZ,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, provider)
);
```

---

## Read Model: AI Insight Feed

```sql
CREATE TABLE rm_insight_feed (
    user_id             UUID NOT NULL,
    insight_id          UUID NOT NULL,
    insight_type        TEXT NOT NULL,
    title               TEXT NOT NULL,
    body                TEXT NOT NULL,
    dimensions          TEXT[] NOT NULL DEFAULT '{}',
    confidence          NUMERIC(4,3),
    date_range_start    DATE,
    date_range_end      DATE,
    is_read             BOOLEAN NOT NULL DEFAULT FALSE,
    is_dismissed        BOOLEAN NOT NULL DEFAULT FALSE,
    evidence_json       JSONB,
    -- {
    --   "entries_referenced": 28,
    --   "date_range": "2026-05-01 to 2026-05-25",
    --   "key_data_points": [
    --     {"date": "2026-05-10", "sleep_score": 55, "dinner_sugar_g": 45},
    --     {"date": "2026-05-15", "sleep_score": 85, "dinner_sugar_g": 8}
    --   ]
    -- }
    llm_model           TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, insight_id)
);
CREATE INDEX idx_insight_feed_unread ON rm_insight_feed (user_id)
    WHERE is_read = FALSE AND is_dismissed = FALSE;
```

---

## Read Model: Medical Export

```sql
CREATE TABLE rm_medical_export (
    user_id             UUID NOT NULL,
    export_id           UUID NOT NULL,
    date_range_start    DATE NOT NULL,
    date_range_end      DATE NOT NULL,
    dimensions          TEXT[] NOT NULL DEFAULT '{}',
    summary_json        JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "nutrition": {
    --     "avg_daily_calories": 2100, "avg_protein_g": 95,
    --     "notable_deficiencies": ["vitamin_d","magnesium"]
    --   },
    --   "sleep": {
    --     "avg_score": 72, "avg_deep_pct": 17,
    --     "concerning_patterns": ["irregular_bedtime"]
    --   },
    --   "symptoms": {
    --     "recurring": [{"type": "headache", "frequency": "2x/week", "avg_severity": 5}]
    --   },
    --   "biometrics": {
    --     "weight_trend": "stable", "hrv_trend": "improving",
    --     "blood_pressure_avg": "118/76"
    --   },
    --   "ai_observations": [
    --     "Headache frequency correlates with days where water intake < 1500ml",
    --     "Sleep quality improves on days with morning exercise"
    --   ]
    -- }
    format              TEXT NOT NULL CHECK (format IN ('pdf','fhir_bundle','json')),
    file_url            TEXT,
    shared_with         TEXT,
    generated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, export_id)
);
```

---

## Example Event Sequences

### Morning routine: wake up → log breakfast → AI insight

```
1. sleep_recorded       {stream: sleep, user: U1}
   → sleep data synced from Oura overnight
   → rm_daily_dashboard.sleep_json updated

2. biometric_synced     {stream: biometric, user: U1}
   → resting HR, HRV, steps from Apple Health
   → rm_daily_dashboard.biometrics_json updated

3. meal_logged          {stream: nutrition, user: U1}
   → breakfast: photo_ai method, Greek yogurt + berries
   → rm_daily_dashboard.nutrition_json.meals appended
   → rm_daily_dashboard.nutrition_json.totals recomputed

4. mood_recorded        {stream: symptom, user: U1}
   → energy: 7, mood: 8
   → rm_daily_dashboard.energy_score, mood_score updated

5. correlation_detected {stream: insight, user: U1, actor: ai}
   → "Sleep score 82 avg on high-protein days vs 68 on low-protein"
   → rm_insight_feed row created
```

### Wearable connection and sync

```
1. wearable_connected   {stream: wearable, user: U1}
   → provider: oura, scopes: [sleep, activity, heart_rate]
   → rm_wearable_status row created

2. wearable_sync_completed {stream: wearable, user: U1, actor: system}
   → entries_synced: 7 (7 days of sleep data backfilled)
   → rm_wearable_status.last_sync_at updated
   → 7x sleep_synced events emitted to sleep stream
   → rm_daily_dashboard rows updated for each day
```

### GDPR erasure via crypto-shredding

```
1. user_erasure_requested {stream: user, user: U1}
   → encryption_key_ref for U1's events is destroyed
   → all events with that key become undecryptable
   → read models for U1 are deleted
   → audit entry preserved (erasure event itself is not encrypted)
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Infrastructure | 3 | event_store (partitioned), stream_snapshots, projection_checkpoints |
| Read Models | 5 | rm_daily_dashboard, rm_trend_analytics, rm_wearable_status, rm_insight_feed, rm_medical_export |
| **Total** | **8** | |

---

## Key Design Decisions

1. **Single event store for all health dimensions** — nutrition, exercise, sleep, symptom, biometric, and insight events all flow through one partitioned event store. The `stream_type` discriminator enables filtered replay, while the unified store supports cross-dimension temporal queries that are the AI correlation engine's primary workload.

2. **CloudEvents envelope** — every event carries `ce_source`, `ce_specversion`, `ce_type`, and `ce_time` for interoperability. Health events can be published to external event brokers or consumed by MCP-connected AI assistants using a standard envelope.

3. **`rm_daily_dashboard` as the primary read model** — the daily dashboard is the most frequently accessed view; materialising it per-user per-day means the dashboard is a single-row read, just like the JSONB hybrid model, but the source of truth remains the event store.

4. **`rm_trend_analytics` with pre-computed correlations** — weekly and monthly trend projections include cross-dimension correlations (nutrition vs. sleep, exercise vs. symptoms) so the trend charts don't require on-the-fly computation from raw events.

5. **`rm_medical_export` as a persistent read model** — clinician-ready health summaries are expensive to compute (aggregating months of cross-dimension data); persisting them as read models allows re-sharing without recomputation, and the event store can regenerate them if the format changes.

6. **`encryption_key_ref` on events** — per-user encryption keys enable GDPR right-to-erasure via crypto-shredding: destroying the key renders all of a user's events undecryptable without deleting the event store rows, preserving aggregate analytics and audit integrity.

7. **Stream-per-dimension design** — each health dimension (nutrition, exercise, sleep, etc.) has its own stream type with dimension-specific event types. This enables targeted replay (rebuild only the sleep read model) without processing irrelevant events.

8. **`rm_insight_feed` with evidence snapshots** — AI-generated insights carry evidence references (the specific data points that triggered the insight) so the insight card can show supporting data without re-querying the event store.

9. **Wearable sync as events** — every sync operation (connect, sync_completed, sync_failed, disconnect) is an event, creating a complete wearable integration audit trail. The `rm_wearable_status` read model provides the current view.

10. **8 tables (3 infrastructure + 5 read models)** — the event-sourced architecture separates the write path (event store) from the read path (materialised views), enabling each to scale independently. Read models are disposable — they can be dropped and rebuilt from the event store at any time.
