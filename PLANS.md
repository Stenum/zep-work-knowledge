# PLANS.md — Implementation Plan (Minimum Lovable Product)

This plan lays out how to implement a **Personal Work Memory Assistant** that ingests Ask’s day-to-day work signals (Teams, Outlook, Calendar + manual notes), stores memory in **Zep**, and uses an **LLM** to reason over that memory with a fast correction/validation loop.

The plan is written so a developer can execute it end-to-end with minimal invention.

---

## 1. What we are building

### Product in one sentence

A web app that acts as Ask’s daily cockpit: it continuously ingests Microsoft 365 activity into Zep and provides a chat assistant plus a “review & validate” loop to keep memory accurate.

### MLP scope (minimum lovable)

* Connect Microsoft 365 (Entra/OIDC)
* Enable/disable sources: Teams, Outlook, Calendar
* Ingest items continuously (webhooks + delta sync fallback)
* Chat that uses tool-calls to query/update Zep
* Daily Brief (one-click)
* Review & Validate page: list beliefs, Accept/Reject/Edit
* Manual notes ingestion
* Reliability: queue + retries + idempotency
* Operational basics: logging, health checks, subscription renewal

Non-scope (for now): attachments parsing, multi-user, long-term analytics dashboards, custom ontology/SHACL/OWL.

---

## 2. Architecture overview

### Components

1. **Next.js App (React + Node BFF)**

   * UI (React)
   * Server endpoints (Next.js route handlers) for chat/review/notes/webhooks
2. **Ingestion Worker (Node)**

   * Processes jobs from a queue
   * Fetches content from Microsoft Graph
   * Normalizes text
   * Ingests documents into Zep
   * Runs subscription renewal + delta sync schedules
3. **Zep (3rd party)**

   * Stores documents and derived memory (temporal facts/entities)
   * Provides query/update APIs
4. **LLM provider (3rd party)**

   * Reasoning + tool-calling

### Minimal internal state

* Redis for:

  * ingestion queue
  * idempotency keys
  * delta sync cursors
  * subscription ids + renewal metadata
  * optional “last daily brief timestamp”

---

## 3. Tech stack (implementation constraints)

### Web

* Next.js (App Router) + React + TypeScript
* Tailwind CSS + shadcn/ui (component library)
* TanStack Query for client data fetching/caching
* Zod for request/response validation (shared)

### Services

* Node (TypeScript)
* BullMQ + Redis for jobs
* pino for structured logs
* Vitest for unit tests
* Playwright for e2e tests (staging)

### Code organization rule

* Prefer many small components split into separate files.
* Prefer feature folders (`features/chat`, `features/review`, `features/connections`).

---

## 4. Milestones and execution order

Each milestone ends with concrete acceptance checks.

### Milestone A — Repo bootstrap and conventions

**Outcome:** a runnable Next.js app + worker skeleton with shared packages.

Implementation steps:

* Create monorepo (pnpm workspaces recommended).
* Add apps/web (Next.js + Tailwind + shadcn/ui).
* Add services/worker (Node TS entrypoint).
* Add packages/shared (types + zod schemas).
* Add packages/clients: graph, zep, llm.
* Add lint + format + typecheck scripts.

Acceptance checks:

* `pnpm dev` runs the web app.
* `pnpm worker` starts a no-op worker.
* `pnpm typecheck` passes.

---

### Milestone B — Auth + Connections UI (Microsoft Entra)

**Outcome:** Ask can sign in and toggle sources.

Implementation steps:

* Register Entra app with redirect URIs for local/staging/prod.
* Implement auth with a mature library (Auth.js/next-auth or equivalent) using OIDC.
* Implement secure server-side token storage/refresh (via auth lib hooks/callbacks).
* Build `/connections` page:

  * Show connected identity
  * Toggles for Teams/Outlook/Calendar
  * Disconnect action
* Persist toggles in Redis keyed by user id (simple JSON blob).

Acceptance checks:

* Ask can log in.
* Toggles persist across refresh.
* Toggling a source changes whether subscriptions/delta sync run for that source.

---

### Milestone C — Zep + LLM tool-calling chat (no ingestion yet)

**Outcome:** chat works end-to-end and can query/update Zep via tools.

Implementation steps:

* Implement Zep client wrapper:

  * ingestDocument(envelope)
  * queryMemory({ query, filters })
  * updateMemory(updatePayload)
* Implement LLM client wrapper:

  * chatCompletionWithTools({ messages, tools, toolHandlers })
* Define tool surface (minimal):

  * `memory_query`: search memory for facts/events/entities
  * `memory_update`: apply correction/verification/deprecation
  * `memory_review`: return “beliefs” for UI review
* Implement BFF endpoint `/api/chat`:

  * Validates input via Zod
  * Calls LLM with tools enabled
  * Executes tool calls by calling Zep
  * Returns assistant response
* Build `/chat` UI (small components):

  * ChatThread
  * ChatComposer
  * MessageBubble

Acceptance checks:

* Ask can chat and see responses.
* If Zep has any seeded docs, the assistant can retrieve them via tool calls.

---

### Milestone D — Queue + webhook receiver + worker plumbing

**Outcome:** system can accept Graph webhook calls and process jobs asynchronously.

Implementation steps:

* Provision Redis (local via docker-compose; managed in staging/prod).
* Add BullMQ queues: `ingestion`.
* Implement webhook route `/api/webhooks/graph`:

  * Handles Graph validation handshake
  * Verifies request signature if used
  * Enqueues a job with (sourceType, resourceId, tenant/user context)
  * Responds quickly
* Implement worker job processor:

  * Fetch job payload
  * Dispatch to source fetcher (Teams/Email/Calendar)
  * Map to Document Envelope
  * Idempotency check (Redis SETNX on `source:source_id`)
  * Ingest to Zep
  * Log outcome
* Implement retry policy:

  * transient Graph/Zep errors retry with exponential backoff
  * dead-letter after N attempts with log event

Acceptance checks:

* Webhook endpoint responds fast (< 1s typical).
* Worker processes a test job and ingests to Zep.

---

### Milestone E — Graph subscriptions + source fetchers + ingestion

**Outcome:** real Teams/Outlook/Calendar events ingest into Zep.

Implementation steps:

* Implement Graph client wrapper:

  * Authenticated client using stored tokens
  * Helpers:

    * fetchTeamsMessageById
    * fetchEmailById
    * fetchCalendarEventById
* Implement subscription manager in worker:

  * Create subscriptions for enabled sources
  * Renew before expiry
  * Store subscription ids + expiry in Redis
  * Handle renewal failures (log + retry)
* Implement HTML→text normalization:

  * Convert message bodies to readable text
  * Preserve URLs where possible
* Implement envelope mapping per source:

  * Teams envelope: chat/thread/channel info, sender, participants
  * Email envelope: subject, from/to/cc, conversationId
  * Calendar envelope: title, time range, attendees, location
* Ingest into Zep using the envelope format.

Acceptance checks:

* Sending Ask a Teams message results in a new Zep document.
* Receiving/sending an email results in a new Zep document.
* Creating/updating a calendar event results in a new Zep document.

---

### Milestone F — Delta sync fallback + bootstrap backfill

**Outcome:** missed events are eventually ingested.

Implementation steps:

* Implement per-source delta sync loop in worker:

  * Maintain per-source cursor in Redis
  * Query Graph delta endpoints (where available)
  * Enqueue ingestion jobs for changed items
* Implement bootstrap backfill:

  * On first connect, backfill last N days (configurable per source)
  * Throttle to avoid rate limits
* Add scheduler:

  * Run delta sync every 10–30 minutes

Acceptance checks:

* If webhooks are disabled temporarily, delta sync still ingests new items.
* Backfill ingests a recent window without hitting hard failures.

---

### Milestone G — Review & Validate UX (lovable correctness loop)

**Outcome:** Ask can inspect and correct beliefs quickly.

Implementation steps:

* Define a UI-facing “Belief” type in `packages/shared`:

  * id
  * statement (human-readable)
  * confidence
  * lastUpdated
  * evidenceLinks (deep links/snippets)
* Implement BFF endpoints:

  * GET `/api/review?query=...`
  * POST `/api/review/accept`
  * POST `/api/review/reject`
  * POST `/api/review/edit`
* Implement UI page `/review`:

  * Search input
  * BeliefList
  * BeliefRow with Accept/Reject
  * BeliefEditorDialog for edits
  * Optional filters: source/time window
* Implement mapping to Zep updates:

  * Accept → “verified” (use Zep-supported mechanism)
  * Reject → “deprecated/invalid”
  * Edit → “correction/supersede”

Acceptance checks:

* Review returns a list of beliefs grounded in memory.
* Accept/Reject/Edit changes what the assistant answers in subsequent chat.

---

### Milestone H — Daily Brief (the “start my day” button)

**Outcome:** one-click brief grounded in the last 24–48h.

Implementation steps:

* Add a `/api/brief` endpoint (or a chat tool preset) that:

  * queries Zep for recent items (time window)
  * asks LLM to produce:

    * top themes
    * decisions
    * follow-ups
    * risks/blockers
  * includes evidence links/snippets where possible
* Add UI: DailyBriefCard and “Generate brief” button in chat.
* Store “last brief time” in Redis to support “since last brief” mode.

Acceptance checks:

* Brief is consistently useful and references recent sources.

---

### Milestone I — Hardening, testing, and runbook

**Outcome:** safe to run daily without babysitting.

Implementation steps:

* Security hygiene:

  * never log tokens
  * redact sensitive content in logs by default
  * confirm Graph scopes are minimal
* Rate limiting handling:

  * respect Graph 429 retry-after
  * add backoff
* Observability:

  * structured logs for ingestion outcomes
  * dead-letter counts
  * subscription renewal failures
* Tests:

  * unit tests for envelope mapping and normalization
  * integration tests with mocked Graph and mocked Zep client
  * minimal e2e in staging for chat + review flows
* Write `RUNBOOK.md`:

  * rotate secrets
  * re-create subscriptions
  * replay dead-letter jobs

Acceptance checks:

* CI green.
* Worker can run for days with stable ingestion.

---

## 5. Concrete implementation tasks by subsystem

### 5.1 Web UI (Next.js)

Routes:

* `/chat`
* `/review`
* `/connections`

Component/file breakdown (example):

* `features/chat/components/ChatThread.tsx`
* `features/chat/components/ChatComposer.tsx`
* `features/chat/components/MessageBubble.tsx`
* `features/chat/components/DailyBriefButton.tsx`
* `features/review/components/BeliefList.tsx`
* `features/review/components/BeliefRow.tsx`
* `features/review/components/BeliefEditorDialog.tsx`
* `features/connections/components/SourceToggle.tsx`
* `features/connections/components/ConnectPanel.tsx`

Client data:

* Use TanStack Query hooks:

  * `useChat()` for mutations
  * `useReview(query)` for beliefs list
  * `useConnections()` for toggles

---

### 5.2 BFF endpoints (Next.js route handlers)

Endpoints:

* `POST /api/chat`
* `GET /api/review`
* `POST /api/review/accept`
* `POST /api/review/reject`
* `POST /api/review/edit`
* `POST /api/notes` (optional if embedded in chat)
* `POST /api/webhooks/graph`
* `GET /api/health`

Request/response validation:

* Define Zod schemas in `packages/shared/schemas/*`.

---

### 5.3 Worker (BullMQ)

Jobs:

* `ingest_teams_message`
* `ingest_email`
* `ingest_calendar_event`
* `delta_sync_teams`
* `delta_sync_email`
* `delta_sync_calendar`
* `renew_subscriptions`
* `bootstrap_backfill`

Operational policies:

* Retries: exponential backoff, capped attempts.
* Dead-letter: log structured event and keep payload for replay.
* Idempotency: `source:source_id` key in Redis.

---

## 6. Document Envelope (shared contract to Zep)

Envelope (JSON shape):

```json
{
  "source": "teams|outlook|calendar|manual",
  "source_id": "stable-id",
  "timestamp": "ISO-8601",
  "author": { "id": "...", "displayName": "..." },
  "participants": [{ "id": "...", "displayName": "..." }],
  "context": {
    "thread_id": "...",
    "channel_id": "...",
    "subject": "...",
    "link": "..."
  },
  "content": {
    "text": "normalized plain text",
    "html": "optional",
    "attachments": []
  }
}
```

Implementation notes:

* Always include `source_id` from Graph so ingestion is idempotent.
* Always include a deep link when Graph can provide it (helps “lovable”).

---

## 7. Assistant tool design (minimal but effective)

Tool: `memory_query`

* Input: `{ query: string, timeWindow?: string, sources?: string[] }`
* Output: list of memory items + evidence pointers

Tool: `memory_review`

* Input: `{ topic: string }`
* Output: beliefs in UI-friendly shape

Tool: `memory_update`

* Input: `{ action: "verify"|"reject"|"correct", beliefId?: string, payload: any }`
* Output: update result

Prompting policy (hard requirement):

* The assistant must prefer tool calls over guessing.
* If evidence is stale/weak, label it and ask minimal validation.

---

## 8. Deployment plan (simple)

Staging first:

* Deploy Next.js app (with server routes enabled).
* Deploy worker (Node process).
* Provision Redis.
* Configure secrets:

  * Graph client id/secret
  * Auth secret
  * Zep API key/base URL
  * LLM API key

Production:

* Same as staging, but using Ask’s real Entra app + tenant.

---

## 9. Definition of Done (MLP)

* Continuous ingestion from Teams, Outlook, Calendar with webhook + delta sync.
* Chat answers grounded in Zep memory.
* Review & Validate works and impacts future answers.
* Daily Brief provides strong day-start value.
* Next.js/React UI with shadcn/ui + Tailwind.
* UI implemented as small components split into separate files.
* Worker retries safely; rate limits handled.
* Runbook exists for common ops.
