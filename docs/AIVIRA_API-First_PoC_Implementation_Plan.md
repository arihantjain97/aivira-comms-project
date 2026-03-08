# AIVIRA: API-First PoC Implementation Plan

**Author:** Manus AI
**Date:** March 7, 2026
**Aligned with:** AIVIRA Technological Feasibility Analysis (v3), AIVIRA Microsoft API Reference Guide (v2)

---

## 1. Purpose & Scope

This document translates the high-level PoC Overview and the architectural recommendations into a concrete, phased implementation plan. It is designed for a development team to execute against, with clear deliverables, module boundaries, and integration checkpoints. The end goal is a functional, externally demonstrable product that validates AIVIRA's core value proposition with a select group of pilot users.

The PoC operates across two Microsoft tenants:

| Tenant | Role | What Lives Here |
| :--- | :--- | :--- |
| **`Tenant_Aivira`** | AIVIRA's own infrastructure | Backend API, Web Portal, PostgreSQL database, Entra ID App Registration |
| **`Tenant_Customer`** | Simulated customer environment | User mailboxes, Outlook Add-in, "AIVIRA Deal Repository" SharePoint site, Security Group |

---

## 2. Architectural Blueprint

The architecture follows a **hybrid service-based model with a modular-monolith backend, hexagonal internal design, and event-driven internal processing**, as recommended in the architectural analysis. This section codifies that recommendation into a concrete structure.

### 2.1 System-Level Architecture

Three separately deployable components communicate over HTTPS:

```
┌─────────────────────────────────────────────────────────────────┐
│                      Tenant_Customer (M365)                     │
│                                                                 │
│   ┌──────────────┐    ┌───────────────────────────────────┐     │
│   │ Outlook       │    │ SharePoint: "AIVIRA Deal Repo"    │     │
│   │ Add-in        │    │ (Artifacts stored here)           │     │
│   │ (Sidebar UI)  │    └───────────────────────────────────┘     │
│   └──────┬───────┘                                              │
│          │ HTTPS                                                │
└──────────┼──────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Tenant_Aivira (SaaS)                       │
│                                                                 │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │                    AIVIRA Backend API                     │  │
│   │  (Modular Monolith: FastAPI + PostgreSQL + pgvector)      │  │
│   │                                                          │  │
│   │  Modules:                                                │  │
│   │  ┌────────────┐ ┌────────────┐ ┌────────────────────┐   │  │
│   │  │ Identity & │ │ Graph      │ │ Thread Tracking     │   │  │
│   │  │ Onboarding │ │ Integration│ │                     │   │  │
│   │  └────────────┘ └────────────┘ └────────────────────┘   │  │
│   │  ┌────────────┐ ┌────────────┐ ┌────────────────────┐   │  │
│   │  │ Email      │ │ AI Analysis│ │ Deal Timeline /     │   │  │
│   │  │ Ingestion  │ │            │ │ Deal Memory         │   │  │
│   │  └────────────┘ └────────────┘ └────────────────────┘   │  │
│   │  ┌────────────┐ ┌────────────┐ ┌────────────────────┐   │  │
│   │  │ Issue Mgmt │ │ Artifact   │ │ RAG / Chat Query   │   │  │
│   │  │ Escalation │ │ References │ │                     │   │  │
│   │  └────────────┘ └────────────┘ └────────────────────┘   │  │
│   └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │                    AIVIRA Web Portal                      │  │
│   │  (React / Next.js: CFO Dashboard, Deal Timeline)          │  │
│   └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│   External Dependencies:                                        │
│   ┌────────────┐  ┌────────────┐  ┌────────────────────┐       │
│   │ PostgreSQL │  │ Redis      │  │ Azure OpenAI       │       │
│   │ + pgvector │  │ (Job Queue)│  │ (LLM + Embeddings) │       │
│   └────────────┘  └────────────┘  └────────────────────┘       │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Backend Internal Design: Hexagonal Architecture

The backend follows a **Ports-and-Adapters (Hexagonal)** pattern. The domain and application logic at the core does not know about Microsoft Graph SDKs, Azure OpenAI client libraries, or PostgreSQL drivers. Instead, it interacts with abstract **Ports**, which are implemented by concrete **Adapters**.

| Port (Interface) | Adapter (Implementation) | Purpose |
| :--- | :--- | :--- |
| `MailSourcePort` | `GraphMailAdapter` | Read emails, list threads, create drafts, send mail via Microsoft Graph. |
| `ArtifactStorePort` | `SharePointAdapter` | Upload/list files in the customer's SharePoint site via Graph. |
| `LLMAnalysisPort` | `AzureOpenAIAdapter` | Classify intent, detect scope deviation, extract entities, generate summaries. |
| `EmbeddingPort` | `AzureOpenAIEmbeddingAdapter` | Generate vector embeddings for email summaries and metadata. |
| `VectorSearchPort` | `PgvectorAdapter` | Store and query vector embeddings for RAG retrieval. |
| `DealRepositoryPort` | `PostgresAdapter` | CRUD operations on deals, threads, email metadata, issues, and artifact references. |
| `NotificationPort` | `GraphMailAdapter` / `WebSocketAdapter` | Send escalation emails and push real-time updates to the sidebar/portal. |
| `JobQueuePort` | `RedisQueueAdapter` | Enqueue and dequeue asynchronous processing jobs (email ingestion). |

This design ensures that if any external dependency changes (e.g., swapping Azure OpenAI for another LLM provider, or migrating from pgvector to a dedicated vector DB), only the adapter changes. The core logic remains untouched.

### 2.3 Interaction Style

The backend uses a **hybrid interaction model**:

**Synchronous (Request/Response)** for user-facing commands:

| Trigger | Endpoint | Behavior |
| :--- | :--- | :--- |
| User clicks "Track this Thread" | `POST /api/threads/track` | Backend fetches history, creates subscription, returns confirmation. |
| User asks a question in Deal Chat | `POST /api/deals/{id}/chat` | Backend performs RAG retrieval, calls LLM, returns answer. |
| CFO makes a decision | `POST /api/issues/{id}/decision` | Backend records decision, updates issue state, returns confirmation. |
| User requests AI draft | `POST /api/deals/{id}/draft-reply` | Backend generates draft, creates it in user's mailbox, returns draft ID. |

**Asynchronous (Event-Driven)** for webhook-triggered processing:

| Trigger | Internal Events | Behavior |
| :--- | :--- | :--- |
| Graph webhook fires | `EmailReceived` | Webhook handler validates, decrypts, and enqueues a processing job. |
| Worker picks up job | `EmailAnalyzed` | Worker fetches email (if needed), runs AI analysis, stores metadata. |
| AI detects deviation | `ScopeDeviationDetected` | Downstream handler creates an Issue record. |
| Attachment found | `ArtifactCaptured` | Downstream handler uploads to SharePoint, stores reference. |
| Issue created | `IssueRegistered` | Downstream handler sends notification to sidebar and portal. |

For the PoC, the internal event bus can be implemented as a simple in-process pub/sub (e.g., Python's `asyncio.Queue` or a lightweight library), with Redis handling the webhook-to-worker job queue.

### 2.4 Data Model

The database stores **metadata only**. No raw email bodies or file contents are persisted.

| Table | Key Columns | Purpose |
| :--- | :--- | :--- |
| `tenants` | `id`, `name`, `entra_tenant_id`, `graph_tokens_encrypted`, `sharepoint_site_id` | Multi-tenant customer registry. |
| `deals` | `id`, `tenant_id`, `name`, `status`, `created_at` | The Deal Object. |
| `tracked_threads` | `id`, `deal_id`, `graph_conversation_id`, `graph_subscription_id`, `subscription_expiry` | Links a Graph conversation to a Deal. |
| `email_metadata` | `id`, `thread_id`, `graph_message_id`, `sender`, `recipients`, `subject`, `timestamp`, `ai_summary`, `ai_classification`, `entities_json`, `risk_level` | The core metadata record. No raw body stored. |
| `issues` | `id`, `deal_id`, `email_metadata_id`, `type`, `severity`, `status`, `decision`, `decision_by`, `decision_at`, `outcome_tag` | Issue lifecycle tracking. |
| `artifacts` | `id`, `deal_id`, `email_metadata_id`, `filename`, `sharepoint_url`, `sharepoint_item_id`, `content_type` | References to files in customer's SharePoint. |
| `embeddings` | `id`, `email_metadata_id`, `deal_id`, `content_text`, `embedding` (vector) | pgvector table for RAG retrieval. |

### 2.5 Backend Module Structure

```
src/
├── main.py                          # FastAPI app entry point
├── config.py                        # Environment variables, tenant config
│
├── core/                            # Domain logic (no external dependencies)
│   ├── models/                      # Domain entities (Deal, Thread, Issue, etc.)
│   ├── ports/                       # Abstract interfaces (MailSourcePort, etc.)
│   ├── events/                      # Internal domain events
│   └── services/                    # Application services (orchestration)
│
├── modules/
│   ├── identity/                    # Tenant onboarding, OAuth token management
│   │   ├── routes.py
│   │   └── service.py
│   ├── tracking/                    # Thread tracking initiation
│   │   ├── routes.py
│   │   └── service.py
│   ├── ingestion/                   # Webhook handler + job queue producer
│   │   ├── routes.py                # POST /graph-webhook
│   │   └── worker.py               # Async job consumer
│   ├── analysis/                    # AI analysis pipeline
│   │   ├── service.py
│   │   └── prompts.py              # Prompt templates (Strategy pattern)
│   ├── deal_memory/                 # Deal timeline + RAG chat
│   │   ├── routes.py
│   │   └── service.py
│   ├── issues/                      # Issue detection, escalation, decision
│   │   ├── routes.py
│   │   └── service.py
│   └── artifacts/                   # Artifact extraction + SharePoint upload
│       ├── routes.py
│       └── service.py
│
├── adapters/                        # Concrete implementations of ports
│   ├── graph_mail.py                # GraphMailAdapter
│   ├── graph_sharepoint.py          # SharePointAdapter
│   ├── azure_openai.py              # AzureOpenAIAdapter
│   ├── postgres.py                  # PostgresAdapter
│   ├── pgvector.py                  # PgvectorAdapter
│   └── redis_queue.py              # RedisQueueAdapter
│
└── infrastructure/
    ├── database.py                  # SQLAlchemy / asyncpg setup
    ├── redis.py                     # Redis connection
    └── encryption.py                # Graph webhook payload decryption
```

---

## 3. Phased Implementation Plan

The implementation is divided into five phases, each producing a testable increment. Phases are ordered by dependency: foundational infrastructure first, then core flows, then polish for demo readiness.

### Phase 0: Foundation & Infrastructure (Week 1)

**Goal:** Establish the development environment, Entra ID registration, and database schema so that all subsequent phases have a working foundation.

| Task | Details | Deliverable |
| :--- | :--- | :--- |
| **0.1** Register multi-tenant Entra ID app | In `Tenant_Aivira`. Request permissions: `Mail.Read`, `Mail.ReadWrite`, `Mail.Send`, `Sites.Selected`, `User.Read`. Configure redirect URIs. | App Registration with Client ID and Client Secret. |
| **0.2** Simulate customer onboarding | In `Tenant_Customer`. Admin consents to the AIVIRA app. Create a security group with PoC users. Create a SharePoint site "AIVIRA Deal Repository." Run permission scoping scripts (RBAC for Applications + Sites.Selected grant). | `Tenant_Customer` fully configured for PoC. |
| **0.3** Scaffold backend project | FastAPI project with the module structure defined in Section 2.5. Set up SQLAlchemy models, Alembic migrations, and the pgvector extension. | Running backend with empty endpoints and a migrated database. |
| **0.4** Set up Redis | Deploy a Redis instance for the job queue. Implement `RedisQueueAdapter` with `enqueue()` and `dequeue()` methods. | Working job queue infrastructure. |
| **0.5** Implement Identity module | OAuth 2.0 client credentials flow to obtain and cache Graph API tokens for `Tenant_Customer`. Token refresh logic. Encrypted storage of tenant credentials. | Backend can authenticate to Microsoft Graph on behalf of the customer. |
| **0.6** Deploy backend to a public URL | Deploy to Azure App Service, a VM, or a tunneling service (e.g., ngrok for dev). Graph webhooks require a publicly accessible HTTPS endpoint. | Public URL for webhook callbacks. |

**Checkpoint:** The backend is running, can authenticate to Graph, and has a public URL. The database schema is deployed. `Tenant_Customer` is fully onboarded.

---

### Phase 1: Thread Tracking & Historical Ingestion (Week 2)

**Goal:** Implement the first critical user flow: a user clicks "Track this Thread" and the system ingests the historical conversation.

| Task | Details | Deliverable |
| :--- | :--- | :--- |
| **1.1** Implement `GraphMailAdapter` | Methods: `list_messages_by_conversation(user_id, conversation_id)`, `get_message(user_id, message_id)`, `list_attachments(user_id, message_id)`, `get_attachment(user_id, message_id, attachment_id)`. | Working adapter that can read emails and attachments from `Tenant_Customer`. |
| **1.2** Implement Tracking module | `POST /api/threads/track` endpoint. Accepts `{ user_id, conversation_id, deal_name }`. Creates a Deal, a TrackedThread, fetches historical messages, and enqueues them for processing. | Endpoint that creates a deal and begins historical ingestion. |
| **1.3** Implement `AzureOpenAIAdapter` | Methods: `analyze_email(email_body, contract_context)` returning structured JSON (classification, summary, entities, risk_level, deviation_detected). `generate_embedding(text)` returning a vector. | Working adapter that returns structured AI analysis. |
| **1.4** Implement Analysis module | The email processing pipeline: receive email text → build prompt → call LLM → parse structured JSON → return analysis result. Use a **Strategy pattern** for prompt templates so deviation detection logic can be swapped or tuned. | Pipeline that takes raw email text and returns structured metadata. |
| **1.5** Implement Ingestion worker | The async worker that dequeues jobs, calls the Analysis module, stores `email_metadata` records, generates embeddings, and stores them in the `embeddings` table. | Worker that processes emails end-to-end. |
| **1.6** Implement `SharePointAdapter` | Methods: `upload_file(site_id, folder_path, filename, content)`, `list_files(site_id, folder_path)`. | Working adapter that can upload files to the customer's SharePoint. |
| **1.7** Implement Artifact module | During historical ingestion, extract attachments, upload them to SharePoint under a deal-specific folder (e.g., `/AIVIRA/DealName/`), and store the reference in the `artifacts` table. | Artifacts from tracked threads are stored in SharePoint with references in the database. |
| **1.8** Create Graph subscription | When a thread is tracked, create a webhook subscription for `created` events on `/users/{user_id}/messages`. Implement the validation endpoint (`POST /api/graph-webhook`). | Real-time monitoring is active for the tracked user's mailbox. |

**Checkpoint:** A developer can call `POST /api/threads/track` with a real conversation ID from `Tenant_Customer`, and the system will ingest the full history, analyze each email, store metadata and embeddings, extract artifacts to SharePoint, and begin monitoring for new emails.

---

### Phase 2: Real-Time Detection & Issue Management (Week 3)

**Goal:** Implement the second critical flow: when a new email arrives in a tracked thread, the system automatically detects scope deviations and registers issues.

| Task | Details | Deliverable |
| :--- | :--- | :--- |
| **2.1** Implement webhook handler | `POST /api/graph-webhook` receives the notification. If using rich notifications, decrypt the payload. Otherwise, extract the `message_id` and `user_id`. Check if the email belongs to a tracked thread (match `conversationId`). If yes, enqueue a processing job. If no, discard. | Webhook correctly filters and enqueues only relevant emails. |
| **2.2** Implement Issue Detection module | After AI analysis, if `deviation_detected == true` or `risk_level >= "high"`, automatically create an Issue record linked to the email metadata and the deal. Set initial status to `Open`. | Issues are automatically created from AI analysis results. |
| **2.3** Implement Escalation logic | `POST /api/issues/{id}/escalate` endpoint. Changes issue status to `Escalated`. Sends a notification email to the configured CFO/decision-maker using `Mail.Send`. | Issues can be escalated with email notification. |
| **2.4** Implement Decision endpoint | `POST /api/issues/{id}/decision` endpoint. Accepts `{ decision: "approve" | "reject" | "defer", rationale, decided_by }`. Updates the issue record. | CFO can record decisions against issues. |
| **2.5** Implement subscription renewal | A scheduled task (cron or background loop) that renews Graph subscriptions before they expire (every ~2 days). Handle lifecycle notifications for `subscriptionRemoved` and `missed` events. | Subscriptions never silently expire. |

**Checkpoint:** Send a real email to a tracked thread in `Tenant_Customer`. The webhook fires, the email is processed, a scope deviation is detected, an Issue is created, and it can be escalated with a notification email.

---

### Phase 3: Deal Memory & RAG Chat (Week 4)

**Goal:** Implement the third critical flow: a user can ask questions about a deal and receive evidence-backed answers from the Deal Memory.

| Task | Details | Deliverable |
| :--- | :--- | :--- |
| **3.1** Implement Deal Timeline endpoint | `GET /api/deals/{id}/timeline` returns a chronological list of `email_metadata` records (summaries, classifications, entities, timestamps) and linked issues and artifact references. | API endpoint that powers the timeline view. |
| **3.2** Implement RAG Chat endpoint | `POST /api/deals/{id}/chat` accepts `{ question }`. The service: (1) generates an embedding for the question, (2) performs a pgvector similarity search against the deal's embeddings, (3) retrieves the top-k matching `email_metadata` records, (4) constructs a prompt with the question + retrieved context, (5) calls the LLM, (6) returns the answer with citations (references to specific email summaries). | Working RAG chat that answers questions grounded in deal data. |
| **3.3** Implement AI Draft Reply | `POST /api/deals/{id}/draft-reply` accepts `{ issue_id, user_id }`. The service: (1) retrieves the issue context and relevant deal memory, (2) generates a professional reply draft, (3) calls `GraphMailAdapter.create_draft_reply()` to place the draft in the user's mailbox. | AI-generated draft reply appears in the user's Outlook drafts. |

**Checkpoint:** A developer can query the chat endpoint with "What commitments have been made about delivery?" and receive an accurate, cited answer. A draft reply can be generated and found in the user's Outlook drafts folder.

---

### Phase 4: User Interfaces (Weeks 5-6)

**Goal:** Build the two user-facing interfaces that make the PoC demonstrable to external pilot users.

#### 4A: Outlook Add-in (Sidebar)

| Task | Details | Deliverable |
| :--- | :--- | :--- |
| **4A.1** Scaffold Outlook Add-in | React-based add-in using the Office.js SDK. Configure the manifest for a `MessageReadCommandSurface` (sidebar that opens when reading an email). | Running add-in that renders in Outlook's sidebar. |
| **4A.2** Implement "Track this Thread" | Read the current email's `conversationId` via `Office.context.mailbox.item`. On button click, call `POST /api/threads/track`. Display confirmation. | User can initiate tracking from the sidebar. |
| **4A.3** Implement Issue View | When viewing an email in a tracked thread, the sidebar displays any associated issues (status, severity, AI summary). Provide "Escalate to CFO" button. | User sees real-time issue status in the sidebar. |
| **4A.4** Implement Deal Chat | A chat interface within the sidebar. User types a question, it calls `POST /api/deals/{id}/chat`, and displays the answer with citations. | Working chat experience within Outlook. |
| **4A.5** Implement "Insert AI Draft" | Button that calls `POST /api/deals/{id}/draft-reply`. On success, notifies the user that a draft has been created in their mailbox (or uses `Office.context.mailbox.item.body.setAsync` to insert text into the compose window if in compose mode). | AI-drafted reply is accessible to the user. |
| **4A.6** Deploy add-in to `Tenant_Customer` | Generate the manifest XML/JSON. The `Tenant_Customer` admin deploys it via Centralized Deployment to the PoC security group. | Add-in is live in Outlook for pilot users. |

#### 4B: AIVIRA Web Portal

| Task | Details | Deliverable |
| :--- | :--- | :--- |
| **4B.1** Scaffold Web Portal | React / Next.js application. Authentication via Entra ID (the user logs in with their M365 account). | Running web app with login. |
| **4B.2** Implement Deal List | Dashboard showing all Deals for the tenant, with summary stats (number of issues, risk level, last activity). | CFO can see all active deals at a glance. |
| **4B.3** Implement Deal Timeline View | Chronological view of a deal's history: AI summaries, detected issues, decisions, and artifact links. Calls `GET /api/deals/{id}/timeline`. | CFO can review the full progression of a deal. |
| **4B.4** Implement Issue Decision UI | For escalated issues: display the decision memo (AI summary, risk assessment, margin impact, precedent data). Provide "Approve / Reject / Defer" buttons that call `POST /api/issues/{id}/decision`. | CFO can make and record decisions. |
| **4B.5** Implement Artifact Links | Display artifact references as clickable links that open the file directly in the customer's SharePoint (the URL points to their tenant). | Users can access original documents without AIVIRA storing them. |

**Checkpoint:** A pilot user can open Outlook, see the AIVIRA sidebar, track a thread, view issues, chat with the deal memory, and generate a draft reply. A CFO can log into the web portal, review deal timelines, and make decisions on escalated issues.

---

### Phase 5: Demo Readiness & Hardening (Week 7)

**Goal:** Polish the PoC for external demonstration. Ensure reliability, error handling, and a smooth demo script.

| Task | Details | Deliverable |
| :--- | :--- | :--- |
| **5.1** Error handling & resilience | Implement retry logic for Graph API calls (rate limiting, transient failures). Graceful error messages in the sidebar and portal. Dead-letter queue for failed processing jobs. | System handles failures without crashing. |
| **5.2** Audit logging | Log all significant actions: thread tracked, email processed, issue created, decision made. Include tenant_id, user_id, timestamp, and action type. Store in a dedicated `audit_log` table. | Full audit trail for governance demonstration. |
| **5.3** Subscription health monitor | A dashboard or log view showing the status of all active Graph subscriptions, their expiry times, and renewal history. Alert if a subscription fails to renew. | Operational visibility for the PoC team. |
| **5.4** Demo data seeding | Create a realistic demo scenario: a deal with a multi-email thread containing a scope deviation, an attachment (SOW), and a follow-up requiring escalation. Pre-seed this into `Tenant_Customer`. | Ready-to-run demo script. |
| **5.5** Security review | Verify that no raw email bodies are persisted in the database. Verify that Graph tokens are encrypted at rest. Verify that the webhook endpoint validates signatures. Verify that RBAC scoping is active. | Security checklist passed. |
| **5.6** External access setup | Configure the Web Portal for external pilot users to access (either via Entra ID guest accounts in `Tenant_Customer`, or a separate auth mechanism). Ensure the Outlook Add-in works for the pilot user group. | Pilot users can access and use the system. |

**Checkpoint:** The PoC is ready for external demonstration. A scripted demo can be executed end-to-end without manual intervention or workarounds.

---

## 4. Alignment Verification: PoC vs. Feasibility Analysis (v3)

This section confirms that every architectural tenet from the v3 feasibility report is directly addressed in this implementation plan.

| v3 Tenet | PoC Implementation | Status |
| :--- | :--- | :--- |
| **"Process, Don't Store" for Emails** | The Ingestion worker processes email text in-memory via the AI Analysis module. Only the resulting `email_metadata` record (summary, classification, entities) is written to the database. The raw email body is never persisted. | **Aligned** |
| **Customer-Owned Artifact Storage** | The Artifact module uses the `SharePointAdapter` to upload files to the customer's SharePoint site via `Sites.Selected`. AIVIRA stores only a reference (URL + item ID) in the `artifacts` table. | **Aligned** |
| **User-Initiated Tracking** | The Tracking module is triggered by an explicit user action (`POST /api/threads/track`) from the Outlook Add-in. No emails are monitored without user intent. | **Aligned** |
| **Clear Trust Boundary** | Raw emails and files remain in `Tenant_Customer`. AIVIRA's database in `Tenant_Aivira` contains only metadata, summaries, embeddings, and artifact references. | **Aligned** |
| **Scoped Permissions (RBAC + Sites.Selected)** | Phase 0.2 explicitly configures RBAC for Applications (scoping mail access to a security group) and Sites.Selected (scoping SharePoint access to one site). | **Aligned** |
| **One-Time Client Setup (30-45 min)** | Phase 0.2 simulates this setup. The Identity module (Phase 0.5) handles the OAuth consent flow. | **Aligned** |
| **Webhook-Driven Real-Time Monitoring** | Phase 1.8 creates Graph subscriptions. Phase 2.1 implements the webhook handler. Phase 2.5 handles subscription renewal. | **Aligned** |
| **RAG-Powered Deal Memory Chat** | Phase 3.2 implements the full RAG pipeline: embedding generation, pgvector similarity search, context retrieval, and LLM-grounded answer generation. | **Aligned** |
| **CFO Escalation & Governance** | Phase 2.3 (escalation), Phase 2.4 (decision recording), and Phase 4B.4 (decision UI) implement the full governance flow. | **Aligned** |

---

## 5. Critical Flows to Validate

These are the four end-to-end scenarios that must work flawlessly for the external demo. Each maps directly to the PoC Overview and the v3 sequence diagrams.

### Flow 1: User-Initiated Thread Tracking

> **Scenario:** Bob opens an email thread in Outlook about "Project Alpha." He clicks the AIVIRA sidebar and presses "Track this Thread."

**Expected behavior:** The backend reads the full thread history, analyzes each email, extracts a SOW attachment to SharePoint, stores all metadata and embeddings, and creates a Graph subscription. The sidebar confirms "Thread tracked. 12 emails analyzed. 1 artifact stored."

**Modules exercised:** Tracking, Graph Integration, Ingestion, Analysis, Artifacts, Deal Memory.

### Flow 2: Automated Scope Deviation Detection

> **Scenario:** A client sends a new email in the tracked thread: "By the way, can you also handle the data migration? We assumed that was included."

**Expected behavior:** The Graph webhook fires. The backend processes the email, the AI detects a scope deviation ("data migration" is not in the SOW), and an Issue is automatically created with severity "High." The sidebar updates to show the new issue. Bob can escalate it to the CFO.

**Modules exercised:** Ingestion, Analysis, Issue Management, Notification.

### Flow 3: Building the Deal Timeline

> **Scenario:** Carol (CFO) logs into the AIVIRA Web Portal and opens the "Project Alpha" deal.

**Expected behavior:** The portal displays a chronological timeline of AI-generated summaries for each key email, with extracted decisions, risk levels, and links to artifacts in SharePoint. No raw email content is visible.

**Modules exercised:** Deal Memory (Timeline), Artifact References.

### Flow 4: Self-Serve RAG Chat

> **Scenario:** Bob opens the Deal Chat in the sidebar and asks: "What was the agreed-upon deadline for Phase 1?"

**Expected behavior:** The system searches the deal's vector store, retrieves the relevant email summary (e.g., "Email from Oct 15: Client confirmed Phase 1 delivery by March 31"), and returns an answer with a citation. Bob then clicks "Draft Reply" and an AI-generated response appears in his drafts.

**Modules exercised:** RAG/Chat Query, AI Analysis, Graph Integration (draft creation).

---

## 6. Technology Stack Summary

| Layer | Technology | Rationale |
| :--- | :--- | :--- |
| **Backend Framework** | Python (FastAPI) | Async-native, excellent for I/O-bound Graph API calls and LLM interactions. Strong typing with Pydantic. |
| **Database** | PostgreSQL + pgvector | Single database for both relational metadata and vector embeddings. Simplifies PoC operations. |
| **Job Queue** | Redis + a lightweight worker (e.g., `arq`, `rq`, or `celery`) | Decouples webhook receipt from processing. Enables retries and dead-letter handling. |
| **LLM** | Azure OpenAI (GPT-4o for analysis, `text-embedding-3-small` for embeddings) | Enterprise-grade, compliant, and available within Azure's trust boundary. |
| **Outlook Add-in** | React + Office.js | Modern web-based add-in framework. Renders in Outlook's sidebar. |
| **Web Portal** | React / Next.js | Fast to build, SSR for initial load, integrates with Entra ID for auth. |
| **Hosting** | Azure App Service or Azure Container Apps | Natural fit for an M365-integrated SaaS. Simplifies networking with Azure OpenAI and Entra ID. |

---

## 7. What to Avoid in the PoC

The following are explicitly out of scope for the PoC, as recommended in the architectural analysis. They represent unnecessary complexity that can be introduced post-validation.

| Avoid | Reason |
| :--- | :--- |
| Microservices | All PoC flows are tightly related. A modular monolith is simpler to deploy, debug, and iterate on. |
| Separate vector database | pgvector within PostgreSQL is sufficient for PoC-scale data. Migrate to a dedicated vector DB (e.g., Qdrant, Pinecone) only if pgvector becomes a bottleneck. |
| Complex event broker (Kafka, RabbitMQ) | Redis queue is sufficient for the PoC's throughput. A full message broker adds operational overhead without proportional benefit. |
| BPM / orchestration engine | The issue lifecycle is simple enough to model with state transitions in code. No need for a workflow engine. |
| Shared domain model between frontend and backend | The Outlook Add-in and Web Portal consume REST APIs. They do not need to share TypeScript types with the Python backend. |
