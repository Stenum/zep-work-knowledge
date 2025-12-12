# PRD.md — Personal Work Memory Assistant (Zep + Microsoft Graph)

**Owner:** Ask Stenum  
**Audience:** Developers (no prior context assumed)  
**Status:** Draft for implementation  
**Primary goal:** Build a simple assistant that stays up-to-date about Ask’s day-to-day work context by ingesting Teams/Outlook/Calendar + manual notes, storing structured memory in Zep, and enabling corrections/validation via conversational UX.

---

## 1) Problem statement

Ask has a high volume of day-to-day work communication spread across:
- Microsoft Teams (chats, channel conversations)
- Outlook (email)
- Calendar (meetings)
- Additional unstructured inputs (manual notes, ad-hoc context)

The problem is not lack of information—it's **lack of a continuously updated “working model” of Ask’s world** that an assistant can use to:
- answer contextual questions (“what’s going on with X?”)
- generate actionable summaries (decisions, blockers, follow-ups)
- detect uncertainty/staleness and ask the right validation questions
- accept corrections quickly and keep memory consistent

The solution should be **minimal** and **leverage 3rd-party components**:
- Microsoft Graph for data access
- Zep for temporal structured memory (entities/facts/relationships over time)
- An LLM provider for reasoning + tool-calling

No ontology work. No semantic web. No heavy bespoke storage.

---

## 2) Goals

### 2.1 Product goals
1. **Up-to-date structured memory** of Ask’s work conversations and relevant entities (people/projects/topics/tasks/decisions).
2. **Useful assistant reasoning** grounded in that memory, with clear handling of uncertainty/staleness.
3. **Fast correction loop**: Ask can correct or validate facts conversationally and through UI actions.
4. **Minimal architecture**: build only glue (ingestion, orchestration, UI), delegate memory to Zep.

### 2.2 UX goals
1. Ask can use this as a daily “start here” cockpit:
   - “What changed since yesterday?”
   - “What should I follow up on?”
   - “What do you know about person/project X?”
2. “Review & validate” mode makes it easy to confirm or reject beliefs with minimal friction.

### 2.3 Engineering goals
1. **React + Next.js + Node BFF** for the app (frontend + backend-for-frontend).
2. Prefer mature libraries over bespoke code.
3. Small components split into separate files; avoid “mega components”.

---

## 3) Non-goals (explicit)

1. Building a canonical enterprise knowledge graph or ontology.
2. Replacing Microsoft tooling (Outlook/Teams/Calendar UI).
3. Full workflow/task system (Jira replacement).
4. Perfect truth / guaranteed correctness—system must surface uncertainty and seek validation.
5. Multi-tenant SaaS. This is single-user (Ask) unless later expanded.

---

## 4) Users & personas

### Primary user
- **Ask** (single-user system): wants a personal assistant that understands ongoing work context.

### Secondary (future)
- Other leaders in similar roles (not part of MVP).

---

## 5) Key user journeys

### J1 — Daily briefing
1. Ask opens the app in the morning.
2. App shows “Since last time” summary (new topics, decisions, follow-ups).
3. Ask asks: “What are the top risks for project X this week?”
4. Assistant answers with references to recent conversations.

### J2 — Investigate a person/topic
1. Ask asks: “What do you know about Adam’s current challenges?”
2. Assistant answers with structured bullets, confidence, and “last updated” markers.
3. Assistant flags stale items and asks 1–3 validation questions.

### J3 — Correct the assistant
1. Ask says: “Adam does not report to me anymore.”
2. Assistant applies correction to memory and confirms the new belief.

### J4 — Review & validate mode
1. Ask triggers: “Review what you know about DACM.”
2. UI lists beliefs (facts) with Accept/Reject.
3. Ask corrects one item, accepts others.
4. Memory updates immediately.

---

## 6) Proposed minimal architecture

### 6.1 Components (minimal)
1. **Next.js Web App**
   - React UI
   - Node BFF endpoints (Next.js route handlers / API routes)
2. **Ingestion Worker (Node)**
   - Handles Microsoft Graph webhook events + delta sync
   - Fetches content
   - Sends documents to Zep
3. **Zep (3rd party)**
   - Primary structured memory store (temporal KG / memory)
4. **LLM Provider (3rd party)**
   - Reasoning, summarization, tool calling

### 6.2 Why worker + BFF split?
- Webhooks and ingestion can be bursty.
- Keeping ingestion in a worker avoids blocking UI requests and simplifies retries.
- The BFF remains responsive and stateless.

### 6.3 Minimal internal storage
- Prefer **no database** for content.
- Use a **queue** (Redis) for ingestion reliability and retries.
- Store secrets in platform-managed secrets store.

---

## 7) Tech stack requirements

### 7.1 Frontend + BFF
- **Next.js (App Router)** + **React** + **TypeScript**
- **Tailwind CSS** + **shadcn/ui** (component library)
- **TanStack Query** for client data fetching/caching
- **Zod** for validation (shared types)
- Authentication: **Microsoft Entra ID (OIDC/OAuth)** using a mature library (e.g., Auth.js/next-auth) where feasible

### 7.2 Node services & libraries
- Microsoft Graph SDK (`@microsoft/microsoft-graph-client`) + MSAL for auth flows
- Queue: **BullMQ** + Redis (or managed Redis like Upstash)
- Logging: **pino**
- Testing: **Vitest** (unit), **Playwright** (e2e)

### 7.3 Code organization constraint
- UI components should be **small** and **split into separate files**.
- Prefer composition over “god components”.
- Feature folders encouraged (e.g., `features/chat/*`, `features/review/*`).

---

## 8) Data sources & ingestion strategy

### 8.1 Sources (MVP)
- Teams messages (chats + optionally channels)
- Outlook emails (in/out)
- Calendar events (create/update)
- Manual notes (typed into app)

### 8.2 Ingestion principles
- **Idempotent:** avoid duplicating documents in Zep.
- **Metadata-rich:** each document must include source, timestamps, and stable IDs.
- **Minimal transformation:** store raw text + metadata; let Zep extract structure.

### 8.3 Sync strategy
- Webhooks/Subscriptions where available.
- Delta queries as a safety net for missed events.
- Backfill window for first-time setup (configurable).

---

## 9) Requirements (EARS)

> Note: Requirements are written only for what *we* implement (glue + UX), assuming Zep provides structured memory/extraction/temporal handling.

### 9.1 Ubiquitous requirements
- The system shall use **Next.js + React** for the frontend and **Node-based BFF** within Next.js for backend endpoints.
- The system shall use a **React component library** (shadcn/ui) and **Tailwind CSS** for UI styling.
- The system shall implement UI as **small components in separate files**.
- The system shall treat Zep as the **system of record** for memory and extracted structure.
- The system shall store no long-term content outside Zep, except for operational needs (queue state, tokens/secrets).

### 9.2 Microsoft Graph connection
- When Ask connects Microsoft 365, the system shall acquire and store tokens securely using OAuth/OIDC.
- While a data source (Teams/Outlook/Calendar) is disabled, the system shall not ingest new items from that source.

### 9.3 Ingestion: webhook → fetch → Zep
- When the system receives a Graph webhook notification for a Teams message, the ingestion worker shall fetch the message content and metadata and ingest it into Zep.
- When the system receives a Graph webhook notification for an email, the ingestion worker shall fetch the email content and metadata and ingest it into Zep.
- When the system receives a Graph webhook notification for a calendar event, the ingestion worker shall fetch the event details and ingest it into Zep.
- When Ask submits a manual note, the BFF shall ingest the note into Zep with source `manual`.

### 9.4 Ingestion reliability
- While Zep is unreachable, the ingestion worker shall enqueue ingestion jobs for retry with exponential backoff.
- While Microsoft Graph is unreachable or returns transient errors, the ingestion worker shall retry fetch operations according to a configurable policy.
- When the ingestion worker processes an item already ingested (same stable source ID), the worker shall avoid creating duplicates (idempotency).

### 9.5 Assistant chat (LLM + Zep tool calls)
- When Ask sends a chat message, the BFF shall call the LLM with tool access to Zep query/update functions.
- When the LLM requests memory retrieval, the BFF shall query Zep and return results to the LLM.
- When the LLM requests a memory update (correction/verification), the BFF shall call the corresponding Zep update mechanism and return the result to the LLM.

### 9.6 Staleness & validation behavior
- While retrieved memory items are older than a configurable freshness window, the assistant shall label them as **stale** in its response.
- When the assistant’s answer relies on stale or low-confidence memory, the assistant shall ask **minimal validation questions** rather than fabricate details.
- When Ask provides a correction, the assistant shall apply it to memory and confirm the new belief succinctly.

### 9.7 Review & validate mode
- When Ask requests “review what you know about X”, the system shall render a structured list of beliefs retrieved from Zep.
- When Ask clicks **Accept** on a belief, the system shall send a verification signal to Zep (using Zep’s supported API/primitive).
- When Ask clicks **Reject** on a belief, the system shall send a rejection/deprecation signal to Zep (using Zep’s supported API/primitive).
- When Ask edits a belief, the system shall send the edited content as a correction to Zep.

### 9.8 Security & privacy (MVP-grade)
- The system shall store OAuth tokens and API keys only in a secure secrets store and never expose them to the client.
- The system shall restrict Graph scopes to the minimum required for enabled sources.
- The system shall provide a “Disconnect” function that revokes local access and stops ingestion.

---

## 10) Functional scope (MVP)

### 10.1 UI screens
1. **Chat**
   - chat thread
   - “daily brief” quick action
   - “review memory about …” quick action
2. **Review & Validate**
   - list of beliefs with Accept/Reject/Edit
   - filter by entity/topic/time window
3. **Connections**
   - connect Microsoft 365
   - toggles: Teams, Outlook, Calendar
4. **Manual Notes**
   - quick note input (also accessible from chat)

### 10.2 Core APIs (BFF)
- `POST /api/chat` — send user message; returns assistant response
- `POST /api/notes` — ingest manual note into Zep
- `GET /api/review?query=...` — retrieve beliefs for review UI
- `POST /api/review/accept` — accept belief
- `POST /api/review/reject` — reject belief
- `POST /api/review/edit` — edit belief (correction)
- `POST /api/webhooks/graph` — Graph webhook receiver (fast, enqueue work)
- `POST /api/connect/microsoft` — auth callback/init (via auth lib)

### 10.3 Worker responsibilities
- Consume queue jobs:
  - fetch item by ID from Graph
  - transform to “Document Envelope” (see below)
  - ingest into Zep
- Run periodic delta sync:
  - recover from missed webhooks

---

## 11) Document envelope (what we send to Zep)

Every ingested item should be wrapped consistently:

```json
{
  "source": "teams|outlook|calendar|manual|chat",
  "source_id": "stable-id-from-graph-or-generated",
  "timestamp": "ISO-8601",
  "author": { "id": "graph-id", "displayName": "..." },
  "participants": [{ "id": "...", "displayName": "..." }],
  "context": {
    "thread_id": "...",
    "channel_id": "...",
    "subject": "...",
    "link": "deep link if available"
  },
  "content": {
    "text": "plain text body",
    "html": "optional raw html",
    "attachments": ["optional metadata only in MVP"]
  }
}
````

Notes:

* Convert Teams/email HTML to readable text for ingestion.
* Keep raw HTML only if cheap; otherwise skip in MVP.

---

## 12) Non-functional requirements

### 12.1 Performance

* Chat responses should feel interactive (target: “fast enough to converse”; avoid blocking on ingestion).
* Webhook endpoint must respond quickly and offload work to queue.

### 12.2 Reliability

* At-least-once ingestion with idempotency.
* Backoff retries for transient failures.
* Clear operational logs for failed items.

### 12.3 Maintainability

* Small React components in separate files.
* Shared types between client/BFF/worker.
* Minimal bespoke logic; prefer libraries.

### 12.4 Observability

* Structured logs (pino) for:

  * webhook receipts
  * ingestion attempts/outcomes
  * Zep errors
  * Graph errors/rate limiting
* Basic metrics (optional MVP): counts of ingested items per source/day.

---

## 13) UI component requirements (small, split files)

### 13.1 Component library

* Use shadcn/ui primitives for:

  * buttons, inputs, cards, dialogs, toasts
  * tables/lists for beliefs review

### 13.2 Suggested file structure (example)

* `app/(routes)/chat/page.tsx`
* `features/chat/components/ChatThread.tsx`
* `features/chat/components/ChatComposer.tsx`
* `features/chat/components/DailyBriefButton.tsx`
* `features/review/components/BeliefList.tsx`
* `features/review/components/BeliefRow.tsx`
* `features/review/components/BeliefEditorDialog.tsx`
* `features/connections/components/SourceToggle.tsx`
* `lib/zep/client.ts`
* `lib/graph/client.ts`
* `lib/llm/client.ts`
* `worker/ingestion/processor.ts`
* `worker/ingestion/deltaSync.ts`

Rule of thumb:

* No UI file > ~200 lines without a strong reason.
* Prefer “dumb components” + a small number of “smart containers”.

---

## 14) Risks & mitigations

1. **Graph API rate limits / complexity**

   * Mitigation: queue + retry, minimal fetch fields, delta sync strategy.
2. **LLM hallucinations**

   * Mitigation: tool-based retrieval, stale labeling, validation prompts.
3. **Memory drift / incorrect beliefs**

   * Mitigation: review & validate mode; easy correction loop.
4. **Webhook reliability**

   * Mitigation: delta sync + idempotency.

---

## 15) Definition of Done (MVP)

1. Ask can connect Microsoft 365 and enable ingestion for Teams + Outlook + Calendar.
2. New items flow into Zep via webhook + worker.
3. Ask can chat with assistant; assistant uses Zep memory.
4. “Review what you know about X” shows beliefs with Accept/Reject/Edit.
5. Corrections update memory and affect subsequent answers.
6. App is built with Next.js/React/TypeScript, shadcn/ui + Tailwind, small components split into separate files.

---

