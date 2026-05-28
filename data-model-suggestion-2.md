# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Pipeline Review Assistant · Created: 2026-05-12

## Philosophy

This model uses an append-only event store as the single source of truth for all pipeline state changes. Every mutation — deal created, stage changed, score recalculated, forecast submitted, email logged — is recorded as an immutable event with a timestamp and payload. Current state is derived by replaying events or, more practically, by maintaining materialised read models (projections) that are updated asynchronously as new events arrive.

This architecture directly addresses the core AI-native opportunity identified in the project research: "always-on probability recalculation." Events flow continuously into the store as CRM syncs, emails arrive, and calls complete. An event processor watches the stream and triggers AI deal scoring within minutes of each signal, rather than waiting for a scheduled batch cycle. The event log also provides the complete audit trail required for GDPR accountability (Article 30), EU AI Act transparency obligations, and SOC 2 compliance — every decision the AI made, every score it assigned, every signal it detected is permanently recorded with its full context.

The CQRS (Command-Query Responsibility Segregation) pattern separates the write path (commands that produce events) from the read path (projections optimised for specific query patterns like pipeline dashboards, forecast roll-ups, and manager review briefs). This allows read models to be independently optimised and scaled without affecting the integrity of the event store.

**Best for:** Teams that need full audit trails, temporal queries ("what was the pipeline on March 1?"), continuous AI processing triggered by real-time signals, and regulatory compliance with immutable records.

**Trade-offs:**
- (+) Complete, immutable audit trail from day one — no retroactive audit trigger setup
- (+) Temporal queries are natural: replay events to reconstruct state at any point in time
- (+) Continuous AI scoring: events trigger score recalculation as signals arrive
- (+) Event replay enables reprocessing with improved AI models against historical data
- (+) Read models can be independently optimised for each use case (dashboards, reports, API)
- (-) Higher architectural complexity: developers must understand event sourcing, projections, and eventual consistency
- (-) Read models are eventually consistent — dashboard may lag writes by seconds
- (-) Event store grows continuously; requires retention policies and archival strategy
- (-) Debugging is harder: current state is derived, not stored directly
- (-) Projection rebuild from scratch can be slow for large event histories

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Salesforce Pub/Sub API | Salesforce's gRPC-based event streaming maps naturally to inbound events in the event store |
| HubSpot Webhooks | Deal-stage change webhooks produce DealStageChanged events |
| GDPR Art. 30 (Records of Processing) | Immutable event log serves as the processing activity record |
| EU AI Act (Transparency) | Every AI scoring event includes model version, input signals, and output — full explainability record |
| ISO/IEC 42001 (AI Management) | Event store captures AI decision provenance: inputs, model version, confidence, explanation |
| OWASP API Security | Command validation layer enforces input sanitisation before events are persisted |
| RFC 6749 (OAuth 2.0) | Token lifecycle events (grant, refresh, revoke) tracked in the event store for security audit |
| MEDDPICC Framework | Qualification score changes are events, enabling temporal scoring analysis per deal |

---

## Event Store (Core)

```sql
-- ============================================================
-- EVENT STORE — The single source of truth
-- ============================================================

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
        -- deal, account, contact, forecast, activity, scoring, review, integration
    stream_id       UUID NOT NULL,           -- the aggregate root ID (e.g., deal ID)
    event_type      VARCHAR(100) NOT NULL,
        -- DealCreated, DealStageChanged, DealAmountUpdated, DealClosed,
        -- DealScoreCalculated, DealSignalDetected, DealSignalResolved,
        -- ActivityLogged, EmailCaptured, CallRecorded,
        -- QualificationScored, MeddpiccUpdated,
        -- ForecastSubmitted, ForecastAdjusted,
        -- CoachingRecommendationGenerated, CoachingAccepted,
        -- ReviewBriefGenerated, ReviewCompleted,
        -- CrmSyncCompleted, TokenRefreshed,
        -- ContactCreated, ContactUpdated, AccountCreated, AccountUpdated
    event_version   INTEGER NOT NULL,        -- version within the stream (optimistic concurrency)
    payload         JSONB NOT NULL,          -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',
        -- { "user_id": "...", "ip": "...", "source": "crm_sync|user|ai_engine",
        --   "correlation_id": "...", "causation_id": "..." }
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, event_version)
);

-- Partition by month for manageable table sizes
-- In production, use PARTITION BY RANGE (occurred_at)

CREATE INDEX idx_event_stream ON event_store(stream_type, stream_id, event_version);
CREATE INDEX idx_event_tenant ON event_store(tenant_id, occurred_at DESC);
CREATE INDEX idx_event_type ON event_store(event_type, occurred_at DESC);
CREATE INDEX idx_event_occurred ON event_store(occurred_at DESC);

-- GIN index for payload queries (e.g., find all events where deal amount > X)
CREATE INDEX idx_event_payload ON event_store USING GIN (payload jsonb_path_ops);
```

### Event Payload Examples

```jsonc
// DealCreated
{
  "deal_name": "Acme Corp - Enterprise License",
  "account_id": "550e8400-e29b-41d4-a716-446655440000",
  "owner_id": "6ba7b810-9dad-11d1-80b4-00c04fd430c8",
  "pipeline_id": "...",
  "stage_id": "...",
  "amount_cents": 15000000,
  "currency_code": "USD",
  "close_date": "2026-06-30",
  "source": "inbound",
  "deal_type": "new_business",
  "crm_external_id": "006xx000001234ABC"
}

// DealStageChanged
{
  "from_stage_id": "...",
  "from_stage_name": "Discovery",
  "to_stage_id": "...",
  "to_stage_name": "Proposal",
  "days_in_previous_stage": 12,
  "trigger": "crm_sync"
}

// DealScoreCalculated
{
  "model_version": "v2.3.1",
  "overall_score": 72.5,
  "win_probability": 0.68,
  "risk_level": "medium",
  "momentum": "decelerating",
  "explanation": "Deal velocity has slowed — no stakeholder meeting in 18 days. Champion (Jane Doe) last responded to email 12 days ago. Competitor (Gong) mentioned in last call transcript.",
  "input_signals": [
    {"signal_type": "champion_dark", "days_since_contact": 12},
    {"signal_type": "competitor_mentioned", "competitor": "Gong", "source": "call_transcript"},
    {"signal_type": "no_meeting_scheduled", "days_without_meeting": 18}
  ],
  "previous_score": 81.2,
  "score_delta": -8.7
}

// ForecastSubmitted
{
  "period_name": "Q2 2026",
  "period_start": "2026-04-01",
  "period_end": "2026-06-30",
  "scenario": "committed",
  "amount_cents": 45000000,
  "deal_count": 12,
  "notes": "Confident on Acme and BigCo; TechStart may slip to Q3"
}

// MeddpiccUpdated
{
  "framework": "MEDDPICC",
  "scores": {
    "M": {"score": 8, "max": 10, "evidence": "ROI model showing 3.2x return validated by CFO"},
    "E": {"score": 9, "max": 10, "evidence": "VP Sales confirmed as economic buyer in last call"},
    "D1": {"score": 6, "max": 10, "evidence": "Decision criteria partially documented; missing technical eval"},
    "D2": {"score": 5, "max": 10, "evidence": "Decision process unclear; legal review timeline unknown"},
    "P": {"score": 3, "max": 10, "evidence": "Paper process not yet discussed"},
    "I": {"score": 9, "max": 10, "evidence": "Pain clearly articulated: 87% forecast miss rate"},
    "C1": {"score": 7, "max": 10, "evidence": "VP Sales is champion; need to multi-thread to CRO"},
    "C2": {"score": 4, "max": 10, "evidence": "Clari and Gong in evaluation; differentiation unclear"}
  },
  "total_score": 51,
  "max_possible": 80,
  "percentage": 63.75,
  "scored_by": "ai_engine",
  "ai_confidence": 0.82
}
```

---

## Projection: Current Deal State (Read Model)

```sql
-- ============================================================
-- READ MODEL: Current deal state (materialised from events)
-- ============================================================

CREATE TABLE rm_deal (
    id                  UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    pipeline_id         UUID NOT NULL,
    stage_id            UUID NOT NULL,
    stage_name          VARCHAR(255) NOT NULL,
    account_id          UUID,
    account_name        VARCHAR(500),
    owner_id            UUID NOT NULL,
    owner_name          VARCHAR(255) NOT NULL,
    name                VARCHAR(500) NOT NULL,
    amount_cents        BIGINT,
    currency_code       CHAR(3) NOT NULL DEFAULT 'USD',
    close_date          DATE,
    probability         DECIMAL(5,2),
    forecast_category   VARCHAR(50) NOT NULL DEFAULT 'pipeline',
    deal_type           VARCHAR(50),
    source              VARCHAR(100),
    -- Current AI score (denormalised for dashboard performance)
    ai_score            DECIMAL(5,2),
    ai_risk_level       VARCHAR(20),
    ai_momentum         VARCHAR(20),
    ai_explanation      TEXT,
    ai_scored_at        TIMESTAMPTZ,
    -- MEDDPICC summary (denormalised)
    meddpicc_pct        DECIMAL(5,2),
    meddpicc_gaps       TEXT[],          -- ['D1: Decision Criteria', 'P: Paper Process']
    -- Activity summary (denormalised)
    last_activity_at    TIMESTAMPTZ,
    last_activity_type  VARCHAR(50),
    days_since_activity INTEGER,
    total_activities    INTEGER NOT NULL DEFAULT 0,
    -- Contact summary
    contact_count       INTEGER NOT NULL DEFAULT 0,
    has_champion        BOOLEAN NOT NULL DEFAULT false,
    has_economic_buyer  BOOLEAN NOT NULL DEFAULT false,
    -- Stage tracking
    stage_entered_at    TIMESTAMPTZ,
    days_in_stage       INTEGER NOT NULL DEFAULT 0,
    is_stalled          BOOLEAN NOT NULL DEFAULT false,
    -- Lifecycle
    is_open             BOOLEAN NOT NULL DEFAULT true,
    is_won              BOOLEAN NOT NULL DEFAULT false,
    created_at          TIMESTAMPTZ NOT NULL,
    updated_at          TIMESTAMPTZ NOT NULL,
    -- Event tracking for projection
    last_event_id       UUID NOT NULL,
    last_event_version  INTEGER NOT NULL
);

CREATE INDEX idx_rm_deal_tenant ON rm_deal(tenant_id);
CREATE INDEX idx_rm_deal_owner ON rm_deal(owner_id);
CREATE INDEX idx_rm_deal_risk ON rm_deal(tenant_id, ai_risk_level) WHERE is_open = true;
CREATE INDEX idx_rm_deal_stalled ON rm_deal(tenant_id) WHERE is_stalled = true AND is_open = true;
CREATE INDEX idx_rm_deal_close ON rm_deal(tenant_id, close_date) WHERE is_open = true;
CREATE INDEX idx_rm_deal_forecast ON rm_deal(tenant_id, forecast_category) WHERE is_open = true;
```

---

## Projection: Pipeline Dashboard (Read Model)

```sql
-- ============================================================
-- READ MODEL: Pipeline dashboard aggregates
-- ============================================================

CREATE TABLE rm_pipeline_summary (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    pipeline_id     UUID NOT NULL,
    snapshot_date   DATE NOT NULL,
    owner_id        UUID,                    -- NULL = pipeline-wide
    stage_id        UUID NOT NULL,
    stage_name      VARCHAR(255) NOT NULL,
    deal_count      INTEGER NOT NULL DEFAULT 0,
    total_amount    BIGINT NOT NULL DEFAULT 0,
    weighted_amount BIGINT NOT NULL DEFAULT 0,
    avg_days_in_stage DECIMAL(7,1),
    stalled_count   INTEGER NOT NULL DEFAULT 0,
    at_risk_count   INTEGER NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, pipeline_id, snapshot_date, owner_id, stage_id)
);

CREATE INDEX idx_rm_pipeline_date ON rm_pipeline_summary(tenant_id, snapshot_date DESC);
```

---

## Projection: Forecast Roll-Up (Read Model)

```sql
-- ============================================================
-- READ MODEL: Forecast roll-up with hierarchy
-- ============================================================

CREATE TABLE rm_forecast (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    period_name     VARCHAR(100) NOT NULL,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    user_id         UUID NOT NULL,
    user_name       VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL,    -- rep, manager, director
    -- Latest submission per scenario
    committed_amount    BIGINT,
    best_case_amount    BIGINT,
    pipeline_amount     BIGINT,
    -- AI forecast
    ai_predicted_amount BIGINT,
    ai_confidence       DECIMAL(5,4),
    -- Manager adjustments
    manager_adjusted_amount BIGINT,
    adjustment_reason   TEXT,
    -- Historical accuracy (for bias detection)
    prior_period_variance_pct DECIMAL(7,2),
    bias_direction      VARCHAR(10),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, period_start, user_id)
);

CREATE INDEX idx_rm_forecast_period ON rm_forecast(tenant_id, period_start);
CREATE INDEX idx_rm_forecast_user ON rm_forecast(user_id);
```

---

## Projection: Activity Timeline (Read Model)

```sql
-- ============================================================
-- READ MODEL: Activity timeline
-- ============================================================

CREATE TABLE rm_activity (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    deal_id         UUID,
    contact_id      UUID,
    user_id         UUID NOT NULL,
    activity_type   VARCHAR(50) NOT NULL,
    subject         VARCHAR(500),
    body_preview    TEXT,
    occurred_at     TIMESTAMPTZ NOT NULL,
    duration_minutes INTEGER,
    sentiment       VARCHAR(20),
    source          VARCHAR(50) NOT NULL,
    external_id     VARCHAR(255),
    is_auto_logged  BOOLEAN NOT NULL DEFAULT false,
    last_event_id   UUID NOT NULL
);

CREATE INDEX idx_rm_activity_deal ON rm_activity(deal_id, occurred_at DESC);
CREATE INDEX idx_rm_activity_contact ON rm_activity(contact_id, occurred_at DESC);
CREATE INDEX idx_rm_activity_user ON rm_activity(user_id, occurred_at DESC);
```

---

## Projection: Coaching & Signals (Read Model)

```sql
-- ============================================================
-- READ MODEL: Active signals and coaching recommendations
-- ============================================================

CREATE TABLE rm_active_signal (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    deal_id         UUID NOT NULL,
    deal_name       VARCHAR(500) NOT NULL,
    owner_id        UUID NOT NULL,
    signal_type     VARCHAR(100) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    description     TEXT NOT NULL,
    detected_at     TIMESTAMPTZ NOT NULL,
    last_event_id   UUID NOT NULL
);

CREATE INDEX idx_rm_signal_deal ON rm_active_signal(deal_id);
CREATE INDEX idx_rm_signal_owner ON rm_active_signal(owner_id);
CREATE INDEX idx_rm_signal_severity ON rm_active_signal(tenant_id, severity);

CREATE TABLE rm_coaching (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    deal_id         UUID NOT NULL,
    deal_name       VARCHAR(500) NOT NULL,
    owner_id        UUID NOT NULL,
    recommendation_type VARCHAR(50) NOT NULL,
    priority        INTEGER NOT NULL,
    title           VARCHAR(255) NOT NULL,
    body            TEXT NOT NULL,
    suggested_action TEXT,
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL,
    last_event_id   UUID NOT NULL
);

CREATE INDEX idx_rm_coaching_owner ON rm_coaching(owner_id) WHERE status = 'pending';
CREATE INDEX idx_rm_coaching_deal ON rm_coaching(deal_id);
```

---

## Supporting Tables (Non-Event-Sourced)

```sql
-- ============================================================
-- REFERENCE DATA & CONFIGURATION (not event-sourced)
-- These are CRUD-managed configuration tables.
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    data_region     VARCHAR(10) NOT NULL DEFAULT 'us',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE "user" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    auth_provider   VARCHAR(50) NOT NULL,
    auth_subject    VARCHAR(512),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE pipeline (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pipeline_stage (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pipeline_id     UUID NOT NULL REFERENCES pipeline(id),
    name            VARCHAR(255) NOT NULL,
    display_order   INTEGER NOT NULL,
    default_probability DECIMAL(5,2) NOT NULL DEFAULT 0,
    forecast_category VARCHAR(50) NOT NULL DEFAULT 'pipeline',
    is_closed_won   BOOLEAN NOT NULL DEFAULT false,
    is_closed_lost  BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE crm_connection (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    provider            VARCHAR(50) NOT NULL,
    instance_url        TEXT,
    access_token_enc    BYTEA,
    refresh_token_enc   BYTEA,
    token_expires_at    TIMESTAMPTZ,
    scopes              TEXT[],
    sync_status         VARCHAR(30) NOT NULL DEFAULT 'pending',
    last_sync_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE email_connection (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES "user"(id),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    provider            VARCHAR(50) NOT NULL,
    email_address       VARCHAR(320) NOT NULL,
    access_token_enc    BYTEA,
    refresh_token_enc   BYTEA,
    token_expires_at    TIMESTAMPTZ,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    last_sync_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Tracks projection rebuild state
CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          VARCHAR(30) NOT NULL DEFAULT 'running',  -- running, paused, rebuilding
    error_message   TEXT
);
```

---

## Event Processing Pipeline

```sql
-- ============================================================
-- EVENT PROCESSING INFRASTRUCTURE
-- ============================================================

-- Outbox table for reliable event publishing to message bus
CREATE TABLE event_outbox (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL,       -- references event_store.event_id
    destination     VARCHAR(100) NOT NULL, -- projection name or external system
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
        -- pending, processing, completed, failed, dead_letter
    attempts        INTEGER NOT NULL DEFAULT 0,
    max_attempts    INTEGER NOT NULL DEFAULT 5,
    last_error      TEXT,
    next_retry_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at    TIMESTAMPTZ
);

CREATE INDEX idx_outbox_pending ON event_outbox(status, next_retry_at)
    WHERE status IN ('pending', 'failed');

-- Dead letter queue for events that failed processing
CREATE TABLE event_dead_letter (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL,
    destination     VARCHAR(100) NOT NULL,
    error_message   TEXT NOT NULL,
    attempts        INTEGER NOT NULL,
    payload_snapshot JSONB NOT NULL,       -- snapshot of the event at time of failure
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ,
    resolution      TEXT
);
```

---

## Temporal Query Examples

```sql
-- Reconstruct deal state at a specific point in time
-- "What was the pipeline worth on March 1, 2026?"
SELECT
    es.stream_id AS deal_id,
    (latest_event.payload->>'name') AS deal_name,
    (latest_event.payload->>'amount_cents')::BIGINT / 100.0 AS amount_usd,
    (latest_event.payload->>'stage_name') AS stage_name,
    latest_event.occurred_at AS state_as_of
FROM (
    SELECT DISTINCT ON (stream_id)
        stream_id,
        payload,
        occurred_at
    FROM event_store
    WHERE tenant_id = :tenant_id
      AND stream_type = 'deal'
      AND event_type IN ('DealCreated', 'DealStageChanged', 'DealAmountUpdated')
      AND occurred_at <= '2026-03-01T23:59:59Z'
    ORDER BY stream_id, event_version DESC
) latest_event
JOIN event_store es ON es.stream_id = latest_event.stream_id
    AND es.event_type = 'DealCreated'
    AND es.stream_type = 'deal';

-- AI score history for a specific deal over time
SELECT
    occurred_at,
    (payload->>'overall_score')::DECIMAL AS score,
    (payload->>'win_probability')::DECIMAL AS probability,
    payload->>'risk_level' AS risk_level,
    payload->>'explanation' AS explanation,
    payload->>'model_version' AS model_version
FROM event_store
WHERE stream_type = 'deal'
  AND stream_id = :deal_id
  AND event_type = 'DealScoreCalculated'
ORDER BY occurred_at;

-- Find all deals where champion went dark (signal detected, never resolved)
SELECT
    detected.stream_id AS deal_id,
    d.name AS deal_name,
    detected.occurred_at AS detected_at,
    (detected.payload->>'days_since_contact')::INTEGER AS days_dark
FROM event_store detected
JOIN rm_deal d ON d.id = detected.stream_id
LEFT JOIN event_store resolved
    ON resolved.stream_type = 'deal'
    AND resolved.stream_id = detected.stream_id
    AND resolved.event_type = 'DealSignalResolved'
    AND (resolved.payload->>'signal_type') = 'champion_dark'
    AND resolved.occurred_at > detected.occurred_at
WHERE detected.stream_type = 'deal'
  AND detected.event_type = 'DealSignalDetected'
  AND (detected.payload->>'signal_type') = 'champion_dark'
  AND detected.tenant_id = :tenant_id
  AND resolved.event_id IS NULL
ORDER BY detected.occurred_at DESC;

-- Forecast variance over time: how did committed forecast change week by week?
SELECT
    DATE_TRUNC('week', occurred_at) AS week,
    (payload->>'amount_cents')::BIGINT / 100.0 AS committed_amount,
    payload->>'period_name' AS period
FROM event_store
WHERE tenant_id = :tenant_id
  AND stream_type = 'forecast'
  AND event_type = 'ForecastSubmitted'
  AND (payload->>'scenario') = 'committed'
  AND occurred_at >= '2026-01-01'
ORDER BY occurred_at;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | Single append-only event table (partitioned by month in production) |
| Event Infrastructure | 3 | event_outbox, event_dead_letter, projection_checkpoint |
| Read Model: Deals | 1 | rm_deal — denormalised current deal state |
| Read Model: Pipeline | 1 | rm_pipeline_summary — aggregated dashboard data |
| Read Model: Forecast | 1 | rm_forecast — roll-up with hierarchy |
| Read Model: Activities | 1 | rm_activity — timeline view |
| Read Model: Signals & Coaching | 2 | rm_active_signal, rm_coaching |
| Reference/Config | 7 | tenant, user, pipeline, pipeline_stage, crm_connection, email_connection, projection_checkpoint |
| **Total** | **17** | Significantly fewer tables than normalized; complexity moves to event processors |

---

## Key Design Decisions

1. **Single event_store table as source of truth** — All state changes across all entity types flow through one table, keyed by (stream_type, stream_id, event_version). This centralisation makes audit queries trivial and ensures nothing is lost. In production, the table should be partitioned by `occurred_at` month to keep each partition manageable.

2. **JSONB payloads instead of typed event tables** — Rather than creating a separate table for each event type, all events share the same table with a JSONB payload column. This keeps the schema stable as new event types are added (just deploy new event handlers) and enables GIN-indexed queries across event payloads.

3. **Optimistic concurrency via event_version** — The UNIQUE constraint on (stream_type, stream_id, event_version) prevents conflicting writes to the same aggregate. Writers must read the current version and increment it; a version collision raises a conflict error, forcing a retry. This eliminates the need for pessimistic locking.

4. **Read models are disposable and rebuildable** — Every `rm_*` table can be dropped and rebuilt from the event store. The `projection_checkpoint` table tracks how far each projection has consumed, enabling incremental catch-up after downtime. This means read model schema changes are non-destructive: drop, rebuild, done.

5. **Denormalised read models for dashboard performance** — `rm_deal` includes pre-joined fields (owner_name, account_name, stage_name, ai_score, meddpicc_pct) that would require multi-table joins in a normalised model. Dashboard queries hit a single table with no JOINs.

6. **Event outbox pattern for reliable projection updates** — Rather than directly updating projections in the event write transaction (which couples write and read), events are published to an outbox table. A separate processor reads the outbox and updates projections. This ensures projections eventually catch up even if they fail temporarily.

7. **Dead letter queue for failed event processing** — Events that fail processing after max retries are moved to a dead letter table with the full error context. Operations teams can investigate and reprocess without losing events.

8. **AI scoring events capture full provenance** — The `DealScoreCalculated` event payload includes model_version, all input signals, the previous score, and the delta. This creates an immutable record of every AI decision, satisfying EU AI Act transparency requirements and enabling model performance analysis across historical data.

9. **Temporal queries use event replay, not snapshot tables** — Instead of maintaining explicit weekly snapshots (as in the normalised model), temporal queries replay events up to a target timestamp. This is more storage-efficient but slower for ad-hoc historical queries. For frequently-accessed temporal views (like week-over-week pipeline), materialised snapshots in `rm_pipeline_summary` provide the performance needed.

10. **Reference data is CRUD, not event-sourced** — Tenant configuration, pipeline stage definitions, user accounts, and OAuth connections are managed with conventional CRUD tables. Event-sourcing these would add complexity without meaningful benefit, since their change history is not analytically valuable. The audit log for these changes is captured as events in the event store with stream_type 'configuration'.
