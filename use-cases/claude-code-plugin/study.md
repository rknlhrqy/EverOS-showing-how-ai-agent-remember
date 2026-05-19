# EverMem Claude Code Plugin — Study Notes

Personal reference notes from reading the source of this use case (`use-cases/claude-code-plugin/`). Written so a future session can pick up cold without re-reading every file. Companion to the repo-level [../../study.md](../../study.md) and the sibling [../game-of-throne-demo/study.md](../game-of-throne-demo/study.md) and [../alzheimer-assistant/study.md](../alzheimer-assistant/study.md).

## TL;DR

A Claude Code plugin called **`evermem`** that adds **persistent cross-session memory** to Claude Code. Four lifecycle hooks automatically inject memories on every prompt and save conversations after every response, scoped per-project. Same plugin also exposes an MCP tool for on-demand searches, 6 slash commands, and a local web dashboard. All memory operations flow through ~3 EverMind Cloud REST endpoints (the **v1 API**) via plain HTTP — no SDK.

## Anatomy

| Piece | Purpose |
|:--|:--|
| [plugin.json](plugin.json) | Plugin manifest (`name: evermem`, v0.2.0) |
| [hooks/hooks.json](hooks/hooks.json) | Wires 4 lifecycle hooks to scripts |
| [hooks/scripts/](hooks/scripts/) | The 4 hook implementations + shared utils |
| [mcp/server.js](mcp/server.js) | MCP stdio server exposing `evermem_search` tool |
| [skills/memory-tools.md](skills/memory-tools.md) | `alwaysInclude` agent skill describing when to search |
| [commands/](commands/) | 6 slash commands (`/evermem:help`, `:search`, `:ask`, `:hub`, `:debug`, `:projects`) |
| [server/proxy.js](server/proxy.js) | Local HTTP server (port 3456) for the Memory Hub dashboard |
| [assets/dashboard.html](assets/dashboard.html) | Memory Hub UI (heatmap, projects, search) |
| [data/](data/) | Local files: `sessions.jsonl` + `groups.jsonl` |

## The four hooks — registered in [hooks/hooks.json](hooks/hooks.json)

```
SessionStart      → session-context-wrapper.sh → session-context.js   (timeout 30s)
UserPromptSubmit  → inject-memories.js                                (timeout 10s)
Stop              → store-memories.js                                 (timeout 30s)
SessionEnd        → session-summary.js                                (timeout 30s)
```

Each hook is a Node ≥18 ES-module script. Claude Code spawns it as a child process, pipes the event JSON to **stdin**, reads a JSON response from **stdout**. The output JSON can include `systemMessage` (shown to user), `systemPrompt` (injected to Claude, invisible), `hookSpecificOutput.additionalContext` (also injected), and `continue: true|false`.

### 1. SessionStart — [hooks/scripts/session-context.js](hooks/scripts/session-context.js)

Runs once when a Claude Code session starts.

1. **Records the project** ([:118](hooks/scripts/session-context.js#L118)) — appends `{keyId, groupId, name, path, timestamp}` to `data/groups.jsonl` via [`saveGroup`](hooks/scripts/utils/groups-store.js) (dedupes on `keyId+groupId`).
2. **Pulls 100 most-recent memories** ([:135](hooks/scripts/session-context.js#L135)) — `getMemories({pageSize: 100, memoryType: 'episodic_memory', rank_by: 'timestamp', rank_order: 'desc'})` then takes the top 5.
3. **Reads last session summary** ([:139](hooks/scripts/session-context.js#L139)) — scans `data/sessions.jsonl` backward for the most recent entry matching current `groupId`.
4. **Outputs** a `systemMessage` ("💡 EverMem: Last (2h ago, 5 turns)…") and a `systemPrompt`:
   ```
   <session-context>
   Last session (2h ago, 5 turns): Implementing JWT…
   Recent memories (5):
   [1] (2/9/2026) JWT token implementation
   ...
   </session-context>
   ```
5. **Node version check** at [:9-26](hooks/scripts/session-context.js#L9-L26): refuses to run on Node <18 (needs ES modules).
6. **Always `continue: true`** even on errors — never blocks session start.

### 2. UserPromptSubmit — [hooks/scripts/inject-memories.js](hooks/scripts/inject-memories.js)

Fires every time the user submits a prompt.

1. **CJK-aware word count** ([:38-56](hooks/scripts/inject-memories.js#L38-L56)) — counts Chinese/Japanese/Korean characters as tokens and Latin words as space-separated; skips if prompt has < 3 "words".
2. **Searches top-15** ([:93-96](hooks/scripts/inject-memories.js#L93-L96)) — `searchMemories(prompt, {topK: 15, retrieveMethod: 'hybrid'})`.
3. **Filters** by `score >= 0.1` ([:107](hooks/scripts/inject-memories.js#L107)), takes top 5.
4. **Two outputs**:
   - `systemMessage` ([buildDisplayMessage](hooks/scripts/inject-memories.js#L173-L189)) — bullet list `[0.85] (2 days ago) subject` shown in the terminal.
   - `hookSpecificOutput.additionalContext` ([buildContext](hooks/scripts/inject-memories.js#L197-L236)) — XML-tagged block with memories **sorted by recency**, including the explicit instruction: *"When there are conflicts or updates between memories, prefer the MORE RECENT information."*
5. **Silent on all errors** — `process.exit(0)` with no output so a broken plugin can never block your prompt.

### 3. Stop — [hooks/scripts/store-memories.js](hooks/scripts/store-memories.js)

Fires when Claude finishes responding. Saves the just-completed turn.

1. **Reads transcript with retry** ([:37-74](hooks/scripts/store-memories.js#L37-L74)) — Claude Code writes a JSONL transcript at `transcript_path`; this script polls up to 5×100 ms until the file's last line is `{type:"system", subtype:"turn_duration"}` (the turn-complete marker). This is the trickiest piece of the plugin — it has to be sure the transcript is fully flushed before parsing it.
2. **Extracts the last turn only** ([extractLastTurn](hooks/scripts/store-memories.js#L110-L191)) — walks back from EOF to find the previous `turn_duration` marker; everything after that is the current turn. Inside the turn:
   - User messages: keeps plain strings and `type:"text"` blocks. **Skips `tool_result`** (workflow noise, not user input).
   - Assistant messages: keeps `type:"text"` blocks only. **Skips `thinking` and `tool_use`** (internal reasoning + tool invocations).
3. **Two parallel `addMemory` calls** ([:206-238](hooks/scripts/store-memories.js#L206-L238)) — one for user content, one for assistant. Each gets `messageId` `u_<ts>` / `a_<ts>` (unused by v1 API but kept for backward compat). Whitespace-only content is skipped.
4. **Outputs** `💾 Memory saved (2) [user: 142, assistant: 1389]` on success, detailed error dump otherwise.

### 4. SessionEnd — [hooks/scripts/session-summary.js](hooks/scripts/session-summary.js)

Fires when the session ends (`/exit`, idle timeout, force quit).

1. **Reads transcript**, walks it once to find: first user prompt (up to 200 chars), last user prompt, turn count, first/last timestamps.
2. **Dedupes** by `sessionId` ([:95-105](hooks/scripts/session-summary.js#L95-L105)) — won't double-save.
3. **Appends one line** to `data/sessions.jsonl`: `{sessionId, groupId, summary, turnCount, reason, startTime, endTime, timestamp}`.
4. **No AI call** — purely local data extraction. Zero latency, zero cost.

### Why split SessionEnd + SessionStart (the deferred-display trick)

Documented at [README.md:536-583](README.md#L536-L583). When SessionEnd runs, the terminal is closing — any `systemMessage` would be lost. Instead SessionEnd writes to disk and **the next** SessionStart reads it and displays "Last session 2h ago: …" as a welcome-back banner. Local-first: works offline, no API cost, no cloud dependency for continuity.

## Config — [hooks/scripts/utils/config.js](hooks/scripts/utils/config.js)

Loaded once at startup of each hook script. Reads `.env` from plugin root, then env vars (env vars don't get overridden, [:23](hooks/scripts/utils/config.js#L23)).

| Variable | Default | Purpose |
|:--|:--|:--|
| `EVERMEM_API_KEY` | (required) | Bearer token |
| `EVERMEM_USER_ID` | `claude-code-user` | Used when no `group_id` |
| `EVERMEM_GROUP_ID` | auto-generated | Multi-tenant scoping |
| `EVERMEM_API_URL` | `https://api.evermind.ai` | EverCore base URL |
| `EVERMEM_DEBUG` | unset | Enables `/tmp/evermem-debug.log` |
| `EVERMEM_CWD` | from hook stdin | Project directory (set internally) |

### Auto-generated `groupId` ([:55-72](hooks/scripts/utils/config.js#L55-L72))

```
{first 4 chars of project name, lowercased, alphanumeric}
+ {first 5 chars of sha256(full_cwd_path)}
= 9-char groupId
```

So `~/Documents/api-server/` becomes something like `apis9a823`. Same path always produces same group → memories persist across sessions in that project. Different paths → different groups → memories stay isolated.

### `keyId` ([:95-102](hooks/scripts/utils/config.js#L95-L102))

`sha256(apiKey)[0:12]`. Used in `groups.jsonl` to associate locally-cached project metadata with a specific account, so switching keys naturally filters out the other account's projects.

---

# How it uses EverCore (in detail)

## API surface: 3 endpoints, all v1

All HTTP calls live in **one file**: [hooks/scripts/utils/evermem-api.js](hooks/scripts/utils/evermem-api.js).

| Function | HTTP | Endpoint | Used by |
|:--|:--|:--|:--|
| `searchMemories(query, opts)` | POST | `/api/v1/memories/search` | inject-memories.js, MCP `evermem_search`, `/search` command |
| `getMemories(opts)` | POST | `/api/v1/memories/get` | session-context.js, dashboard proxy |
| `addMemory(message)` | POST | `/api/v1/memories/group` (when groupId) **or** `/api/v1/memories` (personal) | store-memories.js |

All requests:
- Bearer auth: `Authorization: Bearer ${EVERMEM_API_KEY}`
- `Content-Type: application/json`
- Plain `fetch` against `https://api.evermind.ai`

**Note**: this plugin is on **v1** of the EverMind API, distinct from the v0 used by the Game of Thrones demo. The shape changed significantly — episodes nest under `data.episodes`, search uses `method` not `retrieve_method`, filters live inside the request body as `{group_id|user_id}` objects.

## `searchMemories` — [hooks/scripts/utils/evermem-api.js:22-93](hooks/scripts/utils/evermem-api.js#L22-L93)

### Request

```jsonc
POST /api/v1/memories/search
{
  "query": "<user prompt>",
  "method": "hybrid",                    // keyword|vector|hybrid|agentic
  "top_k": 15,
  "memory_types": ["episodic_memory"],
  "filters": { "group_id": "apis9a823" } // OR { "user_id": "..." }
}
```

30-second timeout via `AbortController` ([:11, :56-57](hooks/scripts/utils/evermem-api.js#L11-L57)). On any non-OK response, returns `{_debug: {url, status, error}}` rather than throwing — so hooks degrade silently.

### Response normalization ([transformSearchResults:101-128](hooks/scripts/utils/evermem-api.js#L101-L128))

EverCore returns `{data: {episodes: [...]}}` where each episode has `{id, user_id, session_id, timestamp, summary, subject, score, participants, group_id, memory_type}`. The transform flattens to:

```ts
{ text: summary, subject, timestamp, memoryType, score,
  metadata: { groupId, type, participants } }
```

Then sorts by `score` desc. Notably **only `summary` becomes `text`**; the longer `episode` field is ignored on the search path (it's used on the `get` path instead).

## `addMemory` — [hooks/scripts/utils/evermem-api.js:140-206](hooks/scripts/utils/evermem-api.js#L140-L206)

### Two URLs, one function

The function picks the endpoint based on whether a `groupId` is configured:

| Configured | URL | Body shape |
|:--|:--|:--|
| `groupId` set | `POST /api/v1/memories/group` | `{group_id, messages: [...], async_mode: true}` |
| no `groupId` | `POST /api/v1/memories` | `{user_id, messages: [...], async_mode: true}` |

Because the plugin always auto-generates a `groupId` from cwd, the `/group` endpoint is the path actually used in practice. The personal endpoint is the fallback when `EVERMEM_GROUP_ID=""` is forced empty.

### Message shape

```jsonc
{
  "sender_id": "claude-assistant" | "<userId>",  // role-based
  "role": "user" | "assistant",
  "timestamp": <Date.now()>,                       // ms epoch, not ISO!
  "content": "<text>"
}
```

`async_mode: true` is interesting — the plugin doesn't wait for EverCore's full extraction+indexing pipeline to finish, just for the ack that the message is queued. That keeps the Stop hook fast (it has a 30 s timeout but typically returns in <1 s).

### Return value

Returns `{url, body, status, ok, response}` (no throw on HTTP errors) — this is what powers store-memories.js's detailed error dump when things go wrong ([store-memories.js:278-291](hooks/scripts/store-memories.js#L278-L291)).

## `getMemories` — [hooks/scripts/utils/evermem-api.js:216-258](hooks/scripts/utils/evermem-api.js#L216-L258)

```jsonc
POST /api/v1/memories/get
{
  "memory_type": "episodic_memory",
  "filters": { "group_id": "apis9a823" },
  "page": 1,
  "page_size": 100,
  "rank_by": "timestamp",
  "rank_order": "desc"
}
```

Unlike `searchMemories`, this one **does throw** on non-OK. Used only by session-context.js to fetch recent memories at session start (no query, just chronological).

[transformGetMemoriesResults](hooks/scripts/utils/evermem-api.js#L265-L280) is different from the search transform: it prefers the full `episode` field over `summary`, sorts by `timestamp` desc, and drops `score`/`participants` since they aren't relevant for the recency display.

## Multi-tenant scoping recap

Three orthogonal IDs interact:

| ID | Scope | Origin |
|:--|:--|:--|
| `userId` | account-wide fallback | `EVERMEM_USER_ID` env, default `claude-code-user` |
| `groupId` | per-project bucket | hash of cwd, 9 chars |
| `keyId` | identifies which account owns local data | `sha256(apiKey)[0:12]` |

EverCore's actual data isolation is server-side via the `Bearer` token. The local `keyId` is only used to filter `groups.jsonl` and `sessions.jsonl` when you switch API keys on the same machine.

---

## The other surfaces (briefly)

### MCP tool — [mcp/server.js](mcp/server.js)

Stdio JSON-RPC server exposing one tool: `evermem_search({query, limit})`. Defaults `limit=10`, capped at 20. Imports the **same** `searchMemories` from `hooks/scripts/utils/evermem-api.js`, so it's just a different access channel to the same client. Returns results as a **markdown table** (`| # | Score | Date | Summary |`) — token-efficient — instead of the XML block the prompt-submit hook injects. Pair this with the `alwaysInclude: true` [skills/memory-tools.md](skills/memory-tools.md) skill that tells Claude when to call it.

### Dashboard proxy — [server/proxy.js](server/proxy.js)

A 195-line local HTTP server on `localhost:3456` that:
- **Proxies** `POST /api/v1/memories/search` and `/api/v1/memories/get` straight to `https://api.evermind.ai` ([:107-143](server/proxy.js#L107-L143)), preserving the Authorization header. This exists because the browser-side dashboard can't add custom CORS headers, so the proxy adds them.
- **Serves** `/api/groups` ([:152-170](server/proxy.js#L152-L170)) — reads `data/groups.jsonl`, filters by `computeKeyId(authToken)`, returns project list for the dashboard's left-sidebar picker.
- **Serves** `dashboard.html` at `/`.

Started by the `/evermem:hub` command. The dashboard itself is a single self-contained HTML file with the activity heatmap, project cards, timeline view, etc.

### Slash commands — [commands/](commands/)

| Command | What |
|:--|:--|
| `/evermem:help` | Status check (env vars set? API reachable?) |
| `/evermem:search <query>` | Manual memory search via [commands/scripts/search-memories.js](commands/scripts/search-memories.js) |
| `/evermem:ask <q>` | Combined memory-search + Claude answer |
| `/evermem:hub` | Starts proxy.js, opens browser to dashboard |
| `/evermem:debug` | Tails `/tmp/evermem-debug.log` |
| `/evermem:projects` | Lists projects from `data/groups.jsonl` |

## What's notably absent

- **No SDK** — pure `fetch`. Easy to grok, no version-coupling to a client library.
- **No streaming** — every API call is a one-shot POST. Hooks complete in under a second typically.
- **No `clear` / `delete`** — the plugin has no way to remove memories; you'd use the dashboard or hit the API directly.
- **No conversation threading** — each Stop hook fires `addMemory` for the last turn only. The server pipeline is responsible for building the longer-term `EpisodicMemory` records from the stream of `MemoryEvent`-style messages. The plugin is a thin pipe.

## How this compares to the other two use cases

| | Game of Thrones | MemoCare | This plugin |
|:--|:--|:--|:--|
| API version | v0 | (Swift SDK over v1) | v1 (direct) |
| Read path | search | search + get | search + get |
| Write path | bulk load (separate script) | memorize (chat + sensors) | addMemory (every turn) |
| Group scoping | `'asoiaf'` (hardcoded) | device-augmented IDs | path-hash per project |
| Retrieval method | `hybrid` (default) | (SDK default) | `hybrid` (default) |
| Memory types filter | none | none | `['episodic_memory']` |
| Failure mode | empty array → degrade | nil client → no-op | exit(0) → silent skip |

## Key files (quick index)

| File | What |
|:--|:--|
| [plugin.json](plugin.json) | Plugin manifest |
| [hooks/hooks.json](hooks/hooks.json) | Hook wiring (4 lifecycle events → 4 scripts) |
| [hooks/scripts/session-context.js](hooks/scripts/session-context.js) | SessionStart: load recent memories + last session summary |
| [hooks/scripts/inject-memories.js](hooks/scripts/inject-memories.js) | UserPromptSubmit: search + inject as additionalContext |
| [hooks/scripts/store-memories.js](hooks/scripts/store-memories.js) | Stop: parse transcript, save last turn |
| [hooks/scripts/session-summary.js](hooks/scripts/session-summary.js) | SessionEnd: save session summary to local JSONL |
| [hooks/scripts/utils/evermem-api.js](hooks/scripts/utils/evermem-api.js) | The HTTP client (search + get + add) |
| [hooks/scripts/utils/config.js](hooks/scripts/utils/config.js) | Env / .env loading, groupId hash, keyId hash |
| [hooks/scripts/utils/groups-store.js](hooks/scripts/utils/groups-store.js) | `data/groups.jsonl` reader/writer |
| [mcp/server.js](mcp/server.js) | MCP stdio server (one tool: evermem_search) |
| [skills/memory-tools.md](skills/memory-tools.md) | Always-included agent skill |
| [server/proxy.js](server/proxy.js) | Local web proxy for dashboard |
| [assets/dashboard.html](assets/dashboard.html) | Memory Hub UI |
| [install.sh](install.sh) | One-line install (registers marketplace, sets API key) |
| [README.md](README.md) | 1100-line user + developer docs |
