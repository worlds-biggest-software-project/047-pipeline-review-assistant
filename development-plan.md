# Pipeline Review Assistant — Development Plan

> Project: Pipeline Review Assistant (Candidate #47)
> Created: 2026-05-25
> Status: Planning

---

## Technology Decisions & Rationale

### Database: PostgreSQL 16+ with Hybrid Relational + JSONB Model (Data Model Suggestion 3)

**Decision:** Adopt the Hybrid Relational + JSONB data model (Suggestion 3) as the primary schema design.

**Rationale:**
- The Pipeline Review Assistant must integrate with 5+ CRM platforms (Salesforce, HubSpot, Dynamics 365, Pipedrive, Zoho), each with different field sets and custom properties. The hybrid model stores normalised core fields (amount, stage, close_date, owner) for performant dashboard queries alongside `provider_data` JSONB columns that preserve full CRM fidelity without schema changes per provider.
- MEDDPICC qualification frameworks vary per tenant (MEDDIC, MEDDPICC, SPICED, BANT, custom). Storing qualification state as a JSONB object per deal avoids a separate framework/criterion/score table triple and enables configuration changes without DDL migrations.
- 17 tables vs. 30+ in the fully normalised model (Suggestion 1) reduces migration complexity and accelerates MVP delivery.
- Compared to full event sourcing (Suggestion 2), the hybrid model adds a simpler append-only `deal_change` table for audit and historical analysis without requiring CQRS projection infrastructure, event outbox patterns, or eventual-consistency reasoning that would slow early development.
- Retains the option to add event sourcing later: the `deal_change` table is structurally similar to an event log and can be evolved into a full event store if the product matures into enterprise-grade temporal analysis.
- GIN-indexed JSONB enables efficient containment queries (`@>`) for signal filtering, custom field searches, and provider-specific queries.

**Key supplement from Suggestion 1:** Adopt Row-Level Security (RLS) policies from Suggestion 1 for tenant isolation at the database level. Adopt the `crm_entity_mapping` pattern from Suggestion 1 as a lightweight sync-state tracker alongside the `sync_hash` columns in Suggestion 3.

**Key supplement from Suggestion 2:** Adopt the `projection_checkpoint` concept from Suggestion 2 for tracking CRM sync and AI scoring job progress. Adopt the detailed AI scoring provenance format from Suggestion 2's `DealScoreCalculated` event payload as the schema for `deal_score_history.scoring_context`.

### Backend: Node.js 22 LTS + TypeScript 5.x

**Rationale:**
- TypeScript provides type safety for the complex JSONB structures (qualification, signals, provider_data) via Zod schemas that double as runtime validators and OpenAPI type generators.
- The CRM integration ecosystem (jsforce for Salesforce, HubSpot Node.js client, Microsoft Graph SDK for JavaScript, Pipedrive Node.js client, Zoho CRM Node.js SDK) is strongest on Node.js.
- Excellent async I/O model for handling concurrent CRM API calls, email sync polling, and webhook processing.
- LLM integration libraries (Anthropic SDK, OpenAI SDK) have first-class TypeScript support.

### API Framework: Fastify 5.x

**Rationale:**
- High-performance HTTP framework with built-in JSON Schema validation, matching the JSON Schema validation strategy for JSONB columns.
- Native OpenAPI 3.1 spec generation via `@fastify/swagger`, fulfilling the standards.md requirement to publish a machine-readable API surface.
- Plugin architecture supports clean separation of CRM sync, scoring engine, and notification subsystems.

### Frontend: Next.js 15 (App Router) + React 19 + Tailwind CSS 4

**Rationale:**
- Server Components reduce client bundle size for data-heavy pipeline dashboards.
- Streaming SSR enables progressive rendering of deal lists while AI scores load.
- shadcn/ui provides the component primitives (tables, cards, charts) needed for pipeline dashboards without design overhead.
- Recharts or Tremor for waterfall charts and forecast visualisations.

### AI/LLM: Claude API (Anthropic) via TypeScript SDK

**Rationale:**
- Claude excels at structured reasoning over multi-document context (email threads + CRM history + call notes), which is the core requirement for deal diagnosis and coaching recommendation generation.
- Prompt caching reduces cost for repeated scoring runs against the same deal context.
- Tool use enables structured output (risk scores, signal classifications, coaching recommendations) with JSON schema enforcement.

### Authentication: OIDC-compliant SSO via Auth.js (NextAuth v5)

**Rationale:**
- Supports enterprise SSO (Okta, Azure AD/Entra ID, Ping) via OpenID Connect, satisfying the standards.md requirement for corporate credential authentication.
- OAuth 2.0 flows for CRM and email provider connections reuse the same token management infrastructure.

### Message Queue: BullMQ (Redis-backed)

**Rationale:**
- Handles background job processing for CRM sync, email polling, AI scoring, snapshot generation, and notification delivery.
- Supports priority queues (critical deal rescoring gets priority over batch snapshots), delayed jobs (retry after token refresh), and rate limiting (respecting CRM API rate limits).
- Simpler operational footprint than Kafka or RabbitMQ for an MVP.

### Infrastructure: Docker Compose (dev) + Kubernetes (production)

**Rationale:**
- Docker Compose provides a single-command local development environment with PostgreSQL, Redis, and the application.
- Kubernetes deployment supports the multi-tenant isolation, horizontal scaling, and data-region requirements from the research.

---

## Project Structure

```
pipeline-review-assistant/
├── apps/
│   ├── web/                          # Next.js 15 frontend
│   │   ├── src/
│   │   │   ├── app/                  # App Router pages
│   │   │   │   ├── (auth)/           # Login, SSO callback
│   │   │   │   ├── (dashboard)/      # Main application
│   │   │   │   │   ├── pipeline/     # Pipeline views, waterfall
│   │   │   │   │   ├── deals/        # Deal detail, scoring, signals
│   │   │   │   │   ├── forecast/     # Forecast roll-up, scenarios
│   │   │   │   │   ├── reviews/      # Pipeline review meetings
│   │   │   │   │   ├── coaching/     # Coaching recommendations
│   │   │   │   │   └── settings/     # Integrations, team, config
│   │   │   │   └── api/              # Next.js API routes (BFF)
│   │   │   ├── components/           # React components
│   │   │   │   ├── pipeline/         # Pipeline-specific UI
│   │   │   │   ├── deals/            # Deal cards, score badges
│   │   │   │   ├── charts/           # Waterfall, forecast charts
│   │   │   │   └── common/           # Shared UI primitives
│   │   │   └── lib/                  # Client-side utilities
│   │   └── next.config.ts
│   │
│   └── api/                          # Fastify API server
│       ├── src/
│       │   ├── server.ts             # Fastify app bootstrap
│       │   ├── routes/               # API route modules
│       │   │   ├── deals.ts
│       │   │   ├── pipeline.ts
│       │   │   ├── forecast.ts
│       │   │   ├── scoring.ts
│       │   │   ├── coaching.ts
│       │   │   ├── reviews.ts
│       │   │   ├── integrations.ts
│       │   │   ├── notifications.ts
│       │   │   └── webhooks.ts
│       │   ├── services/             # Business logic layer
│       │   │   ├── deal-service.ts
│       │   │   ├── scoring-engine.ts
│       │   │   ├── forecast-service.ts
│       │   │   ├── coaching-engine.ts
│       │   │   ├── review-brief-generator.ts
│       │   │   └── notification-service.ts
│       │   ├── integrations/         # CRM and email connectors
│       │   │   ├── crm/
│       │   │   │   ├── crm-adapter.ts        # Abstract CRM interface
│       │   │   │   ├── salesforce.ts
│       │   │   │   ├── hubspot.ts
│       │   │   │   ├── dynamics365.ts
│       │   │   │   ├── pipedrive.ts
│       │   │   │   └── zoho.ts
│       │   │   ├── email/
│       │   │   │   ├── email-adapter.ts
│       │   │   │   ├── gmail.ts
│       │   │   │   └── outlook.ts
│       │   │   └── notifications/
│       │   │       ├── slack.ts
│       │   │       └── email-sender.ts
│       │   ├── jobs/                 # BullMQ job processors
│       │   │   ├── crm-sync.ts
│       │   │   ├── email-sync.ts
│       │   │   ├── deal-scoring.ts
│       │   │   ├── snapshot-generator.ts
│       │   │   └── notification-dispatch.ts
│       │   ├── ai/                   # LLM interaction layer
│       │   │   ├── client.ts         # Claude API client with caching
│       │   │   ├── prompts/          # Prompt templates
│       │   │   │   ├── deal-scoring.ts
│       │   │   │   ├── deal-diagnosis.ts
│       │   │   │   ├── coaching-generation.ts
│       │   │   │   └── review-brief.ts
│       │   │   └── schemas/          # Tool use output schemas
│       │   │       ├── score-output.ts
│       │   │       └── coaching-output.ts
│       │   └── middleware/           # Auth, tenant, logging
│       │       ├── auth.ts
│       │       ├── tenant.ts
│       │       └── rate-limit.ts
│       └── fastify.config.ts
│
├── packages/
│   ├── db/                           # Database package
│   │   ├── migrations/               # SQL migration files
│   │   ├── src/
│   │   │   ├── schema.ts            # Drizzle ORM schema
│   │   │   ├── client.ts            # Database client
│   │   │   └── seed.ts              # Seed data
│   │   └── drizzle.config.ts
│   │
│   ├── shared/                       # Shared types and utilities
│   │   └── src/
│   │       ├── types/                # TypeScript type definitions
│   │       │   ├── deal.ts
│   │       │   ├── scoring.ts
│   │       │   ├── forecast.ts
│   │       │   └── qualification.ts
│   │       ├── schemas/              # Zod schemas (validation)
│   │       │   ├── deal-schema.ts
│   │       │   ├── qualification-schema.ts
│   │       │   └── provider-data-schema.ts
│   │       └── constants/
│   │           ├── signal-types.ts
│   │           └── forecast-categories.ts
│   │
│   └── mcp-server/                   # MCP server for AI agent access
│       └── src/
│           ├── server.ts
│           └── tools/
│               ├── query-pipeline.ts
│               ├── get-deal-score.ts
│               └── get-forecast.ts
│
├── docker-compose.yml
├── turbo.json                        # Turborepo configuration
├── package.json
└── tsconfig.base.json
```

---

## Phase Dependency Graph

```
Phase 1: Foundation
    │
    ├──► Phase 2: CRM Integration
    │       │
    │       ├──► Phase 4: AI Deal Scoring
    │       │       │
    │       │       ├──► Phase 6: Coaching Engine
    │       │       │       │
    │       │       │       └──► Phase 8: Pipeline Review Meetings
    │       │       │
    │       │       └──► Phase 7: Forecasting Engine
    │       │               │
    │       │               └──► Phase 9: Manager Bias & Accuracy
    │       │
    │       └──► Phase 5: Pipeline Dashboard
    │
    ├──► Phase 3: Email & Activity Capture
    │       │
    │       └──► (feeds into Phase 4)
    │
    └──► Phase 10: Notifications & Alerts
            │
            └──► Phase 11: MCP Server & API Publication

Phase 12: Production Hardening (depends on all above)
```

---

## Phase 1: Foundation & Core Infrastructure

**Goal:** Establish the monorepo, database schema, authentication, and tenant management so all subsequent phases build on a working skeleton.

**Duration:** 3 weeks

### Task 1.1: Monorepo Scaffolding

**What:** Initialize a Turborepo monorepo with `apps/web` (Next.js 15), `apps/api` (Fastify 5), `packages/db` (Drizzle ORM + migrations), and `packages/shared` (types and schemas). Configure TypeScript path aliases, ESLint, Prettier, and Vitest.

**Design:**
```typescript
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": [".next/**", "dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "test": { "dependsOn": ["^build"] },
    "lint": {},
    "db:migrate": { "cache": false },
    "db:seed": { "cache": false, "dependsOn": ["db:migrate"] }
  }
}
```
```typescript
// packages/db/drizzle.config.ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./src/schema.ts",
  out: "./migrations",
  dialect: "postgresql",
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
});
```
```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: pipeline_review
      POSTGRES_USER: pra_dev
      POSTGRES_PASSWORD: dev_password
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

volumes:
  pgdata:
```

**Testing:**
- `turbo build` completes without errors across all packages
- `turbo lint` passes with zero violations
- `docker compose up -d` starts PostgreSQL 16 and Redis 7
- `turbo db:migrate` applies all migrations to a clean database
- `vitest` runs with an empty test suite (zero tests, zero failures)

### Task 1.2: Database Schema (Core Tables)

**What:** Implement the 17-table hybrid schema from Data Model Suggestion 3 as Drizzle ORM definitions and SQL migration files. Includes tenant, user, crm_connection, email_connection, account, contact, pipeline, deal, deal_change, activity, deal_score_history, coaching_recommendation, forecast_submission, pipeline_snapshot, forecast_accuracy, pipeline_review, and notification tables. Apply RLS policies for tenant isolation.

**Design:**
```typescript
// packages/db/src/schema.ts (excerpt — deal table)
import { pgTable, uuid, varchar, bigint, char, decimal,
         boolean, timestamp, jsonb, index } from "drizzle-orm/pg-core";

export const deal = pgTable("deal", {
  id: uuid("id").primaryKey().defaultRandom(),
  tenantId: uuid("tenant_id").notNull().references(() => tenant.id),
  pipelineId: uuid("pipeline_id").notNull().references(() => pipeline.id),
  accountId: uuid("account_id").references(() => account.id),
  ownerId: uuid("owner_id").notNull().references(() => user.id),
  crmConnectionId: uuid("crm_connection_id").references(() => crmConnection.id),
  externalId: varchar("external_id", { length: 255 }),
  name: varchar("name", { length: 500 }).notNull(),
  stageId: varchar("stage_id", { length: 100 }).notNull(),
  stageName: varchar("stage_name", { length: 255 }).notNull(),
  amountCents: bigint("amount_cents", { mode: "number" }),
  currencyCode: char("currency_code", { length: 3 }).notNull().default("USD"),
  closeDate: timestamp("close_date", { mode: "date" }),
  forecastCategory: varchar("forecast_category", { length: 50 }).notNull().default("pipeline"),
  dealType: varchar("deal_type", { length: 50 }),
  source: varchar("source", { length: 100 }),
  probability: decimal("probability", { precision: 5, scale: 2 }),
  stageEnteredAt: timestamp("stage_entered_at").notNull().defaultNow(),
  daysInStage: bigint("days_in_stage", { mode: "number" }).notNull().default(0),
  lastActivityAt: timestamp("last_activity_at"),
  isOpen: boolean("is_open").notNull().default(true),
  isWon: boolean("is_won").notNull().default(false),
  aiScore: decimal("ai_score", { precision: 5, scale: 2 }),
  aiRiskLevel: varchar("ai_risk_level", { length: 20 }),
  aiMomentum: varchar("ai_momentum", { length: 20 }),
  aiExplanation: jsonb("ai_explanation"),
  aiScoredAt: timestamp("ai_scored_at"),
  qualification: jsonb("qualification").notNull().default({}),
  contacts: jsonb("contacts").notNull().default([]),
  signals: jsonb("signals").notNull().default([]),
  providerData: jsonb("provider_data").notNull().default({}),
  customFields: jsonb("custom_fields").notNull().default({}),
  syncHash: varchar("sync_hash", { length: 64 }),
  lastSyncedAt: timestamp("last_synced_at"),
  createdAt: timestamp("created_at").notNull().defaultNow(),
  updatedAt: timestamp("updated_at").notNull().defaultNow(),
}, (table) => [
  index("idx_deal_tenant").on(table.tenantId),
  index("idx_deal_owner").on(table.ownerId),
  index("idx_deal_pipeline").on(table.pipelineId),
  index("idx_deal_close").on(table.tenantId, table.closeDate),
]);
```

**Testing:**
- Migration applies cleanly to an empty database: `drizzle-kit migrate` exits with code 0
- Migration is idempotent: running it twice produces no errors
- All 17 tables exist with correct columns verified via `SELECT * FROM information_schema.tables WHERE table_schema = 'public'`
- RLS policies are active: query with `SET app.current_tenant_id = '<tenant-A-id>'` returns only tenant A data when test rows exist for tenants A and B
- Foreign key constraints prevent orphaned records: inserting a deal with a non-existent `tenant_id` raises an FK violation
- JSONB GIN indexes exist: `SELECT indexname FROM pg_indexes WHERE tablename = 'deal' AND indexdef LIKE '%gin%'` returns expected indexes

### Task 1.3: Authentication & Tenant Management

**What:** Implement OIDC-based authentication using Auth.js v5 in the Next.js app. Support Google, Microsoft (Entra ID), and generic OIDC providers. On first login, auto-create a tenant if one does not exist for the user's email domain. Implement middleware that sets `app.current_tenant_id` on every database connection.

**Design:**
```typescript
// apps/api/src/middleware/tenant.ts
import { FastifyRequest, FastifyReply } from "fastify";
import { db } from "@pipeline-review/db";
import { sql } from "drizzle-orm";

export async function tenantMiddleware(
  request: FastifyRequest,
  reply: FastifyReply
) {
  const tenantId = request.user?.tenantId;
  if (!tenantId) {
    return reply.code(401).send({ error: "Tenant context required" });
  }
  // Set RLS context for this connection
  await db.execute(
    sql`SET LOCAL app.current_tenant_id = ${tenantId}`
  );
}
```
```typescript
// apps/web/src/app/api/auth/[...nextauth]/route.ts
import NextAuth from "next-auth";
import { authConfig } from "@/lib/auth-config";

export const { handlers, auth, signIn, signOut } = NextAuth(authConfig);
export const { GET, POST } = handlers;
```

**Testing:**
- OAuth login flow with Google provider completes end-to-end in a test environment (using a test Google account or mock provider)
- First login for a new email domain creates a tenant record with correct `slug` and `data_region`
- Subsequent logins for the same domain associate the user with the existing tenant
- API requests without a valid session token receive 401
- API requests with a valid session token have `app.current_tenant_id` set correctly (verified via `SELECT current_setting('app.current_tenant_id')` in a test query)
- User with tenant A credentials cannot access tenant B data (RLS enforcement verified via API call returning empty results for cross-tenant queries)

### Task 1.4: BullMQ Job Infrastructure

**What:** Set up BullMQ with Redis for background job processing. Create queue definitions for `crm-sync`, `email-sync`, `deal-scoring`, `snapshot-generation`, and `notification-dispatch`. Implement a worker harness with graceful shutdown, dead letter handling, and job progress tracking.

**Design:**
```typescript
// apps/api/src/jobs/queue-config.ts
import { Queue, Worker, QueueEvents } from "bullmq";
import { Redis } from "ioredis";

const connection = new Redis(process.env.REDIS_URL!);

export const queues = {
  crmSync: new Queue("crm-sync", { connection }),
  emailSync: new Queue("email-sync", { connection }),
  dealScoring: new Queue("deal-scoring", { connection, defaultJobOptions: {
    priority: 1,
    attempts: 3,
    backoff: { type: "exponential", delay: 5000 },
  }}),
  snapshotGeneration: new Queue("snapshot-generation", { connection }),
  notificationDispatch: new Queue("notification-dispatch", { connection }),
} as const;

export function createWorker(
  queueName: keyof typeof queues,
  processor: (job: Job) => Promise<void>
) {
  const worker = new Worker(queueName, processor, {
    connection,
    concurrency: 5,
    limiter: { max: 10, duration: 1000 },
  });
  worker.on("failed", (job, err) => {
    logger.error({ jobId: job?.id, queue: queueName, error: err.message },
      "Job failed");
  });
  return worker;
}
```

**Testing:**
- Enqueue a test job to `crm-sync` queue and verify it is picked up by a worker within 2 seconds
- A failing job retries 3 times with exponential backoff (verify via job attempt count)
- After 3 failures, the job moves to the failed state (not retried indefinitely)
- Graceful shutdown: sending SIGTERM to the worker process allows in-progress jobs to complete before exit (verified with a 5-second test job)
- Queue dashboard (Bull Board) is accessible at `/admin/queues` for development environments

### Definition of Done — Phase 1

- [ ] `docker compose up -d && turbo db:migrate && turbo dev` starts the full development stack
- [ ] All 17 database tables created with correct columns, indexes, and RLS policies
- [ ] Authentication flow works end-to-end with at least one OIDC provider
- [ ] Tenant isolation verified: cross-tenant data access returns empty results
- [ ] BullMQ queues accept and process jobs with retry and dead letter handling
- [ ] All tests pass: `turbo test` exits with code 0
- [ ] OpenAPI spec auto-generated at `/api/docs` with Swagger UI

---

## Phase 2: CRM Integration Layer

**Goal:** Build bidirectional sync with Salesforce and HubSpot CRMs, establishing the adapter pattern that subsequent CRM connectors follow.

**Duration:** 4 weeks

**Dependencies:** Phase 1

### Task 2.1: CRM Adapter Interface

**What:** Define an abstract CRM adapter interface that normalises operations across CRM providers. Each adapter must implement: `connect()`, `fetchDeals()`, `fetchAccounts()`, `fetchContacts()`, `fetchActivities()`, `writeDealScore()`, and `handleWebhook()`. The interface uses the shared types from `packages/shared` and returns normalised objects with `providerData` JSONB for provider-specific fields.

**Design:**
```typescript
// apps/api/src/integrations/crm/crm-adapter.ts
export interface CrmAdapter {
  provider: CrmProvider;

  connect(config: CrmConnectionConfig): Promise<OAuthTokens>;
  refreshToken(connection: CrmConnection): Promise<OAuthTokens>;
  testConnection(connection: CrmConnection): Promise<boolean>;

  fetchDeals(connection: CrmConnection, opts: SyncOptions): Promise<NormalisedDeal[]>;
  fetchAccounts(connection: CrmConnection, opts: SyncOptions): Promise<NormalisedAccount[]>;
  fetchContacts(connection: CrmConnection, opts: SyncOptions): Promise<NormalisedContact[]>;
  fetchActivities(connection: CrmConnection, opts: SyncOptions): Promise<NormalisedActivity[]>;

  writeDealScore(connection: CrmConnection, dealExternalId: string,
    score: DealScorePayload): Promise<void>;

  handleWebhook(payload: unknown): Promise<WebhookEvent[]>;
}

export interface SyncOptions {
  since?: Date;          // Incremental sync from this timestamp
  batchSize?: number;    // Page size for paginated APIs
  fields?: string[];     // Specific fields to fetch (sparse fieldset)
}

export interface NormalisedDeal {
  externalId: string;
  name: string;
  stageId: string;
  stageName: string;
  amountCents: number | null;
  currencyCode: string;
  closeDate: Date | null;
  forecastCategory: string;
  ownerExternalId: string;
  accountExternalId: string | null;
  contactExternalIds: string[];
  providerData: Record<string, unknown>;  // Full CRM response preserved
}
```

**Testing:**
- TypeScript compilation succeeds with the interface — all required methods have correct signatures
- A mock adapter implementing the interface passes type checking
- Unit test: `NormalisedDeal` Zod schema validates a correctly shaped deal object
- Unit test: `NormalisedDeal` Zod schema rejects a deal missing required fields (externalId, name, stageId)
- Unit test: `providerData` accepts arbitrary key-value pairs without schema constraints

### Task 2.2: Salesforce Connector

**What:** Implement the CRM adapter for Salesforce using `jsforce` (Node.js library). Support OAuth 2.0 Web Server Flow for connection setup, My Domain-specific endpoints (Spring '26 / v66.0), incremental sync via `SystemModstamp` filtering, and webhook handling for Outbound Messages. Map Salesforce Opportunity, Account, Contact, and Task/Event objects to normalised types. Preserve all Salesforce fields in `providerData`.

**Design:**
```typescript
// apps/api/src/integrations/crm/salesforce.ts
import jsforce from "jsforce";
import { CrmAdapter, NormalisedDeal, SyncOptions } from "./crm-adapter";

export class SalesforceAdapter implements CrmAdapter {
  provider = "salesforce" as const;

  async fetchDeals(connection: CrmConnection, opts: SyncOptions): Promise<NormalisedDeal[]> {
    const conn = new jsforce.Connection({
      instanceUrl: connection.instanceUrl,
      accessToken: await this.getAccessToken(connection),
    });

    const query = opts.since
      ? `SELECT Id, Name, StageName, Amount, CloseDate, ForecastCategory,
               OwnerId, AccountId, Probability, Type, LeadSource, NextStep,
               Description, SystemModstamp
         FROM Opportunity
         WHERE SystemModstamp > ${opts.since.toISOString()}
         ORDER BY SystemModstamp ASC`
      : `SELECT Id, Name, StageName, Amount, CloseDate, ForecastCategory,
               OwnerId, AccountId, Probability, Type, LeadSource, NextStep,
               Description
         FROM Opportunity
         ORDER BY CreatedDate DESC`;

    const result = await conn.query<SalesforceOpportunity>(query);

    return result.records.map((opp) => ({
      externalId: opp.Id,
      name: opp.Name,
      stageId: opp.StageName,    // Mapped to pipeline.stages[] in sync logic
      stageName: opp.StageName,
      amountCents: opp.Amount ? Math.round(opp.Amount * 100) : null,
      currencyCode: "USD",       // Multi-currency handled via CurrencyIsoCode field
      closeDate: opp.CloseDate ? new Date(opp.CloseDate) : null,
      forecastCategory: this.mapForecastCategory(opp.ForecastCategory),
      ownerExternalId: opp.OwnerId,
      accountExternalId: opp.AccountId,
      contactExternalIds: [],    // Fetched separately via OpportunityContactRole
      providerData: opp,         // Entire Salesforce record preserved
    }));
  }

  async writeDealScore(connection: CrmConnection, dealExternalId: string,
    score: DealScorePayload): Promise<void> {
    const conn = new jsforce.Connection({
      instanceUrl: connection.instanceUrl,
      accessToken: await this.getAccessToken(connection),
    });
    // Write score to a custom field on the Opportunity
    await conn.sobject("Opportunity").update({
      Id: dealExternalId,
      PRA_AI_Score__c: score.overallScore,
      PRA_Risk_Level__c: score.riskLevel,
      PRA_Score_Explanation__c: score.explanation.substring(0, 255),
    });
  }
}
```

**Testing:**
- Integration test with Salesforce Developer Edition org: `connect()` completes OAuth flow and returns valid tokens
- Integration test: `fetchDeals()` retrieves at least 1 Opportunity from the dev org with correct field mapping
- Unit test: `mapForecastCategory` correctly maps Salesforce categories ("Pipeline", "Best Case", "Commit", "Closed") to internal categories
- Unit test: Amount conversion — Salesforce `Amount: 150000` maps to `amountCents: 15000000`
- Unit test: null Amount maps to `amountCents: null` (not 0)
- Integration test: `writeDealScore()` updates the custom `PRA_AI_Score__c` field on a test Opportunity
- Error handling: expired token triggers automatic token refresh and retry (verified with a mocked expired-token response)
- Rate limiting: when Salesforce returns HTTP 429, the adapter waits and retries (verified with a mock response)

### Task 2.3: HubSpot Connector

**What:** Implement the CRM adapter for HubSpot using the official HubSpot Node.js client. Support OAuth 2.0 authorization code flow, incremental sync via `lastmodifieddate` property filtering, and webhook handling for deal-stage change events. Map HubSpot Deal, Company, Contact, and Engagement objects to normalised types.

**Design:**
```typescript
// apps/api/src/integrations/crm/hubspot.ts
import { Client } from "@hubspot/api-client";
import { CrmAdapter, NormalisedDeal, SyncOptions } from "./crm-adapter";

export class HubSpotAdapter implements CrmAdapter {
  provider = "hubspot" as const;

  async fetchDeals(connection: CrmConnection, opts: SyncOptions): Promise<NormalisedDeal[]> {
    const client = new Client({ accessToken: await this.getAccessToken(connection) });

    const searchRequest = {
      filterGroups: opts.since ? [{
        filters: [{
          propertyName: "hs_lastmodifieddate",
          operator: "GTE",
          value: opts.since.getTime().toString(),
        }],
      }] : [],
      properties: [
        "dealname", "dealstage", "amount", "closedate",
        "pipeline", "hs_deal_stage_probability",
        "hubspot_owner_id", "hs_analytics_source",
      ],
      limit: opts.batchSize ?? 100,
      sorts: [{ propertyName: "hs_lastmodifieddate", direction: "ASCENDING" }],
    };

    const response = await client.crm.deals.searchApi.doSearch(searchRequest);

    return response.results.map((deal) => ({
      externalId: deal.id,
      name: deal.properties.dealname,
      stageId: deal.properties.dealstage,
      stageName: deal.properties.dealstage,  // Resolved via pipeline stages lookup
      amountCents: deal.properties.amount
        ? Math.round(parseFloat(deal.properties.amount) * 100) : null,
      currencyCode: "USD",
      closeDate: deal.properties.closedate
        ? new Date(deal.properties.closedate) : null,
      forecastCategory: "pipeline",  // Derived from stage mapping
      ownerExternalId: deal.properties.hubspot_owner_id ?? "",
      accountExternalId: null,  // Resolved via associations API
      contactExternalIds: [],   // Resolved via associations API
      providerData: deal.properties,
    }));
  }
}
```

**Testing:**
- Integration test with HubSpot developer test account: `connect()` completes OAuth flow
- Integration test: `fetchDeals()` retrieves deals with correct property mapping
- Unit test: HubSpot amount (string "150000") converts to `amountCents: 15000000`
- Unit test: null amount and empty string amount both map to `amountCents: null`
- Integration test: webhook payload for deal-stage change is parsed into a `WebhookEvent` with correct deal ID and new stage
- Unit test: associations API response correctly populates `accountExternalId` and `contactExternalIds`

### Task 2.4: CRM Sync Engine

**What:** Implement the background sync job that runs on the `crm-sync` BullMQ queue. For each active `crm_connection`, the job: (1) fetches incremental changes since `last_sync_at`, (2) upserts normalised records into the database with `sync_hash` change detection, (3) records all changes in the `deal_change` table, (4) updates `last_sync_at` on the connection. Support both full initial sync and incremental delta sync. Handle OAuth token refresh transparently.

**Design:**
```typescript
// apps/api/src/jobs/crm-sync.ts
import { Job } from "bullmq";
import { getAdapter } from "../integrations/crm";
import { db } from "@pipeline-review/db";
import { deal, dealChange, crmConnection } from "@pipeline-review/db/schema";
import { eq, and } from "drizzle-orm";
import { createHash } from "crypto";

export async function processCrmSync(job: Job<CrmSyncPayload>) {
  const { connectionId } = job.data;
  const connection = await db.query.crmConnection.findFirst({
    where: eq(crmConnection.id, connectionId),
  });
  if (!connection) throw new Error(`Connection ${connectionId} not found`);

  const adapter = getAdapter(connection.provider);

  // Update sync status
  await db.update(crmConnection)
    .set({ syncStatus: "syncing" })
    .where(eq(crmConnection.id, connectionId));

  try {
    const deals = await adapter.fetchDeals(connection, {
      since: connection.lastSyncAt ?? undefined,
      batchSize: 200,
    });

    for (const normalisedDeal of deals) {
      const hash = createHash("sha256")
        .update(JSON.stringify(normalisedDeal))
        .digest("hex");

      const existing = await db.query.deal.findFirst({
        where: and(
          eq(deal.crmConnectionId, connectionId),
          eq(deal.externalId, normalisedDeal.externalId),
        ),
      });

      if (existing && existing.syncHash === hash) continue; // No changes

      if (existing) {
        // Detect and record changes
        const changes = detectChanges(existing, normalisedDeal);
        if (changes.length > 0) {
          await db.insert(dealChange).values(
            changes.map((c) => ({
              dealId: existing.id,
              tenantId: connection.tenantId,
              changeType: c.type,
              source: "crm_sync",
              changes: c.payload,
            }))
          );
        }
        // Update deal
        await db.update(deal)
          .set({ ...mapToDbFields(normalisedDeal), syncHash: hash,
                 lastSyncedAt: new Date(), updatedAt: new Date() })
          .where(eq(deal.id, existing.id));
      } else {
        // Insert new deal
        await db.insert(deal).values({
          tenantId: connection.tenantId,
          crmConnectionId: connectionId,
          externalId: normalisedDeal.externalId,
          ...mapToDbFields(normalisedDeal),
          syncHash: hash,
          lastSyncedAt: new Date(),
        });
      }
    }

    await db.update(crmConnection).set({
      syncStatus: "synced",
      lastSyncAt: new Date(),
      lastError: null,
    }).where(eq(crmConnection.id, connectionId));

  } catch (error) {
    await db.update(crmConnection).set({
      syncStatus: "error",
      lastError: (error as Error).message,
    }).where(eq(crmConnection.id, connectionId));
    throw error; // Re-throw for BullMQ retry
  }
}
```

**Testing:**
- Unit test: sync with no existing deals inserts all fetched deals with correct field mapping
- Unit test: sync with an existing deal where `syncHash` matches skips the deal (no DB writes)
- Unit test: sync with an existing deal where stage changed creates a `deal_change` record with `change_type = 'stage_changed'` and correct `from_stage` / `to_stage` in the changes JSONB
- Unit test: sync with an existing deal where amount changed creates a `deal_change` record with `change_type = 'amount_changed'`
- Unit test: `last_sync_at` is updated on the `crm_connection` after successful sync
- Unit test: failed sync sets `sync_status = 'error'` and `last_error` with the error message
- Integration test: full sync of a Salesforce dev org with 50 Opportunities completes within 30 seconds
- Integration test: incremental sync after a single Opportunity change fetches only the changed record

### Definition of Done — Phase 2

- [ ] Salesforce adapter connects, syncs deals/accounts/contacts, and writes scores back
- [ ] HubSpot adapter connects, syncs deals/accounts/contacts, and writes scores back
- [ ] CRM sync job runs on schedule (configurable, default every 15 minutes)
- [ ] Incremental sync uses `sync_hash` to skip unchanged records
- [ ] All changes recorded in `deal_change` table with correct change types
- [ ] OAuth token refresh handled transparently without user intervention
- [ ] Webhook endpoints accept and process CRM push notifications
- [ ] Integration tests pass against Salesforce Developer Edition and HubSpot developer account

---

## Phase 3: Email & Activity Capture

**Goal:** Auto-log email and calendar activities from Gmail and Outlook, associating them with deals and contacts to eliminate manual rep data entry.

**Duration:** 3 weeks

**Dependencies:** Phase 1

### Task 3.1: Email Adapter Interface & Gmail Connector

**What:** Define an abstract email adapter interface and implement the Gmail connector using Google Workspace APIs. Support OAuth 2.0 with restricted scopes (`gmail.readonly`, `calendar.readonly`). Use Gmail Push Notifications (Google Pub/Sub) for real-time email arrival detection and periodic History API polling for catch-up sync. Match emails to contacts by email address and to deals by contact association.

**Design:**
```typescript
// apps/api/src/integrations/email/email-adapter.ts
export interface EmailAdapter {
  provider: "gmail" | "outlook";
  connect(userId: string, authCode: string): Promise<OAuthTokens>;
  setupPushNotifications(connection: EmailConnection): Promise<void>;
  fetchNewEmails(connection: EmailConnection, since?: string): Promise<EmailMessage[]>;
  fetchCalendarEvents(connection: EmailConnection, timeMin: Date, timeMax: Date): Promise<CalendarEvent[]>;
}

export interface EmailMessage {
  externalId: string;        // Message ID
  threadId: string;
  subject: string;
  bodyPreview: string;       // First 500 chars
  from: string;              // Email address
  to: string[];
  cc: string[];
  sentAt: Date;
  direction: "inbound" | "outbound";  // Relative to the rep
  hasAttachment: boolean;
}

// apps/api/src/integrations/email/gmail.ts
import { google } from "googleapis";

export class GmailAdapter implements EmailAdapter {
  provider = "gmail" as const;

  async fetchNewEmails(connection: EmailConnection, historyId?: string): Promise<EmailMessage[]> {
    const auth = new google.auth.OAuth2();
    auth.setCredentials({
      access_token: await this.decryptToken(connection.accessTokenEnc),
      refresh_token: await this.decryptToken(connection.refreshTokenEnc),
    });
    const gmail = google.gmail({ version: "v1", auth });

    if (historyId) {
      // Incremental: fetch history since last known ID
      const history = await gmail.users.history.list({
        userId: "me",
        startHistoryId: historyId,
        historyTypes: ["messageAdded"],
      });
      // Process each new message...
    } else {
      // Full initial sync: fetch last 30 days
      const messages = await gmail.users.messages.list({
        userId: "me",
        q: "newer_than:30d",
        maxResults: 500,
      });
      // Process each message...
    }
  }
}
```

**Testing:**
- Integration test with Gmail test account: `connect()` obtains valid OAuth tokens with `gmail.readonly` scope
- Integration test: `fetchNewEmails()` retrieves emails from the last 30 days with correct subject, from, and to fields
- Unit test: `direction` is correctly classified as "outbound" when `from` matches the rep's email, "inbound" otherwise
- Unit test: `bodyPreview` is truncated to 500 characters
- Integration test: `fetchCalendarEvents()` retrieves calendar events with correct title, attendees, and time
- Integration test: Google Pub/Sub push notification triggers email fetch within 60 seconds of email arrival
- Unit test: duplicate email (same `externalId`) is not inserted twice

### Task 3.2: Outlook Connector

**What:** Implement the email adapter for Microsoft Outlook using Microsoft Graph API. Support OAuth 2.0 with Entra ID, `Mail.Read` and `Calendars.Read` scopes. Use Microsoft Graph change notifications (webhooks) for real-time email detection and delta queries for incremental sync.

**Design:**
```typescript
// apps/api/src/integrations/email/outlook.ts
import { Client } from "@microsoft/microsoft-graph-client";

export class OutlookAdapter implements EmailAdapter {
  provider = "outlook" as const;

  async fetchNewEmails(connection: EmailConnection, deltaLink?: string): Promise<EmailMessage[]> {
    const client = Client.init({
      authProvider: (done) => done(null, this.getAccessToken(connection)),
    });

    const endpoint = deltaLink
      ? deltaLink  // Use delta link for incremental sync
      : "/me/mailFolders/inbox/messages/delta?$select=subject,from,toRecipients,ccRecipients,bodyPreview,receivedDateTime,hasAttachments&$top=100";

    const response = await client.api(endpoint).get();

    return response.value.map((msg: GraphMessage) => ({
      externalId: msg.id,
      threadId: msg.conversationId,
      subject: msg.subject,
      bodyPreview: msg.bodyPreview?.substring(0, 500) ?? "",
      from: msg.from?.emailAddress?.address ?? "",
      to: msg.toRecipients?.map((r: Recipient) => r.emailAddress.address) ?? [],
      cc: msg.ccRecipients?.map((r: Recipient) => r.emailAddress.address) ?? [],
      sentAt: new Date(msg.receivedDateTime),
      direction: this.classifyDirection(msg, connection.emailAddress),
      hasAttachment: msg.hasAttachments ?? false,
    }));
  }
}
```

**Testing:**
- Integration test with Microsoft 365 developer tenant: `connect()` completes OAuth flow with Entra ID
- Integration test: `fetchNewEmails()` retrieves inbox messages with correct field mapping
- Unit test: delta link from previous sync response is used for incremental fetch
- Integration test: Graph change notification webhook triggers email fetch for new messages
- Unit test: calendar events with multiple attendees correctly list all attendee email addresses

### Task 3.3: Activity Association Engine

**What:** Build a service that matches captured emails and calendar events to deals and contacts in the database. Matching logic: (1) extract all email addresses from the message (from, to, cc), (2) find matching contacts by email, (3) find active deals associated with those contacts via `deal.contacts` JSONB, (4) create `activity` records linked to the matched deal and contact. If multiple deals match, create activity records for all matching deals (with a flag to let the rep disambiguate).

**Design:**
```typescript
// apps/api/src/services/activity-association.ts
export class ActivityAssociationService {
  async associateEmail(
    tenantId: string,
    userId: string,
    email: EmailMessage
  ): Promise<Activity[]> {
    // 1. Find contacts matching any address in the email
    const addresses = [email.from, ...email.to, ...email.cc];
    const matchedContacts = await db.query.contact.findMany({
      where: and(
        eq(contact.tenantId, tenantId),
        inArray(contact.email, addresses),
      ),
    });

    // 2. Find deals involving those contacts
    const contactIds = matchedContacts.map((c) => c.id);
    const matchedDeals = await db.query.deal.findMany({
      where: and(
        eq(deal.tenantId, tenantId),
        eq(deal.isOpen, true),
        // JSONB containment: deal.contacts array contains an object with matching contact_id
        sql`${deal.contacts} @> ANY(ARRAY[${contactIds.map(
          (id) => sql`'[{"contact_id": "${id}"}]'::jsonb`
        )}])`,
      ),
    });

    // 3. Create activity records
    const activities: Activity[] = [];
    for (const matchedDeal of matchedDeals) {
      const activity = await db.insert(activityTable).values({
        tenantId,
        dealId: matchedDeal.id,
        contactId: matchedContacts[0]?.id,
        userId,
        activityType: email.direction === "outbound" ? "email_sent" : "email_received",
        subject: email.subject,
        bodyPreview: email.bodyPreview,
        occurredAt: email.sentAt,
        sentiment: null,  // Set by AI scoring later
        source: "gmail",  // or "outlook"
        externalId: email.externalId,
        isAutoLogged: true,
        metadata: {
          direction: email.direction,
          recipients: email.to,
          threadId: email.threadId,
          hasAttachment: email.hasAttachment,
        },
      }).returning();
      activities.push(activity[0]);
    }

    // 4. Update last_activity_at on matched deals
    for (const matchedDeal of matchedDeals) {
      await db.update(deal)
        .set({ lastActivityAt: email.sentAt, updatedAt: new Date() })
        .where(eq(deal.id, matchedDeal.id));
    }

    return activities;
  }
}
```

**Testing:**
- Unit test: email from `jane@acme.com` matches contact `Jane Doe` at account `Acme Corp` and associates with the open deal involving Jane
- Unit test: email involving contacts on 2 different active deals creates 2 activity records
- Unit test: email with no matching contacts creates zero activity records (not an orphaned activity)
- Unit test: `deal.lastActivityAt` is updated to the email's `sentAt` timestamp
- Unit test: duplicate activity (same `externalId`) is rejected (no duplicate insertion)
- Unit test: calendar meeting with 3 attendees where 2 are known contacts associates with the correct deals
- Integration test: end-to-end flow — Gmail Pub/Sub notification triggers email fetch, which triggers association, which creates an activity record with correct deal linkage

### Definition of Done — Phase 3

- [ ] Gmail OAuth connection setup works with restricted scopes
- [ ] Outlook OAuth connection setup works with Entra ID
- [ ] Emails are auto-logged as activities without rep intervention
- [ ] Calendar meetings are auto-logged as activities with attendee matching
- [ ] Activities are correctly associated with deals and contacts
- [ ] Incremental sync (Gmail History API / Graph delta) reduces API calls
- [ ] Push notifications (Gmail Pub/Sub / Graph webhooks) trigger near-real-time activity capture
- [ ] `deal.lastActivityAt` updated automatically on every new activity

---

## Phase 4: AI Deal Scoring Engine

**Goal:** Build the LLM-powered deal scoring system that evaluates deal health, detects risk signals, and generates plain-language explanations.

**Duration:** 4 weeks

**Dependencies:** Phase 2, Phase 3

### Task 4.1: Scoring Context Builder

**What:** Build a service that assembles the full context for an AI scoring request. For each deal, gather: (1) deal metadata (stage, amount, close date, days in stage), (2) last 20 activities with timeline, (3) contact roles and engagement levels, (4) MEDDPICC qualification scores if available, (5) stage history (progression velocity), (6) comparable historical deals (same stage, similar amount, same pipeline). Output a structured context document that fits within Claude's context window with prompt caching.

**Design:**
```typescript
// apps/api/src/ai/context-builder.ts
export class ScoringContextBuilder {
  async buildContext(dealId: string): Promise<DealScoringContext> {
    const [dealData, activities, stageHistory, comparables] = await Promise.all([
      this.fetchDealWithContacts(dealId),
      this.fetchRecentActivities(dealId, 20),
      this.fetchStageHistory(dealId),
      this.fetchComparableDeals(dealId),
    ]);

    return {
      deal: {
        name: dealData.name,
        amount: dealData.amountCents ? dealData.amountCents / 100 : null,
        stage: dealData.stageName,
        daysInStage: dealData.daysInStage,
        closeDate: dealData.closeDate,
        daysUntilClose: dealData.closeDate
          ? differenceInDays(dealData.closeDate, new Date()) : null,
        forecastCategory: dealData.forecastCategory,
        dealType: dealData.dealType,
        source: dealData.source,
      },
      contacts: dealData.contacts.map((c: DealContact) => ({
        name: `${c.name}`,
        title: c.title,
        role: c.role,
        engagement: c.engagement,
        daysSinceLastInteraction: c.last_interaction
          ? differenceInDays(new Date(), new Date(c.last_interaction)) : null,
      })),
      activities: activities.map((a) => ({
        type: a.activityType,
        subject: a.subject,
        occurredAt: a.occurredAt,
        daysSinceActivity: differenceInDays(new Date(), a.occurredAt),
        sentiment: a.sentiment,
        metadata: a.metadata,
      })),
      qualification: dealData.qualification,
      stageProgression: stageHistory.map((h) => ({
        stage: h.changes.to_stage,
        enteredAt: h.occurredAt,
        daysInStage: h.changes.days_in_previous_stage,
      })),
      benchmarks: {
        avgDaysInStageForComparables: comparables.avgDaysInStage,
        winRateForStage: comparables.winRate,
        avgDealCycleLength: comparables.avgCycleLength,
      },
    };
  }
}
```

**Testing:**
- Unit test: context for a deal with 15 activities returns all 15 (under the 20 limit)
- Unit test: context for a deal with 30 activities returns only the 20 most recent
- Unit test: `daysUntilClose` is negative when close date is in the past
- Unit test: `daysUntilClose` is null when close date is not set
- Unit test: contacts array includes role and engagement level from the `deal.contacts` JSONB
- Unit test: benchmarks include correct average days in stage from comparable deals
- Performance test: context assembly for a deal with 20 activities, 5 contacts, and 10 stage changes completes in under 200ms

### Task 4.2: LLM Scoring Prompt & Tool Use

**What:** Implement the Claude API integration for deal scoring. Use tool use (function calling) to enforce structured output: overall score (0-100), win probability (0.0-1.0), risk level (low/medium/high/critical), momentum (accelerating/steady/decelerating/stalled), explanation (plain language), and detected signals. Enable prompt caching for the system prompt and deal schema.

**Design:**
```typescript
// apps/api/src/ai/prompts/deal-scoring.ts
import Anthropic from "@anthropic-ai/sdk";

const SYSTEM_PROMPT = `You are a revenue intelligence analyst evaluating B2B sales deals.
You will receive structured context about a deal including its metadata, activity history,
contact engagement, qualification scores, and historical benchmarks.

Your task is to:
1. Assess the deal's overall health on a 0-100 scale
2. Estimate win probability (0.0 to 1.0)
3. Classify risk level: low (score 70-100), medium (50-69), high (30-49), critical (0-29)
4. Assess momentum: accelerating, steady, decelerating, or stalled
5. Explain your reasoning in 2-3 sentences of plain language a sales manager would understand
6. Identify specific risk and positive signals from the data

Focus on actionable signals: champion engagement, multi-threading, competitive pressure,
stage velocity relative to benchmarks, and qualification completeness.`;

const SCORING_TOOL: Anthropic.Tool = {
  name: "submit_deal_score",
  description: "Submit the deal health assessment",
  input_schema: {
    type: "object",
    required: ["overall_score", "win_probability", "risk_level", "momentum",
               "explanation", "signals"],
    properties: {
      overall_score: { type: "number", minimum: 0, maximum: 100 },
      win_probability: { type: "number", minimum: 0, maximum: 1 },
      risk_level: { type: "string", enum: ["low", "medium", "high", "critical"] },
      momentum: { type: "string", enum: ["accelerating", "steady", "decelerating", "stalled"] },
      explanation: { type: "string", maxLength: 1000 },
      signals: {
        type: "array",
        items: {
          type: "object",
          required: ["type", "severity", "title", "description"],
          properties: {
            type: { type: "string" },
            severity: { type: "string", enum: ["info", "warning", "critical"] },
            title: { type: "string" },
            description: { type: "string" },
          },
        },
      },
    },
  },
};

export async function scoreDeal(
  client: Anthropic,
  context: DealScoringContext
): Promise<DealScoreResult> {
  const response = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 1024,
    system: [{ type: "text", text: SYSTEM_PROMPT, cache_control: { type: "ephemeral" } }],
    tools: [SCORING_TOOL],
    tool_choice: { type: "tool", name: "submit_deal_score" },
    messages: [{
      role: "user",
      content: `Score this deal:\n\n${JSON.stringify(context, null, 2)}`,
    }],
  });

  const toolUse = response.content.find((c) => c.type === "tool_use");
  if (!toolUse || toolUse.name !== "submit_deal_score") {
    throw new Error("LLM did not return expected tool call");
  }

  return toolUse.input as DealScoreResult;
}
```

**Testing:**
- Integration test: `scoreDeal()` returns a valid `DealScoreResult` with all required fields
- Unit test: tool use output is validated against the Zod schema — invalid scores (e.g., score: 150) are rejected
- Unit test: a deal with no activity in 21 days receives risk_level "high" or "critical" and a "stalled" momentum
- Unit test: a deal with recent activity, multi-threaded contacts, and completed MEDDPICC receives risk_level "low" and "steady" or "accelerating" momentum
- Performance test: scoring a single deal completes within 5 seconds (Claude API latency)
- Cost test: prompt caching verification — second call for a different deal with same system prompt shows cache read in usage metadata

### Task 4.3: Batch Scoring Job & Deal Update

**What:** Implement the background job that scores deals in batch. The job: (1) selects all open deals that have not been scored in the last 4 hours or that have new activities since last scoring, (2) builds context for each deal, (3) calls the LLM scoring function, (4) writes the score to the `deal` table (denormalised fields) and `deal_score_history` table (full provenance), (5) updates `deal.signals` JSONB, (6) creates `deal_change` records for significant score changes (>10 points). Support priority scoring: deals with very recent activity (last 30 minutes) get scored immediately via a high-priority queue.

**Design:**
```typescript
// apps/api/src/jobs/deal-scoring.ts
import { Job } from "bullmq";

export async function processDealScoring(job: Job<DealScoringPayload>) {
  const { dealId, priority } = job.data;

  const contextBuilder = new ScoringContextBuilder();
  const context = await contextBuilder.buildContext(dealId);

  const scoreResult = await scoreDeal(anthropicClient, context);

  // Write to deal_score_history (full provenance)
  await db.insert(dealScoreHistory).values({
    dealId,
    tenantId: context.deal.tenantId,
    modelVersion: "claude-sonnet-4-20250514-v1",
    overallScore: scoreResult.overall_score,
    winProbability: scoreResult.win_probability,
    riskLevel: scoreResult.risk_level,
    momentum: scoreResult.momentum,
    explanation: scoreResult.explanation,
    scoringContext: {
      input_signals: scoreResult.signals,
      previous_score: context.deal.aiScore,
      score_delta: scoreResult.overall_score - (context.deal.aiScore ?? 0),
      trigger_event: priority === "high" ? "recent_activity" : "scheduled_batch",
    },
  });

  // Update deal with latest score (denormalised)
  const previousScore = await db.query.deal.findFirst({
    where: eq(deal.id, dealId),
    columns: { aiScore: true, aiRiskLevel: true },
  });

  await db.update(deal).set({
    aiScore: scoreResult.overall_score,
    aiRiskLevel: scoreResult.risk_level,
    aiMomentum: scoreResult.momentum,
    aiExplanation: scoreResult.explanation,
    aiScoredAt: new Date(),
    signals: scoreResult.signals,
    updatedAt: new Date(),
  }).where(eq(deal.id, dealId));

  // Record significant score changes
  const scoreDelta = scoreResult.overall_score - (previousScore?.aiScore ?? 0);
  if (Math.abs(scoreDelta) > 10) {
    await db.insert(dealChange).values({
      dealId,
      tenantId: context.deal.tenantId,
      changeType: "score_recalculated",
      source: "ai_engine",
      changes: {
        previous_score: previousScore?.aiScore,
        new_score: scoreResult.overall_score,
        delta: scoreDelta,
        risk_level_change: previousScore?.aiRiskLevel !== scoreResult.risk_level
          ? `${previousScore?.aiRiskLevel} -> ${scoreResult.risk_level}` : null,
      },
    });
  }
}
```

**Testing:**
- Unit test: batch scoring selects only open deals not scored in the last 4 hours
- Unit test: deals with new activities since last scoring are included in the batch regardless of time
- Unit test: `deal_score_history` record includes full `scoringContext` with input signals and model version
- Unit test: deal's denormalised AI fields (`ai_score`, `ai_risk_level`, `ai_momentum`, `ai_explanation`, `signals`) are updated
- Unit test: score change of 15 points creates a `deal_change` record; score change of 5 points does not
- Unit test: priority "high" jobs are processed before priority "normal" jobs in the queue
- Integration test: batch scoring of 10 deals completes within 60 seconds
- Error handling: LLM API timeout on one deal does not block scoring of remaining deals in the batch

### Task 4.4: Stalled Deal Detection

**What:** Implement automated stalled deal detection that runs as part of the scoring job. A deal is flagged as stalled when: (1) no activity (email, call, meeting) in the last N days (configurable per tenant, default 14), or (2) deal has been in the same stage for 2x the average time for that stage in the tenant's pipeline. Update `deal.ai_momentum` to "stalled" and add a `stalled_deal` signal to `deal.signals`.

**Design:**
```typescript
// apps/api/src/services/stalled-deal-detector.ts
export class StalledDealDetector {
  async detectStalledDeals(tenantId: string): Promise<string[]> {
    const tenant = await db.query.tenant.findFirst({
      where: eq(tenantTable.id, tenantId),
    });
    const thresholdDays = tenant?.config?.stale_deal_threshold_days ?? 14;

    const stalledByActivity = await db.query.deal.findMany({
      where: and(
        eq(deal.tenantId, tenantId),
        eq(deal.isOpen, true),
        or(
          isNull(deal.lastActivityAt),
          lt(deal.lastActivityAt,
            new Date(Date.now() - thresholdDays * 24 * 60 * 60 * 1000)),
        ),
      ),
    });

    // Also check stage duration vs. pipeline averages
    const stalledByStage = await this.findStalledByStageVelocity(tenantId);

    const allStalled = new Set([
      ...stalledByActivity.map((d) => d.id),
      ...stalledByStage.map((d) => d.id),
    ]);

    // Update each stalled deal
    for (const dealId of allStalled) {
      await this.markAsStalled(dealId, tenantId);
    }

    return Array.from(allStalled);
  }
}
```

**Testing:**
- Unit test: deal with `lastActivityAt` 15 days ago (threshold: 14) is flagged as stalled
- Unit test: deal with `lastActivityAt` 13 days ago (threshold: 14) is NOT flagged
- Unit test: deal with null `lastActivityAt` is flagged as stalled
- Unit test: deal in stage for 30 days when average for that stage is 12 days (>2x) is flagged
- Unit test: deal in stage for 20 days when average for that stage is 12 days (<2x) is NOT flagged
- Unit test: `deal.signals` JSONB includes `{ type: "stalled_deal", severity: "warning", ... }` after detection
- Unit test: deal with `is_open = false` is never flagged regardless of activity age
- Unit test: custom threshold from `tenant.config` (e.g., 7 days) overrides the default 14

### Definition of Done — Phase 4

- [ ] Deal scoring produces consistent, reasonable scores for test deals
- [ ] Scores are persisted in both `deal` (denormalised) and `deal_score_history` (provenance)
- [ ] Risk signals (champion dark, competitor mentioned, stalled, single-threaded) are detected and stored in `deal.signals`
- [ ] Stalled deal detection runs automatically and updates deal status
- [ ] Scoring provenance includes model version, input signals, and score delta for EU AI Act compliance
- [ ] Batch scoring of 50 deals completes within 5 minutes
- [ ] Prompt caching reduces LLM API cost by 50%+ on repeat scoring runs

---

## Phase 5: Pipeline Dashboard

**Goal:** Build the primary user interface showing pipeline health, deal cards with AI scores, and the waterfall (week-over-week movement) view.

**Duration:** 3 weeks

**Dependencies:** Phase 2

### Task 5.1: Pipeline Overview Page

**What:** Build the main pipeline page showing all open deals grouped by stage in a kanban-style board view. Each deal card displays: deal name, amount, close date, AI score badge (colour-coded by risk level), momentum indicator, and key signals. Support filtering by owner, risk level, forecast category, and date range. Support sorting by amount, close date, and AI score.

**Design:**
```tsx
// apps/web/src/app/(dashboard)/pipeline/page.tsx
import { Suspense } from "react";
import { PipelineBoard } from "@/components/pipeline/pipeline-board";
import { PipelineFilters } from "@/components/pipeline/pipeline-filters";
import { fetchPipelineData } from "@/lib/api/pipeline";

export default async function PipelinePage({
  searchParams,
}: {
  searchParams: Promise<Record<string, string>>;
}) {
  const params = await searchParams;
  const data = await fetchPipelineData(params);

  return (
    <div className="flex flex-col gap-4 p-6">
      <h1 className="text-2xl font-semibold">Pipeline</h1>
      <PipelineFilters
        owners={data.owners}
        currentFilters={params}
      />
      <Suspense fallback={<PipelineBoardSkeleton />}>
        <PipelineBoard
          stages={data.stages}
          deals={data.deals}
        />
      </Suspense>
    </div>
  );
}

// apps/web/src/components/pipeline/deal-card.tsx
export function DealCard({ deal }: { deal: DealSummary }) {
  return (
    <div className="rounded-lg border bg-card p-4 shadow-sm hover:shadow-md transition-shadow">
      <div className="flex items-center justify-between">
        <h3 className="font-medium truncate">{deal.name}</h3>
        <RiskBadge level={deal.aiRiskLevel} score={deal.aiScore} />
      </div>
      <div className="mt-2 text-sm text-muted-foreground">
        <span>{formatCurrency(deal.amountCents)}</span>
        <span className="mx-2">·</span>
        <span>{formatDate(deal.closeDate)}</span>
      </div>
      <MomentumIndicator momentum={deal.aiMomentum} />
      {deal.signals.length > 0 && (
        <div className="mt-2 flex flex-wrap gap-1">
          {deal.signals.slice(0, 3).map((signal) => (
            <SignalChip key={signal.type} signal={signal} />
          ))}
        </div>
      )}
    </div>
  );
}
```

**Testing:**
- Visual test: pipeline board renders stages as columns with deal cards in correct positions
- Functional test: filter by `risk_level=high` shows only high and critical risk deals
- Functional test: filter by owner shows only that owner's deals
- Functional test: sort by amount (descending) orders deals from highest to lowest
- Accessibility test: deal cards are keyboard-navigable and screen reader accessible
- Performance test: pipeline with 200 deals renders initial view in under 2 seconds
- Responsive test: pipeline board switches to a list view on mobile breakpoints

### Task 5.2: Deal Detail Page

**What:** Build the deal detail page showing comprehensive deal information: AI score with explanation, active signals, contact roles and engagement, activity timeline, MEDDPICC qualification scorecard (if applicable), stage history, coaching recommendations, and a history of all changes from `deal_change`.

**Design:**
```tsx
// apps/web/src/app/(dashboard)/deals/[id]/page.tsx
export default async function DealDetailPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  const deal = await fetchDealDetail(id);

  return (
    <div className="grid grid-cols-3 gap-6 p-6">
      {/* Left column: deal info + score */}
      <div className="col-span-2 space-y-6">
        <DealHeader deal={deal} />
        <AIScoreCard
          score={deal.aiScore}
          riskLevel={deal.aiRiskLevel}
          momentum={deal.aiMomentum}
          explanation={deal.aiExplanation}
          scoredAt={deal.aiScoredAt}
        />
        <SignalsPanel signals={deal.signals} />
        <QualificationScorecard qualification={deal.qualification} />
        <ActivityTimeline activities={deal.activities} />
      </div>
      {/* Right column: contacts + coaching + history */}
      <div className="space-y-6">
        <ContactsPanel contacts={deal.contacts} />
        <CoachingPanel recommendations={deal.coaching} />
        <ChangeHistory changes={deal.changes} />
      </div>
    </div>
  );
}
```

**Testing:**
- Visual test: deal detail page displays all sections (score, signals, contacts, activities, qualification, history)
- Functional test: AI score card shows correct colour coding — green for low risk, yellow for medium, orange for high, red for critical
- Functional test: MEDDPICC scorecard shows per-element scores with colour coding for gaps (score < 50% of max)
- Functional test: activity timeline shows activities in reverse chronological order with correct icons per type
- Functional test: change history shows stage changes, score changes, and amount changes with timestamps
- Edge case: deal with no activities displays "No activities recorded" message
- Edge case: deal with no MEDDPICC scores displays "Qualification not assessed" message

### Task 5.3: Pipeline Waterfall (Week-over-Week Movement)

**What:** Build the waterfall chart showing how pipeline value moved between stages week over week. Display: opening balance per stage, deals added (new pipeline), deals moved forward, deals moved backward, deals won, deals lost, and closing balance. Powered by `pipeline_snapshot` data. Implement the weekly snapshot generation job.

**Design:**
```typescript
// apps/api/src/jobs/snapshot-generator.ts
export async function generateWeeklySnapshot(tenantId: string) {
  const pipelines = await db.query.pipeline.findMany({
    where: eq(pipeline.tenantId, tenantId),
  });

  for (const p of pipelines) {
    const stages = p.stages as PipelineStage[];
    const stageData = [];

    for (const stage of stages) {
      const deals = await db.query.deal.findMany({
        where: and(
          eq(deal.tenantId, tenantId),
          eq(deal.pipelineId, p.id),
          eq(deal.stageId, stage.id),
          eq(deal.isOpen, true),
        ),
      });
      stageData.push({
        stage_id: stage.id,
        stage_name: stage.name,
        deal_count: deals.length,
        total_amount: deals.reduce((sum, d) => sum + (d.amountCents ?? 0), 0),
        weighted_amount: deals.reduce(
          (sum, d) => sum + Math.round((d.amountCents ?? 0) * (stage.probability / 100)),
          0
        ),
      });
    }

    await db.insert(pipelineSnapshot).values({
      tenantId,
      snapshotDate: new Date(),
      pipelineId: p.id,
      ownerId: null,  // Pipeline-wide snapshot
      stages: stageData,
      totalDeals: stageData.reduce((sum, s) => sum + s.deal_count, 0),
      totalAmount: stageData.reduce((sum, s) => sum + s.total_amount, 0),
      totalWeighted: stageData.reduce((sum, s) => sum + s.weighted_amount, 0),
    });
  }
}
```

**Testing:**
- Unit test: snapshot generator creates correct stage-level totals for a pipeline with 3 stages and 10 deals
- Unit test: weighted amount equals `amount * stage_probability / 100` for each stage
- Visual test: waterfall chart renders with correct bars for each movement category
- Functional test: selecting "last 4 weeks" shows weekly comparison data
- Functional test: pipeline with no previous snapshot shows current state only (no comparison)
- Edge case: snapshot with zero deals in a stage shows $0 (not null or missing row)

### Definition of Done — Phase 5

- [ ] Pipeline board displays deals grouped by stage with AI score badges
- [ ] Deal detail page shows all scoring, signals, contacts, activities, and history
- [ ] Pipeline waterfall chart shows week-over-week movement
- [ ] Filters (owner, risk level, forecast category) work correctly
- [ ] Weekly snapshot generation job runs on schedule
- [ ] All pages are responsive and accessible
- [ ] Page load time under 2 seconds for a pipeline with 200 deals

---

## Phase 6: Coaching Engine

**Goal:** Auto-generate actionable coaching recommendations for at-risk deals, providing reps with specific next steps and managers with coaching questions.

**Duration:** 3 weeks

**Dependencies:** Phase 4

### Task 6.1: Coaching Recommendation Generator

**What:** Build an LLM-powered service that generates coaching recommendations for deals with high or critical risk levels. For each deal, the generator produces 1-3 specific recommendations with: recommendation type (next_step, outreach, meeting_request, proposal_revision, stakeholder_expansion, objection_handling), priority ranking, title, body text, and a specific suggested action. Uses the same deal context as scoring but with a coaching-focused prompt.

**Design:**
```typescript
// apps/api/src/ai/prompts/coaching-generation.ts
const COACHING_SYSTEM_PROMPT = `You are a sales coaching advisor generating specific,
actionable recommendations for at-risk B2B deals. Based on the deal context (risk signals,
activity history, contact engagement, qualification gaps), generate 1-3 concrete
recommendations that a sales rep can act on immediately.

Each recommendation must be:
- Specific to this deal (reference actual contacts, signals, and timeline)
- Actionable (the rep can do it today)
- Prioritised by impact on deal advancement

Types of recommendations:
- next_step: suggest the next concrete action
- outreach: specific person to contact with suggested talking points
- meeting_request: who to meet and what to discuss
- proposal_revision: what to change in the proposal
- stakeholder_expansion: who else to bring into the deal (multi-threading)
- objection_handling: how to address a specific concern`;

const COACHING_TOOL: Anthropic.Tool = {
  name: "submit_coaching_recommendations",
  description: "Submit coaching recommendations for the deal",
  input_schema: {
    type: "object",
    required: ["recommendations"],
    properties: {
      recommendations: {
        type: "array",
        minItems: 1,
        maxItems: 3,
        items: {
          type: "object",
          required: ["type", "priority", "title", "body", "suggested_action"],
          properties: {
            type: { type: "string", enum: ["next_step", "outreach", "meeting_request",
              "proposal_revision", "stakeholder_expansion", "objection_handling"] },
            priority: { type: "integer", minimum: 1, maximum: 100 },
            title: { type: "string", maxLength: 255 },
            body: { type: "string", maxLength: 2000 },
            suggested_action: { type: "string", maxLength: 500 },
          },
        },
      },
    },
  },
};
```

**Testing:**
- Integration test: coaching generator produces 1-3 recommendations for a high-risk deal
- Unit test: recommendations reference actual contact names from the deal context
- Unit test: "champion_dark" signal produces an "outreach" or "meeting_request" recommendation targeting the champion
- Unit test: "single_threaded" signal produces a "stakeholder_expansion" recommendation
- Unit test: MEDDPICC gap in "Decision Process" produces a recommendation to clarify the decision process
- Unit test: recommendations are persisted in `coaching_recommendation` table with status "pending"
- Unit test: existing pending recommendations for the same deal are not duplicated

### Task 6.2: Coaching Dashboard UI

**What:** Build the coaching dashboard showing pending recommendations grouped by rep and deal. Reps see their own recommendations; managers see recommendations for all their reports. Each recommendation card has "Accept" (marks as accepted, creates a task), "Dismiss" (marks as dismissed with optional reason), and "Complete" (marks as done) actions.

**Design:**
```tsx
// apps/web/src/components/coaching/recommendation-card.tsx
export function RecommendationCard({
  recommendation,
  onAction,
}: {
  recommendation: CoachingRecommendation;
  onAction: (action: "accept" | "dismiss" | "complete", id: string) => void;
}) {
  return (
    <div className="rounded-lg border p-4">
      <div className="flex items-center gap-2">
        <RecommendationTypeIcon type={recommendation.type} />
        <h4 className="font-medium">{recommendation.title}</h4>
        <PriorityBadge priority={recommendation.priority} />
      </div>
      <p className="mt-2 text-sm text-muted-foreground">{recommendation.body}</p>
      <div className="mt-3 rounded bg-muted p-2 text-sm">
        <span className="font-medium">Suggested action: </span>
        {recommendation.suggestedAction}
      </div>
      <div className="mt-3 flex gap-2">
        <Button size="sm" onClick={() => onAction("accept", recommendation.id)}>
          Accept
        </Button>
        <Button size="sm" variant="outline" onClick={() => onAction("complete", recommendation.id)}>
          Mark Complete
        </Button>
        <Button size="sm" variant="ghost" onClick={() => onAction("dismiss", recommendation.id)}>
          Dismiss
        </Button>
      </div>
    </div>
  );
}
```

**Testing:**
- Functional test: rep sees only their own pending recommendations
- Functional test: manager sees recommendations for all their reports
- Functional test: clicking "Accept" changes recommendation status to "accepted"
- Functional test: clicking "Dismiss" shows a reason input and marks as "dismissed"
- Functional test: clicking "Mark Complete" changes status to "completed" with timestamp
- Functional test: dismissed recommendations disappear from the active list
- Edge case: deal with no recommendations shows "No coaching recommendations" message

### Definition of Done — Phase 6

- [ ] Coaching recommendations are auto-generated for high/critical risk deals
- [ ] Recommendations are specific to the deal context (reference contacts, signals)
- [ ] Reps can accept, dismiss, or complete recommendations
- [ ] Managers see coaching recommendations for all their reports
- [ ] Coaching generation runs after every deal scoring cycle

---

## Phase 7: Forecasting Engine

**Goal:** Build the multi-scenario forecast roll-up system with rep submission, manager adjustment, and AI-predicted forecast.

**Duration:** 3 weeks

**Dependencies:** Phase 4

### Task 7.1: Forecast Submission & Roll-Up

**What:** Implement forecast submission for reps and managers. Reps submit their forecast for each scenario (committed, best_case, pipeline) with deal-level breakdown. Managers view the roll-up across their reports and can apply adjustments with documented reasons. The system auto-generates an AI forecast using deal scores and win probabilities.

**Design:**
```typescript
// apps/api/src/services/forecast-service.ts
export class ForecastService {
  async getAIForecast(tenantId: string, periodStart: Date, periodEnd: Date): Promise<AIForecast> {
    // Calculate AI forecast from individual deal scores
    const openDeals = await db.query.deal.findMany({
      where: and(
        eq(deal.tenantId, tenantId),
        eq(deal.isOpen, true),
        gte(deal.closeDate, periodStart),
        lte(deal.closeDate, periodEnd),
      ),
    });

    const scenarios = {
      committed: openDeals
        .filter((d) => d.forecastCategory === "commit")
        .reduce((sum, d) => sum + (d.amountCents ?? 0) * (d.probability ?? 0) / 100, 0),
      bestCase: openDeals
        .filter((d) => ["commit", "best_case"].includes(d.forecastCategory))
        .reduce((sum, d) => sum + (d.amountCents ?? 0) * (d.probability ?? 0) / 100, 0),
      pipeline: openDeals
        .reduce((sum, d) => sum + (d.amountCents ?? 0) * (d.probability ?? 0) / 100, 0),
    };

    return {
      committed: Math.round(scenarios.committed),
      bestCase: Math.round(scenarios.bestCase),
      pipeline: Math.round(scenarios.pipeline),
      dealCount: openDeals.length,
      confidence: this.calculateConfidence(openDeals),
    };
  }

  async submitForecast(
    userId: string,
    tenantId: string,
    periodStart: Date,
    periodEnd: Date,
    scenario: string,
    amountCents: number,
    dealBreakdown: DealBreakdown[]
  ): Promise<ForecastSubmission> {
    return db.insert(forecastSubmission).values({
      tenantId,
      userId,
      periodStart,
      periodEnd,
      periodLabel: this.formatPeriodLabel(periodStart),
      scenario,
      amountCents,
      dealCount: dealBreakdown.length,
      breakdown: {
        by_deal: dealBreakdown,
        submitted_at: new Date().toISOString(),
      },
    }).returning();
  }
}
```

**Testing:**
- Unit test: AI forecast `committed` only includes deals with `forecast_category = 'commit'`
- Unit test: AI forecast amounts are weighted by win probability
- Unit test: rep can submit a forecast for "committed" scenario with deal-level breakdown
- Unit test: manager can view roll-up across 5 reports showing sum of their committed forecasts
- Unit test: manager adjustment records the original and adjusted amounts with reason
- Functional test: forecast page shows committed, best case, and pipeline scenarios side by side
- Edge case: new quarter with no submissions shows $0 for all scenarios

### Task 7.2: Forecast Dashboard UI

**What:** Build the forecast page showing: scenario comparison chart (committed vs. best case vs. pipeline), rep-level breakdown table, AI forecast vs. human forecast comparison, and trend over time. Managers see a roll-up view; reps see their individual forecast with submission form.

**Design:**
```tsx
// apps/web/src/app/(dashboard)/forecast/page.tsx
export default async function ForecastPage() {
  const forecastData = await fetchForecastData();

  return (
    <div className="space-y-6 p-6">
      <h1 className="text-2xl font-semibold">Forecast — {forecastData.periodLabel}</h1>
      <ScenarioComparisonChart data={forecastData.scenarios} />
      <div className="grid grid-cols-2 gap-6">
        <AIvsHumanForecast
          ai={forecastData.aiForecast}
          human={forecastData.latestSubmission}
        />
        <ForecastTrendChart history={forecastData.weeklyHistory} />
      </div>
      <RepBreakdownTable
        reps={forecastData.repBreakdown}
        canAdjust={forecastData.isManager}
      />
    </div>
  );
}
```

**Testing:**
- Visual test: scenario comparison chart renders bars for committed, best case, and pipeline
- Functional test: rep can submit forecast via inline form
- Functional test: manager can click "Adjust" on a rep's forecast, enter new amount and reason
- Functional test: AI vs. human chart shows both values for comparison
- Functional test: weekly trend chart shows forecast evolution over the quarter

### Definition of Done — Phase 7

- [ ] Reps can submit forecasts per scenario with deal-level breakdown
- [ ] Managers see roll-up across reports with adjustment capability
- [ ] AI forecast is auto-calculated from deal scores and probabilities
- [ ] Forecast dashboard shows scenario comparison, AI vs. human, and trend
- [ ] Pipeline snapshots power the waterfall and trend views
- [ ] Forecast submissions are persisted with full breakdown in JSONB

---

## Phase 8: Pipeline Review Meetings

**Goal:** Auto-generate AI-powered pre-meeting pipeline review briefs with ranked deal priorities, coaching questions, and structured meeting agendas.

**Duration:** 2 weeks

**Dependencies:** Phase 6

### Task 8.1: Review Brief Generator

**What:** Build an LLM-powered service that generates a structured pipeline review brief for managers. The brief includes: (1) executive summary (total pipeline, gap to quota, key risks), (2) deals ranked by review priority (risk level, amount, coaching need), (3) per-deal coaching questions specific to the deal's signals and qualification gaps, (4) rep-level coaching priorities, (5) forecast summary with variance from last submission.

**Design:**
```typescript
// apps/api/src/services/review-brief-generator.ts
export class ReviewBriefGenerator {
  async generateBrief(managerId: string, tenantId: string): Promise<ReviewBrief> {
    // 1. Gather all deals for the manager's reports
    const reportIds = await this.getDirectReports(managerId, tenantId);
    const deals = await db.query.deal.findMany({
      where: and(
        eq(deal.tenantId, tenantId),
        eq(deal.isOpen, true),
        inArray(deal.ownerId, reportIds),
      ),
      orderBy: [asc(deal.aiScore)],  // Lowest score first = highest priority
    });

    // 2. Build context for LLM
    const context = {
      manager: await this.getUser(managerId),
      deals: deals.map((d) => ({
        name: d.name,
        owner: d.ownerName,
        amount: d.amountCents,
        stage: d.stageName,
        aiScore: d.aiScore,
        riskLevel: d.aiRiskLevel,
        signals: d.signals,
        qualification: d.qualification,
        daysInStage: d.daysInStage,
        closeDate: d.closeDate,
      })),
      forecast: await this.getLatestForecast(tenantId, reportIds),
    };

    // 3. Generate brief via LLM
    const brief = await this.callLLM(context);

    // 4. Save to pipeline_review table
    const review = await db.insert(pipelineReview).values({
      tenantId,
      managerId,
      scheduledAt: null,
      status: "ready",
      brief,
      createdAt: new Date(),
    }).returning();

    return review[0];
  }
}
```

**Testing:**
- Integration test: brief generator produces a structured brief for a manager with 3 reports and 15 deals
- Unit test: deals are ranked by priority (critical risk > high risk > medium risk, then by amount)
- Unit test: coaching questions reference specific deal signals (e.g., "Has the champion responded since May 8?")
- Unit test: forecast summary includes gap-to-quota calculation
- Unit test: brief JSONB includes all required fields (summary, deals_to_review, forecast_summary, rep_coaching_priorities)
- Functional test: manager can view the brief in the Reviews page
- Functional test: manager can add notes and action items during the review
- Functional test: review status changes from "ready" to "in_progress" to "completed"

### Definition of Done — Phase 8

- [ ] AI-generated review briefs are produced on demand or on schedule
- [ ] Briefs include deal ranking, coaching questions, and forecast context
- [ ] Managers can conduct reviews with inline note-taking and action items
- [ ] Review history is persisted for future reference

---

## Phase 9: Manager Bias & Forecast Accuracy Tracking

**Goal:** Track forecast accuracy over time and detect systematic over- or under-calling patterns by manager.

**Duration:** 2 weeks

**Dependencies:** Phase 7

### Task 9.1: Accuracy Calculation Engine

**What:** At the end of each forecast period (quarter close), calculate the variance between each user's forecast submissions and actual closed-won revenue. Determine bias direction (over/under/accurate) and variance percentage. Store results in `forecast_accuracy` table.

**Design:**
```typescript
// apps/api/src/services/forecast-accuracy.ts
export class ForecastAccuracyService {
  async calculatePeriodAccuracy(tenantId: string, periodStart: Date, periodEnd: Date) {
    // Get actual closed-won revenue per user
    const actuals = await db.execute(sql`
      SELECT owner_id, SUM(amount_cents) AS actual_amount
      FROM deal
      WHERE tenant_id = ${tenantId}
        AND is_won = true
        AND close_date >= ${periodStart}
        AND close_date <= ${periodEnd}
      GROUP BY owner_id
    `);

    // Get latest forecast submissions per user per scenario
    const submissions = await db.query.forecastSubmission.findMany({
      where: and(
        eq(forecastSubmission.tenantId, tenantId),
        eq(forecastSubmission.periodStart, periodStart),
      ),
      orderBy: [desc(forecastSubmission.submittedAt)],
    });

    // Calculate accuracy for each user
    for (const actual of actuals.rows) {
      const userSubmissions = submissions.filter((s) => s.userId === actual.owner_id);
      for (const scenario of ["committed", "best_case", "pipeline"]) {
        const submission = userSubmissions.find((s) => s.scenario === scenario);
        if (!submission) continue;

        const variancePct = submission.amountCents === 0 ? 0
          : ((actual.actual_amount - submission.amountCents) / submission.amountCents) * 100;

        const biasDirection = variancePct > 5 ? "under"
          : variancePct < -5 ? "over" : "accurate";

        await db.insert(forecastAccuracy).values({
          tenantId,
          userId: actual.owner_id,
          periodStart,
          periodEnd,
          scenario,
          forecastAmount: submission.amountCents,
          actualAmount: actual.actual_amount,
          variancePct: Math.round(variancePct * 100) / 100,
          biasDirection,
        }).onConflictDoUpdate({
          target: [forecastAccuracy.tenantId, forecastAccuracy.userId,
                   forecastAccuracy.periodStart, forecastAccuracy.scenario],
          set: {
            forecastAmount: submission.amountCents,
            actualAmount: actual.actual_amount,
            variancePct: Math.round(variancePct * 100) / 100,
            biasDirection,
            calculatedAt: new Date(),
          },
        });
      }
    }
  }
}
```

**Testing:**
- Unit test: rep who forecast $100K committed and closed $95K has variance_pct = -5% and bias_direction = "accurate"
- Unit test: rep who forecast $100K committed and closed $130K has variance_pct = 30% and bias_direction = "under"
- Unit test: rep who forecast $100K committed and closed $70K has variance_pct = -30% and bias_direction = "over"
- Unit test: rep with $0 forecast and actual closed-won has variance_pct = 0
- Unit test: multiple quarters of "over" bias creates a visible pattern in the accuracy table
- Functional test: manager bias dashboard shows per-rep accuracy over last 4 quarters with trend lines
- Functional test: colour coding highlights consistent over-callers (red) and under-callers (blue)

### Definition of Done — Phase 9

- [ ] Forecast accuracy calculated at period close for all users and scenarios
- [ ] Manager bias detection shows patterns over multiple quarters
- [ ] Accuracy dashboard with per-rep variance and trend visualisation
- [ ] Bias direction feeds back into AI forecast confidence weighting

---

## Phase 10: Notifications & Alerts

**Goal:** Deliver real-time deal risk alerts, coaching nudges, and forecast change notifications via in-app, email, and Slack channels.

**Duration:** 2 weeks

**Dependencies:** Phase 1

### Task 10.1: Notification Engine

**What:** Build a notification service that generates and dispatches alerts for: deal risk level change (medium->high, high->critical), stalled deal detection, new coaching recommendation, forecast submission deadline approaching, pipeline review brief ready, and significant score drops (>15 points). Support in-app notifications (stored in `notification` table), email (via transactional email service), and Slack (via Incoming Webhooks or Web API).

**Design:**
```typescript
// apps/api/src/services/notification-service.ts
export class NotificationService {
  async notifyDealRiskChange(deal: Deal, previousRisk: string, newRisk: string) {
    const recipients = await this.getRecipients(deal);
    const title = `Deal risk changed: ${deal.name}`;
    const body = `Risk level changed from ${previousRisk} to ${newRisk}. ${deal.aiExplanation}`;

    for (const recipient of recipients) {
      const channels = recipient.profile?.notification_channels ?? ["in_app"];
      for (const channel of channels) {
        await this.dispatch(channel, {
          tenantId: deal.tenantId,
          userId: recipient.id,
          notificationType: "deal_at_risk",
          title,
          body,
          dealId: deal.id,
          context: { previous_risk: previousRisk, new_risk: newRisk },
        });
      }
    }
  }

  private async dispatch(channel: string, notification: NotificationPayload) {
    switch (channel) {
      case "in_app":
        await db.insert(notificationTable).values(notification);
        break;
      case "slack":
        await queues.notificationDispatch.add("slack", {
          ...notification,
          webhookUrl: await this.getSlackWebhook(notification.tenantId),
        });
        break;
      case "email":
        await queues.notificationDispatch.add("email", notification);
        break;
    }
  }
}
```

**Testing:**
- Unit test: deal risk change from "medium" to "high" creates an in-app notification for the deal owner
- Unit test: deal risk change from "medium" to "high" sends a Slack message to the configured webhook
- Unit test: user with notification channels `["in_app", "slack"]` receives notifications on both channels
- Unit test: user with notification channels `["in_app"]` does not receive Slack notifications
- Integration test: Slack webhook receives a Block Kit formatted message with deal name, risk level, and explanation
- Unit test: notification deduplication — same notification type for the same deal within 1 hour is not sent again
- Functional test: in-app notification bell shows unread count and notification list
- Functional test: clicking a notification marks it as read and navigates to the deal detail page

### Definition of Done — Phase 10

- [ ] In-app notifications work for all event types (risk change, stalled deal, coaching, forecast)
- [ ] Slack integration delivers alerts to configured webhook URLs
- [ ] Email notifications sent for critical-priority events
- [ ] Users can configure which channels they receive notifications on
- [ ] Notification deduplication prevents alert fatigue
- [ ] Unread notification count visible in the app header

---

## Phase 11: MCP Server & API Publication

**Goal:** Expose pipeline data through a Model Context Protocol (MCP) server and publish an OpenAPI 3.1 specification, enabling AI agents and third-party integrations to query pipeline health programmatically.

**Duration:** 2 weeks

**Dependencies:** Phase 10

### Task 11.1: MCP Server Implementation

**What:** Build an MCP server (using the TypeScript MCP SDK) that exposes tools for: querying pipeline health, getting deal scores, fetching forecast data, listing at-risk deals, and retrieving coaching recommendations. The MCP server authenticates via API key and enforces tenant isolation.

**Design:**
```typescript
// packages/mcp-server/src/server.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({
  name: "pipeline-review-assistant",
  version: "1.0.0",
});

server.tool(
  "query_pipeline",
  "Query the current pipeline state with optional filters",
  {
    risk_level: z.enum(["low", "medium", "high", "critical"]).optional(),
    owner: z.string().optional(),
    forecast_category: z.string().optional(),
    min_amount: z.number().optional(),
  },
  async (params) => {
    const deals = await dealService.queryPipeline(params);
    return {
      content: [{
        type: "text",
        text: JSON.stringify({
          total_deals: deals.length,
          total_amount: deals.reduce((sum, d) => sum + (d.amountCents ?? 0), 0) / 100,
          deals: deals.map((d) => ({
            name: d.name,
            amount: (d.amountCents ?? 0) / 100,
            stage: d.stageName,
            risk_level: d.aiRiskLevel,
            score: d.aiScore,
            close_date: d.closeDate,
          })),
        }, null, 2),
      }],
    };
  }
);

server.tool(
  "get_deal_score",
  "Get the AI health score and risk assessment for a specific deal",
  { deal_name: z.string() },
  async (params) => {
    const deal = await dealService.findByName(params.deal_name);
    return {
      content: [{
        type: "text",
        text: JSON.stringify({
          name: deal.name,
          score: deal.aiScore,
          risk_level: deal.aiRiskLevel,
          momentum: deal.aiMomentum,
          explanation: deal.aiExplanation,
          signals: deal.signals,
          qualification: deal.qualification,
        }, null, 2),
      }],
    };
  }
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

**Testing:**
- Integration test: MCP client connects to the server and calls `query_pipeline` successfully
- Integration test: `query_pipeline` with `risk_level: "critical"` returns only critical-risk deals
- Integration test: `get_deal_score` returns correct score, explanation, and signals for a known deal
- Unit test: MCP server enforces tenant isolation — requests for deals in other tenants return empty
- Unit test: invalid tool parameters are rejected with a descriptive error

### Task 11.2: OpenAPI Specification & Documentation

**What:** Ensure the Fastify API auto-generates an OpenAPI 3.1 specification at `/api/docs`. Add descriptive summaries, examples, and response schemas for all API endpoints. Publish a Swagger UI for interactive API exploration.

**Testing:**
- OpenAPI spec validates against the OpenAPI 3.1 JSON Schema
- Swagger UI is accessible at `/api/docs` and allows interactive API calls
- All API endpoints have request/response schemas documented
- Example payloads are valid and representative

### Definition of Done — Phase 11

- [ ] MCP server exposes tools for pipeline queries, deal scores, and forecasts
- [ ] MCP server authenticates and enforces tenant isolation
- [ ] OpenAPI 3.1 spec auto-generated and accessible at `/api/docs`
- [ ] Swagger UI available for interactive API exploration

---

## Phase 12: Production Hardening

**Goal:** Prepare the application for production deployment with security, observability, performance, and compliance requirements.

**Duration:** 3 weeks

**Dependencies:** All previous phases

### Task 12.1: Security Audit & Hardening

**What:** Conduct an OWASP API Security Top 10 review. Implement: rate limiting on all API endpoints, input validation on all JSONB writes (Zod schemas), CORS configuration, CSP headers, secure cookie settings, API key rotation for MCP server, and encryption-at-rest for OAuth tokens (already using BYTEA encrypted columns). Ensure all endpoints enforce tenant isolation via RLS.

**Testing:**
- Security test: API rate limiter returns 429 after exceeding configured threshold
- Security test: CORS rejects requests from non-allowed origins
- Security test: XSS payload in deal name is sanitised on output
- Security test: SQL injection attempt via JSONB filter parameter is blocked
- Security test: API endpoint without authentication returns 401
- Security test: RLS prevents cross-tenant data access at the database level (not just application level)
- Security test: OAuth tokens are encrypted in the database (verified by reading raw BYTEA column)

### Task 12.2: Observability & Monitoring

**What:** Implement structured logging (Pino), distributed tracing (OpenTelemetry), and metrics (Prometheus). Key metrics: CRM sync latency, AI scoring duration, API response times, queue depth, and active deal count. Set up health check endpoints (`/health`, `/ready`).

**Testing:**
- Health check: `/health` returns 200 when database and Redis are connected
- Health check: `/ready` returns 503 when database is unreachable
- Metrics: Prometheus endpoint at `/metrics` exposes `deal_scoring_duration_seconds`, `crm_sync_total`, and `api_request_duration_seconds`
- Tracing: a single API request produces correlated trace spans across API handler, database query, and external API call

### Task 12.3: Performance Optimisation

**What:** Optimise database queries with EXPLAIN ANALYZE on critical paths (pipeline dashboard, deal detail, forecast roll-up). Add database connection pooling (PgBouncer or built-in pool). Implement Redis caching for frequently-accessed dashboard aggregates. Target: pipeline dashboard loads in under 1 second for 500 deals, deal detail loads in under 500ms.

**Testing:**
- Performance test: pipeline dashboard with 500 deals loads in under 1 second (server-side rendering time)
- Performance test: deal detail page loads in under 500ms
- Performance test: forecast roll-up for a team of 20 reps calculates in under 200ms
- Load test: 50 concurrent users browsing the pipeline dashboard with no errors and p95 < 2 seconds

### Task 12.4: GDPR & Compliance

**What:** Implement GDPR right-to-erasure for contacts (delete contact PII, clear `provider_data` contact fields, retain anonymised deal history). Add data export endpoint for data portability. Document data processing activities for GDPR Article 30 compliance. Add AI transparency disclosures (what the AI scored, what model version, what inputs) per EU AI Act requirements — already stored in `deal_score_history.scoring_context`.

**Testing:**
- GDPR test: right-to-erasure for a contact removes PII (name, email, phone) but retains deal history with anonymised references
- GDPR test: data export endpoint returns all user data in JSON format within 30 seconds
- Compliance test: `deal_score_history` records include model version, input signals, and confidence for every score
- Compliance test: audit log records all data access and modifications with timestamps

### Task 12.5: CI/CD & Deployment

**What:** Set up GitHub Actions CI pipeline: lint, type-check, unit tests, integration tests (with PostgreSQL and Redis services), build, and Docker image creation. Configure deployment to Kubernetes via Helm chart with environment-specific values (staging, production). Implement database migration as a Kubernetes Job run before deployment.

**Testing:**
- CI: push to `main` triggers lint, test, build, and image push
- CI: PR checks run tests and report coverage
- Deployment: staging deployment via Helm chart succeeds with migrations applied
- Deployment: zero-downtime rolling update verified by sending requests during deployment
- Rollback: `helm rollback` restores previous version within 2 minutes

### Definition of Done — Phase 12

- [ ] OWASP API Top 10 review completed with all findings addressed
- [ ] Structured logging, tracing, and metrics operational
- [ ] Performance targets met (dashboard <1s, detail <500ms, 50 concurrent users)
- [ ] GDPR right-to-erasure and data export functional
- [ ] EU AI Act transparency requirements met via scoring provenance
- [ ] CI/CD pipeline runs tests and deploys to staging automatically
- [ ] Production deployment documented with runbook

---

## Summary

| Phase | Name | Duration | Dependencies |
|-------|------|----------|-------------|
| 1 | Foundation & Core Infrastructure | 3 weeks | None |
| 2 | CRM Integration Layer | 4 weeks | Phase 1 |
| 3 | Email & Activity Capture | 3 weeks | Phase 1 |
| 4 | AI Deal Scoring Engine | 4 weeks | Phase 2, 3 |
| 5 | Pipeline Dashboard | 3 weeks | Phase 2 |
| 6 | Coaching Engine | 3 weeks | Phase 4 |
| 7 | Forecasting Engine | 3 weeks | Phase 4 |
| 8 | Pipeline Review Meetings | 2 weeks | Phase 6 |
| 9 | Manager Bias & Forecast Accuracy | 2 weeks | Phase 7 |
| 10 | Notifications & Alerts | 2 weeks | Phase 1 |
| 11 | MCP Server & API Publication | 2 weeks | Phase 10 |
| 12 | Production Hardening | 3 weeks | All |
| **Total** | | **34 weeks** | |

**Critical path:** Phase 1 -> Phase 2 -> Phase 4 -> Phase 6 -> Phase 8 (16 weeks)

**Parallelisable work:**
- Phase 2 (CRM) and Phase 3 (Email) can run in parallel after Phase 1
- Phase 5 (Dashboard) can start after Phase 2, in parallel with Phase 4
- Phase 6 (Coaching) and Phase 7 (Forecasting) can run in parallel after Phase 4
- Phase 10 (Notifications) can start after Phase 1, running in parallel with Phases 2-9

**With full parallelisation, the estimated calendar time is approximately 22 weeks.**
