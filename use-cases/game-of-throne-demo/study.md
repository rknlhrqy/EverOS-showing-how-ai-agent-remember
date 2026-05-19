# Game of Thrones Demo — Study Notes

Personal reference notes from reading the source of this use case (`use-cases/game-of-throne-demo/`). Written so a future session can pick up cold without re-reading every file. Companion to the repo-level [../../study.md](../../study.md).

## What this demo is

A web app that **proves the value of long-term memory by running two LLM streams side-by-side** on the same question about *A Game of Thrones* — one with EverCore-retrieved book passages, one with only the model's training-set knowledge. The contrast (precise + citation-backed vs. vague/hallucinated) is the whole point.

- ~4,700 lines of TypeScript across frontend, backend, and CLI scripts.
- Single page, no router, no auth.
- Frontend: React 18 + Vite. Backend: Express + Bun. LLM: Claude Haiku via OpenRouter. Memory: EverMind Cloud (or local EverCore).

## Three flows

### 1. Construction (one-time, offline) — [scripts/load-novel-cloud.ts](scripts/load-novel-cloud.ts)

CLI script that pours the novel into EverMind Cloud.

- **Chapter detection** ([:187-269](scripts/load-novel-cloud.ts#L187-L269)) — regex for `PROLOGUE`, `EPILOGUE`, all-caps POV names (`EDDARD`, `BRAN`, `ARYA`), `CHAPTER N`.
- **Paragraph splitting + grouping** ([:283-376](scripts/load-novel-cloud.ts#L283-L376)) — split on blank lines, then merge consecutive short paragraphs until each chunk hits ≥200 chars (configurable). Better recall context, fewer tiny embeddings.
- **Deterministic IDs** ([:378-382](scripts/load-novel-cloud.ts#L378-L382)): `asoiaf-got-ch01-p001`.
- **Per-paragraph upload** ([:461-525](scripts/load-novel-cloud.ts#L461-L525)) — prepends a header `[A Game of Thrones - Ch1: Bran]\n\n…paragraph…` then `POST /api/v0/memories` with `group_id: 'asoiaf'`. Bearer auth, 30 s timeout, 3 retries with exponential backoff (1 s / 2 s / 4 s).
- **Resumable**: writes `.novel-progress-cloud-got.json` after each paragraph so a crash or rate-limit can pick up where it left off (`--fresh-start` to override).

The EverCore construction track (atomic-fact extraction, summary/subject/episode generation, dense+sparse indexing) happens server-side. This script doesn't see any of that — it just feeds raw chunks.

### 2. Perception (per question) — [backend/src/routes/chat.ts:109-216](backend/src/routes/chat.ts#L109-L216)

User submits a question. Interesting endpoint is `/api/chat/compare`:

1. **Retrieve once** ([:128-135](backend/src/routes/chat.ts#L128-L135)) — `memoryService.retrieveMemories(query, 5)`.
2. **SSE `memories` event** sent first so the UI renders the chip row instantly.
3. **Two streams in parallel** ([:160-170](backend/src/routes/chat.ts#L160-L170)) — same model, same prompt scaffold, **only the context differs**:
   - `streamChatCompletion(message, memories, history)` → tagged `withMemory`
   - `streamChatCompletion(message, [], history)`     → tagged `withoutMemory`
   - `Promise.all` consumes both; each token fans out as `{type:'token', stream:'withMemory'|'withoutMemory', token}`.
4. **Two distinct system prompts** ([backend/src/services/OpenAIService.ts:27-54](backend/src/services/OpenAIService.ts#L27-L54)):
   - **With memory**: "ONLY use the provided excerpts. Cite them `[1][2]`."
   - **Without memory**: "ONLY use training data. Do NOT search the internet. If you don't recall specifics, say so — don't make up page numbers or quotes." Honest-mode prompt designed to suppress hallucinations rather than encourage them, so the comparison is fair.
5. **After both finish**, a third Claude call generates 2-3 follow-up questions ([:177-186](backend/src/routes/chat.ts#L177-L186)) seeded only from the with-memory answer.
6. Final `complete` event closes the SSE.

The non-comparison endpoint `/api/chat` is the same logic minus the second stream — vestigial (`App.tsx` only uses `useCompareChat`).

### 3. Display — frontend

- [frontend/src/App.tsx:5-32](frontend/src/App.tsx#L5-L32) — single-screen `ComparisonChatInterface`.
- [frontend/src/hooks/useCompareChat.ts:53-148](frontend/src/hooks/useCompareChat.ts#L53-L148) — state machine: `isRetrievingMemories` → memory chips render → `isStreaming` on both panels → tokens append per-stream → `isDone` toggles each side independently → on `complete`, the with-memory answer is committed to chat history.
- [frontend/src/services/api.ts:107-195](frontend/src/services/api.ts#L107-L195) — manual SSE parser over `fetch` + `ReadableStream` (not `EventSource`, because `POST` body is needed).
- **localStorage** caches history + last memory set across reloads.

## Pluggable memory backend

[`IMemoryService`](backend/src/services/IMemoryService.ts) is a 3-method interface. [backend/src/server.ts:26-32](backend/src/server.ts#L26-L32) picks one at boot:

- `USE_EVERMEMOS=true` + API key → **EverMind Cloud** (`https://api.evermind.ai`, Bearer auth)
- `USE_EVERMEMOS=true`, no key → **local EverCore** (`http://localhost:1995`, no auth)
- otherwise → **MockMemoryService** (canned data) — useful for UI dev without the backend.

---

# How the demo uses EverCore (in detail)

## The shape: 1 interface, 4 endpoints, 5 call sites

The demo never imports an EverCore SDK — it speaks **plain HTTP** to the public REST API behind an `IMemoryService` abstraction. Total surface is **four endpoints**:

| HTTP | Path | Used by | Purpose |
|:--|:--|:--|:--|
| `GET` | `/health` | EverCoreService, both ops scripts | Health probe |
| `GET` | `/api/v0/memories/search` | **Backend request path** | Hybrid retrieve top-K for a query |
| `POST` | `/api/v0/memories` | `load-novel-cloud.ts` | Write one paragraph as a memory |
| `GET` | `/api/v0/memories` | `get-memories-cloud.ts` | List/inspect stored memories |
| `DELETE` | `/api/v0/memories` | `clear-memories-cloud.ts` | Bulk delete by group |

Auth: `Authorization: Bearer <key>` for Cloud, none for local. The "Cloud vs local" switch is just whether `apiKey` is set.

## Layer 1 — The interface contract

[backend/src/services/IMemoryService.ts:17-32](backend/src/services/IMemoryService.ts#L17-L32):

```ts
interface IMemoryService {
  retrieveMemories(query: string, limit?: number): Promise<Memory[]>;
  isAvailable(): Promise<boolean>;
  clearMemories?(): Promise<void>; // optional, only Mock implements
}
```

The whole runtime read path is funneled through `retrieveMemories`. Routes never touch `EverCoreService` directly — they only see `IMemoryService`. That's how `MockMemoryService` ([backend/src/services/MockMemoryService.ts](backend/src/services/MockMemoryService.ts)) can be swapped in for offline UI dev.

The normalized `Memory` shape ([backend/src/services/IMemoryService.ts:1-15](backend/src/services/IMemoryService.ts#L1-L15)) is the only EverCore vocabulary the rest of the app ever sees:

```ts
interface Memory {
  id: string;
  content: string;                  // cleanedSummary (chip label)
  metadata: { bookTitle, chapterNumber?, chapterName? };
  relevanceScore?: number;
  subject?: string;                 // EverCore: concise title/headline
  summary?: string;                 // EverCore: short paragraph
  episode?: string;                 // EverCore: detailed narrative
  originalContent?: string;         // EverCore: source paragraph(s) joined
}
```

## Layer 2 — Boot-time selection

[backend/src/server.ts:15-32](backend/src/server.ts#L15-L32):

```ts
const USE_EVERMEMOS    = process.env.USE_EVERMEMOS === 'true';
const EVERMEMOS_URL    = process.env.EVERMEMOS_URL    || 'http://localhost:1995';
const EVERMEMOS_API_KEY= process.env.EVERMEMOS_API_KEY|| '';
const EVERMEMOS_GROUP_ID=process.env.EVERMEMOS_GROUP_ID||'asoiaf';

const memoryService = USE_EVERMEMOS
  ? new EverCoreService({ baseUrl: EVERMEMOS_URL, apiKey: EVERMEMOS_API_KEY || undefined, groupId: EVERMEMOS_GROUP_ID })
  : new MockMemoryService();
```

The `EverCoreService` constructor records `isCloudMode = !!config.apiKey` ([backend/src/services/EverMemOSService.ts:102](backend/src/services/EverMemOSService.ts#L102)) and that single flag toggles every subsequent behavioral difference (auth header, group filter, timeout).

## Layer 3 — The hot path: `retrieveMemories`

[backend/src/services/EverMemOSService.ts:109-149](backend/src/services/EverMemOSService.ts#L109-L149).

### Request

```
GET ${baseUrl}/api/v0/memories/search
  ?query=<URL-encoded user question>
  &retrieve_method=hybrid          # BM25 + vector + RRF rerank
  &top_k=5                         # backend always asks for 5
  &include_metadata=true
  [&group_ids=asoiaf]              # only in cloud mode
```

Headers: `Authorization: Bearer <key>` only when cloud. Timeout: **15 s cloud, 10 s local** ([:135](backend/src/services/EverMemOSService.ts#L135)). On any non-OK or thrown error → returns `[]` (graceful degradation, [:138-148](backend/src/services/EverMemOSService.ts#L138-L148)) — the "with memory" stream then runs prompt-less and effectively degrades into the "without memory" panel. No crash, no error event.

### Response (what EverCore returns)

Typed in [backend/src/services/EverMemOSService.ts:3-54](backend/src/services/EverMemOSService.ts#L3-L54). Shape:

```jsonc
{
  "status": "ok",
  "result": {
    "profiles": [                  // semantic facts/traits — IGNORED by demo
      { "item_type": "explicit_info", "category": "...", "description": "...", "score": 0.83 }
    ],
    "memories": [                  // the episodes the demo cares about
      {
        "memory_type": "...",
        "subject": "Bran witnesses Ned execute a deserter",
        "summary": "On January 18, 2026, Bran rode out...",
        "episode": "Full narrative with timestamps...",
        "keywords": [...], "linked_entities": [...],
        "score": 0.91,
        "original_data": [
          { "data_type": "...",
            "messages": [
              { "content": "[A Game of Thrones - Ch1: Bran]\n\nThe morning had dawned clear and cold...",
                "extend": { "message_id": "asoiaf-got-ch01-p001", "speaker_name": "Narrator" } }
            ]}
        ]
      }
    ],
    "scores": [0.91, ...],
    "total_count": 5, "has_more": false
  }
}
```

### Normalization

`mapSearchResultsToMemories` ([:183-206](backend/src/services/EverMemOSService.ts#L183-L206)) iterates `result.memories[i]` paired with `result.scores[i]` (per-item `score` wins). The demo **completely ignores `result.profiles`** — only the episodic `memories[]` array matters.

`mapMemoryItem` ([:211-272](backend/src/services/EverMemOSService.ts#L211-L272)) does three jobs:

1. **Reconstruct book metadata** from the leading `[Book - ChN: Name]\n\n…` header on the first `original_data.messages[*].content` ([parseContent](backend/src/services/EverMemOSService.ts#L278-L310), regex `^\[(.+?)\s+-\s+Ch(\d+):\s+(.+?)\]\n\n/`). Fallback: parse the `message_id` (`asoiaf-got-ch01-p001` → `BOOK_TITLES['got']` lookup at [:64-70](backend/src/services/EverMemOSService.ts#L64-L70)).
2. **Concatenate source paragraphs** into `originalContent` (header-stripped). Powers the "Show original" reveal in the UI.
3. **Scrub date hallucinations** with `cleanDateArtifacts` ([:349-376](backend/src/services/EverMemOSService.ts#L349-L376)) on both `summary` and `subject`. EverMind's auto-summarizer injects today's date into prose ("On January 18, 2026, Bran rode out…"); three regexes strip leading "On <DOW>, <Month> N, YYYY,", trailing " - <Month> N, YYYY", and inline " on <Month> N, YYYY", then re-uppercases the first letter if needed.

Returned `Memory.id` = first message's `message_id` (e.g. `asoiaf-got-ch01-p001`) — deterministic per paragraph, so React keys stay stable.

### Call site

Just one consumer ([backend/src/routes/chat.ts:129](backend/src/routes/chat.ts#L129)):

```ts
const memories = await memoryService.retrieveMemories(message, 5);
res.write(`data: ${JSON.stringify({ type: 'memories', memories })}\n\n`);
```

The 5 memories then get formatted into the system prompt's context block ([backend/src/services/OpenAIService.ts:77-101](backend/src/services/OpenAIService.ts#L77-L101)) — both `summary` and `originalContent` are pasted in so Claude can cite from either:

```
[1] A Game of Thrones - Chapter 1: Bran
Summary: Bran witnesses Ned execute a deserter…

Original Text:
The morning had dawned clear and cold…
```

The `[1]`, `[2]` numbering is what the system prompt tells Claude to use for in-text citations.

## Layer 4 — `isAvailable` (health probe)

[backend/src/services/EverMemOSService.ts:154-178](backend/src/services/EverMemOSService.ts#L154-L178). `GET /health` with the same auth conditional, 5 s timeout. Accepts **both** `status: "healthy"` (local EverCore convention) **and** `status: "ok"` (Cloud convention) — that line at [:173](backend/src/services/EverMemOSService.ts#L173) is the only place the demo papers over a Cloud-vs-local API difference at the **response** level (everything else handles it on the request side via `isCloudMode`). Used by `/api/health` to expose service status to the frontend.

## Layer 5 — Write path: novel ingest

[scripts/load-novel-cloud.ts:461-525](scripts/load-novel-cloud.ts#L461-L525). For each chunked paragraph:

### Request

```
POST ${apiUrl}/api/v0/memories
Headers: Authorization: Bearer <key>, Content-Type: application/json
Timeout: 30 s
Body:
{
  "message_id":  "asoiaf-got-ch01-p001",      // deterministic, idempotent
  "group_id":    "asoiaf",                     // shared bucket for all ASOIAF books
  "group_name":  "A Song of Ice and Fire",
  "create_time": "<ISO timestamp>",
  "role":        "assistant",                  // narrator role
  "sender":      "asoiaf_narrator",
  "sender_name": "Narrator",
  "content":     "[A Game of Thrones - Ch1: Bran]\n\n<paragraph text>",
  "refer_list":  []
}
```

The `[Book - ChN: Name]\n\n` prefix on `content` is the **side-channel for the search-side `parseContent` parser** — not a feature EverCore knows about; the loader writes it in, the reader peels it back out. EverCore just stores `content` opaquely and indexes it.

### Retry strategy

Three attempts, exponential backoff 1 s / 2 s / 4 s ([:485-522](scripts/load-novel-cloud.ts#L485-L522)). Distinguishes "timeout" vs other errors in the log line. On final failure, the paragraph is marked `failed` in the progress file but the loader keeps going.

### Idempotency & resumption

Deterministic `message_id` per paragraph — re-uploads overwrite, not duplicate (assumes EverCore upserts on `message_id`). Side-file `.novel-progress-cloud-got.json` ([:50-58](scripts/load-novel-cloud.ts#L50-L58) schema, [:417-426](scripts/load-novel-cloud.ts#L417-L426) update) records `success`/`failed` per `message_id`, so a crashed run resumes mid-chapter without re-uploading completed paragraphs.

The EverCore server-side pipeline (atomic-fact extraction → summary → subject → episode → embed → BM25+vector index) is **completely invisible to this script** — it just POSTs and trusts the response.

## Layer 6 — Ops scripts

### [scripts/get-memories-cloud.ts:149-191](scripts/get-memories-cloud.ts#L149-L191)

Reads back what was stored. `GET /api/v0/memories?group_ids=asoiaf&page=N&page_size=K[&memory_type=…&start_time=…&end_time=…]`. Listing endpoint. Not used at runtime; developer inspection only.

### [scripts/clear-memories-cloud.ts:173-219](scripts/clear-memories-cloud.ts#L173-L219)

`DELETE /api/v0/memories` with a JSON body — interesting bit:

```json
{ "event_id": "__all__", "user_id": "__all__", "group_id": "asoiaf" }
```

The API requires all three keys and accepts `"__all__"` as a wildcard. The demo always passes `__all__` for the first two and a specific `group_id` to wipe only the ASOIAF bucket. **404 is treated as success** ("already clean") — idempotent cleanup. 30 s timeout.

Only `DELETE` in the codebase. `IMemoryService.clearMemories?` is optional and not implemented by `EverCoreService` — only `MockMemoryService` has it. Wiping Cloud memory requires running this script, not a call into the running backend.

## End-to-end: one user question → EverCore round trip

1. User submits "How did Bran find the direwolves?" in the React UI.
2. `useCompareChat.sendMessage` ([frontend/src/hooks/useCompareChat.ts:53-148](frontend/src/hooks/useCompareChat.ts#L53-L148)) POSTs to `/api/chat/compare` with the prior history.
3. Backend's `/api/chat/compare` handler ([backend/src/routes/chat.ts:109-216](backend/src/routes/chat.ts#L109-L216)) immediately sets SSE headers, then calls `memoryService.retrieveMemories(message, 5)`.
4. `EverCoreService.retrieveMemories` issues:
   ```
   GET https://api.evermind.ai/api/v0/memories/search?query=How+did+Bran+find+the+direwolves%3F
       &retrieve_method=hybrid&top_k=5&include_metadata=true&group_ids=asoiaf
   Authorization: Bearer evk_...
   ```
5. EverCore runs hybrid retrieval (BM25 + vector + RRF rerank) over the ASOIAF group → returns 5 episode memories with summaries, subjects, scores, and the original book paragraphs.
6. `mapMemoryItem` flattens each one, strips date artifacts, reconstructs `bookTitle`/`chapterNumber`/`chapterName` from the content-prefix side channel.
7. Backend writes `data: {"type":"memories","memories":[…5 items…]}\n\n` to the SSE — UI renders the 5 numbered chips before a single token of LLM output arrives.
8. Backend opens two `streamChatCompletion` calls to Claude Haiku via OpenRouter — one with the formatted `[1]…[5]` context + a "cite these excerpts" prompt, one with empty memories + a "training data only" prompt — and pipes both streams to the SSE tagged `withMemory` / `withoutMemory`.
9. EverCore is **done** at step 7. It is touched **exactly once per user question** and it serves only retrieval.

## Things worth knowing

- **All Cloud-vs-local differences live in one class.** Everything else in the demo treats them identically (routes, LLM service, frontend). Adding a third backend just means another `IMemoryService` impl.
- **No conversation memorization.** The demo **never POSTs user chats back to EverCore** — per-question retrieve is one-shot, history-free from EverCore's viewpoint. Conversation history is kept entirely client-side in `localStorage` and replayed to Claude on every request. So EverCore in this demo is a **read-only RAG corpus**, not a growing agent memory. (Contrast with MemoCare in [../alzheimer-assistant/](../alzheimer-assistant/), which both `memorize` and `searchMemories`.)
- **`profiles` array is dropped on the floor.** EverCore returns semantic profiles too, but the demo only renders episodic memories. For a literary corpus that's fine — characters aren't "users with traits" — but it means the demo doesn't exercise the full EverCore output.
- **Hybrid is hardcoded.** No `retrieve_method=keyword`/`vector`/`agentic` knob exposed. README claims "hybrid" but doesn't mention the other modes EverCore supports.
- **`top_k=5` is hardcoded** in the route call. Interface accepts `limit` but the route always passes 5.
- **Side-channel encoding via content prefix.** The `[Book - ChN: Name]\n\n` header trick is fragile — written by the loader, parsed by the reader, with no schema between them. If EverCore ever transformed the content (e.g. stripping leading brackets), the read-side metadata would silently disappear and fall through to `parseMessageId`.
- **No EverCore SDK dependency.** Plain `fetch` against documented REST endpoints — easy to port, easy to mock, no version-coupling.
- **Default model mismatch.** [backend/src/server.ts:13](backend/src/server.ts#L13) defaults to `openai/gpt-5.2` if `OPENAI_MODEL` is unset; README says `anthropic/claude-3-haiku`. Set the env var explicitly or you're silently on a different model.
- **Three brand names for one product line.** "EverMem" (badges), "EverCore" (types/class names), "EverMemOS" (file names). Same backend.
- **`cleanDateArtifacts` is a workaround, not a feature.** It exists because EverMind's summarizer injects today's date into prose. If that gets fixed upstream, those regexes become dead code.

## Key files (quick index)

| File | What |
|:--|:--|
| [backend/src/server.ts](backend/src/server.ts) | Boots Express, picks memory backend |
| [backend/src/routes/chat.ts](backend/src/routes/chat.ts) | `/api/chat` + `/api/chat/compare` SSE endpoints |
| [backend/src/services/IMemoryService.ts](backend/src/services/IMemoryService.ts) | Memory backend interface + `Memory` shape |
| [backend/src/services/EverMemOSService.ts](backend/src/services/EverMemOSService.ts) | EverCore HTTP client (search + health + normalization) |
| [backend/src/services/MockMemoryService.ts](backend/src/services/MockMemoryService.ts) | Offline canned memories for UI dev |
| [backend/src/services/OpenAIService.ts](backend/src/services/OpenAIService.ts) | Claude Haiku via OpenRouter; two system prompts; follow-up generator |
| [frontend/src/App.tsx](frontend/src/App.tsx) | Single-screen entry |
| [frontend/src/hooks/useCompareChat.ts](frontend/src/hooks/useCompareChat.ts) | Compare-mode state machine + localStorage |
| [frontend/src/services/api.ts](frontend/src/services/api.ts) | Manual SSE parser over `fetch` |
| [scripts/load-novel-cloud.ts](scripts/load-novel-cloud.ts) | Ingest novel → POST `/api/v0/memories` |
| [scripts/get-memories-cloud.ts](scripts/get-memories-cloud.ts) | GET `/api/v0/memories` (listing) |
| [scripts/clear-memories-cloud.ts](scripts/clear-memories-cloud.ts) | DELETE `/api/v0/memories` (bulk wipe by group) |
