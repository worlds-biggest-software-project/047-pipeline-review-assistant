# Standards & API Reference

> Project: Pipeline Review Assistant · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 27001:2022 — Information Security Management Systems**
- Official URL: https://www.iso.org/standard/27001
- Relevance: The de facto enterprise security certification required by large buyers when evaluating SaaS vendors that handle CRM, deal, and contact data. Pipeline tools processing conversation recordings and financial forecast data must demonstrate an ISMS aligned with ISO 27001 to pass enterprise procurement reviews. Achieving certification is a hard requirement for many Fortune 500 procurement checklists.

**ISO/IEC 42001:2023 — Artificial Intelligence Management Systems**
- Official URL: https://www.iso.org/standard/81230.html
- Relevance: Provides a governance framework for responsible AI system design, deployment, and management. Applicable when the Pipeline Review Assistant uses AI deal scoring or automated forecasting that influences business decisions, especially in jurisdictions where AI governance is regulated.

**ISO/IEC 27701:2019 — Privacy Information Management**
- Official URL: https://www.iso.org/standard/71670.html
- Relevance: An extension to ISO 27001 that adds privacy-specific controls aligned with GDPR. Directly relevant because pipeline tools store personal data about sales contacts and prospects; a privacy-by-design certification strengthens GDPR compliance evidence.

---

### W3C & IETF Standards

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- Official URL: https://www.rfc-editor.org/rfc/rfc6749.html
- Relevance: The foundational standard governing how the Pipeline Review Assistant obtains delegated access to Gmail, Outlook/Microsoft Graph, Salesforce, HubSpot, and other CRM APIs on behalf of users. All CRM and email/calendar integrations require OAuth 2.0 authorization code flows. As of 2026, Salesforce specifically recommends My Domain–specific OAuth endpoints over legacy generic endpoints.

**RFC 8693 — OAuth 2.0 Token Exchange**
- Official URL: https://datatracker.ietf.org/doc/html/rfc8693
- Relevance: Enables token exchange across microservices and across service boundaries — relevant when the pipeline assistant aggregates data from multiple OAuth-protected sources (e.g., Salesforce + Gmail + Gong) within a single request pipeline, allowing downstream services to operate with appropriately scoped tokens.

**OpenID Connect Core 1.0**
- Official URL: https://openid.net/specs/openid-connect-core-1_0.html
- Relevance: The identity layer on top of OAuth 2.0 used for enterprise SSO. Buyers deploying the Pipeline Review Assistant in an enterprise context will require OIDC-based SSO integration with their identity provider (Okta, Azure AD, Ping). Ensures reps and managers authenticate with corporate credentials rather than separate product accounts.

**RFC 7519 — JSON Web Tokens (JWT)**
- Official URL: https://datatracker.ietf.org/doc/html/rfc7519
- Relevance: The token format used in OAuth 2.0 and OIDC flows across all CRM APIs. JWTs carry claims about the authenticated user and their scoped permissions; the Pipeline Review Assistant must validate and securely store JWT access and refresh tokens for each connected CRM and communication system.

**RFC 9110 — HTTP Semantics**
- Official URL: https://datatracker.ietf.org/doc/html/rfc9110
- Relevance: The normative standard for HTTP methods, headers, and status codes that all REST API integrations in the stack (Salesforce, HubSpot, Gong, Slack, Microsoft Graph) are built upon. Understanding RFC 9110 is essential for correct error handling, caching headers, and content-type negotiation in the integration layer.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1 / 3.2**
- Official URL: https://spec.openapis.org/oas/v3.2.0.html; https://www.openapis.org/
- Relevance: The industry-standard machine-readable API description format. In 2026, API-first development is the norm for over 80% of organizations. The Pipeline Review Assistant should publish an OpenAPI 3.1+ specification for its own API surface, and consume OpenAPI specs published by CRM vendors (Salesforce, HubSpot) for code-generated SDK clients. As of 2026, OpenAPI specs are also the primary bridge enabling AI agents and MCP servers to discover and call API endpoints.

**JSON Schema (Draft 2020-12)**
- Official URL: https://json-schema.org/specification
- Relevance: The standard for validating the structure of JSON data exchanged with CRM APIs and for defining the schema of pipeline deal records, deal scores, and forecast submissions. Used in conjunction with OpenAPI 3.1 (which embeds JSON Schema) for validating AI-generated deal scores before writing them back to CRM systems.

**OData v4 (ISO/IEC 20802)**
- Official URL: https://www.odata.org/documentation/
- Relevance: Microsoft Dynamics 365 Sales and the underlying Dataverse platform expose their APIs as OData v4 endpoints. The Pipeline Review Assistant must parse OData query responses and construct OData filter expressions when querying Dynamics 365 opportunity records and forecast data.

---

### Security & Authentication Standards

**GDPR — General Data Protection Regulation (EU) 2016/679**
- Official URL: https://gdpr-info.eu/
- Relevance: Applies whenever the Pipeline Review Assistant processes personal data about EU-based contacts, prospects, or sales staff. Article 22 grants data subjects the right not to be subject to decisions based solely on automated processing — directly applicable to AI deal scoring that influences compensation or rep performance reviews. Data processor agreements are required for all sub-processors. Fines reach up to €20M or 4% of global annual turnover.

**EU AI Act (Regulation 2024/1689)**
- Official URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32024R1689
- Relevance: The EU's risk-based AI regulatory framework enacted in 2024. Sales forecasting software and automated deal-scoring systems that materially influence HR or business decisions may qualify as "limited-risk" or higher under the Act, requiring transparency obligations and human oversight mechanisms. Non-compliance fines reach up to €35M or 7% of global annual turnover.

**CCPA / CPRA (California Consumer Privacy Act)**
- Official URL: https://cppa.ca.gov/regulations/
- Relevance: As of January 2026, CCPA explicitly covers B2B contact data and introduces new requirements for Automated Decision-Making Technology (ADMT) including privacy risk assessments before using AI to score, rank, or profile individuals. The Pipeline Review Assistant's deal-scoring and rep-performance tracking features fall within ADMT scope for California-based users.

**OWASP API Security Top 10 (2023)**
- Official URL: https://owasp.org/API-Security/editions/2023/en/0x00-header/
- Relevance: The authoritative checklist for API security vulnerabilities. The Pipeline Review Assistant exposes API endpoints for CRM data ingestion and deal score write-back; the OWASP API Top 10 (covering broken object-level authorisation, excessive data exposure, lack of rate limiting, etc.) should guide the API security review process.

**NIST SP 800-53 Rev 5 — Security and Privacy Controls**
- Official URL: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- Relevance: The US federal security control framework. Enterprise buyers in regulated industries (financial services, healthcare, government) will benchmark the Pipeline Review Assistant's security controls against NIST 800-53, particularly for access control (AC), audit and accountability (AU), and system protection (SC) families.

---

### Model Context Protocol (MCP) Specifications

**Anthropic Model Context Protocol (MCP) — Specification v1.x**
- Official URL: https://modelcontextprotocol.io/; https://www.anthropic.com/news/model-context-protocol
- Relevance: The open standard introduced by Anthropic in November 2024 for connecting LLMs to external tools, data sources, and APIs. By May 2026 there are over 10,000 active public MCP servers and 97 million monthly SDK downloads. The Pipeline Review Assistant should expose an MCP server interface so that Claude Desktop, sales AI assistants, and other MCP-compliant clients can directly query pipeline health, deal scores, and forecast data via natural language without custom integration code. The TypeScript and Python reference SDKs are available at https://github.com/modelcontextprotocol.

---

## Similar Products — Developer Documentation & APIs

### Salesforce Sales Cloud (REST API + CRM Analytics)

- **Description:** The dominant CRM platform; virtually all pipeline intelligence tools must integrate with Salesforce as the primary data source. Exposes Opportunities, OpportunityStage, ForecastingItem, and ActivityHistory objects over REST.
- **API Documentation:** https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/resources_list.htm
- **CRM Analytics REST API:** https://resources.docs.salesforce.com/latest/latest/en-us/sfdc/pdf/bi_dev_guide_rest.pdf
- **API Library (all APIs):** https://developer.salesforce.com/docs/apis
- **SDKs/Libraries:** jsforce (Node.js, MIT); simple-salesforce (Python); Apex for native server-side; Salesforce Mobile SDK (iOS/Android)
- **Developer Guide:** https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/intro_what_is_rest_api.htm
- **Standards:** REST/JSON; Bulk API 2.0 for async large-volume queries; Pub/Sub API (gRPC-based) for real-time event streaming; OData v4 not used natively
- **Authentication:** OAuth 2.0 Web Server Flow (authorization code); Connected Apps required; My Domain–specific endpoints as of Spring '26 (v66.0)

---

### HubSpot CRM (CRM Pipelines API + Deals API)

- **Description:** Primary CRM for SMB and mid-market; offers Pipelines API for stage management and Deals API for opportunity CRUD. Forecast categories and weighted pipeline math are configurable via API.
- **API Documentation (Pipelines):** https://developers.hubspot.com/docs/guides/api/crm/pipelines
- **API Reference:** https://developers.hubspot.com/docs/reference/api/crm/pipelines
- **SDKs/Libraries:** HubSpot Node.js client; HubSpot Python client; available at https://github.com/HubSpot
- **Developer Guide:** https://developers.hubspot.com/docs/guides
- **Standards:** REST/JSON; OpenAPI 3.0 spec available; webhook subscriptions for deal-stage change events
- **Authentication:** OAuth 2.0 (authorization code flow); Private App tokens for server-side integrations; API key deprecated

---

### Microsoft Dynamics 365 Sales (Dataverse Web API)

- **Description:** Enterprise CRM built on Microsoft Dataverse; exposes opportunity, lead, and forecast entities through an OData v4 Web API. Dynamics 365 Sales AI surfaces deal insights and relationship analytics.
- **API Documentation:** https://learn.microsoft.com/en-us/dynamics365/sales/developer/developer-guide
- **Dynamics 365 REST APIs:** https://learn.microsoft.com/en-us/rest/dynamics365/
- **Dataverse Developer Guide:** https://learn.microsoft.com/en-us/power-apps/developer/data-platform/
- **SDKs/Libraries:** Microsoft.PowerPlatform.Dataverse.Client (.NET); dataverse-ify (TypeScript); community Python wrappers
- **Standards:** OData v4; REST/JSON; OpenAPI 3.0 descriptions available via Power Platform; supports WebHooks and Azure Service Bus for event streaming
- **Authentication:** OAuth 2.0 with Azure Active Directory (now Entra ID); MSAL libraries recommended; supports application permissions for server-to-server scenarios

---

### Gong (Conversation Intelligence + Forecast API)

- **Description:** Conversation intelligence platform that ingests call recordings, transcripts, and CRM data to produce deal risk signals and forecast submissions. Gong Forecast API exposes deal scorecards and pipeline data with 300+ signal attributes.
- **API Documentation:** https://help.gong.io/docs/what-the-gong-api-provides
- **API Access Setup:** https://help.gong.io/docs/receive-access-to-the-api
- **Engage API Capabilities:** https://help.gong.io/docs/gong-engage-api-capabilities
- **Postman Collection:** https://www.postman.com/growment/gong-meetup/collection/yuikwaq/gong-api-beginners-guide
- **SDKs/Libraries:** No official SDK; community Ruby client at https://github.com/matteeyah/gong-api; REST/JSON consumed directly
- **Standards:** REST/JSON; base URL https://api.gong.io/v2/; OAuth 2.0 app creation documented at https://help.gong.io/docs/create-an-app-for-gong
- **Authentication:** OAuth 2.0 (Gong OAuth app); Gong administrator access required to generate credentials; bearer token in Authorization header

---

### Clari (Revenue Operations Platform API)

- **Description:** Enterprise revenue forecasting and pipeline management platform. Clari's Export API allows forecast submission data to be exported to data warehouses; Copilot API provides call recording and conversation intelligence data.
- **API Documentation:** https://developer.clari.com/documentation/external_spec
- **Copilot API:** https://api-doc.copilot.clari.com/
- **REST API Base URL:** https://api.clari.com/v2
- **SDKs/Libraries:** No official SDK; Python context via dltHub at https://dlthub.com/context/source/clari; Ampersand integration at https://docs.withampersand.com/provider-guides/clari
- **Standards:** REST/JSON; OpenAPI/Swagger spec available; supports webhooks for forecast change events; Postman and Insomnia collections available
- **Authentication:** API token in `apikey` header; partner/ingest APIs also require `partnerkey` header; Copilot API uses `X-Api-Key` and `X-Api-Password` headers

---

### Salesloft (now part of Clari — Sales Engagement API)

- **Description:** Sales engagement platform (merged with Clari December 2025) providing sequences, cadences, and sales activity data. API enables read/write access to accounts, people, cadence memberships, and call recordings relevant to pipeline activity capture.
- **Developer Portal:** https://developers.salesloft.com/
- **API Basics:** https://developers.salesloft.com/docs/platform/api-basics/
- **Postman Documentation:** https://www.postman.com/salesloft-dev/salesloft/documentation/u1411ak/salesloft-developer-api
- **SDKs/Libraries:** No official SDK; GitHub example repository at https://github.com/SalesLoft/api-example; REST consumed directly
- **Standards:** REST/JSON; OAuth 2.0 authorization
- **Authentication:** OAuth 2.0 (authorization code flow); bearer token; contact support@salesloft.com for API access

---

### Pipedrive (Deals & Pipeline API)

- **Description:** SMB-focused CRM with a well-documented REST API for deal and pipeline management. API v2 exited beta March 2025 and is the recommended integration target in 2026; v1 remains for endpoints not yet migrated.
- **API Documentation:** https://developers.pipedrive.com/docs/api/v1
- **Deals Reference:** https://developers.pipedrive.com/docs/api/v1/Deals
- **Developer Getting Started:** https://pipedrive.readme.io/docs/getting-started
- **SDKs/Libraries:** Official Node.js client (MIT licence) at https://github.com/pipedrive/client-nodejs; community Python wrappers available
- **Standards:** REST/JSON; token-budget rate limiting introduced 2025 (daily API token budget shared per company account); webhooks for real-time deal-stage events
- **Authentication:** API key (legacy) or OAuth 2.0; OAuth 2.0 recommended for multi-user integrations; bearer token in Authorization header

---

### Zoho CRM (V8 API — Deals & Pipeline)

- **Description:** Mid-market CRM with pipeline management APIs allowing full CRUD on deal stages, pipeline definitions, and AI assistant (Zia) signals. V8 API is the current stable version as of 2026.
- **API Documentation:** https://www.zoho.com/crm/developer/docs/api/v8/
- **Pipeline Endpoints:** https://www.zoho.com/crm/developer/docs/api/v8/get-pipelines.html
- **Developer Resources:** https://www.zoho.com/crm/developer/docs/
- **SDKs/Libraries:** Official Zoho CRM SDK for Python, Java, PHP, Node.js, and .NET at https://www.zoho.com/crm/developer/api.html
- **Standards:** REST/JSON; Bulk API for async large-volume data sync; supports webhooks (Zoho Flow) for deal-stage change events
- **Authentication:** OAuth 2.0 (Zoho Accounts); server-to-server via client credentials; domain-specific token endpoints per region (US, EU, IN, AU, JP)

---

### Microsoft Graph API (Outlook Mail + Calendar)

- **Description:** Unified Microsoft API for accessing Outlook email, calendar, contacts, and Teams data. Required for activity capture from Microsoft 365 users — auto-logging emails and meetings against CRM deals.
- **Mail API Overview:** https://learn.microsoft.com/en-us/graph/outlook-mail-concept-overview
- **Calendar API Overview:** https://learn.microsoft.com/en-us/graph/outlook-calendar-concept-overview
- **Full Reference:** https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview?view=graph-rest-1.0
- **SDKs/Libraries:** Microsoft Graph SDK for JavaScript, Python, Java, .NET, Go at https://learn.microsoft.com/en-us/graph/sdks/sdks-overview
- **Standards:** REST/JSON; OData v4 query parameters for filtering; OpenAPI 3.0 spec published by Microsoft; webhook change notifications via Microsoft Graph subscriptions
- **Authentication:** OAuth 2.0 with Azure AD (Entra ID); MSAL libraries (msal-node, msal-python); delegated permissions for user-context; application permissions for background sync; scopes: `Mail.Read`, `Calendars.Read`, `offline_access`

---

### Google Workspace APIs (Gmail + Calendar)

- **Description:** Google's APIs for accessing Gmail and Google Calendar data, required for activity capture from Google Workspace users. Used to auto-log emails and meeting events against CRM deals without manual rep entry.
- **Gmail API Scopes:** https://developers.google.com/workspace/gmail/api/auth/scopes
- **Calendar API Auth:** https://developers.google.com/workspace/calendar/api/auth
- **OAuth 2.0 for Google APIs:** https://developers.google.com/identity/protocols/oauth2
- **SDKs/Libraries:** google-api-python-client; googleapis/google-api-nodejs-client; Java and Go official libraries at https://developers.google.com/workspace/products
- **Standards:** REST/JSON; OAuth 2.0 with Google Identity; domain-wide delegation for enterprise Google Workspace deployments (service account impersonation); Google Pub/Sub for push notifications on mailbox changes
- **Authentication:** OAuth 2.0 (authorization code flow for user-delegated access); Service Account + domain-wide delegation for enterprise background sync; restricted scopes (`https://www.googleapis.com/auth/gmail.readonly`, `https://www.googleapis.com/auth/calendar.readonly`) require Google OAuth App Verification for production use

---

### Slack API (Incoming Webhooks + Web API)

- **Description:** Messaging platform API used to deliver real-time deal alerts, forecast change notifications, and pipeline review briefs to sales team channels and direct messages.
- **Incoming Webhooks:** https://api.slack.com/incoming-webhooks
- **Web API Reference:** https://api.slack.com/web
- **Block Kit UI:** https://api.slack.com/block-kit
- **SDKs/Libraries:** `@slack/web-api` (Node.js, official); `slack-sdk` (Python, official); Bolt framework for Slack apps (Node.js, Python, Java)
- **Standards:** REST/JSON with Slack-specific Block Kit payload format for rich messages; WebSocket-based Socket Mode for real-time event listening without exposing a public endpoint; rate limit ~1 message/second per webhook URL
- **Authentication:** Incoming webhooks are self-contained (no token required after setup); Web API uses Bearer token (Bot Token, `xoxb-`); OAuth 2.0 app installation flow for distributing to multiple workspaces

---

## Notes

**MEDDIC/MEDDPICC as a data model concern:** The MEDDIC framework (Metrics, Economic Buyer, Decision Criteria, Decision Process, Identify Pain, Champion) and its variants (MEDDICC, MEDDPICC) are not formal ISO or IETF standards, but are widely adopted qualification frameworks. Originally developed at Parametric Technology Corporation in the 1990s and now used by 73% of SaaS companies selling above $100K ARR. Any pipeline scoring system that evaluates deal qualification completeness should model MEDDPICC fields as a schema and allow configurable weighting per organisation.

**Unified CRM abstraction layers:** Rather than implementing individual integrations for each CRM, the Pipeline Review Assistant may benefit from consuming Merge.dev, Unified.to, or Truto — unified CRM API abstraction layers that expose a single normalised API across Salesforce, HubSpot, Dynamics 365, Pipedrive, and Zoho. These products publish their own OpenAPI specs and handle OAuth token management per CRM, reducing integration surface area significantly.

**Emerging standard — OpenAPI + MCP convergence:** In 2026, AI agents are increasingly using OpenAPI specs as machine-readable descriptions to discover and invoke REST APIs through MCP tool-call mechanisms. The Pipeline Review Assistant should ensure its own API surface is described with an OpenAPI 3.1 spec and optionally wrapped in an MCP server to make pipeline data queryable by any MCP-compliant AI agent.

**Data residency gaps:** None of the CRM APIs surveyed provide a formal data residency guarantee at the API contract level. Data residency must be negotiated at the SaaS subscription level with each vendor (Salesforce EU org, HubSpot EU data centre, etc.). The Pipeline Review Assistant should document which geographic region data is processed in and obtain Data Processing Agreements (DPAs) from all CRM sub-processors.
