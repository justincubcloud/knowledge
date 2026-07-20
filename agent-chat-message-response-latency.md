# Per-Message Agent Response Latency — Development Strategy

> Roadmap item **#4** from `agent-chat-features-roadmap.md`  
> Architecture baseline: `agent-chat-system-architecture.md`  
> Sibling pattern: `agent-chat-context-buffer.md` (same messagemeta + `stream_end` + HMAC save pipeline)  
> Goal: record wall-clock duration of each agent turn (ms), persist on the assistant message, expose via APIs / ChatV2 / admin so latency is inspectable per conversation and model  
> Status: **Not started** — ship early so historical timing data starts accruing  
> Last updated: 2026-07-20

---

## 1. Problem

Agent turns already persist:

- `token_usage` (prompt / completion / total) on assistant messages  
- `context_buffer` (window / used / ratio) from Phase A of roadmap #5  

Relay already logs per-tool elapsed time to stderr (`tool_start` → `Date.now() - tool_start` in `relay.ts`), and WS emits `stream_start` → deltas → `stream_end` — but we do **not**:

1. Capture **end-to-end turn latency** (prep + LLM rounds + tools + stream finish)  
2. Persist that number on the assistant message in `wp_cub_chat_messagemeta`  
3. Return it on message REST responses or show it in ChatV2 / admin analytics  

Result: slow turns are felt in the UI but invisible in history, support triage, and model/provider comparison. Once multimodal (#2), HITL approvals (#10), and background jobs (#13) land, latency variance will grow — we need a baseline before those inflate the signal.

---

## 2. Product definition

| User-facing / ops behavior | Detail |
|---|---|
| Per-message duration | Every completed agent assistant message stores **response latency in milliseconds** |
| Scope of the clock | From **turn start** on the relay (WS `chat` handler / `stream_start` boundary) through **final assistant content ready** (`stream_end` / HMAC save) — **includes** tool rounds |
| API exposure | Latency available on message list/detail (same path that already hydrates `token_usage`) |
| ChatV2 (optional UI) | Subtle per-bubble duration next to existing token usage (e.g. `1.2s`) — not a dashboard |
| Admin / analytics | Conversation and model-level aggregates (p50 / p95 / mean) using the new meta |
| Guest | Same persistence on guest assistant saves so guest UX and quotas are comparable |

Non-goals for v1:

- Client-side RTT / browser paint latency as the source of truth (relay wall-clock only)  
- Full OpenTelemetry spans / external APM (nice later; structured logs + messagemeta first)  
- Changing LLM gateways or adding artificial timeouts beyond existing behavior  
- Per-token streaming “time to first byte” as a required UI metric (optional Phase B field)  
- User↔user (non-agent) message latency  

---

## 3. Architectural invariants (do not break)

```
cubcloud.ai            → optional bubble display + types from stream_end / REST
cub-agent-chat-relay   → owns the stopwatch; attaches latency on stream_end + HMAC body
cub-chat-wp            → persists response_latency messagemeta; exposes on message APIs
cub-agents-wp          → unchanged for v1
```

- WordPress remains the persistence boundary.  
- Relay remains the intelligence plane — the only place that sees prep + tools + multi-round LLM.  
- Next.js does **not** invent latency from `Date.now()` around `sendMessage` as the stored value (that would mix network RTT and miss tool work on reconnect). Client timing may be used for UX-only debug later.  
- Same HMAC save path as `usage` / `context` — no parallel channel.  
- Credentials and LLM routing unchanged.

---

## 4. What already exists (reuse)

| Asset | Location | Reuse |
|---|---|---|
| Per-turn usage normalize/accumulate | `cub-agent-chat-relay/src/token-usage.ts` | Sibling module or extend with latency payload helpers |
| `stream_start` / `stream_end` + HMAC save | `ws-server.ts` | Start clock at `stream_start` (or immediately before); attach `latency_ms` on `stream_end` + save body |
| Tool-round local timing | `relay.ts` (`tool_start` / `tool_ms` logs) | Optional Phase B breakdown; proves wall-clock pattern |
| `token_usage` + `context_buffer` messagemeta | `cub-chat-wp/includes/message-meta.php` | Add `response_latency` to allowed keys + normalize/save/parse |
| HMAC save body | `relay-chat-save.php`, guest save | Accept `latency` / `latency_ms` alongside `usage` / `context` |
| Message meta batch hydrate | `cub_chat_get_message_meta_batch` + message API attach | Include latency when attaching usage |
| ChatV2 `usageByMessageId` | `ChatV2Interface.tsx` | Sibling `latencyByMessageId` or fold into bubble props |
| `useViresAgentWs` `onStreamEnd(…, usage?, …, context?)` | `hooks/useVireschatRelay.ts` | Extend with `latencyMs?` |
| Admin token usage sums | `cub_chat_sum_conversation_token_usage()` | Pattern for conversation latency aggregates |
| Structured logging baseline | relay `logger` + Vercel/WP runtime logs | Log `latency_ms` on each turn for ops without waiting on UI |

---

## 5. Core concepts

### 5.1 Definition: `latency_ms`

Single integer: **wall-clock milliseconds** for one agent turn on the relay.

```
t0 = moment turn work begins on relay
     (after auth/quota OK; ideally at/just before stream_start emit)
t1 = moment final assistant text is complete and stream_end is about to emit
     (after last LLM round + tools; before or as HMAC save fires)

latency_ms = max(0, round(t1 - t0))
```

**Includes:** LLM config resolve (if inside turn), RAG / tool-knowledge fetch, history build, all tool rounds (up to `MAX_ROUNDS`), all streamed LLM rounds, vision path when used.

**Excludes:** Client WebSocket connect/auth, browser render, WordPress HMAC round-trip after `t1` (save may run in parallel or just after — do not wait on WP HTTP for the clock stop unless save is on the critical path before `stream_end`).

**On error / abort:** If the turn fails after `t0`, still record `latency_ms` when an assistant error frame or partial save occurs (mark `status: "error"` in optional Phase B object). Quota-exceeded early exits before work → omit latency (no meaningful turn).

### 5.2 Storage shape (v1)

Prefer a small JSON object for forward compatibility (matches `token_usage` / `context_buffer`), not a bare integer — while keeping the primary field obvious:

```json
{
  "latency_ms": 4820,
  "source": "relay_wall"
}
```

Meta key: **`response_latency`**.

Allowed via `cub_chat_allowed_message_meta_keys`. Normalize in PHP the same way as token usage (coerce number, reject negatives / non-finite).

### 5.3 Optional breakdown (Phase B — not required to ship)

```json
{
  "latency_ms": 4820,
  "source": "relay_wall",
  "ttft_ms": 610,
  "llm_ms": 3100,
  "tools_ms": 1500,
  "prep_ms": 220,
  "rounds": 2,
  "tool_calls": 3
}
```

`ttft_ms` = time from `t0` to first `stream_delta` (or first non-empty content). Useful for “feels slow to start” vs “slow tools”. Keep v1 to `latency_ms` only so persistence ships this week.

### 5.4 Aggregates (analytics)

| Aggregate | Formula | Use |
|---|---|---|
| Conversation mean / p50 / p95 | Over assistant messages with `response_latency` | Support + admin conversation view |
| By model | Join message → conversation `agent_id` / saved model if available | Compare providers |
| By agent | Group on `agent_id` | Slow-integration detection |

v1 can ship **per-message persist + API field**; admin charts are Phase B if they block shipping.

---

## 6. Protocol changes

### 6.1 WebSocket frames (agent `/ws`)

Extend existing frames; no new required frame type.

**`stream_end`:**

```json
{
  "type": "stream_end",
  "content": "…",
  "usage": { "prompt_tokens": 42000, "completion_tokens": 800, "total_tokens": 42800 },
  "context": { "window": 131072, "used": 42000, "ratio": 0.32, "source": "measured" },
  "latency_ms": 4820
}
```

Frontend: pass through `useViresAgentWs` → ChatV2 bubble / state. Older clients ignore unknown fields.

### 6.2 HMAC save body (relay → WP)

Extend `/relay/save-message` and `/relay/save-guest-message`:

```json
{
  "usage": { … },
  "context": { … },
  "latency": { "latency_ms": 4820, "source": "relay_wall" }
}
```

Accept aliases: top-level `latency_ms` number **or** `latency` object — normalize to `response_latency` meta.

### 6.3 Message REST (cub-chat-wp)

When hydrating messages for list/detail (same batch path as `token_usage`):

```json
{
  "id": 12345,
  "message": "…",
  "token_usage": { … },
  "context_buffer": { … },
  "response_latency": { "latency_ms": 4820, "source": "relay_wall" }
}
```

Keep field optional when meta missing (historical messages).

### 6.4 HTTP `/relay/chat` parity

Non-streaming JSON path should include the same `latency_ms` in the response body and on HMAC save so scheduled / mcp-proxy turns accrue data too. Share one helper with WS.

---

## 7. Measurement algorithm (relay)

Instrument in `ws-server.ts` chat handler (and shared helper used by `/relay/chat`):

```
1. After quota/auth OK and conversation resolved:
     turnStartedAt = Date.now()
2. Emit stream_start (existing)
3. Run relay_chat_stream / tool loop as today
4. Just before stream_end:
     latency_ms = Math.max(0, Date.now() - turnStartedAt)
5. Attach latency_ms to stream_end payload
6. Pass latency into HMAC save params (alongside usage + context)
7. Structured log: { msg: "turn_complete", conversationId, agentId, model?, latency_ms, rounds?, usage? }
```

Rules:

- One clock per user turn — do not reset between tool rounds.  
- Vision dual-write path: same clock around the vision LLM call in that turn.  
- Guest: identical.  
- If `stream_end` fires with `quotaExceeded` before LLM work, omit or set `latency_ms: 0` with no meta save.  
- Never block streaming on latency persistence — compute locally; HMAC save already async/fire-and-forget style where possible.

Clock source: `Date.now()` is enough (same as existing tool logs). Prefer `performance.now()` only if you need sub-ms mono clocks inside one process; store still as integer ms.

---

## 8. Frontend (cubcloud.ai)

### 8.1 Types / hook

- `lib/vireschatRelay.ts`: extend `stream_end` with `latency_ms?: number`; optional `ResponseLatency` type mirroring meta.  
- `useViresAgentWs`: extend `onStreamEnd` signature with `latencyMs?: number` (or pass full object).  
- ChatV2: map latency onto the streaming/persisted assistant bubble id the same way as `usageByMessageId`.

### 8.2 Bubble UI (optional Phase A polish)

- Show compact duration under/beside token usage when present (`4820ms` → format as `4.8s` if ≥1000).  
- No cards, no stats strip — one quiet secondary line on agent bubbles only.  
- Guest / widget: inherit frame data; UI optional.

### 8.3 REST hydrate

When loading conversation history, if API returns `response_latency`, seed the same map so refresh does not lose labels.

---

## 9. Phased delivery

### Phase A — Capture & persist (ship first)

**Outcome:** Every successful agent assistant save has `response_latency`; `stream_end` carries `latency_ms`; message APIs expose it.

1. Relay: turn stopwatch in WS (+ shared helper for `/relay/chat`)  
2. Attach `latency_ms` on `stream_end` + HMAC body  
3. WP: allow + normalize + save `response_latency` meta (auth + guest)  
4. WP: hydrate on message responses (batch with token_usage)  
5. Frontend types + hook; ChatV2 optional bubble display  
6. Structured turn_complete log line  

**Exit criteria:** New agent replies in staging show meta in DB / REST; ChatV2 can display duration when wired; historical messages without meta remain valid.

### Phase B — Breakdown & analytics

1. Optional `ttft_ms` / `tools_ms` / `llm_ms` / `prep_ms` on the same meta object  
2. Admin conversation latency summary (p50/p95) next to token totals  
3. Account / hosting usage surfaces: latency trend by model (reuse token-usage UI patterns lightly)  
4. Alerting hooks: log warn when `latency_ms` > threshold (e.g. 60s) for ops  

**Exit criteria:** Ops can answer “is this model/tool path getting slower?” from admin or logs without reading stderr only.

### Phase C — Observability handoff

1. Optional OpenTelemetry / drain export of turn spans (Vercel Drains / external APM) — **do not block** Phase A  
2. Correlate `latency_ms` with `context_buffer.ratio` and tool count for slow-turn diagnosis  
3. Align mobile / cubden-web consumers if they render agent bubbles  

---

## 10. Work breakdown by repo

| Repo | Phase A | Phase B |
|---|---|---|
| `cub-agent-chat-relay` | Stopwatch; `stream_end.latency_ms`; pass on save; HTTP parity; log | Breakdown fields; warn thresholds |
| `cub-chat-wp` | `response_latency` allowlist + normalize/save/parse; relay save accept; message API hydrate | Admin aggregates; optional analytics endpoints |
| `cubcloud.ai` | Types, hook, ChatV2 bubble (optional but cheap) | Richer formatting / filters if needed |
| `cub-agents-wp` | None | None |

---

## 11. Suggested implementation order (concrete files)

1. `cub-agent-chat-relay/src/ws-server.ts` — `turnStartedAt` at stream start; `latency_ms` on `stream_end` + `saveRelayMessage` / guest save params  
2. `cub-agent-chat-relay/src/relay.ts` — ensure HTTP `/relay/chat` return includes `latency_ms` (shared helper)  
3. `cub-chat-wp/includes/message-meta.php` — allow `response_latency`; `cub_chat_normalize_response_latency` / save / parse; batch hydrate  
4. `cub-chat-wp/public/api/relay-chat-save.php` (+ guest save) — accept `latency` / `latency_ms` into `$message_meta`  
5. Message list/detail serializers that already attach `token_usage` — attach `response_latency`  
6. `cubcloud.ai/lib/vireschatRelay.ts` + `hooks/useVireschatRelay.ts` + `ChatV2Interface.tsx` / `ChatBubble.tsx` — types + display  

---

## 12. Thresholds & knobs

| Knob | Default | Where |
|---|---|---|
| Persist always when turn completes | yes | Relay |
| Omit on pre-work quota reject | yes | Relay |
| Log warn if `latency_ms` ≥ | `60000` (optional Phase B) | Relay const |
| UI format | `<1000` → `Nms`; else one decimal seconds | Frontend |
| Meta key | `response_latency` | WP |

No WP options required for v1.

---

## 13. Testing plan

### Unit (relay)
- Clock: mock `Date.now` sequence → expected `latency_ms`  
- Zero/negative guard (`max(0, …)`)  
- Quota early exit does not write meta  

### Unit (WP)
- Normalize: number, object, aliases, reject garbage  
- Save upsert; allowed keys include `response_latency`  
- Batch hydrate mixed messages (some with meta, some without)  

### Integration
- WS turn with tools → `stream_end.latency_ms` > tool-only log sum (prep + LLM included)  
- HMAC body stores meta; REST message returns it  
- Guest save path mirrors auth  
- `/relay/chat` JSON includes latency  

### Manual / browser (ChatV2)
- Send message → bubble shows duration after `stream_end`  
- Reload conversation → duration still present from REST  
- Fast vs tool-heavy turns show plausible relative values  

### Regression
- `token_usage` / `context_buffer` still save  
- Vision dual-write unchanged  
- User↔user Pusher threads unchanged  

---

## 14. Risks & mitigations

| Risk | Mitigation |
|---|---|
| Clock stops after HMAC save → inflated latency | Stop at `stream_end` content-ready; do not await WP for `t1` |
| Client measures RTT and overwrites server value | Server meta is source of truth; client display uses relay/REST only |
| Partial turns / errors skip persist | Persist latency on error path when any assistant row is saved; document omissions |
| Clock skew / process sleep | Wall-clock is intentional for UX; mono clock optional later for internal segments |
| Meta key sprawl | One JSON object `response_latency`, extend in place for Phase B |
| REST path forgets latency | Shared save helper; Phase A exit checklist includes `/relay/chat` |
| UI noise | Keep duration secondary; hide under a details toggle if bubble clutter appears |

---

## 15. Success metrics

- **≥95%** of new agent assistant messages (WS + HTTP relay) in production have `response_latency` within one release after ship  
- Support can answer “how long did that reply take?” from admin/message API without SSH logs  
- p95 latency by model visible within Phase B (or raw SQL/meta export until then)  
- No increase in failed saves / WS errors attributable to the feature  

---

## 16. Handoff to later roadmap items

| Later item | How latency helps |
|---|---|
| **#2 Full multimodal** | Quantify cost of vision+tools turns vs text-only |
| **#3 Relay-backed voice** | Compare voice turn duration to text baseline |
| **#5 Context compact** | Detect whether compaction adds unacceptable prep time (`prep_ms` / total) |
| **#10 HITL approvals** | Separate “waiting on human” from agent compute (Phase B+: pause clock or mark `status`) |
| **#13 Background jobs** | Job completion latency is a sibling metric — reuse meta shape where jobs post chat messages |

Do **not** block this item on OTel or admin charts — persistence first.

---

## 17. Decision log (defaults)

1. **Relay wall-clock is canonical** — not browser RTT.  
2. **Measure full tool loop**, not first token only (TTFT is Phase B additive).  
3. **Same pipeline as usage/context** — `stream_end` + HMAC + messagemeta.  
4. **Ship Phase A immediately** so data accrues before multimodal / voice inflate variance.  
5. **JSON meta object** with `latency_ms` primary field for forward-compatible breakdowns.  
6. ChatV2 display is valuable but **optional** relative to persist+API; do both if cheap in the same PR.

---

## 18. Open questions (resolve during Phase A)

1. Should `t0` be exactly at `stream_start` emit, or earlier (include agent/config fetch)? **Recommend:** start when the chat handler begins turn work after auth/quota — includes prep that users wait on.  
2. On multi-round tool turns, is one total enough for v1? **Yes** — breakdown in Phase B.  
3. Store on user message as well? **No** — assistant message only (the agent response).  
4. Include HMAC save HTTP time? **No** for v1 user-facing latency.  
5. cubden-mobile / cubden-web in Phase A? **No** — API field is enough; they inherit when ready.

---

*When implementation starts, update this doc’s Status line and link PRs against Phase A / B exit criteria.*
