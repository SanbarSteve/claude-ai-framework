# Next.js Refactor Design — Entity File Interview Tool

**Date:** 2026-03-20
**Status:** Approved
**Author:** SanbarSteve + Claude

---

## Overview

Refactor the current single-file HTML application (`entity-interview-v2.html`) into a Next.js App Router project deployable to Vercel, with a PostgreSQL database via Prisma. The core interview experience is preserved. The primary architectural improvements are: moving the Anthropic API key server-side, replacing localStorage with a persistent DB session, and establishing a file structure a junior developer can iterate on quickly.

---

## Project Structure

```
claude-ai-framework/
├── app/
│   ├── layout.tsx                # Root layout, fonts, global styles
│   ├── page.tsx                  # Home — use case selection (reads from lib/topics.ts)
│   ├── interview/
│   │   └── [useCase]/
│   │       └── page.tsx          # Interview page; reads ?sessionId= from URL
│   └── api/
│       ├── interview/
│       │   ├── session/
│       │   │   └── route.ts      # POST — creates a new Session, returns sessionId
│       │   └── route.ts          # POST — sends a user message, returns { display, data }
│       └── entity/
│           └── route.ts          # POST — generates entity file, returns { content, filename }
├── components/
│   ├── TopBar.tsx                # App title bar
│   ├── UseCaseBar.tsx            # Use case selector tabs
│   ├── ChatPane.tsx              # Scrollable message list + input bar
│   ├── MessageBubble.tsx         # Single bot or user message
│   ├── PreviewPane.tsx           # Live entity file preview (right column)
│   └── ProgressBar.tsx           # Pass dots + topic progress bar
├── lib/
│   ├── anthropic.ts              # Anthropic client + parseDataBlock + buildSystemPrompt (server-only)
│   ├── db.ts                     # Prisma client singleton
│   └── topics.ts                 # USE_CASES array, TOPICS map, QUESTIONS map
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── types/
│   └── index.ts                  # Shared TypeScript interfaces for API request/response bodies
├── docker-compose.yml            # Local Postgres for dev
├── .env.local                    # ANTHROPIC_API_KEY, DATABASE_URL
└── package.json
```

---

## Data Model

```prisma
// prisma/schema.prisma

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model Session {
  id         String      @id @default(cuid())
  useCase    String      // "personal" | "sales" | "consumer"
  pass       Int         @default(1)
  topicIdx   Int         @default(0)
  data       Json        @default("{}")   // { [topicId]: "extracted answer" }
  convo      Json        @default("[]")   // last 10 individual messages: { role, content }[]
  createdAt  DateTime    @default(now())
  updatedAt  DateTime    @updatedAt
  entityFile EntityFile?
}

model EntityFile {
  id        String   @id @default(cuid())
  sessionId String   @unique
  session   Session  @relation(fields: [sessionId], references: [id])
  content   String   // the generated markdown
  useCase   String
  createdAt DateTime @default(now())
}
```

**Key decisions:**
- `Session` replaces both `localStorage` and the in-memory JS `state` object.
- `sessionId` is passed as a URL query param: `/interview/personal?sessionId=abc123`. The interview page reads it with `useSearchParams()`. All subsequent API calls include it in the POST body.
- `data` and `convo` are stored as `Json` — flexible enough to add fields without migrations as use cases evolve.
- `EntityFile` is a separate model so it can be attached to a `User` when auth is added — no breaking schema changes needed at that point.
- No `User` model in v1. When auth is added, add `userId String?` to both models and backfill. This is an intentional trade-off: sessions are single-use and accessed only by `sessionId`. No additional security is implemented in v1.

---

## Topics Data Structure

Use cases, topics, and questions all live in `lib/topics.ts` — the single source of truth. This is a direct port of the `TOPICS` and `QUESTIONS` objects from the original HTML file.

```ts
// lib/topics.ts

export const USE_CASES = ['personal', 'sales', 'consumer'] as const
export type UseCase = typeof USE_CASES[number]

export interface Topic {
  id: string    // e.g. "identity", "goals"
  icon: string  // emoji
  label: string // display name
}

export const TOPICS: Record<UseCase, Topic[]> = {
  personal: [
    { id: 'identity',     icon: '🧠', label: 'Identity' },
    { id: 'goals',        icon: '🎯', label: 'Goals' },
    // ... (port all from HTML)
  ],
  sales: [ /* ... */ ],
  consumer: [ /* ... */ ],
}

export const QUESTIONS: Record<UseCase, Record<string, string>> = {
  personal: {
    identity: 'Tell me about yourself...',
    goals:    'What are your primary goals...',
    // ...
  },
  sales: { /* ... */ },
  consumer: { /* ... */ },
}
```

The home page (`app/page.tsx`) imports `USE_CASES` to render the use case selector. No DB seed needed — use cases are static config.

---

## Pass and Topic Progression

The interview has up to 3 passes. Within each pass, every topic is asked in order. Progression is controlled by the `action` field in the DATA block returned by Anthropic:

| `action` | Meaning | Effect |
|---|---|---|
| `ANSWER` | Claude is still on the current topic | `topicIdx` unchanged |
| `NEXT` | Move to next topic | `topicIdx = next_idx` |
| `SKIP` | User skipped this topic | `topicIdx = next_idx` |
| `COMPLETE` | All topics done for this pass | Enable Generate button |

When `topicIdx` reaches the end of the topic list, the server calls `endPass()` logic, increments `pass`, and resets `topicIdx` to 0 for pass 2/3 if the user opts in.

**Early Generate unlock:** The Generate button is also enabled once 4 or more topics have been answered (i.e., `Object.keys(session.data).length >= 4`), regardless of whether a `COMPLETE` action has been received. This mirrors the original HTML behavior and allows users to generate a partial entity file without completing all topics.

---

## DATA Block Protocol

Claude appends a structured JSON block to every reply. The API route strips it before sending the display text to the client.

**Format:**
```
<DATA>{"action":"NEXT","topic_id":"goals","next_idx":2,"value":"Steve wants to grow invested assets and build web-based retail apps."}</DATA>
```

**Schema:**
```ts
interface DataBlock {
  action:   'ANSWER' | 'NEXT' | 'SKIP' | 'COMPLETE'
  topic_id: string   // current topic id
  next_idx: number   // index of the next topic to ask (same as current for ANSWER)
  value:    string   // 1-2 sentence summary of what the user shared
}
```

**`parseDataBlock` contract (`lib/anthropic.ts`):**
```ts
function parseDataBlock(text: string): DataBlock | null
// Returns null if: no <DATA> block found, JSON is malformed, or required fields are missing
// On null: reply is still displayed; topic state does not advance
// Side effect: logs malformed input to console.error for debugging
```

---

## System Prompt

Built in `lib/anthropic.ts` → `buildSystemPrompt(session: Session): string`. Mirrors the original HTML system prompt, with dynamic injection of current state:

- Use case name
- Current pass (1–3) and pass label (Initial / Refinement / Polish)
- Current topic label and index
- Topics answered so far (keys of `session.data`)
- Full collected data (`JSON.stringify(session.data)`)
- The DATA block format and rules (verbatim from original HTML)

---

## Conversation History

**Format:** `session.convo` is an array of `{ role: 'user' | 'assistant', content: string }` objects — the shape Anthropic's API expects directly.

**Truncation rule:** Before every Anthropic call, the API route takes `session.convo.slice(-10)` — the last 10 individual messages (not pairs). The system prompt is passed separately via the `system` parameter and does not count against this limit. The truncated array is written back to `session.convo` after each turn.

**Where it lives:** Truncation and formatting happen in `lib/anthropic.ts` → `callClaude()`. The route handler passes the full session object; the lib handles trimming.

---

## Data Flow

### Interview lifecycle

```
1. User visits /
   └── Renders use case selector from lib/topics.ts USE_CASES
   └── User selects use case → POST /api/interview/session { useCase }
       └── Creates Session record in DB
       └── Returns { sessionId }
   └── Client navigates to /interview/[useCase]?sessionId=abc123

2. Interview page loads
   └── Reads sessionId from useSearchParams()
   └── Fetches session state (or creates welcome message client-side)
   └── Renders TopBar, UseCaseBar, ChatPane, PreviewPane, ProgressBar

3. User types answer → POST /api/interview
   Body: { sessionId: string, userText: string }
   └── Fetch Session from DB (404 if not found)
   └── Build system prompt via buildSystemPrompt(session)
   └── Call Anthropic with session.convo.slice(-10) + new user message
   └── Parse <DATA> block
   └── Update session: topicIdx, pass, data, convo (trimmed to last 10)
   └── Return { display: string, data: Record<string, string> }
   └── Client: append message to ChatPane, update PreviewPane + ProgressBar

4. User clicks Generate → POST /api/entity
   Body: { sessionId: string }
   └── Fetch Session.data from DB
   └── Call Anthropic with a generation prompt + full collected data
   └── Save EntityFile record to DB
   └── Return { content: string, filename: string }
       e.g. { content: "# Personal Co-Pilot Entity File...", filename: "entity-personal.md" }
   └── Client: create Blob, call URL.createObjectURL(), trigger download
```

> **`max_tokens` values:** Use `1024` for `/api/interview/route.ts` (conversational replies). Use `4096` for `/api/entity/route.ts` — the output is a full markdown document and a smaller value will silently truncate it.

### State ownership

| Concern | Owner |
|---|---|
| API key | `.env.local` / Vercel env var |
| Interview state (pass, topicIdx, data) | DB via `Session` |
| Conversation history (last 10 msgs) | DB via `Session.convo` |
| UI state (typing indicator, busy flag) | React `useState` — ephemeral, never persisted |
| Generated entity file | DB via `EntityFile` + client `.md` download |

---

## TypeScript Interfaces

Defined in `types/index.ts` and shared between client and server:

```ts
// API request bodies
interface CreateSessionRequest  { useCase: string }
interface InterviewRequest      { sessionId: string; userText: string }
interface EntityRequest         { sessionId: string }

// API response bodies
interface CreateSessionResponse { sessionId: string }
interface InterviewResponse     { display: string; data: Record<string, string> }
interface EntityResponse        { content: string; filename: string }
interface ErrorResponse         { error: string; message?: string }
```

---

## Error Handling

### Zone 1 — Anthropic API errors (server-side)

```ts
try {
  const reply = await callClaude(session, userText)
  return Response.json({ display: reply.display, data: reply.data })
} catch (err) {
  if (err.status === 401) return Response.json({ error: 'invalid_key' }, { status: 401 })
  if (err.status === 429) return Response.json({ error: 'rate_limited' }, { status: 429 })
  return Response.json({ error: 'api_error', message: err.message }, { status: 500 })
}
```

Client maps error codes to user-friendly messages in the ChatPane.

### Zone 2 — DATA block parse failures

```ts
function parseDataBlock(text: string): DataBlock | null {
  const match = text.match(/<DATA>(.*?)<\/DATA>/s)
  if (!match) return null
  try {
    return JSON.parse(match[1].trim())
  } catch {
    console.error('DATA block parse failed:', match[1])
    return null
  }
}
```

If parsing fails, the reply still displays but topic state does not advance. User can reply again or skip.

### Zone 3 — DB errors

- Missing session → `404` → client redirects to `/` with "session expired" toast
- Write failure → `500` → client shows retry prompt; previous DB state is intact (partial writes do not occur — Prisma uses transactions for multi-field updates)

### Explicitly out of scope for v1

- Auth / session ownership validation (any client that knows a `sessionId` can access it — acceptable for single-user v1)
- Offline / network loss handling

### Environment variable validation

Both `lib/anthropic.ts` and `lib/db.ts` throw a clear error at module load time if their required env vars are missing:

```ts
// lib/anthropic.ts
if (!process.env.ANTHROPIC_API_KEY) {
  throw new Error('Missing env var: ANTHROPIC_API_KEY')
}
```

This surfaces misconfiguration immediately at startup rather than deep in a call stack.

---

## Prisma Client Singleton

Next.js hot reload creates new module instances in development, which causes multiple `PrismaClient` instances and exhausts DB connections. Use the standard singleton pattern:

```ts
// lib/db.ts
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient }

export const db =
  globalForPrisma.prisma ?? new PrismaClient({ log: ['error'] })

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = db
```

Always import `db` from `lib/db.ts` — never call `new PrismaClient()` directly.

---

## Testing

### Layer 1 — Unit tests (Vitest)

Pure logic, no framework dependency:

- `lib/topics.ts` — all use cases have required fields, no missing topic IDs
- `lib/anthropic.ts` → `parseDataBlock()` — valid JSON parses, malformed returns null, missing block returns null
- `lib/anthropic.ts` → `buildSystemPrompt()` — correct pass/topic injected into output string

Run with: `npm test`

### Layer 2 — API route integration tests (Vitest + Prisma test DB)

- `POST /api/interview/session` — creates a Session, returns sessionId
- `POST /api/interview` — updates session state, returns display text
- `POST /api/entity` — creates EntityFile record, returns markdown content

Uses a second Docker DB container (`DATABASE_URL_TEST`). Prisma migrates it at test start. Run with: `npm run test:integration`

### Layer 3 — Smoke test (Playwright)

One end-to-end test:
- Open `/`, select Personal Co-Pilot, type an answer, verify bot reply appears in ChatPane

### Explicitly not tested in v1

- React component rendering
- Anthropic response quality

---

## Local Development Setup

**`docker-compose.yml`:**
```yaml
services:
  db:
    image: postgres:16
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: entityfile
  db_test:
    image: postgres:16
    ports:
      - "5433:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: entityfile_test
```

**`.env.local`:**
```
ANTHROPIC_API_KEY=sk-ant-...
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/entityfile
DATABASE_URL_TEST=postgresql://postgres:postgres@localhost:5433/entityfile_test
```

**Setup steps:**
```bash
docker compose up -d          # start local Postgres
npm install                   # install dependencies
npx prisma migrate dev        # apply schema to local DB
npm run dev                   # start Next.js dev server
```

---

## Vercel Deployment

1. Set `ANTHROPIC_API_KEY` in Vercel environment variables
2. Provision Vercel Postgres (Neon) — copy the **pooled** connection string for `DATABASE_URL`

> **Important:** Neon requires connection pooling for serverless environments. Use the pooled connection string (contains `?pgbouncer=true` or uses port `6543`). Using the direct connection string will exhaust connections under any real load.

3. Run `npx prisma migrate deploy` as part of the Vercel build command, or run it manually after first deploy
4. No other build configuration changes needed beyond standard Next.js defaults

---

## Future Considerations (out of scope for this refactor)

- User auth (NextAuth or Clerk) — `userId String?` field planned for Session and EntityFile
- Custom Roles use case — AI-generated topic sets from plain-language role description
- Role Library — community-contributed role templates
- Session export as JSON
- Edit mode for collected answers before generation
- Streaming Anthropic responses (deferred — requires Server-Sent Events or ReadableStream handling)
- Voice input
