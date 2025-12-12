# AGENTS.md — Developer/Agent Guide

This document explains **what this project is**, **where things live**, and **how to work effectively** in the repo.

Key companion docs:

* **PRD.md** — Product requirements and EARS requirements (what must exist).
* **PLANS.md** — Implementation plan / milestones (how to build it end-to-end).

---

## 1. Project at a glance

### What we’re building

A **Personal Work Memory Assistant**:

* Ingests Microsoft 365 data (Teams, Outlook, Calendar) + manual notes
* Stores memory in **Zep** (3rd party)
* Uses an **LLM** to answer questions and perform memory review/corrections via tool-calls

### Core principles

* **Build only glue**: ingestion, orchestration, UX. Zep is the memory system-of-record.
* **Stateless BFF**: keep server routes thin; no long-term storage outside Zep.
* **Queue for reliability**: webhooks enqueue; worker fetches + ingests.
* **Small components**: split UI into small files.
* **Library-first**: prefer stable OSS libraries over custom frameworks.

---

## 2. Repo structure (where things are)

Recommended layout (monorepo):

```
/ (repo root)
  PRD.md
  PLANS.md
  AGENTS.md

  /apps
    /web                  # Next.js app (React UI + Node BFF)

  /packages
    /shared               # Shared types, Zod schemas, common utils
    /clients
      /graph              # Microsoft Graph client wrapper
      /zep                # Zep client wrapper
      /llm                # LLM client wrapper

  /services
    /worker               # Ingestion worker (BullMQ processors, delta sync, subscription renewal)

  /infra
    /docker               # Local docker compose for Redis, etc. (optional)

  /docs                   # Optional extra docs (runbook, architecture notes)
```

If the repo differs slightly, keep this mapping conceptually the same.

---

## 3. Where the main flows live

### 3.1 UI (React)

Location: `apps/web` and `apps/web/features/*`

Conventions:

* Prefer **feature folders**:

  * `features/chat/*`
  * `features/review/*`
  * `features/connections/*`
* Split UI into small components:

  * components in `features/<x>/components/*`
  * hooks in `features/<x>/hooks/*`
  * client API helpers in `features/<x>/api/*`

Key pages:

* `/chat` → assistant chat + daily brief action
* `/review` → belief list + Accept/Reject/Edit
* `/connections` → connect M365 + source toggles

### 3.2 BFF (Next.js route handlers)

Location: `apps/web/app/api/**` (or equivalent Next.js API routing)

Key endpoints (MVP/MLP):

* `POST /api/chat` — LLM conversation with Zep tool-calls
* `GET /api/review?query=...` — retrieve beliefs for review UI
* `POST /api/review/accept` — verify a belief
* `POST /api/review/reject` — reject/deprecate a belief
* `POST /api/review/edit` — correct/supersede a belief
* `POST /api/notes` — ingest manual note
* `POST /api/webhooks/graph` — Graph webhook receiver (fast; enqueue only)
* `GET /api/health` — minimal health/diagnostic endpoint

### 3.3 Worker (ingestion)

Location: `services/worker/*`

Responsibilities:

* Consumes BullMQ queue jobs
* Fetches content from Microsoft Graph by id
* Normalizes text (HTML → plain text)
* Wraps into a **Document Envelope**
* Idempotency check
* Ingests into Zep

Worker scheduled loops:

* Subscription renewal
* Delta sync (fallback)
* Bootstrap/backfill

### 3.4 Client wrappers (3rd party integration)

Location: `packages/clients/*`

* `clients/graph`:

  * authenticated Graph client
  * helpers for Teams/email/calendar fetches
* `clients/zep`:

  * `ingestDocument(envelope)`
  * `queryMemory({ query, filters })`
  * `updateMemory(updatePayload)`
* `clients/llm`:

  * `chatCompletionWithTools(...)`

### 3.5 Shared types and schemas

Location: `packages/shared/*`

* Zod request/response schemas for BFF endpoints
* Shared TS types (e.g., `Belief`, `Envelope`, `SourceToggleConfig`)

---

## 4. Contracts and key data shapes

### 4.1 Document Envelope (sent to Zep)

Canonical shape lives in `packages/shared`.

Fields to include at minimum:

* `source` (teams|outlook|calendar|manual)
* `source_id` (stable Graph id)
* `timestamp`
* `author`, `participants`
* `context` (thread/channel/subject/link)
* `content.text` (normalized)

### 4.2 Belief (review UI)

UI-friendly belief shape should be stable and live in `packages/shared`:

* `id`
* `statement`
* `confidence`
* `lastUpdated`
* `evidenceLinks[]`

---

## 5. Coding conventions (very important)

### 5.1 “Small components” rule

* Prefer composing small components over large ones.
* Avoid pages/components that exceed ~200 LOC unless justified.
* Extract:

  * presentational components
  * hooks
  * utility functions

### 5.2 Library choices

* UI: Tailwind + shadcn/ui
* Data fetching: TanStack Query
* Validation: Zod
* Queue: BullMQ + Redis
* Logging: pino
* Tests: Vitest (unit), Playwright (e2e)

### 5.3 Error handling

* Webhook route must respond quickly and never block on downstream calls.
* Worker must implement retries with exponential backoff.
* Respect Graph rate limiting (429 + Retry-After).

### 5.4 Security hygiene

* Never log tokens or raw secrets.
* Redact message content in logs by default (log ids/lengths).
* Keep Graph scopes minimal and gated by enabled sources.

---

## 6. How to work from requirements

When implementing a feature:

1. Read **PRD.md** and identify the relevant EARS requirements.
2. Use **PLANS.md** to find the milestone and acceptance checks.
3. Implement in the smallest possible vertical slice:

   * endpoint + UI + minimal worker support
4. Add tests where it’s cheap and valuable.

If a change conflicts with “build only glue”, it likely belongs in Zep/LLM configuration rather than custom code.

---

## 7. Local development (expected commands)

These commands may vary slightly; keep scripts updated in root `package.json`:

* `pnpm dev` — run Next.js app
* `pnpm worker` — run ingestion worker
* `pnpm lint` — lint
* `pnpm typecheck` — typecheck
* `pnpm test` — unit tests
* `pnpm e2e` — Playwright (staging)

Local dependencies (optional):

* Redis via docker compose in `infra/docker`

---

## 8. Environment variables (do not hardcode)

Maintain a `.env.example` at repo root.

Typical vars:

* Microsoft/Entra:

  * `AZURE_AD_CLIENT_ID`
  * `AZURE_AD_CLIENT_SECRET`
  * `AZURE_AD_TENANT_ID`
  * `AUTH_REDIRECT_URL`
* Auth:

  * `AUTH_SECRET`
* Zep:

  * `ZEP_BASE_URL`
  * `ZEP_API_KEY`
* LLM:

  * `LLM_API_KEY`
  * `LLM_MODEL`
* Redis:

  * `REDIS_URL`

---

## 9. PR review checklist

* UI split into small components and separate files
* Zod schemas updated for endpoint changes
* No long-term storage introduced outside Zep
* Webhook route enqueues only
* Worker remains idempotent
* Logs do not leak secrets
* Acceptance checks in PLANS.md still hold
