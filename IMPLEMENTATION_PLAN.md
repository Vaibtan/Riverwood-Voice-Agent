# Implementation Plan — Riverwood AI Voice Agent

A checklist-based implementation plan organized into 8 phases. Each task is atomic and verifiable. Complete tasks in order within each phase; phases should be completed sequentially.

---

## Phase 1: Monorepo Foundation & Database

**Goal:** Working monorepo with Docker services running, database schema pushed, seed data loaded, and a health-check endpoint returning data.

### 1.1 Root Monorepo Setup
- [ ] Create root `package.json` with pnpm workspace scripts (dev, build, lint, db:push, db:seed, docker:up/down)
- [ ] Create `pnpm-workspace.yaml` pointing to `apps/*` and `packages/*`
- [ ] Create `turbo.json` with build/dev/lint pipeline tasks
- [ ] Create root `.gitignore` (node_modules, dist, .env, .turbo, docker volumes, logs)

### 1.2 Shared Types Package
- [ ] Create `packages/shared/package.json` (name: `@riverwood/shared`)
- [ ] Create `packages/shared/tsconfig.json`
- [ ] Create `packages/shared/src/index.ts` with all shared enums:
  - PaymentStage, MilestonePhase, CampaignStatus, CampaignLeadStatus (`PENDING`, `QUEUED`, `DIALING`, `IN_CALL`, `COMPLETED`, `FAILED`, `SKIPPED_NO_CONSENT`, `SKIPPED_OPTED_OUT`, `CANCELLED`), SiteVisitIntent, CallStatus, MemoryType, ConsentStatus
- [ ] Add shared interfaces: WebSocket event types (WsCallStatusPayload, WsCallCompletedPayload, WsCampaignProgressPayload, WsSiteVisitPayload)
- [ ] Add shared interfaces: VAPI webhook types (VapiWebhookPayload, VapiCall, VapiToolCall, VapiMessage, VapiAnalysis, VapiArtifact)
- [ ] Add shared interfaces: API response types (PaginatedResponse, DashboardMetrics)
- [ ] Add shared interfaces: Chat types (ChatRequest, ChatResponse)

### 1.3 Docker Compose
- [ ] Create `docker-compose.yml` with postgres (pgvector/pgvector:pg16) and redis (redis:7-alpine) services
- [ ] Create `docker/postgres/init.sql` — `CREATE EXTENSION IF NOT EXISTS vector`
- [ ] Create `docker/redis/redis.conf` — maxmemory 256mb, appendonly yes
- [ ] Verify: `docker compose up -d` starts both services, `docker compose ps` shows healthy

### 1.4 NestJS API Scaffold
- [ ] Create `apps/api/package.json` with all dependencies (NestJS, Prisma, BullMQ, Socket.io, OpenAI, Langfuse, Joi, class-validator, ioredis)
- [ ] Create `apps/api/tsconfig.json` and `tsconfig.build.json`
- [ ] Create `apps/api/nest-cli.json`
- [ ] Create `apps/api/.env.example` with all env vars documented
- [ ] Create `apps/api/src/main.ts` — bootstrap NestJS app on PORT from config
- [ ] Create `apps/api/src/app.module.ts` — root module importing ConfigModule with Joi validation
- [ ] Create `apps/api/src/config/validation.ts` — Joi schema for all env vars (DATABASE_URL, REDIS_HOST/PORT, VAPI keys, OpenAI key, etc.)

### 1.5 Prisma Schema & Seed
- [ ] Create `apps/api/prisma/schema.prisma` with all 8 models:
  - **Lead:** phone + plotNumber compound unique (@@unique([phone, plotNumber])), phone indexed, name, plotArea, paymentStage, siteVisitIntent, preferredLang, consentStatus (default PENDING), consentDate, optedOut (default false), optedOutAt, quietHoursStart (default "09:00"), quietHoursEnd (default "20:00"), notes
  - **ConstructionMilestone:** phase (enum unique), title, titleHi, description, descriptionHi, completionPct, isActive, sortOrder
  - **Campaign:** name, status, totalLeads, completedLeads, failedLeads, skippedLeads, dialWindowStart (default "09:00"), dialWindowEnd (default "20:00"), timestamps
  - **CampaignLead:** campaignId+leadId unique, status (includes `SKIPPED_NO_CONSENT`, `SKIPPED_OPTED_OUT`, `CANCELLED`), attempts, vapiCallId, terminalReason, errorMessage
  - **CallLog:** vapiCallId unique, transcript (JSON), summary, structuredData (JSON), recordingUrl, costUsd, langfuseTraceId
  - **ConversationMemory:** embedding Unsupported("vector(1536)"), memoryType, content, relevance
  - **WebhookEvent:** eventKey unique, messageType, responseJson (JSON nullable), processedResultHash, processedAt
  - **ChatSession:** leadId, messages (JSON array), createdAt, updatedAt
- [ ] Create `apps/api/src/prisma/prisma.service.ts` — NestJS-managed Prisma client with onModuleInit/enableShutdownHooks
- [ ] Create `apps/api/src/prisma/prisma.module.ts` — global module exporting PrismaService
- [ ] Create `apps/api/prisma/seed.ts` with:
  - 25 mock leads with realistic Haryana profiles (Hindi/English names, +91 phones, varied payment stages, plot numbers A-1 to E-5, mixed preferredLang, all consentStatus: GRANTED, 2-3 with optedOut: true)
  - 10 construction milestones (phases 1-4 at 100%, phase 5 at 65% and isActive, phases 6-10 at 0%)
  - 3-4 pre-existing conversation memories for demo leads (with content, no embedding — embedding generated on first post-call pipeline run)
  - 1 sample completed campaign with mixed CampaignLead statuses
- [ ] Run: `pnpm install` from root
- [ ] Run: `docker compose up -d` to start Postgres + Redis
- [ ] Run: `pnpm db:push` to push schema to database
- [ ] Run: raw SQL to create IVFFlat vector index on conversation_memories.embedding
- [ ] Run: `pnpm db:seed` to load seed data

### 1.6 CRM Module & Verification
- [ ] Create `apps/api/src/modules/crm/crm.module.ts`, `crm.controller.ts`, `crm.service.ts`
  - `GET /api/leads` — returns paginated leads (query: page, limit, search, siteVisitIntent, paymentStage)
  - `GET /api/leads/:id` — returns single lead with memories
  - `PATCH /api/leads/:id` — update lead fields (with UpdateLeadDto validation)
  - `POST /api/leads/:id/opt-out` — mark lead as opted out
  - `POST /api/leads/:id/consent` — update consent status with timestamp
  - `GET /api/milestones` — returns all construction milestones ordered by sortOrder
- [ ] **Verify:** `curl http://localhost:3000/api/leads` returns 25 seeded leads
- [ ] **Verify:** `curl http://localhost:3000/api/milestones` returns 10 milestones with phase 5 active

---

## Phase 2: VAPI Webhook Core

**Goal:** Receive VAPI webhooks with idempotency, return a dynamic assistant config, persist call status and transcripts to database, emit WebSocket events.

### 2.1 Common Infrastructure
- [ ] Create `apps/api/src/common/filters/http-exception.filter.ts` — global exception filter with structured JSON error responses
- [ ] Create `apps/api/src/common/interceptors/logging.interceptor.ts` — request/response logging with duration
- [ ] Register both in `main.ts` as global providers

### 2.2 VAPI Module Setup
- [ ] Create `apps/api/src/modules/vapi/vapi.module.ts`
- [ ] Create `apps/api/src/modules/vapi/vapi.guard.ts` — `VapiWebhookGuard` that verifies `x-vapi-secret` header against `VAPI_WEBHOOK_SECRET` env var
- [ ] Create `apps/api/src/modules/vapi/vapi.controller.ts` — single `POST /api/vapi/webhook` endpoint with VapiWebhookGuard
- [ ] Create `apps/api/src/modules/vapi/vapi.service.ts` — routes webhook messages to appropriate handler by `message.type`, wraps non-idempotent handlers in dedup guard

### 2.3 Webhook Idempotency Guard
- [ ] Create `apps/api/src/modules/vapi/webhook-dedup.service.ts`:
  - `reserveEvent(eventKey: string, messageType: string): Promise<{ isDuplicate: boolean; responseJson?: unknown }>` — tries to insert into `webhook_events`; on unique violation reads existing `responseJson`
  - `storeEventResponse(eventKey: string, responseJson: unknown, processedResultHash: string): Promise<void>` — persists deterministic response payload for replay-safe retries
  - Event key derivation:
    - `end-of-call-report`: `"{callId}:end-of-call-report"`
    - `tool-calls`: `"{callId}:tool:{toolCallId}"` (per tool call)
    - `status-update`: no dedup — uses upsert with ordering guard instead
    - `assistant-request`: no dedup — stateless re-computation
- [ ] Wire into `vapi.service.ts`: call `reserveEvent()` before forwarding to `end-of-call-report` and `tool-calls`; for duplicate `tool-calls`, immediately return stored `responseJson`

### 2.4 Assistant Request Handler (Critical Path)
- [ ] Create `apps/api/src/modules/vapi/handlers/assistant-request.handler.ts`
- [ ] Implement safe lead resolution:
  - Outbound campaign calls: `call.metadata.leadId` is required
  - Optional phone fallback (future inbound support only): query by phone and use only when exactly one lead matches
  - If lead cannot be resolved or is ambiguous: return generic assistant config and disable CRM-mutating tools for that call
- [ ] Implement: parallel fetch — all from DB/Redis, NO external API calls:
  - Lead record (Prisma)
  - Active ConstructionMilestone (Prisma: `where: { isActive: true }`)
  - Memory context: Redis `GET memory:context:{leadId}`, fallback to `SELECT content, memory_type FROM conversation_memories WHERE lead_id = $1 ORDER BY created_at DESC LIMIT 5`
- [ ] Implement: build full system prompt with template variables:
  - `{{customerName}}`, `{{plotNumber}}`, `{{paymentStage}}`, `{{milestoneTitle}}`, `{{milestoneDescription}}`, `{{completionPct}}`, `{{memoryContext}}`
- [ ] Implement: write the production Hindi/English system prompt template (Riverwood context, personality, conversation flow, strict rules, opt-out handling)
- [ ] Implement: dynamic transcriber language from `lead.preferredLang`:
  - `"hi"` → `language: "hi"`, `"en"` → `language: "en"`, `"hinglish"` / default → `language: "multi"`
  - Include `keywords: ["Riverwood", "Kharkhoda", "DDJAY", "Maruti"]`
- [ ] Implement: return complete VAPI assistant config JSON:
  - `model` block (provider: openai, model: gpt-4o-mini, systemPrompt, temperature: 0.7)
  - `voice` block (provider: 11labs, voiceId from env)
  - `transcriber` block (provider: deepgram, model: nova-2, dynamic language, keywords)
  - `analysisPlan` block (summaryPrompt, structuredDataSchema with customerMood/topicsDiscussed/concerns/siteVisitInterest/callbackRequested, successEvaluationPrompt/rubric)
  - `tools` array (schedule_site_visit, log_callback_request, get_plot_details, opt_out_customer — defined as VAPI function tools with parameter schemas)
  - `firstMessage` language-aware by `preferredLang` (`hi` Hindi, `en` English, `hinglish/default` Hinglish)
  - `voicemailDetection` enabled
- [ ] **Verify:** Unit test — given a mock webhook payload with `metadata.leadId`, handler returns valid assistant config JSON with interpolated customer data in <200ms

### 2.5 Status Update Handler
- [ ] Create `apps/api/src/modules/vapi/handlers/status-update.handler.ts`
- [ ] Implement: map VAPI status strings to CallStatus enum
- [ ] Implement: status ordering guard — define allowed transitions (e.g., INITIATED→RINGING→IN_PROGRESS→COMPLETED, never backwards). Skip update if current DB status is "ahead" of incoming status
- [ ] Implement: upsert CallLog record (create on first status, update on subsequent)
- [ ] Implement: update CampaignLead status if call is part of a campaign (extract campaignId from call metadata)
- [ ] Implement: emit WebSocket event via RealtimeGateway (placeholder — gateway created in 2.7)

### 2.6 End-of-Call Report Handler
- [ ] Create `apps/api/src/modules/vapi/handlers/end-of-call-report.handler.ts`
- [ ] Implement: (protected by dedup guard from 2.3)
- [ ] Implement: update CallLog with transcript, summary, structuredData, recordingUrl, costUsd, endedReason, durationSeconds
- [ ] Implement: update CampaignLead to COMPLETED/FAILED (determine from endedReason)
- [ ] Implement: update Campaign completed/failed counts (atomic increment)
- [ ] Implement: emit WebSocket call:completed event
- [ ] Implement: enqueue `campaign:post-call` BullMQ job for memory generation + cache rebuild (placeholder — queue created in Phase 4)

### 2.7 WebSocket Gateway
- [ ] Create `apps/api/src/modules/realtime/realtime.module.ts`
- [ ] Create `apps/api/src/modules/realtime/realtime.gateway.ts` — Socket.io gateway with:
  - `handleConnection` / `handleDisconnect` logging
  - `@SubscribeMessage('join:campaign')` — join campaign room
  - `@SubscribeMessage('leave:campaign')` — leave campaign room
  - Public methods: `emitCallStatus()`, `emitCallCompleted()`, `emitCampaignProgress()`, `emitSiteVisitScheduled()`

### 2.8 Calls Module
- [ ] Create `apps/api/src/modules/calls/calls.module.ts`, `calls.controller.ts`, `calls.service.ts`
  - `GET /api/calls` — paginated call logs with lead name, filterable by status/date/campaignId
  - `GET /api/calls/:id` — single call with full transcript, summary, structured data
  - `GET /api/calls/active` — currently active calls (status IN_PROGRESS or RINGING)
  - `GET /api/dashboard/metrics` — DashboardMetrics aggregate (active calls, calls today, visits booked, queue depth)

### 2.9 Verification
- [ ] Start NestJS server
- [ ] Send mock `assistant-request` webhook via curl → confirm valid assistant config JSON returned with dynamic language, tools, personalized prompt
- [ ] Send mock `assistant-request` without `leadId` and with ambiguous phone match → confirm generic non-CRM-safe assistant returned (no mutating tools)
- [ ] Send mock `status-update` webhook → confirm CallLog created in DB
- [ ] Send same `status-update` webhook again → confirm idempotent (no duplicate, no status regression)
- [ ] Send mock `end-of-call-report` webhook → confirm transcript/summary persisted
- [ ] Send same `end-of-call-report` again → confirm dedup (WebhookEvent exists, processing skipped)
- [ ] **Verify:** WebSocket connection via Socket.io client → receive events on status update
- [ ] **Optional:** ngrok tunnel + VAPI test call → real end-to-end verification

---

## Phase 3: Semantic Memory

**Goal:** Two-tier memory system: async post-call embedding + cache write, fast cache-first retrieval on assistant-request.

### 3.1 Memory Module
- [ ] Create `apps/api/src/modules/memory/memory.module.ts`
- [ ] Create `apps/api/src/modules/memory/memory.types.ts` — RetrievedMemory interface
- [ ] Create `apps/api/src/modules/memory/memory.service.ts` with:

**Embedding generation (async only — never called on blocking path):**
  - `generateEmbedding(text: string): Promise<number[]>` — calls OpenAI text-embedding-3-small

**Memory storage (async):**
  - `storeMemory(leadId, callLogId, type, content): Promise<void>` — generates embedding + raw SQL INSERT (Prisma doesn't support vector natively)

**Memory retrieval — Tier 1 (blocking-path safe, <100ms):**
  - `getMemoryContext(leadId: string): Promise<string>`:
    1. Redis `GET memory:context:{leadId}` → if hit, return cached string
    2. Cache miss: `SELECT content, memory_type FROM conversation_memories WHERE lead_id = $1 ORDER BY created_at DESC LIMIT 5` → format and return
    3. No memories found: return empty string

**Memory retrieval — Tier 2 (async, full semantic search):**
  - `rebuildMemoryCache(leadId: string): Promise<void>`:
    1. Generate embedding for query text
    2. Run hybrid scoring SQL:
       ```sql
       SELECT *,
         (1 - (embedding <=> $query::vector)) * 0.6 +
         EXP(-0.1 * EXTRACT(EPOCH FROM (NOW() - created_at)) / 604800) * 0.3 +
         relevance * 0.1 AS score
       FROM conversation_memories
       WHERE lead_id = $leadId AND embedding IS NOT NULL
       ORDER BY score DESC
       LIMIT 5
       ```
    3. Format top-5 into context string
    4. Redis `SET memory:context:{leadId} <context> EX 604800` (7-day TTL)

**Formatting:**
  - `formatMemoriesForPrompt(memories): string` — formats retrieved memories into natural language context

### 3.2 Wire Memory into Assistant Request
- [ ] Update `assistant-request.handler.ts`: call `memoryService.getMemoryContext(leadId)` in parallel with Lead/Milestone fetch (Tier 1 — cache or recency, no external calls)
- [ ] Inject returned context string into system prompt `{{memoryContext}}` variable
- [ ] If context is empty, use a default like "This is the first interaction with this customer."

### 3.3 Post-Call Memory Creation + Cache Rebuild
- [ ] In `end-of-call-report.handler.ts`: after persisting call data, enqueue BullMQ `campaign:post-call` job with `{ leadId, callLogId, summary, structuredData }`
- [ ] In post-call processor (Phase 4): call `memoryService.storeMemory()` with:
  - Type: CALL_SUMMARY, content: VAPI summary
  - Type: CONCERN if structuredData.concerns is non-empty
  - Type: SITE_VISIT_INTENT if site visit was discussed
- [ ] After storing memories: call `memoryService.rebuildMemoryCache(leadId)` to run Tier 2 semantic search and update Redis cache

### 3.4 Verification
- [ ] Seed a lead with no memories → simulate end-of-call-report → verify memory record created with embedding, Redis cache populated
- [ ] Call `getMemoryContext()` for that lead → verify cache hit, returns formatted context
- [ ] Delete Redis cache → call `getMemoryContext()` again → verify recency fallback works
- [ ] Simulate a second call → verify assistant-request includes first call's context in the system prompt
- [ ] **Verify:** `SELECT * FROM conversation_memories WHERE lead_id = $id` returns rows with non-null embedding

---

## Phase 4: Campaign Orchestration

**Goal:** Create, launch, pause, and cancel calling campaigns. BullMQ handles concurrency-controlled outbound call initiation with compliance pre-flight checks.

### 4.1 BullMQ Setup
- [ ] Add BullModule.forRoot() to AppModule with Redis connection from config
- [ ] Register two queues: `campaign:outbound` and `campaign:post-call`

### 4.2 VAPI Outbound Call Service
- [ ] Create `apps/api/src/modules/vapi/vapi-outbound.service.ts`
- [ ] Implement: `initiateCall(leadId, campaignId, phoneNumber): Promise<string>` — calls VAPI REST API `POST /call` with:
  - `assistantId: null` (triggers assistant-request webhook)
  - `customer.number` (lead phone)
  - `phoneNumberId` (from env)
  - `metadata: { leadId, campaignId }`
  - Returns VAPI call ID

### 4.3 Campaigns Module
- [ ] Create `apps/api/src/modules/campaigns/campaigns.module.ts`
- [ ] Create `apps/api/src/modules/campaigns/campaigns.controller.ts`:
  - `POST /api/campaigns` — create campaign (name, description, leadIds[], dialWindowStart?, dialWindowEnd?)
  - `GET /api/campaigns` — list campaigns with progress stats
  - `GET /api/campaigns/:id` — single campaign with CampaignLead statuses
  - `POST /api/campaigns/:id/launch` — launch campaign (validate, queue all leads)
  - `POST /api/campaigns/:id/pause` — pause campaign
  - `POST /api/campaigns/:id/cancel` — cancel campaign
- [ ] Create `apps/api/src/modules/campaigns/dto/create-campaign.dto.ts` — validation with class-validator
- [ ] Create `apps/api/src/modules/campaigns/campaigns.service.ts`:
  - `createCampaign()` — create Campaign + CampaignLead records, set totalLeads
  - `launchCampaign()` — validate status is DRAFT/PAUSED, addBulk to BullMQ with staggered delays (30s between batches of 50), update status to IN_PROGRESS
  - `pauseCampaign()` — pause queue processing, update status to PAUSED
  - `cancelCampaign()` — remove pending jobs, update remaining PENDING CampaignLeads to `CANCELLED`, update status to CANCELLED

### 4.4 Campaign Processor (Outbound Queue)
- [ ] Create `apps/api/src/modules/campaigns/campaigns.processor.ts` — BullMQ processor:
  - Queue: `campaign:outbound`, concurrency: 50
  - **Compliance pre-flight check for each job:**
    1. Fetch lead → if `optedOut === true` → skip, mark CampaignLead `SKIPPED_OPTED_OUT`, increment `Campaign.skippedLeads`
    2. If `consentStatus !== 'GRANTED'` → skip, mark CampaignLead `SKIPPED_NO_CONSENT`, increment `Campaign.skippedLeads`
    3. Check current time (IST) against campaign `dialWindowStart`/`dialWindowEnd` AND lead `quietHoursStart`/`quietHoursEnd` → if outside window, re-queue with calculated delay
  - After pre-flight: update CampaignLead to DIALING → call `vapiOutboundService.initiateCall()` → update CampaignLead with vapiCallId
  - On failure: update CampaignLead to FAILED, increment attempts, store errorMessage
  - Circuit breaker: track consecutive failures in memory, open after 5 → pause processing for 60s → probe with single call → recover on success

### 4.5 Post-Call Processor
- [ ] Create `apps/api/src/modules/campaigns/post-call.processor.ts` — BullMQ processor:
  - Queue: `campaign:post-call`, concurrency: 10
  - Job data: `{ leadId, callLogId, summary, structuredData }`
  - Steps:
    1. Store memories via `memoryService.storeMemory()` (summary + any concerns/intents)
    2. Rebuild memory cache via `memoryService.rebuildMemoryCache(leadId)`
    3. Update Lead CRM fields from structuredData (e.g., siteVisitIntent if site visit discussed)
    4. Emit WebSocket events on completion

### 4.6 Campaign Progress Tracking
- [ ] In status-update handler: emit `campaign:progress` WebSocket event with updated counts when call is part of campaign
- [ ] In end-of-call-report handler: after updating campaign counts, check if all CampaignLeads are terminal (`COMPLETED`, `FAILED`, `SKIPPED_*`, `CANCELLED`) → if so, mark Campaign as COMPLETED with completedAt timestamp

### 4.7 Verification
- [ ] Create a campaign with 5 test leads via `POST /api/campaigns` (include 1 opted-out lead, 1 with no consent)
- [ ] Launch campaign via `POST /api/campaigns/:id/launch`
- [ ] **Verify:** BullMQ queue has 5 jobs
- [ ] **Verify:** Opted-out lead is immediately marked `SKIPPED_OPTED_OUT`
- [ ] **Verify:** No-consent lead is marked `SKIPPED_NO_CONSENT`
- [ ] **Verify:** If VAPI keys configured, remaining 3 calls are initiated and statuses update in DB
- [ ] **Verify:** Pause campaign → pending jobs stop processing
- [ ] **Verify:** Cancel campaign → remaining pending CampaignLeads marked `CANCELLED`

---

## Phase 5: Tool Calling

**Goal:** VAPI mid-call tool execution — agent can schedule site visits, log callbacks, fetch plot details, and process opt-outs during conversation.

### 5.1 Tool Call Handler
- [ ] Create `apps/api/src/modules/vapi/handlers/tool-call.handler.ts`
- [ ] Implement: dedup guard per tool call using `reserveEvent("{callId}:tool:{toolCallId}")` — if duplicate, return stored `responseJson`
- [ ] Implement dispatcher: route by `toolCall.function.name`
- [ ] Implement `schedule_site_visit` (params: preferredDate string, preferredTime string, notes? string):
  - Update Lead: siteVisitIntent = SCHEDULED, siteVisitDate = parsed date
  - Emit WebSocket `site-visit:scheduled` event
  - Return confirmation message (Hindi/English based on lead.preferredLang)
- [ ] Implement `log_callback_request` (params: reason string, urgency enum low/medium/high):
  - Store memory with type CALLBACK_REQUEST via memoryService
  - Update Lead notes
  - Return acknowledgment
- [ ] Implement `get_plot_details` (params: none):
  - Fetch Lead plot info from DB
  - Return formatted plot details (number, area, sector, "DDJAY Scheme", nearest landmark)
- [ ] Implement `opt_out_customer` (params: reason? string):
  - Update Lead: optedOut = true, optedOutAt = now()
  - Store memory with type PREFERENCE and content about opt-out reason
  - Return graceful goodbye message ("Bilkul, hum aapko aage call nahi karenge. Dhanyavaad.")

### 5.2 Wire into VAPI Controller
- [ ] Update `vapi.service.ts` to handle `tool-calls` message type
- [ ] Return tool results in VAPI-expected format: `{ results: [{ toolCallId, result }] }`
- [ ] After successful first execution of a tool call, persist the exact response payload via `storeEventResponse(eventKey, responseJson, processedResultHash)` for deterministic replay

### 5.3 Tool Definitions Consistency Check
- [ ] Verify tool definitions in assistant-request handler match the handler implementations:
  - `schedule_site_visit` params: preferredDate (string, required), preferredTime (string, required), notes (string, optional)
  - `log_callback_request` params: reason (string, required), urgency (string enum: low/medium/high, required)
  - `get_plot_details` params: (none)
  - `opt_out_customer` params: reason (string, optional)

### 5.4 CRM Integration
- [ ] Ensure `crm.service.ts` has `updateSiteVisitIntent()`, `updateLeadNotes()`, `optOutLead()` methods
- [ ] Tool handlers call CRM service methods (not raw Prisma) for consistency

### 5.5 Verification
- [ ] Send mock `tool-calls` webhook with `schedule_site_visit` → verify Lead.siteVisitIntent updated to SCHEDULED, siteVisitDate set
- [ ] Send same webhook again → verify dedup (WebhookEvent exists, stored `responseJson` replayed)
- [ ] Send mock `tool-calls` webhook with `opt_out_customer` → verify Lead.optedOut = true
- [ ] Send mock `tool-calls` webhook with `get_plot_details` → verify correct plot data returned
- [ ] **Verify:** WebSocket client receives `site-visit:scheduled` event
- [ ] **Optional:** During live VAPI call, say "main March 25 ko aana chahta hoon" → verify site visit saved
- [ ] **Optional:** During live VAPI call, say "mujhe call mat karo" → verify opt-out processed

---

## Phase 6: Text Chat Channel

**Goal:** Text-based interaction with the same AI persona, accessible via API and dashboard chat panel.

### 6.1 Chat Module
- [ ] Create `apps/api/src/modules/chat/chat.module.ts`
- [ ] Create `apps/api/src/modules/chat/chat.service.ts`:
  - `startSession(leadId: string): Promise<ChatSession>` — creates a new ChatSession record
  - `sendMessage(sessionId: string, message: string): Promise<string>`:
    1. Fetch ChatSession with message history
    2. Fetch Lead record
    3. Fetch active ConstructionMilestone
    4. Fetch memory context via `memoryService.getMemoryContext(leadId)` (Tier 1, cache-first)
    5. Build system prompt using same prompt builder as assistant-request (minus voice-specific instructions: no word count limit, no filler guidance)
    6. Call OpenAI GPT-4o-mini directly with system prompt + message history + new message
    7. Append user message + assistant response to ChatSession.messages
    8. Return assistant response text
  - `getSession(sessionId: string): Promise<ChatSession>` — fetch session with full history
  - `listSessions(leadId: string): Promise<ChatSession[]>` — list sessions for a lead
- [ ] Create `apps/api/src/modules/chat/dto/send-message.dto.ts` — validation (message: string, non-empty)
- [ ] Create `apps/api/src/modules/chat/chat.controller.ts`:
  - `POST /api/chat/sessions` — body: `{ leadId }` → create session, return sessionId
  - `POST /api/chat/sessions/:sessionId/messages` — body: `{ message }` → send message, return response
  - `GET /api/chat/sessions/:sessionId` — get session with full message history
  - `GET /api/chat/sessions?leadId=xxx` — list sessions for a lead

### 6.2 Shared Prompt Builder
- [ ] Extract prompt building logic from `assistant-request.handler.ts` into a shared `prompt-builder.service.ts` (or a shared method in vapi.service)
- [ ] Voice prompt variant: includes word count limit, filler guidance, conversation flow timing
- [ ] Text prompt variant: relaxed word count, no filler/timing instructions, allows slightly longer responses
- [ ] Both handlers (assistant-request + chat) use the same builder with a `channel: 'voice' | 'text'` parameter

### 6.3 Post-Session Memory
- [ ] After a chat session ends (or after N messages), generate a memory from the conversation
- [ ] Ensure text conversations also contribute to cross-channel memory

### 6.4 Verification
- [ ] `POST /api/chat/sessions` with a seeded leadId → session created
- [ ] `POST /api/chat/sessions/:id/messages` with "Hello, mera plot ka kya update hai?" → AI responds with personalized construction update for that lead
- [ ] Send follow-up message → AI maintains conversation context from session history
- [ ] **Verify:** Response includes the same Riverwood personality and knowledge as voice calls
- [ ] **Verify:** Memory context is injected (if lead has prior call memories, they appear in chat responses)

---

## Phase 7: React Admin Dashboard

**Goal:** Real-time admin dashboard with live call monitoring, campaign management, CRM lead table, call detail view, and text chat panel.

### 7.1 React App Scaffold
- [ ] Create `apps/web/package.json` with dependencies (React 18, Vite, TailwindCSS, TanStack Query, Socket.io-client, React Router, Axios, Lucide icons)
- [ ] Create `apps/web/vite.config.ts` with API proxy to `http://localhost:3000`
- [ ] Create `apps/web/tsconfig.json`
- [ ] Create `apps/web/tailwind.config.js` and `postcss.config.js`
- [ ] Create `apps/web/index.html`
- [ ] Create `apps/web/src/main.tsx` — React root with QueryClientProvider and BrowserRouter
- [ ] Create `apps/web/src/index.css` — Tailwind directives
- [ ] Create `apps/web/src/App.tsx` — layout with sidebar nav + route outlet

### 7.2 API & WebSocket Client
- [ ] Create `apps/web/src/lib/api.ts` — Axios instance with baseURL `/api`
- [ ] Create `apps/web/src/lib/queryClient.ts` — TanStack QueryClient config
- [ ] Create `apps/web/src/hooks/useWebSocket.ts` — Socket.io hook:
  - Connect on mount, disconnect on unmount
  - `joinCampaign(id)` / `leaveCampaign(id)` methods
  - Event listeners for all WsEvent types
  - Return reactive state for active calls, campaign progress

### 7.3 Dashboard Page
- [ ] Create `apps/web/src/pages/Dashboard.tsx`:
  - MetricsBar component: active calls, queue depth, calls today, visits booked (from `GET /api/dashboard/metrics`)
  - LiveCallGrid: cards for each active call showing lead name, status, duration timer
  - Recent completions list: last 10 completed calls with summary
  - Auto-refresh via TanStack Query + WebSocket updates
- [ ] Create `apps/web/src/components/MetricsBar.tsx`
- [ ] Create `apps/web/src/components/LiveCallCard.tsx`

### 7.4 Campaigns Page
- [ ] Create `apps/web/src/pages/Campaigns.tsx`:
  - Campaign list with status badges, progress bars, lead counts
  - "New Campaign" button → CampaignWizard modal
  - Launch/Pause/Cancel action buttons per campaign
- [ ] Create `apps/web/src/components/CampaignWizard.tsx`:
  - Step 1: Name + description + dial window
  - Step 2: Select leads (checkbox table with search/filter, shows consent/opt-out status)
  - Step 3: Review + launch
- [ ] Create `apps/web/src/hooks/useCampaigns.ts` — TanStack Query hooks for campaigns CRUD

### 7.5 Leads Page
- [ ] Create `apps/web/src/pages/Leads.tsx`:
  - Paginated table with columns: name, phone, plot, payment stage, site visit intent, consent status, opted out, last call date
  - Search by name/phone
  - Filter by site visit intent, payment stage, consent status
  - Click row → navigate to lead detail (or expand inline)
- [ ] Create `apps/web/src/hooks/useLeads.ts` — TanStack Query hooks for leads

### 7.6 Call Detail Page
- [ ] Create `apps/web/src/pages/CallDetail.tsx`:
  - Call metadata: lead name, duration, cost, status, recording player
  - Full transcript view with speaker labels (user/assistant) and timestamps
  - AI summary card
  - Structured data card: customer mood, topics discussed, concerns, site visit interest
  - Success evaluation badge
- [ ] Create `apps/web/src/components/CallTimeline.tsx` — transcript rendered as chat bubbles
- [ ] Create `apps/web/src/hooks/useCalls.ts` — TanStack Query hooks for calls

### 7.7 Chat Panel
- [ ] Create `apps/web/src/components/ChatPanel.tsx`:
  - Lead selector dropdown (search leads by name)
  - Chat message list (user messages right-aligned, AI left-aligned)
  - Text input with send button
  - "New Session" button to start fresh conversation
  - Shows lead context (name, plot, payment stage) in header
  - Uses `POST /api/chat/sessions/:id/messages` for each message
- [ ] Create `apps/web/src/hooks/useChat.ts` — TanStack Query mutations for chat API
- [ ] Integrate ChatPanel into Dashboard page as a sidebar/modal or as a standalone route

### 7.8 Layout & Navigation
- [ ] Create sidebar component with nav links: Dashboard, Campaigns, Leads, Chat
- [ ] Add Riverwood branding/logo
- [ ] Responsive layout (collapsible sidebar on mobile)

### 7.9 Verification
- [ ] `pnpm --filter web dev` starts Vite dev server
- [ ] Dashboard page loads and shows metrics from API
- [ ] Leads page shows 25 seeded leads with pagination, consent/opt-out columns visible
- [ ] Campaign page can create and list campaigns
- [ ] Chat panel: select a lead, send a message, receive AI response with Riverwood persona
- [ ] **Verify:** WebSocket connection established — live call cards appear on status updates
- [ ] **Verify:** Call detail page renders transcript correctly

---

## Phase 8: Observability, Polish & Demo

**Goal:** Langfuse tracing, error handling, Bull Board, and demo-ready state.

### 8.1 Langfuse Integration
- [ ] Create `apps/api/src/modules/observability/observability.module.ts`
- [ ] Create `apps/api/src/modules/observability/langfuse.service.ts`:
  - Initialize Langfuse client from env vars
  - `startCallTrace(callId, leadId)` — create trace with metadata
  - `addGeneration(traceId, input, output, model, tokens)` — log LLM interaction
  - `endCallTrace(traceId, metadata)` — finalize trace with duration, cost
- [ ] Wire into assistant-request handler: start trace, log prompt construction
- [ ] Wire into end-of-call-report handler: finalize trace with cost/duration
- [ ] Wire into chat service: trace each chat message as a generation
- [ ] Store `langfuseTraceId` in CallLog record

### 8.2 Bull Board
- [ ] Install `@bull-board/api` and `@bull-board/express`
- [ ] Mount Bull Board at `/admin/queues` in main.ts
- [ ] **Verify:** Navigate to `http://localhost:3000/admin/queues` → see campaign queues

### 8.3 Error Handling Polish
- [ ] Ensure HttpExceptionFilter catches all errors and returns structured JSON
- [ ] Add request validation DTOs (class-validator) for all POST/PATCH endpoints:
  - CreateCampaignDto, UpdateLeadDto, SendMessageDto, ConsentDto
- [ ] Add Prisma error handling (unique constraint violations → 409, not found → 404)

### 8.4 Dockerfile
- [ ] Create `apps/api/Dockerfile` — multi-stage build (builder → production)
- [ ] Add `api` service to docker-compose.yml (optional — for containerized deployment)

### 8.5 Expanded Seed Data
- [ ] Ensure seed data includes:
  - 25 leads with diverse profiles (varied consent/opt-out states)
  - At least 3 leads with pre-existing CallLog + ConversationMemory records (to demo memory retrieval)
  - Pre-populated Redis memory cache for demo leads (via seed script)
  - 10 milestones with realistic dates
  - 1 sample completed campaign with mixed statuses for dashboard demo

### 8.6 Demo Preparation
- [ ] Write demo script (ordered steps to show in Loom recording):
  1. Show monorepo structure, docker services running
  2. Show Prisma Studio with seeded data
  3. Open Dashboard — show metrics, explain WebSocket integration
  4. Open Leads page — show 25 leads, show consent/opt-out columns, filter by site visit intent
  5. Open Chat panel — select a lead, have a text conversation demonstrating Hinglish persona and context awareness
  6. Create a campaign with 3-5 leads, show that opted-out leads are flagged, launch it
  7. Show live call cards appearing on Dashboard (if VAPI configured)
  8. Open a completed call detail — show transcript, AI summary, structured data
  9. Show Langfuse dashboard with call traces (if configured)
  10. Explain system prompt, assistant-request pattern, two-tier memory, webhook idempotency
  11. Briefly discuss 1000-call architecture (BullMQ, concurrency, circuit breaker, compliance)
- [ ] Test full flow end-to-end with VAPI (or mock webhooks if no API keys)

### 8.7 Final Verification Checklist
- [ ] `docker compose up -d` → Postgres + Redis healthy
- [ ] `pnpm install && pnpm db:push && pnpm db:seed` → schema pushed, 25 leads + 10 milestones + sample campaign seeded
- [ ] `pnpm dev` → API on :3000, Web on :5173
- [ ] `GET /api/leads` → 25 leads returned with consent/opt-out fields
- [ ] `GET /api/milestones` → 10 milestones, phase 5 active
- [ ] `POST /api/vapi/webhook` with mock assistant-request → valid config with personalized prompt, dynamic language, 4 tools, voicemailDetection enabled
- [ ] `POST /api/vapi/webhook` with mock end-of-call-report → CallLog + ConversationMemory created, Redis cache populated
- [ ] Duplicate end-of-call-report → dedup guard prevents reprocessing
- [ ] `POST /api/chat/sessions` + `POST .../messages` → AI responds with Riverwood persona
- [ ] WebSocket connects and receives events
- [ ] Dashboard, Campaigns, Leads, CallDetail, ChatPanel all render correctly
- [ ] Campaign create → opted-out/no-consent leads skipped → remaining calls initiated (or mock verified)
- [ ] Memory retrieval: second call to same lead includes first call context (via Redis cache)
- [ ] Bull Board accessible at `/admin/queues`
- [ ] Opt-out tool: mock tool-call webhook with opt_out_customer → lead marked opted out

---

## Architecture Quick Reference

```
                                    ┌───────────────────────┐
                                    │   React Dashboard     │
                                    │ (Vite + Socket + Chat)│
                                    └──────┬────────────────┘
                                           │ REST + WS
                                    ┌──────▼────────────────┐
                   VAPI Webhooks ──▶│   NestJS API          │
                                    │  ┌──────────────────┐ │
                                    │  │ VAPI Module      │ │◀── assistant-request (7.5s, DB+cache only)
                                    │  │  └ Dedup Guard   │ │◀── tool-calls (20s, dedup by toolCallId)
                                    │  │ Campaign Module  │ │◀── status-update (upsert + ordering)
                                    │  │  └ Compliance    │ │◀── end-of-call-report (dedup by callId)
                                    │  │ CRM Module       │ │
                                    │  │ Memory Module    │ │
                                    │  │  └ Redis Cache   │ │
                                    │  │ Chat Module      │ │◀── /api/chat/sessions/* (OpenAI direct)
                                    │  │ Realtime GW      │ │
                                    │  └──────────────────┘ │
                                    └──┬─────┬──────────────┘
                                       │     │
                              ┌────────▼┐ ┌──▼──────┐
                              │Postgres │ │  Redis  │
                              │pgvector │ │ BullMQ  │
                              │8 tables │ │ + Cache │
                              └─────────┘ └─────────┘
```

**Critical Path:** assistant-request handler must respond in <200ms with personalized config. Zero external API calls — DB + Redis only.

**Data flow:** VAPI manages calls → webhooks (idempotent) update our DB → post-call BullMQ job generates memories + rebuilds cache → WebSocket pushes to dashboard.

**Compliance flow:** Campaign launch → BullMQ job → pre-flight check (consent + opt-out + quiet hours) → dial or skip.
