# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Pipeline Review Assistant · Created: 2026-05-12

## Philosophy

This model follows traditional third-normal-form relational design, giving every concept its own table with strict foreign key enforcement. Each CRM entity (account, contact, deal, activity) has a dedicated table with typed columns, and relationships are expressed through junction tables and foreign keys. Reference data (pipeline stages, forecast categories, MEDDPICC criteria) lives in lookup tables that enforce data integrity at the database level.

This approach mirrors how enterprise CRM platforms like Salesforce structure their data internally. Salesforce's Opportunity, OpportunityStage, ForecastingItem, and OpportunityContactRole objects map directly to separate normalized tables. The model prioritises query flexibility and data integrity over write performance, making it well-suited for a platform where complex cross-entity queries (e.g., "show all deals owned by reps under Manager X where the champion has gone dark and MEDDPICC score is below 60%") are the primary access pattern.

The normalized structure also simplifies regulatory compliance: personal data (contacts) is cleanly separated from business data (deals, forecasts), making GDPR right-to-erasure and data residency requirements straightforward to implement.

**Best for:** Teams that value data integrity, need complex ad-hoc queries, and have a stable, well-understood domain model.

**Trade-offs:**
- (+) Strong referential integrity prevents orphaned records and inconsistent state
- (+) Standard SQL queries work naturally without JSONB operators or event replay
- (+) GDPR compliance is simpler with cleanly separated personal data tables
- (+) Well-understood by most developers and database administrators
- (-) High table count (40+) increases schema migration complexity
- (-) Schema changes require DDL migrations for every new field
- (-) Multi-CRM field variations (Salesforce custom fields vs. HubSpot properties) are awkward to model without adding many nullable columns
- (-) Historical state queries require explicit snapshot tables or audit triggers

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Salesforce Opportunity Object | Deal table fields (amount, stage, forecast_category, close_date) align with Salesforce standard field names for seamless sync |
| HubSpot Deals API | Deal properties mapped to normalised columns; association model (deal-contact, deal-company) reflected in junction tables |
| MEDDPICC Framework | Each MEDDPICC element modelled as a separate scoring record with evidence text, enabling per-element coaching |
| ISO 3166-1/2 | Jurisdiction codes on accounts and tenant configuration for data residency compliance |
| OAuth 2.0 (RFC 6749) | Integration credentials table stores encrypted tokens per CRM connection with refresh/expiry tracking |
| ISO/IEC 27001 | Separation of PII (contacts) from business data (deals) supports information classification controls |
| GDPR Art. 17 | Contact table isolation enables targeted erasure without cascading business data loss |
| OpenAPI 3.1 | Table structure maps directly to OpenAPI resource schemas for API generation |

---

## Tenant & Identity Management

```sql
-- ============================================================
-- TENANT & IDENTITY
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',  -- free, pro, enterprise
    data_region     VARCHAR(10) NOT NULL DEFAULT 'us',     -- ISO 3166-1 alpha-2
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE "user" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    auth_provider   VARCHAR(50) NOT NULL,   -- oidc, saml, google, microsoft
    auth_subject    VARCHAR(512),           -- external IdP subject ID
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_user_tenant ON "user"(tenant_id);
CREATE INDEX idx_user_email ON "user"(email);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,  -- admin, manager, rep, viewer
    permissions     TEXT[] NOT NULL DEFAULT '{}',
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES "user"(id),
    PRIMARY KEY (user_id, role_id)
);

-- Manager-rep hierarchy
CREATE TABLE team_membership (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    manager_id      UUID NOT NULL REFERENCES "user"(id),
    rep_id          UUID NOT NULL REFERENCES "user"(id),
    effective_from  DATE NOT NULL DEFAULT CURRENT_DATE,
    effective_to    DATE,  -- NULL = current
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, manager_id, rep_id, effective_from)
);

CREATE INDEX idx_team_membership_manager ON team_membership(manager_id) WHERE effective_to IS NULL;
CREATE INDEX idx_team_membership_rep ON team_membership(rep_id) WHERE effective_to IS NULL;
```

---

## CRM Integration

```sql
-- ============================================================
-- CRM INTEGRATION
-- ============================================================

CREATE TABLE crm_connection (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    provider            VARCHAR(50) NOT NULL,  -- salesforce, hubspot, dynamics365, pipedrive, zoho
    instance_url        TEXT,                   -- e.g., https://myorg.my.salesforce.com
    access_token_enc    BYTEA,                  -- encrypted OAuth access token
    refresh_token_enc   BYTEA,                  -- encrypted OAuth refresh token
    token_expires_at    TIMESTAMPTZ,
    scopes              TEXT[],
    sync_status         VARCHAR(30) NOT NULL DEFAULT 'pending', -- pending, syncing, synced, error
    last_sync_at        TIMESTAMPTZ,
    last_error          TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_crm_connection_tenant ON crm_connection(tenant_id);

CREATE TABLE email_connection (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES "user"(id),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    provider            VARCHAR(50) NOT NULL,  -- gmail, outlook
    email_address       VARCHAR(320) NOT NULL,
    access_token_enc    BYTEA,
    refresh_token_enc   BYTEA,
    token_expires_at    TIMESTAMPTZ,
    scopes              TEXT[],
    is_active           BOOLEAN NOT NULL DEFAULT true,
    last_sync_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_email_connection_user ON email_connection(user_id);

-- Mapping between internal IDs and external CRM IDs
CREATE TABLE crm_entity_mapping (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    crm_connection_id   UUID NOT NULL REFERENCES crm_connection(id),
    entity_type         VARCHAR(50) NOT NULL,   -- account, contact, deal, activity
    internal_id         UUID NOT NULL,
    external_id         VARCHAR(255) NOT NULL,
    external_url        TEXT,
    last_synced_at      TIMESTAMPTZ,
    sync_hash           VARCHAR(64),            -- SHA-256 of last synced payload for change detection
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (crm_connection_id, entity_type, external_id)
);

CREATE INDEX idx_crm_mapping_internal ON crm_entity_mapping(entity_type, internal_id);
CREATE INDEX idx_crm_mapping_external ON crm_entity_mapping(crm_connection_id, entity_type, external_id);
```

---

## Account & Contact Management

```sql
-- ============================================================
-- ACCOUNTS & CONTACTS
-- ============================================================

CREATE TABLE account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(500) NOT NULL,
    domain          VARCHAR(255),
    industry        VARCHAR(100),
    employee_count  INTEGER,
    annual_revenue  BIGINT,              -- in cents
    country_code    CHAR(2),             -- ISO 3166-1 alpha-2
    region          VARCHAR(100),
    owner_id        UUID REFERENCES "user"(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_account_tenant ON account(tenant_id);
CREATE INDEX idx_account_owner ON account(owner_id);
CREATE INDEX idx_account_domain ON account(domain);

CREATE TABLE contact (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    account_id      UUID REFERENCES account(id),
    first_name      VARCHAR(255),
    last_name       VARCHAR(255),
    email           VARCHAR(320),
    phone           VARCHAR(50),
    title           VARCHAR(255),        -- job title
    department      VARCHAR(100),
    linkedin_url    TEXT,
    is_champion     BOOLEAN NOT NULL DEFAULT false,
    is_economic_buyer BOOLEAN NOT NULL DEFAULT false,
    last_contacted_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contact_tenant ON contact(tenant_id);
CREATE INDEX idx_contact_account ON contact(account_id);
CREATE INDEX idx_contact_email ON contact(email);
```

---

## Pipeline & Deal Management

```sql
-- ============================================================
-- PIPELINE & DEALS
-- ============================================================

CREATE TABLE pipeline (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    display_order   INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pipeline_stage (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pipeline_id     UUID NOT NULL REFERENCES pipeline(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    display_order   INTEGER NOT NULL,
    default_probability DECIMAL(5,2) NOT NULL DEFAULT 0,  -- 0.00 to 100.00
    forecast_category VARCHAR(50) NOT NULL DEFAULT 'pipeline',
        -- pipeline, best_case, commit, closed_won, closed_lost, omitted
        -- Aligns with Salesforce ForecastCategory values
    is_closed_won   BOOLEAN NOT NULL DEFAULT false,
    is_closed_lost  BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pipeline_stage_pipeline ON pipeline_stage(pipeline_id);

CREATE TABLE deal (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    pipeline_id         UUID NOT NULL REFERENCES pipeline(id),
    stage_id            UUID NOT NULL REFERENCES pipeline_stage(id),
    account_id          UUID REFERENCES account(id),
    owner_id            UUID NOT NULL REFERENCES "user"(id),
    name                VARCHAR(500) NOT NULL,
    amount              BIGINT,                 -- in cents (aligns with Salesforce Amount)
    currency_code       CHAR(3) NOT NULL DEFAULT 'USD',  -- ISO 4217
    close_date          DATE,                   -- expected close (aligns with Salesforce CloseDate)
    probability         DECIMAL(5,2),           -- AI-adjusted probability (0.00-100.00)
    forecast_category   VARCHAR(50) NOT NULL DEFAULT 'pipeline',
    deal_type           VARCHAR(50),            -- new_business, expansion, renewal
    source              VARCHAR(100),           -- inbound, outbound, referral, partner
    loss_reason         VARCHAR(255),
    competitor          VARCHAR(255),
    days_in_stage       INTEGER NOT NULL DEFAULT 0,
    last_activity_at    TIMESTAMPTZ,
    stage_entered_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_stalled          BOOLEAN NOT NULL DEFAULT false,
    stalled_since       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deal_tenant ON deal(tenant_id);
CREATE INDEX idx_deal_owner ON deal(owner_id);
CREATE INDEX idx_deal_stage ON deal(stage_id);
CREATE INDEX idx_deal_account ON deal(account_id);
CREATE INDEX idx_deal_pipeline ON deal(pipeline_id);
CREATE INDEX idx_deal_close_date ON deal(tenant_id, close_date);
CREATE INDEX idx_deal_stalled ON deal(tenant_id, is_stalled) WHERE is_stalled = true;
CREATE INDEX idx_deal_forecast_cat ON deal(tenant_id, forecast_category);

-- Many-to-many: deals involve multiple contacts
CREATE TABLE deal_contact (
    deal_id         UUID NOT NULL REFERENCES deal(id) ON DELETE CASCADE,
    contact_id      UUID NOT NULL REFERENCES contact(id) ON DELETE CASCADE,
    role            VARCHAR(100),  -- champion, economic_buyer, technical_evaluator, coach, blocker
    engagement_level VARCHAR(50) DEFAULT 'unknown',  -- high, medium, low, unknown, disengaged
    last_interaction_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (deal_id, contact_id)
);

-- Stage change history for waterfall analysis
CREATE TABLE deal_stage_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id         UUID NOT NULL REFERENCES deal(id) ON DELETE CASCADE,
    from_stage_id   UUID REFERENCES pipeline_stage(id),
    to_stage_id     UUID NOT NULL REFERENCES pipeline_stage(id),
    changed_by      UUID REFERENCES "user"(id),
    changed_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    days_in_previous_stage INTEGER
);

CREATE INDEX idx_stage_history_deal ON deal_stage_history(deal_id);
CREATE INDEX idx_stage_history_date ON deal_stage_history(changed_at);
```

---

## MEDDPICC Qualification Scoring

```sql
-- ============================================================
-- MEDDPICC QUALIFICATION
-- ============================================================

-- Configurable scoring criteria per tenant
CREATE TABLE qualification_framework (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,  -- MEDDPICC, MEDDIC, SPICED, BANT
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE qualification_criterion (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id    UUID NOT NULL REFERENCES qualification_framework(id) ON DELETE CASCADE,
    code            VARCHAR(5) NOT NULL,    -- M, E, D1, D2, P, I, C1, C2
    name            VARCHAR(100) NOT NULL,  -- Metrics, Economic Buyer, Decision Criteria, etc.
    description     TEXT,
    weight          DECIMAL(5,2) NOT NULL DEFAULT 1.0,
    max_score       INTEGER NOT NULL DEFAULT 10,
    display_order   INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_qual_criterion_framework ON qualification_criterion(framework_id);

-- Per-deal MEDDPICC scores
CREATE TABLE deal_qualification_score (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id         UUID NOT NULL REFERENCES deal(id) ON DELETE CASCADE,
    criterion_id    UUID NOT NULL REFERENCES qualification_criterion(id),
    score           INTEGER NOT NULL DEFAULT 0,  -- 0 to max_score
    evidence        TEXT,                         -- rep's notes / AI-extracted evidence
    ai_suggested_score INTEGER,                   -- AI recommendation
    ai_confidence   DECIMAL(5,4),                 -- 0.0000 to 1.0000
    scored_by       UUID REFERENCES "user"(id),   -- NULL if AI-scored
    scored_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (deal_id, criterion_id)
);

CREATE INDEX idx_deal_qual_deal ON deal_qualification_score(deal_id);
```

---

## Activity Tracking

```sql
-- ============================================================
-- ACTIVITIES
-- ============================================================

CREATE TABLE activity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    deal_id         UUID REFERENCES deal(id),
    contact_id      UUID REFERENCES contact(id),
    user_id         UUID NOT NULL REFERENCES "user"(id),   -- the rep who performed it
    activity_type   VARCHAR(50) NOT NULL,   -- email_sent, email_received, call, meeting, task, note
    subject         VARCHAR(500),
    body_preview    TEXT,                    -- first 500 chars for search
    occurred_at     TIMESTAMPTZ NOT NULL,
    duration_minutes INTEGER,
    sentiment       VARCHAR(20),            -- positive, neutral, negative (AI-detected)
    source          VARCHAR(50) NOT NULL,   -- gmail, outlook, crm_manual, gong, salesloft
    external_id     VARCHAR(255),           -- message ID or event ID from source system
    is_auto_logged  BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_activity_tenant ON activity(tenant_id);
CREATE INDEX idx_activity_deal ON activity(deal_id);
CREATE INDEX idx_activity_contact ON activity(contact_id);
CREATE INDEX idx_activity_user ON activity(user_id);
CREATE INDEX idx_activity_type ON activity(tenant_id, activity_type);
CREATE INDEX idx_activity_occurred ON activity(deal_id, occurred_at DESC);
```

---

## AI Deal Scoring & Signals

```sql
-- ============================================================
-- AI DEAL SCORING & SIGNALS
-- ============================================================

CREATE TABLE deal_score (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id         UUID NOT NULL REFERENCES deal(id) ON DELETE CASCADE,
    model_version   VARCHAR(50) NOT NULL,
    overall_score   DECIMAL(5,2) NOT NULL,   -- 0.00 to 100.00
    win_probability DECIMAL(5,4) NOT NULL,   -- 0.0000 to 1.0000
    risk_level      VARCHAR(20) NOT NULL,    -- low, medium, high, critical
    momentum        VARCHAR(20) NOT NULL,    -- accelerating, steady, decelerating, stalled
    explanation     TEXT NOT NULL,            -- LLM-generated plain-language explanation
    scored_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ,             -- when this score should be recalculated
    is_current      BOOLEAN NOT NULL DEFAULT true
);

CREATE INDEX idx_deal_score_deal ON deal_score(deal_id) WHERE is_current = true;
CREATE INDEX idx_deal_score_risk ON deal_score(risk_level) WHERE is_current = true;

-- Individual signals that contribute to deal scores
CREATE TABLE deal_signal (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id         UUID NOT NULL REFERENCES deal(id) ON DELETE CASCADE,
    deal_score_id   UUID REFERENCES deal_score(id),
    signal_type     VARCHAR(100) NOT NULL,
        -- champion_dark, single_threaded, competitor_mentioned, legal_review_started,
        -- pricing_objection, no_next_steps, engagement_declining, close_date_pushed,
        -- positive_sentiment, multi_threaded, executive_engaged
    severity        VARCHAR(20) NOT NULL,    -- info, warning, critical
    title           VARCHAR(255) NOT NULL,
    description     TEXT NOT NULL,
    source_activity_id UUID REFERENCES activity(id),
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ,
    is_active       BOOLEAN NOT NULL DEFAULT true
);

CREATE INDEX idx_deal_signal_deal ON deal_signal(deal_id) WHERE is_active = true;
CREATE INDEX idx_deal_signal_type ON deal_signal(signal_type);
```

---

## Coaching Recommendations

```sql
-- ============================================================
-- COACHING RECOMMENDATIONS
-- ============================================================

CREATE TABLE coaching_recommendation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id         UUID NOT NULL REFERENCES deal(id) ON DELETE CASCADE,
    deal_score_id   UUID REFERENCES deal_score(id),
    recommendation_type VARCHAR(50) NOT NULL,
        -- next_step, outreach, meeting_request, proposal_revision,
        -- stakeholder_expansion, objection_handling
    priority        INTEGER NOT NULL DEFAULT 50,  -- 1 = highest
    title           VARCHAR(255) NOT NULL,
    body            TEXT NOT NULL,               -- AI-generated coaching text
    suggested_action TEXT,                       -- specific action for the rep
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
        -- pending, accepted, dismissed, completed
    accepted_by     UUID REFERENCES "user"(id),
    accepted_at     TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_coaching_deal ON coaching_recommendation(deal_id);
CREATE INDEX idx_coaching_status ON coaching_recommendation(status) WHERE status = 'pending';
```

---

## Forecasting

```sql
-- ============================================================
-- FORECASTING
-- ============================================================

CREATE TABLE forecast_period (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,   -- "Q2 2026", "2026-06"
    period_type     VARCHAR(20) NOT NULL,    -- quarterly, monthly
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, period_type, start_date)
);

CREATE TABLE forecast_submission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    period_id       UUID NOT NULL REFERENCES forecast_period(id),
    user_id         UUID NOT NULL REFERENCES "user"(id),  -- the rep or manager submitting
    scenario        VARCHAR(50) NOT NULL,    -- committed, best_case, pipeline
    amount          BIGINT NOT NULL,         -- in cents
    deal_count      INTEGER NOT NULL,
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    notes           TEXT
);

CREATE INDEX idx_forecast_sub_period ON forecast_submission(period_id);
CREATE INDEX idx_forecast_sub_user ON forecast_submission(user_id);

-- Weekly pipeline snapshots for waterfall/movement analysis
CREATE TABLE pipeline_snapshot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    snapshot_date   DATE NOT NULL,
    pipeline_id     UUID NOT NULL REFERENCES pipeline(id),
    owner_id        UUID REFERENCES "user"(id),  -- NULL = pipeline-wide snapshot
    stage_id        UUID NOT NULL REFERENCES pipeline_stage(id),
    deal_count      INTEGER NOT NULL DEFAULT 0,
    total_amount    BIGINT NOT NULL DEFAULT 0,     -- in cents
    weighted_amount BIGINT NOT NULL DEFAULT 0,     -- amount * probability, in cents
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_snapshot_tenant_date ON pipeline_snapshot(tenant_id, snapshot_date);
CREATE INDEX idx_snapshot_pipeline ON pipeline_snapshot(pipeline_id, snapshot_date);

-- Manager override / adjustment on top of AI forecast
CREATE TABLE forecast_adjustment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    period_id       UUID NOT NULL REFERENCES forecast_period(id),
    adjusted_by     UUID NOT NULL REFERENCES "user"(id),  -- the manager
    target_user_id  UUID NOT NULL REFERENCES "user"(id),  -- whose forecast is being adjusted
    scenario        VARCHAR(50) NOT NULL,
    original_amount BIGINT NOT NULL,
    adjusted_amount BIGINT NOT NULL,
    reason          TEXT,
    adjusted_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_forecast_adj_period ON forecast_adjustment(period_id);

-- Manager bias tracking
CREATE TABLE forecast_accuracy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    period_id       UUID NOT NULL REFERENCES forecast_period(id),
    user_id         UUID NOT NULL REFERENCES "user"(id),
    scenario        VARCHAR(50) NOT NULL,
    forecast_amount BIGINT NOT NULL,
    actual_amount   BIGINT NOT NULL,
    variance_pct    DECIMAL(7,2),            -- (actual - forecast) / forecast * 100
    bias_direction  VARCHAR(10),             -- over, under, accurate
    calculated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (period_id, user_id, scenario)
);
```

---

## Pipeline Review Meetings

```sql
-- ============================================================
-- PIPELINE REVIEW MEETINGS
-- ============================================================

CREATE TABLE pipeline_review (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    manager_id      UUID NOT NULL REFERENCES "user"(id),
    period_id       UUID REFERENCES forecast_period(id),
    scheduled_at    TIMESTAMPTZ,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft', -- draft, ready, in_progress, completed
    brief_text      TEXT,                    -- AI-generated pre-meeting brief
    brief_generated_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pipeline_review_deal (
    review_id       UUID NOT NULL REFERENCES pipeline_review(id) ON DELETE CASCADE,
    deal_id         UUID NOT NULL REFERENCES deal(id),
    priority_rank   INTEGER NOT NULL,        -- AI-ranked review priority
    review_notes    TEXT,                     -- manager's notes during review
    coaching_questions TEXT[],                -- AI-suggested questions for the manager
    action_items    TEXT[],
    reviewed        BOOLEAN NOT NULL DEFAULT false,
    PRIMARY KEY (review_id, deal_id)
);
```

---

## Notification & Alerts

```sql
-- ============================================================
-- NOTIFICATIONS & ALERTS
-- ============================================================

CREATE TABLE notification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID NOT NULL REFERENCES "user"(id),
    channel         VARCHAR(30) NOT NULL,    -- in_app, email, slack
    notification_type VARCHAR(50) NOT NULL,
        -- deal_at_risk, stalled_deal, forecast_change, coaching_nudge,
        -- pipeline_review_ready, close_date_approaching
    title           VARCHAR(255) NOT NULL,
    body            TEXT,
    deal_id         UUID REFERENCES deal(id),
    is_read         BOOLEAN NOT NULL DEFAULT false,
    read_at         TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notification_user ON notification(user_id, is_read);
CREATE INDEX idx_notification_deal ON notification(deal_id);
```

---

## Audit Log

```sql
-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID REFERENCES "user"(id),  -- NULL for system actions
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(30) NOT NULL,   -- create, update, delete, score, sync
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    user_agent      TEXT,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, occurred_at DESC);
CREATE INDEX idx_audit_log_user ON audit_log(user_id, occurred_at DESC);
```

---

## Row-Level Security

```sql
-- ============================================================
-- ROW-LEVEL SECURITY POLICIES
-- ============================================================

-- Enable RLS on all tenant-scoped tables
ALTER TABLE account ENABLE ROW LEVEL SECURITY;
ALTER TABLE contact ENABLE ROW LEVEL SECURITY;
ALTER TABLE deal ENABLE ROW LEVEL SECURITY;
ALTER TABLE activity ENABLE ROW LEVEL SECURITY;
ALTER TABLE notification ENABLE ROW LEVEL SECURITY;

-- Example policy: users can only see data in their own tenant
CREATE POLICY tenant_isolation_account ON account
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE POLICY tenant_isolation_deal ON deal
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE POLICY tenant_isolation_contact ON contact
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE POLICY tenant_isolation_activity ON activity
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE POLICY tenant_isolation_notification ON notification
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

---

## Example Queries

```sql
-- Deals at risk with MEDDPICC gaps, ordered by amount
SELECT
    d.name AS deal_name,
    d.amount / 100.0 AS amount_usd,
    d.close_date,
    ds.overall_score,
    ds.risk_level,
    ds.explanation,
    COALESCE(SUM(dqs.score)::DECIMAL / NULLIF(SUM(qc.max_score), 0) * 100, 0) AS meddpicc_pct
FROM deal d
JOIN deal_score ds ON ds.deal_id = d.id AND ds.is_current = true
LEFT JOIN deal_qualification_score dqs ON dqs.deal_id = d.id
LEFT JOIN qualification_criterion qc ON qc.id = dqs.criterion_id
WHERE d.tenant_id = :tenant_id
  AND d.stage_id NOT IN (SELECT id FROM pipeline_stage WHERE is_closed_won OR is_closed_lost)
  AND ds.risk_level IN ('high', 'critical')
GROUP BY d.id, ds.id
ORDER BY d.amount DESC;

-- Week-over-week pipeline movement (waterfall)
SELECT
    ps_prev.stage_id,
    pst.name AS stage_name,
    ps_prev.total_amount / 100.0 AS last_week_amount,
    ps_curr.total_amount / 100.0 AS this_week_amount,
    (ps_curr.total_amount - ps_prev.total_amount) / 100.0 AS movement
FROM pipeline_snapshot ps_curr
JOIN pipeline_snapshot ps_prev
    ON ps_prev.tenant_id = ps_curr.tenant_id
    AND ps_prev.pipeline_id = ps_curr.pipeline_id
    AND ps_prev.stage_id = ps_curr.stage_id
    AND ps_prev.owner_id IS NOT DISTINCT FROM ps_curr.owner_id
    AND ps_prev.snapshot_date = ps_curr.snapshot_date - INTERVAL '7 days'
JOIN pipeline_stage pst ON pst.id = ps_curr.stage_id
WHERE ps_curr.tenant_id = :tenant_id
  AND ps_curr.snapshot_date = CURRENT_DATE
  AND ps_curr.owner_id IS NULL
ORDER BY pst.display_order;

-- Forecast accuracy / manager bias over last 4 quarters
SELECT
    u.display_name,
    fp.name AS period,
    fa.forecast_amount / 100.0 AS forecast_usd,
    fa.actual_amount / 100.0 AS actual_usd,
    fa.variance_pct,
    fa.bias_direction
FROM forecast_accuracy fa
JOIN "user" u ON u.id = fa.user_id
JOIN forecast_period fp ON fp.id = fa.period_id
WHERE u.tenant_id = :tenant_id
  AND fa.scenario = 'committed'
ORDER BY fp.start_date DESC, u.display_name
LIMIT 40;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Identity | 5 | tenant, user, role, user_role, team_membership |
| CRM Integration | 3 | crm_connection, email_connection, crm_entity_mapping |
| Accounts & Contacts | 2 | account, contact |
| Pipeline & Deals | 4 | pipeline, pipeline_stage, deal, deal_contact, deal_stage_history |
| MEDDPICC Qualification | 3 | qualification_framework, qualification_criterion, deal_qualification_score |
| Activities | 1 | activity |
| AI Scoring & Signals | 2 | deal_score, deal_signal |
| Coaching | 1 | coaching_recommendation |
| Forecasting | 5 | forecast_period, forecast_submission, pipeline_snapshot, forecast_adjustment, forecast_accuracy |
| Pipeline Reviews | 2 | pipeline_review, pipeline_review_deal |
| Notifications | 1 | notification |
| Audit | 1 | audit_log |
| **Total** | **30** | |

---

## Key Design Decisions

1. **Monetary values stored in cents (BIGINT)** — Avoids floating-point precision errors inherent in DECIMAL for currency calculations. All amounts are in the smallest currency unit. Currency code stored per deal supports multi-currency tenants.

2. **UUID primary keys everywhere** — Enables distributed ID generation without coordination, simplifies CRM sync (no ID collisions), and prevents enumeration attacks on API endpoints.

3. **Separate deal_score table with is_current flag** — Preserves score history while enabling fast lookups of the latest score. Partial index on `is_current = true` keeps queries efficient as score history grows.

4. **MEDDPICC modelled as framework/criterion/score triple** — Fully configurable: tenants can use MEDDIC (6 criteria), MEDDPICC (8), SPICED, BANT, or custom frameworks. Weights are per-criterion, allowing organisations to emphasise different elements.

5. **Pipeline snapshots for waterfall analysis** — Rather than computing historical state from event logs, weekly snapshots are materialised for fast week-over-week comparison queries. This trades storage for query simplicity.

6. **Team hierarchy via team_membership with temporal range** — Supports manager-rep relationships that change over time (effective_from/effective_to), enabling accurate historical forecast roll-ups even after org chart changes.

7. **Row-Level Security for tenant isolation** — PostgreSQL RLS policies enforce multi-tenant data isolation at the database level, preventing data leaks even if application code has bugs. Uses `current_setting('app.current_tenant_id')` set at connection time.

8. **CRM entity mapping table** — Decouples internal UUIDs from external CRM IDs (Salesforce 18-char, HubSpot numeric, etc.), supports multi-CRM environments, and tracks sync state per entity for incremental sync.

9. **Audit log with old/new JSONB values** — Captures field-level change history without requiring triggers on every table. Supports GDPR accountability requirements and SOC 2 audit evidence.

10. **Forecast accuracy tracking table** — Explicitly records forecast vs. actual at the user/period/scenario level, enabling manager bias detection (a key differentiator from Clari) without complex historical reconstruction.
