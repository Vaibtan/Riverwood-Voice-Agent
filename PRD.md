# Product Requirements Document (PRD): Riverwood AI Voice Agent

## 1. Executive Summary

Riverwood Projects LLP is building an AI-powered CRM system for real estate. This PRD outlines the architecture, features, and technical strategy for the core component: a human-like AI voice agent capable of making outbound calls to plot buyers at Riverwood Estate (Sector 7, Kharkhoda, Haryana) with personalized construction updates in Hindi/English, maintaining conversational memory across calls, qualifying site visit intent, and providing a real-time admin dashboard.

This is a prototype for the Riverwood AI Systems Internship Challenge, designed with production-grade architecture to demonstrate senior-level engineering rigor.

---

## 2. Objectives & Goals

| Objective | Description |
|---|---|
| **Primary** | Deploy a low-latency, highly realistic AI voice agent that converses seamlessly in Hindi and English |
| **Scalability** | System must efficiently handle 1000+ outbound calls every morning |
| **User Experience** | Warm, conversational tone with < 1 second response latency |
| **Business Value** | Automate customer updates, qualify site visit intent, log interaction data into CRM |
| **Memory** | Maintain conversational context across multiple calls to the same customer |
| **Observability** | Full call tracing, admin dashboard with live monitoring |
| **Multi-channel** | Accept user input by both voice (phone calls) and text (dashboard chat) |

---

## 3. Tech Stack

### 3.1 Backend: NestJS (TypeScript)

- **Framework:** NestJS — scalable, strongly-typed, first-class WebSocket/queue support
- **Database:** PostgreSQL 16 with `pgvector` extension + Prisma ORM
- **Queueing:** Redis + BullMQ for campaign orchestration (concurrency-controlled outbound calling)
- **Caching:** Redis for memory context caching (pre-computed per-lead prompt context)
- **Real-time:** Socket.io WebSocket gateway for live dashboard updates

### 3.2 Frontend: React (TypeScript)

- **Build tool:** Vite
- **Styling:** TailwindCSS
- **Server-state:** TanStack Query (React Query)
- **Real-time:** Socket.io client for live call status updates

### 3.3 AI & Telephony Infrastructure

| Component | Choice | Rationale |
|---|---|---|
| **Orchestration** | VAPI | Low-latency wrapper around STT/LLM/TTS, sub-second conversational latency, webhook-driven |
| **LLM** | OpenAI GPT-4o-mini | Best speed/cost ratio for voice agents; also used directly for text chat channel |
| **TTS** | ElevenLabs | Industry leader for natural Hindi/English voices |
| **STT** | Deepgram nova-2 | Low-cost, high-accuracy, language dynamically set per lead's preferredLang |
| **Telephony** | Twilio Voice (via VAPI) | Reliable, BYOC to VAPI |
| **Embeddings** | OpenAI text-embedding-3-small | 1536 dims, 5x cheaper than ada-002, better quality |
| **Observability** | Langfuse | Open-source LLM tracing, per-call trace visibility |

### 3.4 Monorepo & DevOps

- **Monorepo:** pnpm workspaces + Turborepo
- **Containerization:** Docker Compose (Postgres + Redis for local dev)
- **Shared code:** `@riverwood/shared` package for TypeScript types/enums

---

## 4. Architecture Philosophy

1. **Async-first:** VAPI handles all real-time call state; our backend only reacts to webhook events and commands VAPI via REST
2. **Event-driven jobs:** BullMQ is the control plane for campaign orchestration. Every outbound call is a queue job with concurrency control
3. **Dynamic per-call config:** The `assistant-request` webhook pattern — instead of a static VAPI assistant, every call gets a fully customized assistant config with personalized prompt, memories, and CRM context
4. **Zero external calls on the blocking path:** The assistant-request handler (7.5s SLA) must never call external APIs (OpenAI, etc.). All enrichment data (memories, context) is pre-computed and cached
5. **Idempotent webhook processing:** Every webhook event is deduplicated via a `WebhookEvent` table to handle VAPI retries safely
6. **Compliance-aware dialing:** Consent tracking, quiet hours, and opt-out suppression baked into the campaign processor
7. **Observability by default:** Every call traced in Langfuse, every event persisted to Postgres, dashboard reads from WebSocket + REST

---

## 5. Core Features (MVP)

### 5.1 Multi-lingual Conversation
- Agent dynamically switches between Hindi, English, and Hinglish based on user responses
- Natural code-switching with warm fillers ("bilkul", "zaroor", "haan ji")
- Culturally aware — references local context (Maruti plant, DDJAY scheme, Kharkhoda)
- **Transcriber language set dynamically** from `Lead.preferredLang`:

| preferredLang | Deepgram language | Rationale |
|---|---|---|
| `"hi"` | `"hi"` | Pure Hindi |
| `"en"` | `"en"` | Pure English |
| `"hinglish"` | `"multi"` | Deepgram multilingual auto-detection for code-switching |
| (unset) | `"multi"` | Safe default |

- Transcriber config includes `keywords: ["Riverwood", "Kharkhoda", "DDJAY", "Maruti"]` for proper noun accuracy
- **First message language is also dynamic** from `preferredLang`:
  - `hi` → Hindi greeting
  - `en` → English greeting
  - `hinglish` / unset → Hinglish greeting

### 5.2 Contextual Awareness (Dynamic `assistant-request`)
**This is the architectural centerpiece.** Instead of creating a static assistant in VAPI dashboard, every call triggers an `assistant-request` webhook. Our server has 7.5 seconds to return a fully customized assistant config:

```
Outbound Call Initiated
  → VAPI sends assistant-request webhook
  → Our server: parallel fetch from DB + Redis only (~50-100ms total):
      • Lead record (Prisma)
      • Active ConstructionMilestone (Prisma)
      • Pre-computed memory context (Redis cache, fallback: recency query)
  → Build personalized system prompt
  → Return full assistant JSON config (voice, transcriber, model, tools, system prompt)
```

**Critical constraint:** No external API calls (OpenAI, etc.) on this path. All data comes from Postgres + Redis. Target response time: <200ms.

This enables:
- Customer name, plot number, payment stage interpolated into system prompt
- Past conversation context injected (from pre-computed cache)
- Current construction milestone description included
- Per-call tool definitions and transcriber language

### 5.3 Site Visit Qualification (VAPI Tool Calling)
Agent organically asks if customer wants to schedule a site visit and uses VAPI tool calling to:
- `schedule_site_visit` — captures preferred date/time, updates CRM
- `log_callback_request` — logs reason for callback, flags lead for follow-up
- `get_plot_details` — returns plot info on demand
- `opt_out_customer` — if customer requests no further calls, marks lead as opted out and ends call gracefully

### 5.4 Semantic Memory (pgvector / Two-Tier Retrieval)

Memory retrieval uses a two-tier architecture to keep the blocking path fast:

**Tier 1 — Assistant-request path (blocking, <100ms):**
1. Check Redis cache: `GET memory:context:{leadId}` → if hit, use pre-formatted context string (~5ms)
2. Cache miss fallback: simple recency query — `SELECT content, memory_type FROM conversation_memories WHERE lead_id = $1 ORDER BY created_at DESC LIMIT 5` (~10ms, no vector search)

**Tier 2 — Post-call pipeline (async, BullMQ job):**
1. Extract summary from VAPI `analysisPlan` structured output
2. Generate OpenAI embedding (text-embedding-3-small, 1536 dims)
3. INSERT into `ConversationMemory` table with embedding
4. Run full hybrid semantic search for this lead using pgvector:
   ```
   score = cosine_similarity × 0.6 + recency_decay × 0.3 + relevance × 0.1
   ```
5. Format top-5 results into a natural language context string
6. Cache to Redis: `SET memory:context:{leadId} <context> EX 604800` (7-day TTL)

**Trade-off:** First-ever call to a lead with existing memories but no cache gets recency-ranked (not semantically-ranked) memories. Acceptable because most leads have <10 memories where recency ≈ semantic order, and cache populates after every call.

### 5.5 Campaign Orchestration (BullMQ)
- **Queue:** `campaign:outbound` — one job per lead, concurrency=50, rate limit 100 jobs/min
- **Post-processing queue:** `campaign:post-call` — memory generation, CRM update, cache rebuild, concurrency=10
- **Campaign launch flow:**
  1. `POST /api/campaigns/:id/launch`
  2. Query all pending CampaignLeads
  3. **Compliance pre-flight** on each lead (see §5.7)
  4. `addBulk()` to BullMQ with staggered delays (30s between batches of 50)
  5. Each job calls VAPI REST API to initiate outbound call
- **Circuit breaker:** 5 consecutive failures → open for 60s → probe → recover
- **Campaign controls:** pause, resume, cancel

### 5.6 Text Chat Channel
To meet the challenge requirement of accepting both voice and text input, the system includes a session-based text chat API that reuses the same prompt builder and LLM:

- **`POST /api/chat/sessions`** — accepts `{ leadId }`, creates a chat session and returns `sessionId`
- **`POST /api/chat/sessions/:sessionId/messages`** — accepts `{ message }`, builds the same personalized prompt (same builder as assistant-request), calls OpenAI GPT-4o-mini directly (no VAPI), returns text response
- **`GET /api/chat/sessions/:sessionId`** — returns full conversation history
- **Session management:** `ChatSession` model stores message history per session. Messages included in LLM context for multi-turn coherence
- **Dashboard chat panel:** "Test Agent" panel in the React dashboard where admins/evaluators can type messages to any lead's agent persona and see responses. Demonstrates the same AI personality works via text
- **Memory integration:** Text chat sessions also generate post-session memories via the same pipeline, and the prompt includes the same cached memory context as voice calls

### 5.7 Outbound Compliance Controls
Bulk dialing requires consent tracking, quiet hours, and opt-out suppression:

- **Consent gate:** Only leads with `consentStatus = GRANTED` are dialed. Others are marked terminal with status `SKIPPED_NO_CONSENT`
- **Opt-out suppression:** Leads with `optedOut = true` are permanently suppressed and marked terminal with status `SKIPPED_OPTED_OUT`
- **Quiet hours:** Calls are only placed within the campaign's `dialWindowStart`–`dialWindowEnd` (default 09:00–20:00 IST). If a lead is reached outside their personal `quietHoursStart`–`quietHoursEnd`, the job is re-queued with delay until the window opens
- **Mid-call opt-out:** The `opt_out_customer` VAPI tool allows the agent to immediately honor "don't call me again" requests during a live call
- **Endpoints:** `POST /api/leads/:id/opt-out`, `POST /api/leads/:id/consent`

### 5.8 Real-time Admin Dashboard
- **Live call monitoring:** Active calls grid with per-call status cards
- **Metrics bar:** Active calls, queue depth, calls today, site visits booked
- **Campaign management:** Create/launch/pause campaigns, progress bars
- **CRM lead table:** Paginated, searchable, filterable by site visit intent
- **Call detail view:** Full transcript, AI summary, structured data, mood/topics/concerns
- **Text chat panel:** "Test Agent" interface for text-based interaction with any lead's agent persona
- **WebSocket:** Socket.io with room-based subscriptions per campaign

---

## 6. Database Schema

### Models

| Model | Purpose | Key Fields |
|---|---|---|
| `Lead` | CRM customer data | phone, plotNumber (compound unique), name, plotArea, paymentStage, siteVisitIntent, preferredLang, consentStatus, optedOut, quietHours |
| `ConstructionMilestone` | 10 construction phases | phase (enum unique), title, titleHi, description, descriptionHi, completionPct, isActive |
| `Campaign` | Batch calling campaigns | name, status, totalLeads, completedLeads, failedLeads, skippedLeads, dialWindowStart/End |
| `CampaignLead` | Join table: lead status within campaign | status (includes `SKIPPED_NO_CONSENT`, `SKIPPED_OPTED_OUT`, `CANCELLED`), attempts, vapiCallId, terminalReason |
| `CallLog` | Full call record | vapiCallId (unique), transcript (JSON), summary, structuredData (JSON), recordingUrl, costUsd |
| `ConversationMemory` | Semantic memory with vector embedding | content, embedding (vector(1536)), memoryType, relevance |
| `WebhookEvent` | Idempotency + replay-response table | eventKey (unique), messageType, responseJson (JSON nullable), processedResultHash, processedAt |
| `ChatSession` | Text chat conversation history | leadId, messages (JSON), createdAt, updatedAt |

### Key Design Decisions
- `Lead` unique on `[phone, plotNumber]` (not phone alone) — handles shared family numbers common in Indian real estate. Phone still indexed for fast lookup
- For outbound calls: `leadId` is mandatory in VAPI call metadata. If missing/invalid, the call is treated as non-CRM-safe (generic assistant only, no CRM-mutating tools)
- For phone-based fallback (future inbound support): only use if exactly one lead matches. If ambiguous, do not auto-attach to a lead
- `CallLog.transcript` and `structuredData` stored as JSON (VAPI output structure can evolve)
- `ConversationMemory.embedding` uses `Unsupported("vector(1536)")` in Prisma — IVFFlat index created via raw SQL migration
- `CampaignLead` unique on `[campaignId, leadId]` to prevent duplicate enrollments
- `WebhookEvent.eventKey` is a composite string (e.g., `"{callId}:{messageType}:{subkey}"`) for replay-safe deduplication, and `responseJson` enables deterministic replay responses for blocking events

---

## 7. VAPI Webhook Architecture

### Webhook Events

| Event | Blocking? | Timeout | Handler | Idempotency |
|---|---|---|---|---|
| `assistant-request` | Yes | 7.5s | Return dynamic assistant config | Re-compute (stateless, same inputs → same output) |
| `tool-calls` | Yes | 20s | Execute tools | Dedup by `{callId}:tool:{toolCallId}`, replay stored `responseJson` on retries |
| `status-update` | No | — | Update CallLog status, emit WS | Upsert with status ordering guard (never regress) |
| `end-of-call-report` | No | — | Persist transcript, enqueue post-call job | Dedup by `{callId}:end-of-call-report`, skip if already processed |

### Webhook Security
- `VapiWebhookGuard` — NestJS guard that verifies VAPI webhook shared secret from `x-vapi-secret` request header
- Secret stored in environment variable `VAPI_WEBHOOK_SECRET`

### Idempotency Protocol
All non-idempotent webhook handlers are wrapped with a dedup guard:
1. Derive `eventKey` from the webhook payload (call ID + message type + sub-key)
2. Attempt `INSERT INTO webhook_events (event_key, message_type, response_json) VALUES (..., ..., NULL)`
3. Unique constraint violation → webhook is a replay:
   - For blocking events (`tool-calls`): read and return stored `responseJson`
   - For non-blocking events (`end-of-call-report`): return early
4. Insert succeeds → process normally, then persist `responseJson` (for blocking events) + `processedResultHash`

**Status update ordering:** Status updates use upsert semantics, but include an ordering guard — a status-update for `IN_PROGRESS` is only applied if current status is `RINGING` or `INITIATED`, never if current status is `COMPLETED`. This prevents out-of-order webhook delivery from corrupting call state.

### VAPI `analysisPlan` (Structured Data Extraction)
Each assistant config includes an `analysisPlan` that instructs VAPI to extract:
- `summary` — 2-3 sentence call summary
- `structuredData` — JSON with: customerMood, topicsDiscussed[], concerns[], siteVisitInterest (boolean), callbackRequested (boolean)
- `successEvaluation` — "successful" / "partially-successful" / "unsuccessful"

---

## 8. System Prompt Design

The production-quality Hindi/English system prompt includes:

- **Riverwood Estate context:** Sector 7 Kharkhoda, DDJAY scheme, RERA registered, Maruti Suzuki plant proximity, 25-acre township
- **Customer context (injected per call):** `{{customerName}}`, `{{plotNumber}}`, `{{paymentStage}}`, `{{memoryContext}}`
- **Construction milestones:** 10 phases from Land Leveling to Handover Ready, with current active phase highlighted
- **Language instructions:** Natural Hinglish, code-switching rules, warm fillers ("bilkul", "zaroor", "haan ji")
- **Personality:** Warm and patient (not salesy), honest about timelines, family-oriented framing ("aapka ghar"), Maruti enthusiasm as talking point
- **Conversation flow:** GREET (10s) → UPDATE (30-45s) → ENGAGE (30-60s) → QUALIFY (30-60s) → CLOSE (15-20s)
- **Strict rules:** Never fabricate data, no competitor discussion, <40 words per response, no lists/bullet points in speech
- **Opt-out handling:** If customer requests no further calls, use `opt_out_customer` tool immediately and end with a respectful goodbye

The same prompt template is reused for the text chat channel (§5.6), minus voice-specific instructions (word count limit relaxed, no filler guidance).

---

## 9. Scalability Strategy: 1000 Calls/Morning

1. **Batch Processing:** Campaign launch enqueues all leads into BullMQ `campaign:outbound` queue
2. **Compliance Pre-flight:** Each job checks consent, opt-out, quiet hours before dialing (§5.7). Non-compliant leads are marked terminal (`SKIPPED_NO_CONSENT` / `SKIPPED_OPTED_OUT`)
3. **Concurrency Control:** BullMQ workers process with concurrency=50 (50 simultaneous active calls), rate limited to 100 jobs/min
4. **Staggered Dispatch:** Batches of 50 with 30s delays to avoid Twilio/VAPI rate limits
5. **Webhook Architecture:** Fire-and-forget call initiation via VAPI REST API; NestJS reacts asynchronously to webhook events, idempotently (§7)
6. **Circuit Breaker:** 5 consecutive failures → 60s cooldown → probe call → recover
7. **Retry Logic:** Failed calls get up to 2 retries with exponential backoff
8. **Voicemail Detection:** Enable VAPI `voicemailDetection` to hang up early on answering machines, reducing wasted cost

**Estimated throughput:** 50 concurrent × 1.5 min avg = ~33 calls complete/min → 1000 calls in ~30 minutes

---

## 10. Observability & Tracing

- **Langfuse integration:** Every call generates a trace with: system prompt, retrieved CRM context, tool calls, duration, cost
- **Admin dashboard:**
  - Live call status (connecting, in-progress, completed, failed)
  - Full transcript view with tool invocation timeline
  - Aggregated metrics (avg latency, call duration, success rates)
  - Error tracking (dropped calls, STT failures, LLM timeouts)
- **BullMQ monitoring:** Bull Board UI at `/admin/queues` for queue inspection
- **Structured logging:** NestJS `LoggingInterceptor` on all API endpoints

---

## 11. Seed Data Strategy

25 mock leads with realistic Haryana profiles:
- Hindi/English name mix (e.g., "Rajesh Kumar", "Sunita Devi", "Amit Sharma")
- E.164 phone numbers (+91 prefixed)
- Varied payment stages across all enum values
- Plot numbers matching Riverwood Estate layout (A-1 through E-5)
- Plot areas: 100, 125, 150, 200 sq yards
- Mixed `preferredLang`: hi, en, hinglish
- All leads seeded with `consentStatus: GRANTED` (mock buyers with signed booking agreements)
- 2-3 leads with `optedOut: true` to demonstrate suppression
- Some leads with pre-existing conversation memories and cached memory context (for demo)

10 construction milestones seeded with:
- Phase 1-4 completed (100%)
- Phase 5 active (65%)
- Phases 6-10 upcoming (0%)

1 sample completed campaign with mixed CampaignLead statuses for dashboard demo.

---

## 12. Cost Estimation (Per 1000 Call Targets)

### Sensitivity Analysis

| Scenario | Pickup Rate | Avg Connected Duration | Retry Attempts (No-Answer) | Connected Calls | Total Attempts | Estimated Cost |
|---|---|---|---|---|---|---|
| **Optimistic** | 80% | 0.8 min | 0× | 800 | 1000 | ~$98 |
| **Realistic** | 55% | 1.2 min | 1× | 550 | 1450 | ~$110 |
| **Conservative** | 40% | 1.4 min | 2× | 400 | 2200 | ~$118 |

### Cost Formula (Used for All Scenarios)

```
connected_minutes = connected_calls * avg_connected_duration_min
total_attempts = 1000 + ((1000 - connected_calls) * retry_attempts)

total_cost =
  (connected_minutes * connected_minute_rate) +
  (total_attempts * per_attempt_base_cost) +
  (connected_calls * (llm_cost_per_connected_call + embedding_cost_per_connected_call)) +
  infra_overhead
```

Where:
- `connected_minute_rate = 0.05 (VAPI) + 0.03 (TTS at 50% speech) + 0.0043 (STT) + 0.014 (Twilio) = $0.0983/min`
- `per_attempt_base_cost = 0.02 (VAPI base) + 0.006 (telephony setup/ring overhead) = $0.026/attempt`

### Cost Breakdown (Realistic Scenario)

| Component | Rate | Cost |
|---|---|---|
| VAPI Orchestration | $0.05/min × 660 connected min | ~$33.00 |
| VAPI base per-attempt | $0.02 × 1450 attempts | ~$29.00 |
| ElevenLabs TTS | $0.06/min × (50% of 660 min) | ~$19.80 |
| Deepgram STT | $0.0043/min × 660 min | ~$2.84 |
| OpenAI GPT-4o-mini | ~800 tokens × 550 connected calls | ~$4.53 |
| Twilio Voice | ($0.014/min × 660 min) + ($0.006 × 1450 attempts) | ~$17.94 |
| OpenAI Embeddings | ~1 embedding × 550 connected calls | ~$0.02 |
| Langfuse + Redis + Postgres | Infrastructure overhead | ~$2.50 |
| **Total (realistic)** | | **~$109.63 (~$110)** |

**Additional cost factors:**
- No-answer attempts still incur Twilio + VAPI base costs (~$0.03-0.05 per attempt)
- Voicemail without AMD: agent may run 30-60s before realizing → enable `voicemailDetection` to hang up early
- Monthly at 20 business days (realistic): ~$2,200 for 20,000 call targets

---

## 13. Key Architecture Decisions

| Decision | Choice | Rationale |
|---|---|---|
| `assistant-request` vs static assistant | Dynamic | Every call gets personalized prompt with memory — impossible with static |
| Embedding on blocking path? | No — two-tier cache | OpenAI API on 7.5s SLA is a timeout risk under load; cache-first + recency fallback eliminates it |
| BullMQ vs VAPI batch calling | BullMQ | Per-call customization, compliance pre-flight, fine-grained retry/pause, Bull Board visibility |
| IVFFlat vs HNSW vector index | IVFFlat | Faster build, lower memory for <1M rows (our scale: ~250K max) |
| Socket.io vs native WebSocket | Socket.io | NestJS first-class support, room system maps to campaign subscriptions |
| text-embedding-3-small vs ada-002 | 3-small | Better quality, 5x cheaper, same 1536 dimensions |
| Prisma vs TypeORM | Prisma | Better DX, type-safe queries, schema-as-code, but raw SQL for vector ops |
| pnpm + Turborepo | Monorepo | Shared types between API and web, single repo for challenge submission |
| Phone unique vs compound unique | `[phone, plotNumber]` | Handles shared family numbers common in Indian real estate |
| Lead identity on webhook | `leadId` required for CRM writes | Prevents wrong lead attachment when a phone maps to multiple family members |
| Text channel via OpenAI direct | Yes | Reuses prompt builder, no VAPI dependency, makes demo accessible without phone |
| Webhook idempotency | Dedup + replay-response table | VAPI can retry; prevents duplicate side-effects and supports deterministic response replay for blocking events |

---

## 14. Success Metrics (Evaluation Alignment)

| Criteria | Weight | How We Address It |
|---|---|---|
| **Voice Realism** | 25% | ElevenLabs Hindi/English voice, natural Hinglish system prompt with fillers and warmth |
| **Latency** | 20% | VAPI orchestration (~600-800ms), assistant-request handler <200ms (DB+cache only, zero external API calls), parallel fetches |
| **Infrastructure Design** | 20% | NestJS + BullMQ + pgvector + WebSocket + Docker + Turborepo monorepo + webhook idempotency + compliance controls |
| **Context Understanding** | 20% | pgvector semantic memory (two-tier retrieval), dynamic per-call prompt injection, VAPI analysisPlan structured extraction, text chat with same context |
| **Creativity** | 15% | Culturally aware Hinglish persona, Riverwood-specific knowledge, real-time admin dashboard, text chat panel for accessible demo |

---

## 15. Project Structure

```
riverwood-voice-agent/
├── apps/
│   ├── api/                                # NestJS backend
│   │   ├── src/
│   │   │   ├── main.ts
│   │   │   ├── app.module.ts
│   │   │   ├── modules/
│   │   │   │   ├── vapi/                   # VAPI webhook handler
│   │   │   │   │   ├── vapi.module.ts
│   │   │   │   │   ├── vapi.controller.ts
│   │   │   │   │   ├── vapi.service.ts
│   │   │   │   │   ├── vapi.guard.ts
│   │   │   │   │   ├── vapi-outbound.service.ts
│   │   │   │   │   ├── dto/
│   │   │   │   │   └── handlers/
│   │   │   │   │       ├── assistant-request.handler.ts
│   │   │   │   │       ├── tool-call.handler.ts
│   │   │   │   │       ├── status-update.handler.ts
│   │   │   │   │       └── end-of-call-report.handler.ts
│   │   │   │   ├── campaigns/              # BullMQ campaign orchestration
│   │   │   │   ├── crm/                    # Lead/customer CRUD + compliance
│   │   │   │   ├── memory/                 # pgvector semantic memory + Redis cache
│   │   │   │   ├── chat/                   # Text chat channel (OpenAI direct)
│   │   │   │   ├── calls/                  # Call log read API
│   │   │   │   ├── realtime/               # Socket.io WebSocket gateway
│   │   │   │   └── observability/          # Langfuse tracing
│   │   │   ├── common/                     # Guards, filters, interceptors
│   │   │   └── config/                     # Joi env validation
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   └── seed.ts
│   │   └── Dockerfile
│   └── web/                                # React admin dashboard
│       ├── src/
│       │   ├── pages/
│       │   │   ├── Dashboard.tsx
│       │   │   ├── Campaigns.tsx
│       │   │   ├── Leads.tsx
│       │   │   └── CallDetail.tsx
│       │   ├── components/
│       │   │   ├── ChatPanel.tsx            # Text chat "Test Agent" panel
│       │   │   └── ...
│       │   ├── hooks/
│       │   └── lib/
│       └── vite.config.ts
├── packages/
│   └── shared/                             # Shared TypeScript types/enums
├── docker/
│   ├── postgres/init.sql
│   └── redis/redis.conf
├── docker-compose.yml
├── turbo.json
├── pnpm-workspace.yaml
└── package.json
```
