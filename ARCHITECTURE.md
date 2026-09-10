# Katiba AI - Architecture Overview

This document describes the architecture of Katiba AI, a production RAG SaaS for
constitutional and civic intelligence in Kenya. The application source is private; this
overview is published so the design can be reviewed without exposing the codebase. Every
detail below reflects the real system.

---

## 1. System purpose

Katiba AI combines:
- Retrieval-Augmented Generation (RAG) over the Kenyan Constitution, with article-level citations
- Tier-gated access (WANANCHI / WAKILI / MCHANGANUZI)
- Keycloak-based OIDC authentication (Authorization Code + PKCE)
- Lago + Stripe + M-Pesa billing
- File attachment, Speech-to-Text, and Text-to-Speech features
- Real-time streaming chat with Server-Sent Events

---

## 2. Architecture diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                           USERS (Browsers)                          │
│                    React 18 / Vite SPA over HTTPS                   │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ REST + SSE streaming
┌──────────────────────────▼──────────────────────────────────────────┐
│                    CORE BACKEND API                                  │
│                 FastAPI · Python 3.11 · Port 8000                    │
│  Routers: /auth /chats /chats/events (SSE) /query (RAG)             │
│           /files /audio /entitlements /admin /health                │
│  Middleware: Security telemetry · CORS · Guardrails                 │
│              (Pre-Flight / Input / Retrieval / Output)              │
└────┬───────┬────────┬──────────┬──────────┬────────────────────────┘
     ▼       ▼        ▼          ▼          ▼
┌────────┐ ┌──────┐ ┌───────┐ ┌───────┐ ┌──────────────┐
│MongoDB │ │Redis │ │Milvus │ │ GCS   │ │  Keycloak    │
│chats,  │ │broker│ │vector │ │files, │ │  OIDC / JWKS │
│users,  │ │cache,│ │index  │ │audio  │ │              │
│files,  │ │pub/  │ │HNSW,  │ │       │ └──────────────┘
│audio   │ │sub   │ │COSINE │ └───────┘
└────────┘ └──┬───┘ └───────┘
              │ Celery tasks
              ▼
┌────────────────────────────────────────────────────────┐
│                  CELERY WORKERS                        │
│  Queues: priority · ingestion · media · default        │
│  ingestion: scan, extract, chunk_and_embed             │
│  media:     transcribe_audio, synthesize_speech        │
│  cleanup:   expired uploads, stale jobs, shares        │
│  celery-beat: cron scheduler                           │
└────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    BILLING LAYER                                     │
│  billing-service (FastAPI · Python 3.12 · Port 8001)               │
│  Routers: /plans /customers /checkout /subscriptions               │
│           /invoices /events /webhooks                              │
│  Keycloak (roles/SSO) · Lago (metering/subscriptions) ·            │
│  Stripe (cards) · Flutterwave (M-Pesa KE) · PostgreSQL · Redis     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│               EXTERNAL LLM PROVIDERS                                │
│  OpenAI-compatible API (configurable base URL)                     │
│  Chat: o4-mini (Tier 1) / o3 (Tier 2/3)                           │
│  Embeddings: text-embedding-3-small (1536 dim)                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           GOOGLE CLOUD PLATFORM (PRODUCTION)                        │
│  Cloud Run: api, billing-service, frontend                         │
│  GKE: milvus, keycloak (stateful)   ·   Memorystore: Redis         │
│  Cloud SQL: Keycloak PG, Lago PG    ·   GCS: uploads/audio/TTS     │
│  Cloud Speech-to-Text / Text-to-Speech · BigQuery: analytics       │
│  Secret Manager · Artifact Registry · Cloud Build CI/CD            │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Service inventory

| Service | Technology | Role |
|---|---|---|
| portal-frontend | React 18 + Vite + TypeScript | SPA user interface |
| core-backend (api) | FastAPI + Python 3.11 | Constitutional RAG API |
| celery-worker | Celery 5.3 | Background task processor |
| celery-beat | Celery Beat | Periodic task scheduler |
| flower | Flower | Celery monitoring UI |
| billing-service | FastAPI + Python 3.12 | Billing bridge (Lago + Stripe + M-Pesa) |
| mongodb | MongoDB 7 | Primary application database |
| redis | Redis 7.2 | Celery broker/backend, pub/sub, cache |
| milvus | Milvus 2.3 | Vector database (constitution embeddings) |
| keycloak | Keycloak 24 | OIDC identity provider |
| lago-api / lago-worker | Lago (Rails + Sidekiq) | Billing/metering engine |

---

## 4. Data flow: chat query (RAG pipeline)

```
User message in SPA → POST /api/query/constitution-stream
  1. Compliance check (guardrail middleware)
  2. Keycloak JWT validation (RS256 via JWKS)
  3. Tier extraction from realm_access.roles
  4. Retrieval: embed query → Milvus vector search
       Lane A: semantic similarity (COSINE, HNSW)
       Lane B: authority-weighted search
  5. Context assembly (evidence hierarchy: Chapter | Article | Clause)
  6. LLM streaming (async httpx → OpenAI-compatible API)
  7. Output guardrails (hallucination check, citation validation)
  8. Persist message to MongoDB
  9. Publish chat event to Redis channel chat_events:{user_id}
 10. SSE stream completes with session.completed (chat_id + message_id)
```

The core reliability idea: retrieval evidence is assembled by legal hierarchy and
authority level, and output guardrails validate that every cited article actually appears
in the retrieved evidence before the answer is returned.

---

## 5. Authentication flow (Keycloak PKCE)

```
SPA /login → Keycloak Authorization Code + PKCE (code_verifier / code_challenge)
Callback  → exchange code for access_token + refresh_token
API calls → Authorization: Bearer {access_token}
   Backend fetches JWKS from Keycloak, validates RS256 signature,
   checks issuer, extracts realm_access.roles → tier
SSE       → token passed as query param (EventSource cannot set headers)
Refresh   → Keycloak token endpoint (long-lived refresh tokens)
```

---

## 6. User tier system

| Tier | Keycloak role | LLM model | Files | Features |
|---|---|---|---|---|
| WANANCHI (T1) | wananchi | o4-mini | No | Basic constitutional Q&A |
| WAKILI (T2) | wakili | o3 | Yes (<= 10MB, 3/query) | + File attachments |
| MCHANGANUZI (T3) | mchanganuzi | o3 | Yes (<= 25MB, 10/query) | + Full research features |

Role precedence: mchanganuzi > wakili > wananchi.

---

## 7. Billing flow

```
Sign up / log in (Keycloak)
  → billing-service provisions a Lago customer, stores lago_customer_id on the user
Select plan at /pricing
  → billing-service creates a Stripe Checkout session
Stripe webhook (checkout.session.completed) → billing-service
  → creates the Lago subscription
  → updates the Keycloak role (wananchi → wakili / mchanganuzi)
  → sends a SendGrid confirmation email
Frontend reads tier from the JWT; the core backend enforces tier gates per request
```

M-Pesa payments (Kenya) are handled through Flutterwave alongside Stripe cards.

---

## 8. Background tasks (Celery)

```
Broker/Backend: Redis
Queues: priority (max-priority 10) · ingestion · media · default

ingestion_tasks: scan_file, extract_text, chunk_and_embed, process_file_pipeline
media_tasks:     transcribe_audio (GCP STT), synthesize_speech (GCP TTS)
cleanup_tasks:   expired uploads (hourly), stale jobs (6h),
                 soft-deleted chats (daily), expired chat shares (hourly)

Reliability: task_acks_late, prefetch_multiplier = 1,
             reject_on_worker_lost, max_retries = 3 with exponential backoff
```

---

## 9. Milvus vector schema

```
Collection: katiba_embeddings
  chunk_id          VARCHAR  (PK)
  source_key        VARCHAR
  source_type       VARCHAR   CONSTITUTION | CONTEXT
  is_binding        BOOL
  retrieval_allowed BOOL      access-control flag
  article_number    INT64
  authority_level   INT64     1-5 (higher = more authoritative)
  chunk_hash        VARCHAR   SHA-256 content hash
  embedding         FLOAT_VECTOR(1536)   text-embedding-3-small

Index: HNSW (M=16, efConstruction=200)   ·   Metric: COSINE
```

---

## 10. Security trust boundaries

```
EXTERNAL (untrusted)    Browser SPA, mobile clients, Stripe/Flutterwave webhooks
EDGE (validate all)     Core API: CORS + JWT + rate limiting + guardrails
                        Billing: JWT + webhook signature verification
INTERNAL (networked)    Workers → Redis, API → MongoDB/Milvus, billing → Keycloak/Lago
EXTERNAL SERVICES       API-key auth to LLM provider, SendGrid, Stripe, M-Pesa, GCP
```

All secrets are supplied through Google Secret Manager (production) or environment
variables (local); none are committed to source control.

---

## 11. Notes on design decisions

- **Two-lane retrieval.** Semantic similarity alone over-retrieves generic passages, so a
  second, authority-weighted lane biases toward higher-authority constitutional text.
- **Citation validation as an output guardrail.** The model is only trusted to phrase the
  answer; whether a cited article is real is checked against the retrieved evidence, not
  the model's memory.
- **Tiering enforced server-side.** The frontend reads the tier from the JWT for UX, but
  every gate (model, token budget, file limits) is enforced in the backend per request.
- **Decoupled billing service.** Billing runs as its own FastAPI service on its own
  network so a billing outage cannot take down constitutional Q&A.
