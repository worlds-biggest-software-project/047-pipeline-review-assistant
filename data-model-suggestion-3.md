# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Pipeline Review Assistant · Created: 2026-05-12

## Philosophy

This model combines the structural rigour of relational tables for core entities with JSONB columns for variable, provider-specific, and extensible data. The key insight driving this approach is that the Pipeline Review Assistant must integrate with multiple CRM platforms (Salesforce, HubSpot, Dynamics 365, Pipedrive, Zoho) — each with different field sets, custom properties, and data conventions. Rather than creating the union of all possible fields across all CRMs as nullable columns (leading to a sparse, wide table), or maintaining separate table variants per CRM, the hybrid model stores a stable core of normalised fields alongside a `provider_data` JSONB column that preserves the full fidelity of each CRM's native representation.

This pattern is widely used in modern multi-integration SaaS platforms. Stripe stores payment method details in JSONB alongside normalised transaction fields. Segment stores event properties in JSONB alongside normalised event metadata. The pattern works because the application layer (dashboards, scoring, forecasting) operates on the normalised core fields, while CRM sync, data export, and troubleshooting access the full JSONB payload.

The same JSONB flexibility extends to MEDDPICC scoring (configurable frameworks with varying elements), AI signal metadata (each signal type has different attributes), and tenant-specific settings — all areas where the schema varies per customer or changes frequently without requiring DDL migrations.

**Best for:** Teams integrating with multiple CRMs, needing rapid schema evolution, wanting a faster MVP path, and willing to trade some query elegance for deployment flexibility.

**Trade-offs:**
- (+) Multi-CRM integration without schema changes: Salesforce, HubSpot, Dynamics fields coexist naturally
- (+) Fewer tables (20 vs. 30+): less migration overhead, faster development
- (+) New CRM fields or custom properties require no DDL changes — just update JSONB
- (+) MEDDPICC framework customisation is data-driven, not schema-driven
- (+) GIN-indexed JSONB enables fast containment queries
- (+) Good balance between data integrity (core fields) and flexibility (JSONB extensions)
- (-) JSONB fields lack database-level type enforcement — application must validate
- (-) JSONB queries use PostgreSQL-specific operators (`->`, `->>`, `@>`, `jsonb_path_query`) — less portable
- (-) Reporting on JSONB fields requires extraction; BI tools may not handle JSONB natively
- (-) Schema documentation is split between DDL (core) and application code (JSONB shapes)
- (-) Risk of "JSONB everything" — discipline needed to promote frequently-queried JSONB fields to proper columns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Salesforce Opportunity Object | Core deal fields (amount, stage, close_date) normalised; Salesforce-specific fields (ForecastCategoryName, HasOpportunityLineItem, etc.) in `provider_data` |
| HubSpot Deals API | Core properties normalised; HubSpot-specific properties (hs_deal_stage_probability, hs_analytics_source) in `provider_data` |
| Dynamics 365 OData | Core fields normalised; Dataverse-specific metadata in `provider_data` |
| MEDDPICC Framework | Qualification scores stored as JSONB object per deal — configurable elements without schema changes |
| JSON Schema (2020-12) | JSONB columns validated against JSON Schema definitions at the application layer |
| ISO 4217 | Currency codes as normalised CHAR(3) column |
| ISO 3166-1 | Country codes as normalised CHAR(2) column |
| OAuth 2.0 (RFC 6749) | Token storage normalised; provider-specific OAuth metadata in JSONB |
| GDPR Art. 17 | PII in contact table (normalised columns) enables targeted erasure; `provider_data` cleared separately |

---

## Tenant & Identity

```sql
-- ============================================================
-- TENANT & IDENTITY
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    data_region     VARCHAR(10) NOT NULL DEFAULT 'us',   -- ISO 3166-1 alpha-2
    -- Tenant-specific configuration as JSONB
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "qualification_framework": "meddpicc",
    --   "stale_deal_threshold_days": 14,
    --   "forecast_scenarios": ["committed", "best_case", "pipeline"],
    --   "scoring_model": "v2.3",
    --   "slack_webhook_url": "https://hooks.slack.com/...",
    --   "notification_preferences": { "stalled_deal": true, "score_drop": true },
    --   "custom_deal_fields": [
    --     {"key": "implementation_complexity", "label": "Implementation Complexity", "type": "select", "options": ["low","medium","high"]}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE "user" (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'rep',  -- admin, manager, rep, viewer
    manager_id      UUID REFERENCES "user"(id),          -- direct manager (self-referencing)
    auth_provider   VARCHAR(50) NOT NULL,
    auth_subject    VARCHAR(512),
    permissions     TEXT[] NOT NULL DEFAULT '{}',
    profile         JSONB NOT NULL DEFAULT '{}',
    -- profile example:
    -- {
    --   "avatar_url": "...",
    --   "timezone": "America/New_York",
    --   "notification_channels": ["in_app", "slack"],
    --   "dashboard_layout": "compact"
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_user_tenant ON "user"(tenant_id);
CREATE INDEX idx_user_manager ON "user"(manager_id);
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
    provider            VARCHAR(50) NOT NULL,
    instance_url        TEXT,
    access_token_enc    BYTEA,
    refresh_token_enc   BYTEA,
    token_expires_at    TIMESTAMPTZ,
    scopes              TEXT[],
    sync_status         VARCHAR(30) NOT NULL DEFAULT 'pending',
    last_sync_at        TIMESTAMPTZ,
    last_error          TEXT,
    -- Provider-specific connection metadata
    provider_config     JSONB NOT NULL DEFAULT '{}',
    -- Salesforce example:
    -- {
    --   "org_id": "00Dxx0000001234",
    --   "my_domain": "myorg.my.salesforce.com",
    --   "api_version": "v66.0",
    --   "field_mapping": {
    --     "opportunity.amount": "Amount",
    --     "opportunity.stage": "StageName",
    --     "opportunity.close_date": "CloseDate"
    --   },
    --   "custom_field_mapping": {
    --     "opportunity.competitor__c": "competitor",
    --     "opportunity.meddic_score__c": "meddpicc_total"
    --   }
    -- }
    -- HubSpot example:
    -- {
    --   "portal_id": "12345678",
    --   "pipeline_id": "default",
    --   "field_mapping": {
    --     "deal.amount": "amount",
    --     "deal.dealstage": "stage_id",
    --     "deal.closedate": "close_date"
    --   }
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_crm_conn_tenant ON crm_connection(tenant_id);

CREATE TABLE email_connection (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES "user"(id),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    provider            VARCHAR(50) NOT NULL,   -- gmail, outlook
    email_address       VARCHAR(320) NOT NULL,
    access_token_enc    BYTEA,
    refresh_token_enc   BYTEA,
    token_expires_at    TIMESTAMPTZ,
    provider_config     JSONB NOT NULL DEFAULT '{}',
    -- Gmail example:
    -- { "history_id": "12345", "watch_expiration": "2026-06-01T00:00:00Z" }
    -- Outlook example:
    -- { "subscription_id": "...", "delta_link": "..." }
    is_active           BOOLEAN NOT NULL DEFAULT true,
    last_sync_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_email_conn_user ON email_connection(user_id);
```

---

## Accounts & Contacts

```sql
-- ============================================================
-- ACCOUNTS & CONTACTS
-- ============================================================

CREATE TABLE account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    crm_connection_id UUID REFERENCES crm_connection(id),
    external_id     VARCHAR(255),            -- CRM-native ID
    name            VARCHAR(500) NOT NULL,
    domain          VARCHAR(255),
    industry        VARCHAR(100),
    country_code    CHAR(2),                 -- ISO 3166-1
    owner_id        UUID REFERENCES "user"(id),
    -- All CRM-specific and custom fields preserved here
    provider_data   JSONB NOT NULL DEFAULT '{}',
    -- Salesforce example:
    -- {
    --   "Type": "Customer",
    --   "AnnualRevenue": 50000000,
    --   "NumberOfEmployees": 200,
    --   "BillingState": "CA",
    --   "Rating": "Hot",
    --   "custom_segment__c": "mid-market"
    -- }
    sync_hash       VARCHAR(64),
    last_synced_at  TIMESTAMPTZ,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_account_tenant ON account(tenant_id);
CREATE INDEX idx_account_external ON account(crm_connection_id, external_id);
CREATE INDEX idx_account_domain ON account(domain);
-- GIN index for JSONB queries on provider-specific fields
CREATE INDEX idx_account_provider ON account USING GIN (provider_data jsonb_path_ops);

CREATE TABLE contact (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    account_id      UUID REFERENCES account(id),
    crm_connection_id UUID REFERENCES crm_connection(id),
    external_id     VARCHAR(255),
    first_name      VARCHAR(255),
    last_name       VARCHAR(255),
    email           VARCHAR(320),
    phone           VARCHAR(50),
    title           VARCHAR(255),
    -- Provider-specific and custom contact fields
    provider_data   JSONB NOT NULL DEFAULT '{}',
    -- { "LeadSource": "Web", "MailingCity": "San Francisco", "custom_persona__c": "technical_buyer" }
    last_contacted_at TIMESTAMPTZ,
    sync_hash       VARCHAR(64),
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contact_tenant ON contact(tenant_id);
CREATE INDEX idx_contact_account ON contact(account_id);
CREATE INDEX idx_contact_email ON contact(email);
CREATE INDEX idx_contact_external ON contact(crm_connection_id, external_id);
```

---

## Pipeline & Deals (Core + JSONB Hybrid)

```sql
-- ============================================================
-- PIPELINE & DEALS
-- ============================================================

CREATE TABLE pipeline (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    stages          JSONB NOT NULL DEFAULT '[]',
    -- stages example:
    -- [
    --   {"id": "uuid-1", "name": "Prospecting", "order": 1, "probability": 10, "forecast_category": "pipeline"},
    --   {"id": "uuid-2", "name": "Discovery", "order": 2, "probability": 20, "forecast_category": "pipeline"},
    --   {"id": "uuid-3", "name": "Proposal", "order": 3, "probability": 50, "forecast_category": "best_case"},
    --   {"id": "uuid-4", "name": "Negotiation", "order": 4, "probability": 75, "forecast_category": "commit"},
    --   {"id": "uuid-5", "name": "Closed Won", "order": 5, "probability": 100, "forecast_category": "closed_won", "is_closed_won": true},
    --   {"id": "uuid-6", "name": "Closed Lost", "order": 6, "probability": 0, "forecast_category": "closed_lost", "is_closed_lost": true}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE deal (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    pipeline_id         UUID NOT NULL REFERENCES pipeline(id),
    account_id          UUID REFERENCES account(id),
    owner_id            UUID NOT NULL REFERENCES "user"(id),
    crm_connection_id   UUID REFERENCES crm_connection(id),
    external_id         VARCHAR(255),

    -- ========== NORMALISED CORE (used by scoring, dashboards, forecasting) ==========
    name                VARCHAR(500) NOT NULL,
    stage_id            VARCHAR(100) NOT NULL,           -- references pipeline.stages[].id
    stage_name          VARCHAR(255) NOT NULL,           -- denormalised for dashboard perf
    amount_cents        BIGINT,
    currency_code       CHAR(3) NOT NULL DEFAULT 'USD',  -- ISO 4217
    close_date          DATE,
    forecast_category   VARCHAR(50) NOT NULL DEFAULT 'pipeline',
    deal_type           VARCHAR(50),
    source              VARCHAR(100),
    probability         DECIMAL(5,2),
    stage_entered_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    days_in_stage       INTEGER NOT NULL DEFAULT 0,
    last_activity_at    TIMESTAMPTZ,
    is_open             BOOLEAN NOT NULL DEFAULT true,
    is_won              BOOLEAN NOT NULL DEFAULT false,

    -- ========== AI SCORING (denormalised for fast dashboard queries) ==========
    ai_score            DECIMAL(5,2),
    ai_risk_level       VARCHAR(20),         -- low, medium, high, critical
    ai_momentum         VARCHAR(20),         -- accelerating, steady, decelerating, stalled
    ai_explanation      TEXT,
    ai_scored_at        TIMESTAMPTZ,

    -- ========== QUALIFICATION (MEDDPICC as JSONB — configurable per tenant) ==========
    qualification       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "framework": "MEDDPICC",
    --   "total_score": 51,
    --   "max_possible": 80,
    --   "percentage": 63.75,
    --   "elements": {
    --     "M": {"score": 8, "max": 10, "label": "Metrics", "evidence": "ROI model showing 3.2x return"},
    --     "E": {"score": 9, "max": 10, "label": "Economic Buyer", "evidence": "VP Sales confirmed"},
    --     "D1": {"score": 6, "max": 10, "label": "Decision Criteria", "evidence": "Partially documented"},
    --     "D2": {"score": 5, "max": 10, "label": "Decision Process", "evidence": "Timeline unclear"},
    --     "P": {"score": 3, "max": 10, "label": "Paper Process", "evidence": "Not discussed"},
    --     "I": {"score": 9, "max": 10, "label": "Identify Pain", "evidence": "87% forecast miss rate"},
    --     "C1": {"score": 7, "max": 10, "label": "Champion", "evidence": "VP Sales; need CRO access"},
    --     "C2": {"score": 4, "max": 10, "label": "Competition", "evidence": "Clari and Gong in eval"}
    --   },
    --   "gaps": ["D1", "D2", "P", "C2"],
    --   "scored_by": "ai_engine",
    --   "scored_at": "2026-05-10T14:30:00Z"
    -- }

    -- ========== CONTACTS ON DEAL (denormalised for fast single-query access) ==========
    contacts            JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"contact_id": "uuid", "name": "Jane Doe", "title": "VP Sales", "role": "champion", "engagement": "high", "last_interaction": "2026-05-08"},
    --   {"contact_id": "uuid", "name": "Bob Smith", "title": "CFO", "role": "economic_buyer", "engagement": "medium", "last_interaction": "2026-04-25"}
    -- ]

    -- ========== ACTIVE SIGNALS (denormalised for dashboard) ==========
    signals             JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"type": "champion_dark", "severity": "critical", "title": "Champion inactive 12 days", "detected_at": "2026-05-08"},
    --   {"type": "competitor_mentioned", "severity": "warning", "title": "Gong mentioned in last call", "detected_at": "2026-05-07"}
    -- ]

    -- ========== CRM-SPECIFIC FIELDS (full fidelity preservation) ==========
    provider_data       JSONB NOT NULL DEFAULT '{}',
    -- Salesforce example:
    -- {
    --   "Id": "006xx000001234ABC",
    --   "ForecastCategoryName": "Commit",
    --   "HasOpportunityLineItem": true,
    --   "IsPrivate": false,
    --   "NextStep": "Send revised proposal",
    --   "Description": "Enterprise license for 200 seats",
    --   "LeadSource": "Web",
    --   "CampaignId": "701xx000001234",
    --   "custom_impl_complexity__c": "high",
    --   "custom_contract_term__c": 36
    -- }

    -- ========== CUSTOM TENANT FIELDS (tenant-defined extensions) ==========
    custom_fields       JSONB NOT NULL DEFAULT '{}',
    -- { "implementation_complexity": "high", "partner_involved": true, "partner_name": "Acme Consulting" }

    sync_hash           VARCHAR(64),
    last_synced_at      TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deal_tenant ON deal(tenant_id);
CREATE INDEX idx_deal_owner ON deal(owner_id);
CREATE INDEX idx_deal_account ON deal(account_id);
CREATE INDEX idx_deal_pipeline ON deal(pipeline_id);
CREATE INDEX idx_deal_close ON deal(tenant_id, close_date) WHERE is_open = true;
CREATE INDEX idx_deal_risk ON deal(tenant_id, ai_risk_level) WHERE is_open = true;
CREATE INDEX idx_deal_stalled ON deal(tenant_id, ai_momentum) WHERE ai_momentum = 'stalled' AND is_open = true;
CREATE INDEX idx_deal_forecast ON deal(tenant_id, forecast_category) WHERE is_open = true;
CREATE INDEX idx_deal_external ON deal(crm_connection_id, external_id);

-- GIN indexes for JSONB queries
CREATE INDEX idx_deal_qualification ON deal USING GIN (qualification jsonb_path_ops);
CREATE INDEX idx_deal_signals ON deal USING GIN (signals jsonb_path_ops);
CREATE INDEX idx_deal_provider ON deal USING GIN (provider_data jsonb_path_ops);
CREATE INDEX idx_deal_custom ON deal USING GIN (custom_fields jsonb_path_ops);
```

---

## Deal History & Audit

```sql
-- ============================================================
-- DEAL HISTORY (append-only change log)
-- ============================================================

CREATE TABLE deal_change (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id         UUID NOT NULL REFERENCES deal(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL,
    change_type     VARCHAR(50) NOT NULL,
        -- stage_changed, amount_changed, score_recalculated, qualification_updated,
        -- signal_detected, signal_resolved, contact_added, contact_removed,
        -- forecast_category_changed, close_date_changed, owner_changed, field_updated
    changed_by      UUID REFERENCES "user"(id),  -- NULL for system/AI changes
    source          VARCHAR(50) NOT NULL DEFAULT 'system',  -- user, crm_sync, ai_engine
    changes         JSONB NOT NULL,
    -- stage_changed example:
    -- {
    --   "from_stage": "Discovery",
    --   "to_stage": "Proposal",
    --   "from_stage_id": "uuid-2",
    --   "to_stage_id": "uuid-3",
    --   "days_in_previous_stage": 14
    -- }
    -- score_recalculated example:
    -- {
    --   "previous_score": 81.2,
    --   "new_score": 72.5,
    --   "delta": -8.7,
    --   "model_version": "v2.3.1",
    --   "risk_level_change": "medium -> high",
    --   "trigger": "champion_dark_signal"
    -- }
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deal_change_deal ON deal_change(deal_id, occurred_at DESC);
CREATE INDEX idx_deal_change_tenant ON deal_change(tenant_id, occurred_at DESC);
CREATE INDEX idx_deal_change_type ON deal_change(change_type);
```

---

## Activities

```sql
-- ============================================================
-- ACTIVITIES
-- ============================================================

CREATE TABLE activity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    deal_id         UUID REFERENCES deal(id),
    contact_id      UUID REFERENCES contact(id),
    user_id         UUID NOT NULL REFERENCES "user"(id),
    activity_type   VARCHAR(50) NOT NULL,
    subject         VARCHAR(500),
    body_preview    TEXT,
    occurred_at     TIMESTAMPTZ NOT NULL,
    duration_minutes INTEGER,
    sentiment       VARCHAR(20),
    source          VARCHAR(50) NOT NULL,
    external_id     VARCHAR(255),
    is_auto_logged  BOOLEAN NOT NULL DEFAULT false,
    -- Activity-type-specific metadata in JSONB
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- email example:
    -- {
    --   "direction": "outbound",
    --   "recipients": ["jane@acme.com", "bob@acme.com"],
    --   "thread_id": "msg-thread-123",
    --   "has_attachment": true,
    --   "response_time_hours": 4.5
    -- }
    -- call example:
    -- {
    --   "direction": "outbound",
    --   "disposition": "connected",
    --   "recording_url": "https://...",
    --   "transcript_id": "tr-456",
    --   "topics_detected": ["pricing", "timeline", "competitor"],
    --   "talk_ratio": 0.35
    -- }
    -- meeting example:
    -- {
    --   "attendees": ["jane@acme.com", "bob@acme.com", "rep@ourcompany.com"],
    --   "location": "Zoom",
    --   "meeting_url": "https://zoom.us/j/...",
    --   "agenda": "Proposal review and pricing discussion"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_activity_deal ON activity(deal_id, occurred_at DESC);
CREATE INDEX idx_activity_contact ON activity(contact_id, occurred_at DESC);
CREATE INDEX idx_activity_user ON activity(user_id, occurred_at DESC);
CREATE INDEX idx_activity_tenant ON activity(tenant_id, occurred_at DESC);
CREATE INDEX idx_activity_type ON activity(tenant_id, activity_type);
CREATE INDEX idx_activity_metadata ON activity USING GIN (metadata jsonb_path_ops);
```

---

## AI Score History

```sql
-- ============================================================
-- AI SCORE HISTORY
-- ============================================================

CREATE TABLE deal_score_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id         UUID NOT NULL REFERENCES deal(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL,
    model_version   VARCHAR(50) NOT NULL,
    overall_score   DECIMAL(5,2) NOT NULL,
    win_probability DECIMAL(5,4) NOT NULL,
    risk_level      VARCHAR(20) NOT NULL,
    momentum        VARCHAR(20) NOT NULL,
    explanation     TEXT NOT NULL,
    -- Full scoring context for AI transparency / EU AI Act compliance
    scoring_context JSONB NOT NULL,
    -- {
    --   "input_signals": [
    --     {"type": "activity_recency", "value": 18, "unit": "days"},
    --     {"type": "champion_engagement", "value": "declining"},
    --     {"type": "meddpicc_score", "value": 63.75},
    --     {"type": "stage_velocity", "value": "below_average"},
    --     {"type": "competitor_signal", "competitor": "Gong", "source": "call_transcript"}
    --   ],
    --   "previous_score": 81.2,
    --   "score_delta": -8.7,
    --   "trigger_event": "champion_dark_14d",
    --   "processing_time_ms": 342
    -- }
    scored_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_score_history_deal ON deal_score_history(deal_id, scored_at DESC);
CREATE INDEX idx_score_history_tenant ON deal_score_history(tenant_id, scored_at DESC);
```

---

## Coaching Recommendations

```sql
-- ============================================================
-- COACHING RECOMMENDATIONS
-- ============================================================

CREATE TABLE coaching_recommendation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    deal_id         UUID NOT NULL REFERENCES deal(id) ON DELETE CASCADE,
    owner_id        UUID NOT NULL REFERENCES "user"(id),
    recommendation_type VARCHAR(50) NOT NULL,
    priority        INTEGER NOT NULL DEFAULT 50,
    title           VARCHAR(255) NOT NULL,
    body            TEXT NOT NULL,
    suggested_action TEXT,
    context         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "trigger_signal": "champion_dark",
    --   "related_contacts": ["uuid-1"],
    --   "related_activities": ["uuid-2"],
    --   "model_version": "v2.3.1"
    -- }
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    actioned_by     UUID REFERENCES "user"(id),
    actioned_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_coaching_deal ON coaching_recommendation(deal_id);
CREATE INDEX idx_coaching_owner ON coaching_recommendation(owner_id) WHERE status = 'pending';
```

---

## Forecasting

```sql
-- ============================================================
-- FORECASTING
-- ============================================================

CREATE TABLE forecast_submission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID NOT NULL REFERENCES "user"(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    period_label    VARCHAR(100) NOT NULL,   -- "Q2 2026"
    scenario        VARCHAR(50) NOT NULL,    -- committed, best_case, pipeline
    amount_cents    BIGINT NOT NULL,
    deal_count      INTEGER NOT NULL,
    -- Detailed breakdown
    breakdown       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "by_stage": {
    --     "Proposal": {"count": 5, "amount": 8000000},
    --     "Negotiation": {"count": 3, "amount": 12000000}
    --   },
    --   "by_rep": [
    --     {"user_id": "uuid", "name": "Alice", "amount": 5000000, "deal_count": 4}
    --   ],
    --   "adjustments": [
    --     {"deal_id": "uuid", "deal_name": "Acme", "original": 5000000, "adjusted": 4000000, "reason": "Likely to slip"}
    --   ]
    -- }
    notes           TEXT,
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_forecast_tenant ON forecast_submission(tenant_id, period_start);
CREATE INDEX idx_forecast_user ON forecast_submission(user_id, period_start);

-- Pipeline snapshots (weekly materialised)
CREATE TABLE pipeline_snapshot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    snapshot_date   DATE NOT NULL,
    pipeline_id     UUID NOT NULL REFERENCES pipeline(id),
    owner_id        UUID,
    -- Stage-level breakdown in JSONB (avoids N rows per stage)
    stages          JSONB NOT NULL,
    -- [
    --   {"stage_id": "uuid-1", "stage_name": "Prospecting", "deal_count": 15, "total_amount": 30000000, "weighted_amount": 3000000},
    --   {"stage_id": "uuid-2", "stage_name": "Discovery", "deal_count": 8, "total_amount": 24000000, "weighted_amount": 4800000},
    --   ...
    -- ]
    total_deals     INTEGER NOT NULL,
    total_amount    BIGINT NOT NULL,
    total_weighted  BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, snapshot_date, pipeline_id, owner_id)
);

CREATE INDEX idx_snapshot_tenant ON pipeline_snapshot(tenant_id, snapshot_date DESC);

-- Forecast accuracy tracking
CREATE TABLE forecast_accuracy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID NOT NULL REFERENCES "user"(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    scenario        VARCHAR(50) NOT NULL,
    forecast_amount BIGINT NOT NULL,
    actual_amount   BIGINT NOT NULL,
    variance_pct    DECIMAL(7,2),
    bias_direction  VARCHAR(10),            -- over, under, accurate
    calculated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, user_id, period_start, scenario)
);

CREATE INDEX idx_accuracy_user ON forecast_accuracy(user_id);
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
    scheduled_at    TIMESTAMPTZ,
    period_label    VARCHAR(100),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    -- AI-generated review brief as structured JSONB
    brief           JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "generated_at": "2026-05-10T08:00:00Z",
    --   "period": "Q2 2026",
    --   "summary": "12 deals in pipeline worth $4.5M. 3 deals at high risk...",
    --   "deals_to_review": [
    --     {
    --       "deal_id": "uuid",
    --       "deal_name": "Acme Corp",
    --       "amount": 1500000,
    --       "risk_level": "high",
    --       "priority_rank": 1,
    --       "coaching_questions": [
    --         "Has the champion responded to the last two outreach attempts?",
    --         "What is the timeline for the legal review?"
    --       ],
    --       "key_signals": ["champion_dark", "competitor_mentioned"]
    --     }
    --   ],
    --   "forecast_summary": {
    --     "committed": 3200000,
    --     "best_case": 4500000,
    --     "gap_to_quota": 800000
    --   },
    --   "rep_coaching_priorities": [
    --     {"rep_id": "uuid", "rep_name": "Alice", "focus": "Multi-threading on enterprise deals"}
    --   ]
    -- }
    review_notes    TEXT,
    action_items    JSONB NOT NULL DEFAULT '[]',
    -- [{"deal_id": "uuid", "action": "Schedule champion call", "assigned_to": "uuid", "due_date": "2026-05-15"}]
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_review_manager ON pipeline_review(manager_id);
CREATE INDEX idx_review_tenant ON pipeline_review(tenant_id, scheduled_at DESC);
```

---

## Notifications

```sql
-- ============================================================
-- NOTIFICATIONS
-- ============================================================

CREATE TABLE notification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID NOT NULL REFERENCES "user"(id),
    channel         VARCHAR(30) NOT NULL,
    notification_type VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    body            TEXT,
    deal_id         UUID REFERENCES deal(id),
    context         JSONB NOT NULL DEFAULT '{}',
    -- { "risk_level": "critical", "signal_type": "champion_dark", "score_drop": -8.7 }
    is_read         BOOLEAN NOT NULL DEFAULT false,
    read_at         TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notification_user ON notification(user_id, is_read);
```

---

## Row-Level Security

```sql
-- ============================================================
-- ROW-LEVEL SECURITY
-- ============================================================

ALTER TABLE deal ENABLE ROW LEVEL SECURITY;
ALTER TABLE account ENABLE ROW LEVEL SECURITY;
ALTER TABLE contact ENABLE ROW LEVEL SECURITY;
ALTER TABLE activity ENABLE ROW LEVEL SECURITY;
ALTER TABLE notification ENABLE ROW LEVEL SECURITY;
ALTER TABLE coaching_recommendation ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON deal
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON account
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON contact
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON activity
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON notification
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation ON coaching_recommendation
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

---

## Example Queries

```sql
-- Dashboard: all open deals at high/critical risk with their signals
-- Note: signals are embedded in the deal row — no joins needed
SELECT
    d.name,
    d.amount_cents / 100.0 AS amount_usd,
    d.close_date,
    d.ai_score,
    d.ai_risk_level,
    d.ai_explanation,
    d.qualification->>'percentage' AS meddpicc_pct,
    d.qualification->'gaps' AS meddpicc_gaps,
    d.signals AS active_signals,
    d.contacts AS deal_contacts
FROM deal d
WHERE d.tenant_id = :tenant_id
  AND d.is_open = true
  AND d.ai_risk_level IN ('high', 'critical')
ORDER BY d.amount_cents DESC;

-- Find deals with a specific MEDDPICC gap (Paper Process score < 5)
SELECT d.name, d.amount_cents / 100.0 AS amount_usd,
    (d.qualification->'elements'->'P'->>'score')::INTEGER AS paper_process_score
FROM deal d
WHERE d.tenant_id = :tenant_id
  AND d.is_open = true
  AND (d.qualification->'elements'->'P'->>'score')::INTEGER < 5
ORDER BY d.amount_cents DESC;

-- Find deals with a specific signal type active
SELECT d.name, d.amount_cents / 100.0 AS amount_usd, d.ai_risk_level
FROM deal d
WHERE d.tenant_id = :tenant_id
  AND d.is_open = true
  AND d.signals @> '[{"type": "champion_dark"}]'::JSONB;

-- Week-over-week pipeline movement using JSONB snapshot
SELECT
    snap_curr.snapshot_date,
    curr_stage->>'stage_name' AS stage,
    (curr_stage->>'total_amount')::BIGINT / 100.0 AS this_week,
    (prev_stage->>'total_amount')::BIGINT / 100.0 AS last_week,
    ((curr_stage->>'total_amount')::BIGINT - (prev_stage->>'total_amount')::BIGINT) / 100.0 AS movement
FROM pipeline_snapshot snap_curr,
    LATERAL jsonb_array_elements(snap_curr.stages) AS curr_stage
JOIN pipeline_snapshot snap_prev
    ON snap_prev.tenant_id = snap_curr.tenant_id
    AND snap_prev.pipeline_id = snap_curr.pipeline_id
    AND snap_prev.owner_id IS NOT DISTINCT FROM snap_curr.owner_id
    AND snap_prev.snapshot_date = snap_curr.snapshot_date - 7
CROSS JOIN LATERAL jsonb_array_elements(snap_prev.stages) AS prev_stage
WHERE snap_curr.tenant_id = :tenant_id
  AND snap_curr.snapshot_date = CURRENT_DATE
  AND snap_curr.owner_id IS NULL
  AND curr_stage->>'stage_id' = prev_stage->>'stage_id'
ORDER BY (curr_stage->>'stage_name');

-- Salesforce-specific query: find deals with a custom field value
SELECT d.name, d.provider_data->>'custom_impl_complexity__c' AS complexity
FROM deal d
WHERE d.tenant_id = :tenant_id
  AND d.is_open = true
  AND d.provider_data @> '{"custom_impl_complexity__c": "high"}'::JSONB;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Identity | 2 | tenant, user (roles embedded via column + permissions array) |
| CRM Integration | 2 | crm_connection, email_connection |
| Accounts & Contacts | 2 | account, contact |
| Pipeline & Deals | 2 | pipeline (stages in JSONB), deal (qualification, signals, contacts in JSONB) |
| Deal History | 1 | deal_change (append-only change log) |
| Activities | 1 | activity (type-specific metadata in JSONB) |
| AI Scoring | 1 | deal_score_history |
| Coaching | 1 | coaching_recommendation |
| Forecasting | 3 | forecast_submission, pipeline_snapshot, forecast_accuracy |
| Pipeline Reviews | 1 | pipeline_review (brief and action items in JSONB) |
| Notifications | 1 | notification |
| **Total** | **17** | Roughly half the table count of the normalised model |

---

## Key Design Decisions

1. **Core fields normalised, variable fields in JSONB** — Amount, stage, close_date, owner, and risk_level are typed columns with indexes because every query needs them. CRM-specific fields (Salesforce custom fields, HubSpot properties), MEDDPICC scores, and active signals live in JSONB because their structure varies per tenant, per CRM, and per framework configuration.

2. **Pipeline stages stored as JSONB array in pipeline table** — Eliminates the pipeline_stage junction table. Stages rarely exceed 10 per pipeline, so JSONB array is more practical than a separate table. Stage IDs within the JSONB are referenced by the deal's `stage_id` VARCHAR column.

3. **Deal contacts denormalised into deal.contacts JSONB** — Most dashboard queries need contact names and roles alongside deal data. Embedding them in the deal row eliminates the deal_contact junction table join. The authoritative contact records remain in the contact table; the JSONB is a synchronised summary.

4. **Active signals denormalised into deal.signals JSONB** — Dashboard queries frequently filter or display signals alongside deals. Embedding them avoids a join. The JSONB containment operator (`@>`) with a GIN index enables queries like "find all deals with champion_dark signal" efficiently.

5. **MEDDPICC as JSONB object, not separate tables** — Different tenants use MEDDIC (6 elements), MEDDPICC (8 elements), SPICED, BANT, or custom frameworks. Rather than maintaining framework/criterion/score tables that require DDL changes for new frameworks, the entire qualification state is a single JSONB object per deal. JSON Schema validation at the application layer ensures structural correctness.

6. **provider_data preserves full CRM fidelity** — When syncing from Salesforce, the entire Opportunity API response (minus sensitive fields) is stored in `provider_data`. This enables round-trip sync without data loss, troubleshooting CRM sync issues, and supporting CRM-specific features without schema changes.

7. **deal_change table as append-only change log** — Rather than full event sourcing (suggestion 2), this model uses a simpler change log that records diffs as JSONB. It provides audit trail and historical analysis without the architectural complexity of CQRS projections.

8. **Pipeline snapshot stages as JSONB array** — Rather than one row per stage per snapshot (which can produce thousands of rows), each snapshot stores all stage data in a single JSONB array. This reduces table bloat and simplifies retention management, at the cost of slightly more complex waterfall queries using `jsonb_array_elements`.

9. **GIN indexes on all JSONB columns** — The `jsonb_path_ops` GIN index class is used consistently across provider_data, qualification, signals, and custom_fields columns. This enables efficient containment queries (`@>`) which are the primary JSONB access pattern for filtering and searching.

10. **Tenant configuration in JSONB rather than config tables** — Stale deal thresholds, scoring model version, notification preferences, and custom field definitions are stored in `tenant.config` JSONB. This avoids a proliferation of small configuration tables and makes tenant setup a single-document operation. JSON Schema validation in the application layer ensures configuration correctness.
