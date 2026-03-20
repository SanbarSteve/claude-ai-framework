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
│   ├── page.tsx                  # Home — use case selection
│   ├── interview/
│   │   └── [useCase]/
│   │       └── page.tsx          # Interview page (personal | sales | consumer)
│   └── api/
│       ├── interview/
│       │   └── route.ts          # Server-side Anthropic proxy
│       └── entity/
│           └── route.ts          # Generate + save entity file
├── components/
│   ├── TopBar.tsx
│   ├── UseCaseBar.tsx
│   ├── ChatPane.tsx
│   ├── MessageBubble.tsx
│   ├── PreviewPane.tsx
│   └── ProgressBar.tsx
├── lib/
│   ├── anthropic.ts              # Anthropic client (server-only)
│   ├── db.ts                     # Prisma client singleton
│   └── topics.ts                 # Topic & question data (ported from HTML)
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── docker-compose.yml            # Local Postgres for dev
├── .env.local                    # ANTHROPIC_API_KEY, DATABASE_URL
└── package.json
```

---

## Data Model

```prisma
model Session {
  id         String      @id @default(cuid())
  useCase    String      // "personal" | "sales" | "consumer"
  pass       Int         @default(1)
  topicIdx   Int         @default(0)
  data       Json        @default("{}")
  convo      Json        @default("[]")  // last 10 messages for API context
  createdAt  DateTime    @default(now())
  updatedAt  DateTime    @updatedAt
  entityFile EntityFile?
}

model EntityFile {
  id        String   @id @default(cuid())
  sessionId String   @unique
  session   Session  @relation(fields: [sessionId], references: [id])
  content   String
  useCase   String
  createdAt DateTime @default(now())
}
```

**Key decisions:**
- `Session` replaces both `localStorage` and the in-memory JS `state` object
- `sessionId` lives in the URL (`/interview/personal?session=abc123`) — sessions are resumable
- `data` and `Json` fields are flexible — new fields added without migrations
- `EntityFile` is a separate model so it can be attached to a `User` later with no breaking changes
- No `User` model in v1 — when auth is added, `userId String?` is added to both models

---

## Data Flow

### Interview lifecycle

```
1. User visits /
   └── Selects use case → navigates to /interview/[useCase]

2. Page loads → POST /api/interview/session
   └── Creates Session record in DB
   └── Returns sessionId → stored in URL (?session=abc123)
   └── Client renders welcome message + first topic question

3. User types answer → POST /api/interview
   Body: { sessionId, userText }
   └── API route fetches Session from DB
   └── Builds system prompt
   └── Calls Anthropic API (server-side, key in env)
   └── Parses <DATA> block from response
   └── Updates Session (topicIdx, pass, data, convo) in DB
   └── Returns { display, data } to client
   └── Client updates ChatPane + PreviewPane + ProgressBar

4. User clicks Generate → POST /api/entity
   Body: { sessionId }
   └── Fetches complete Session.data from DB
   └── Calls Anthropic to render final markdown entity file
   └── Saves EntityFile record to DB
   └── Returns markdown to client → triggers .md download
```

### State ownership

| Concern | Owner |
|---|---|
| API key | `.env.local` / Vercel env var |
| Interview state (pass, topicIdx, data) | DB via `Session` |
| Conversation history (last 10 msgs) | DB via `Session.convo` |
| UI state (typing indicator, busy flag) | React `useState` — ephemeral |
| Generated entity file | DB via `EntityFile` + client download |

**Conversation trimming:** The API route always trims `Session.convo` to the last 10 messages before sending to Anthropic, and writes the trimmed array back to DB. Mirrors current HTML behavior.

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

Client maps error codes to user-friendly messages in the chat pane.

### Zone 2 — DATA block parse failures

```ts
function parseDataBlock(text: string): ParsedData | null {
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

- Missing session → `404` → client redirects to `/` with "session expired" message
- Write failure → `500` → client shows retry prompt, previous DB state intact

### Explicitly out of scope for v1

- Auth / session ownership validation
- Offline / network loss handling

---

## Testing

### Layer 1 — Unit tests (Vitest)

Pure logic, no framework dependency:

- `lib/topics.ts` — all use cases have required fields, no missing topic IDs
- `lib/anthropic.ts` → `parseDataBlock()` — valid JSON parses, malformed returns null, missing block returns null
- `lib/anthropic.ts` → system prompt builder — correct pass/topic injected

Run with: `npm test`

### Layer 2 — API route integration tests (Vitest + Prisma test DB)

- `POST /api/interview` — creates session, returns display text, updates session in DB
- `POST /api/entity` — fetches session data, returns markdown, creates EntityFile record

Uses a second Docker DB container (`DATABASE_URL_TEST`). Run with: `npm run test:integration`

### Layer 3 — Smoke test (Playwright)

One end-to-end test:
- Open `/`, select Personal Co-Pilot, type an answer, verify bot reply appears

### Explicitly not tested in v1

- React component rendering
- Anthropic response quality

---

## Local Development Setup

```bash
# 1. Start local Postgres
docker compose up -d

# 2. Install dependencies
npm install

# 3. Push schema to local DB
npx prisma migrate dev

# 4. Start dev server
npm run dev
```

`.env.local` requires:
```
ANTHROPIC_API_KEY=sk-ant-...
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/entityfile
```

---

## Vercel Deployment

- Set `ANTHROPIC_API_KEY` and `DATABASE_URL` in Vercel environment variables
- `DATABASE_URL` points to Vercel Postgres (Neon) — same connection string format as local
- No build configuration changes needed beyond standard Next.js defaults

---

## Future Considerations (out of scope for this refactor)

- User auth (NextAuth or Clerk) — `userId` field already planned for in schema
- Role Library / Custom Roles use case
- Session export as JSON
- Edit mode for collected answers before generation
- Voice input
