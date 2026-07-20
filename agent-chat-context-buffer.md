# Chat Context Buffer — Development Strategy

> Roadmap item **#4** from `agent-chat-features-roadmap.md`  
> Architecture baseline: `agent-chat-system-architecture.md`  
> Goal: show session context fullness (% used / % remaining) and auto-compact near the limit so long tool / multimodal threads stay predictable  
> Status: **Phase A implemented** (visibility meter) — Phase B (auto-compact) not started  
> Last updated: 2026-07-19

---

## 1. Problem

Agent turns already accumulate:

- System prompt + date + RAG + tool-knowledge + tool defs  
- Up to **20** recent history messages (`HISTORY_LIMIT` in relay `ws-server.ts`)  
- Up to **5** tool rounds per turn (`MAX_ROUNDS`)  
- Upcoming multimodal history (roadmap #2) will grow prompt tokens much faster  

Today we persist **per-turn token usage** (`prompt_tokens` / `completion_tokens` / `total_tokens`) on assistant messages, and ChatV2 can show per-bubble usage — but we do **not**:

1. Know the model’s **context window** for the resolved provider/model  
2. Estimate **session fill** before / after a turn  
3. Compact older turns when the window is nearly full  

Result: long chats fail silently (gateway truncation / 4xx) or drop older detail without the user knowing why.

---

## 2. Product definition

| User-facing behavior | Detail |
|---|---|
| Context meter | In authenticated agent chat (ChatV2), show **% of context window used** for the current conversation + selected model |
| Threshold warning | Soft warning around **~80%**; stronger signal approaching **~95%** |
| Auto-compact | At **≥95%** (or when the next turn would exceed ~90%), relay **summarizes older turns**, keeps a recent verbatim tail, persists the summary, and continues |
| Transparency | After compact, UI shows a subtle “Conversation compacted” system note (not a fake assistant message) |
| Guest | Meter optional/simplified; auto-compact still runs on relay so guests don’t hard-fail |

Non-goals for v1:

- Cross-session durable memory (roadmap **#5**)  
- Manual “compact now” as the only path (nice-to-have after auto)  
- Exact byte-perfect token accounting for every provider (estimate + measured last prompt is enough)  
- Changing LiteLLM / Bifrost / xAI themselves  

---

## 3. Architectural invariants (do not break)

```
cubcloud.ai     → UX meter + compact notice
cub-agent-chat-relay → owns estimate, threshold, compaction LLM call, history rewrite
cub-chat-wp     → persists summaries, context snapshots, token_usage (already)
cub-agents-wp   → unchanged for v1 (unless catalog needs model window metadata later)
```

- WordPress remains the persistence boundary.  
- Relay remains the intelligence plane (same place that already loads history + calls the LLM).  
- Next.js does not invent its own context math from raw message text alone.  
- Credentials stay in WP; compaction uses the same `/llm-config` path as chat.

---

## 4. What already exists (reuse)

| Asset | Location | Reuse |
|---|---|---|
| Per-turn usage normalize/accumulate | `cub-agent-chat-relay/src/token-usage.ts` | Extend with session/context helpers |
| Usage on `stream_end` + HMAC save | `ws-server.ts`, `relay-chat-save.php` | Add `context` payload alongside `usage` |
| `token_usage` messagemeta | `cub-chat-wp/includes/message-meta.php` | Keep; add sibling meta keys for context |
| Conversation token sum (admin) | `cub_chat_sum_conversation_token_usage()` | Pattern for session totals API |
| ChatV2 `usageByMessageId` | `ChatV2Interface.tsx` | Sibling state for `contextBuffer` |
| `useViresAgentWs` `onStreamEnd(usage)` | `hooks/useVireschatRelay.ts` | Extend callback with `context` |
| History cache (last 20 msgs) | `ws-server.ts` `HISTORY_LIMIT` | Compaction rewrites this cache + WP |
| LLM resolve via `/llm-config` | cub-chat-wp + relay | Add `context_window` to response |

---

## 5. Core concepts

### 5.1 Context window (`W`)

Max input tokens for the **resolved** model from `cub_chat_resolve_llm()`.

Resolution order for `W`:

1. Catalog / provider metadata if present (`context_window` on model row or LiteLLM `/models` field)  
2. Curated override map in WP option `cub_llm_context_windows` keyed by catalog id (`provider:model`)  
3. Safe default by provider family (e.g. 128k / 32k) — **documented fallback, never silent 0**

Expose on `GET /cubchat/v1/llm-config` (and optionally `/model-providers`) as:

```json
{
  "provider": "litellm",
  "model": "…",
  "context_window": 131072
}
```

### 5.2 Measured vs estimated fill

| Signal | Source | Role |
|---|---|---|
| `last_prompt_tokens` | Last LLM round `usage.prompt_tokens` | Ground truth for “how full was the last request” |
| `estimated_tokens` | Relay estimator before call | Pre-flight gate + UI when no prior usage |
| `used_ratio` | `used / context_window` | Meter + thresholds |

**v1 estimator (good enough):**

```
estimate ≈ tokens(system_messages)
         + tokens(history text)
         + tokens(new user message)
         + tokens(tool JSON schemas)   // rough: chars/4 or tiktoken-lite
         + reserve_for_completion      // e.g. min(4096, 10% of W)
```

Prefer a small shared tokenizer util in the relay (e.g. `js-tiktoken` / `gpt-tokenizer` with cl100k fallback). Exact model tokenizer matching is phase-2 polish.

**Important:** Session meter should track **prompt fill for the next/last turn**, not cumulative billing totals across the conversation. Cumulative `total_tokens` sum is cost analytics, not context buffer.

### 5.3 Compaction artifact

When compacting, produce a durable **thread summary** stored in WP:

| Field | Storage |
|---|---|
| Summary text | Conversation meta or a special system message (`type` / messagemeta `role=context_summary`) |
| Covered message id range | meta: `context_summary_from_id`, `context_summary_to_id` |
| Created at / model used | meta |
| Token estimate of summary | meta |

Relay history after compact becomes:

```
[system: thread summary]
[… last N verbatim messages …]   // N chosen so estimate << W (e.g. keep ~20–30% of W)
[new user turn]
```

Older verbatim messages remain in MySQL for UI scrollback; they are simply **omitted from the model context** until/unless user expands (out of v1).

---

## 6. Protocol changes

### 6.1 WebSocket frames (agent `/ws`)

Extend existing frames; avoid a parallel channel.

**`ready`** (after auth):

```json
{
  "type": "ready",
  "agentId": 12,
  "history": […],
  "context": {
    "window": 131072,
    "used": 18400,
    "ratio": 0.14,
    "source": "estimate"
  }
}
```

**`stream_end`** (after each turn):

```json
{
  "type": "stream_end",
  "content": "…",
  "usage": { "prompt_tokens": 42000, "completion_tokens": 800, "total_tokens": 42800 },
  "context": {
    "window": 131072,
    "used": 42000,
    "ratio": 0.32,
    "source": "measured",
    "compacted": false
  }
}
```

**New optional frame** when compact runs mid-turn prep:

```json
{
  "type": "context_compact",
  "summaryPreview": "Earlier you discussed …",
  "context": { "window": 131072, "used": 12000, "ratio": 0.09, "source": "estimate", "compacted": true }
}
```

Frontend: handle in `useViresAgentWs`; ChatV2 updates meter + shows compact notice.

### 6.2 HMAC save body (relay → WP)

Extend `/relay/save-message` (assistant) with optional:

```json
{
  "usage": { … },
  "context": {
    "window": 131072,
    "used": 42000,
    "ratio": 0.32,
    "compacted": false
  }
}
```

Persist as messagemeta `context_buffer` (JSON) on the assistant message — same pattern as `token_usage`.

### 6.3 Compaction persist API (new, small)

`POST /cubchat/v1/conversations/{id}/context-summary` (HMAC or Basic owner):

- Body: `{ summary, from_message_id, to_message_id, model, token_estimate }`  
- Writes conversation-level meta (or a single system row)  
- Idempotent replace of “active” summary for that conversation  

Relay calls this when compact succeeds, **before** continuing the user turn.

---

## 7. Compaction algorithm (relay)

Trigger in `prepare_turn` / WS chat handler **before** `call_llm_stream`:

```
1. Resolve llm + context_window W
2. Build candidate messages (system + history + user [+ images])
3. estimated = estimate_tokens(candidate) + completion_reserve
4. If estimated / W < 0.95 → proceed (emit context on stream_end from measured usage)
5. Else COMPACT:
   a. Split history into head (older) + tail (recent)
      - Tail: newest messages until estimate(tail) ≈ 0.25–0.35 * W
      - Head: everything else (must be non-empty)
   b. Summarize head with same llm-config (no tools, low max_tokens)
      Prompt: preserve decisions, entities, tool outcomes, user prefs; drop chit-chat
   c. Persist summary via WP context-summary API
   d. Replace session historyCache with [summary_as_systemish_or_user_context] + tail
   e. Emit context_compact frame
   f. Re-estimate; if still ≥ 0.95, shrink tail further once; then proceed or error gracefully
6. Run normal tool loop
7. On stream_end, set context.used from last measured prompt_tokens (best signal)
```

Rules:

- Never compact away the **current** user message.  
- Vision turns: count image tokens with a fixed surcharge per image (provider-agnostic table) until full multimodal (#2) lands.  
- Guest: same algorithm; persist via guest save path / guest conversation meta.  
- Compaction LLM usage is recorded as its own assistant or system meta turn for analytics (optional v1.1).

---

## 8. Frontend (cubcloud.ai)

### 8.1 Meter UI (ChatV2 only for v1)

- Place near model selector / composer (one glance, not a dashboard card).  
- Show `Math.round(ratio * 100)%` used; tooltip: `used / window` tokens.  
- States: normal / warn (≥80%) / critical (≥95% or compacting).  
- On `context_compact`, toast or inline chip: “Older messages summarized to free context.”  

Keep styling in existing chat-v2 language (no new design system). Prefer a slim progress or numeric badge — not a stats strip of cards.

### 8.2 Hook / types

- Extend `TokenUsage` neighbor type `ContextBuffer` in `lib/vireschatRelay.ts`.  
- `onStreamEnd(content, usage?, quotaExceeded?, context?)`  
- `onContextCompact?(context)`  
- Seed meter from `ready.context` when present; fall back to last measured until first turn.

### 8.3 Scrollback vs model context

UI continues to load full conversation from WP REST. Compact only affects **what the relay sends to the LLM**. Optionally badge messages that are “outside active context” later (v1.1).

---

## 9. Phased delivery

### Phase A — Visibility (ship first) ✅

**Outcome:** Accurate-enough meter; no auto-compact yet.

1. ✅ Add `context_window` to LLM catalog / `/llm-config` (+ curated option `cub_llm_context_windows`, default 128k)  
2. ✅ Relay: estimate + attach `context` on `ready` and `stream_end` (peak measured `prompt_tokens` across tool rounds)  
3. ✅ Persist `context_buffer` messagemeta on HMAC save (auth + guest)  
4. ✅ ChatV2 meter + hook types (`ContextBufferMeter` beside model selector)  

**Exit criteria:** Authenticated agent chat shows % that moves after each turn; matches last `prompt_tokens / window` within reasonable error when usage is present.

**Key files shipped:**
- `cub-chat-wp/includes/llm-catalog.php` — `cub_chat_resolve_context_window()`
- `cub-chat-wp/public/api/llm-config.php` / `llm-models.php` — expose `context_window`
- `cub-chat-wp/includes/message-meta.php` — `context_buffer` meta
- `cub-agent-chat-relay/src/context-buffer.ts` — estimator + payload builder
- `cub-agent-chat-relay/src/relay.ts` / `ws-server.ts` — ready + stream_end + save
- `cubcloud.ai/components/chat/ContextBufferMeter.tsx` + ChatV2 wiring

### Phase B — Auto-compact

**Outcome:** Sessions survive past ~95% without hard failure.

1. Compaction summarizer in relay (no tools)  
2. WP `context-summary` persist endpoint + conversation meta  
3. Rewrite `historyCache`; emit `context_compact`  
4. UI compact notice  
5. Guardrails: max 1 compact per turn; fail open with clear `error` if summarize fails  

**Exit criteria:** Synthetic long thread (tool-heavy) auto-compacts and continues; summary visible in admin/meta; meter drops after compact.

### Phase C — Harden & handoff to #5

1. Better tokenizer / per-provider image token table  
2. Expose active summary in REST conversation detail for clients  
3. “Outside context” message dimming (optional)  
4. Design summary schema so roadmap **#5 Long-Context Memory** can promote summaries into durable knowledge without a rewrite  
5. Align REST `/relay/chat` path with same prepare_turn compact logic (parity with WS)

---

## 10. Work breakdown by repo

| Repo | Phase A | Phase B |
|---|---|---|
| `cub-chat-wp` | `context_window` on llm-config + model-providers; save `context_buffer` meta; curated window option | `POST …/context-summary`; conversation meta getters |
| `cub-agent-chat-relay` | Estimator module; attach context on ready/stream_end; pass through save | Compact pipeline in WS (+ shared helper for `/relay/chat`) |
| `cubcloud.ai` | Types, hook, ChatV2 meter | Compact frame + notice |
| `cub-agents-wp` | None (unless agent default model should advertise window in chat-context later) | None |

---

## 11. Suggested implementation order (concrete files)

1. `cub-chat-wp/includes/llm-catalog.php` + `public/api/llm-config.php` — `context_window`  
2. `cub-agent-chat-relay/src/context-buffer.ts` (new) — estimate, ratio, thresholds  
3. `cub-agent-chat-relay/src/relay.ts` + `ws-server.ts` — wire context into prepare/stream_end/ready  
4. `cub-chat-wp/public/api/relay-chat-save.php` — persist `context_buffer`  
5. `cubcloud.ai/lib/vireschatRelay.ts` + `hooks/useVireschatRelay.ts` + `ChatV2Interface.tsx` — meter  
6. Compaction: new relay summarize helper → WP context-summary API → history rewrite → UI notice  

---

## 12. Thresholds & knobs

| Knob | Default | Where |
|---|---|---|
| Warn ratio | `0.80` | Relay const / optional WP option |
| Compact ratio | `0.95` | Relay |
| Completion reserve | `min(4096, 0.10 * W)` | Relay |
| Tail budget after compact | `0.30 * W` | Relay |
| History fetch cap | Keep `HISTORY_LIMIT=20` until Phase B; then prefer **token budget** over fixed 20 | Relay |
| Fallback window | `128000` | WP curated map |

Make thresholds constants first; promote to WP options only if ops needs them.

---

## 13. Testing plan

### Unit (relay)
- Estimator: empty / long system / tools-heavy / multibyte text  
- Ratio + threshold boundaries (0.799 / 0.80 / 0.949 / 0.95)  
- Compact split: head/tail non-empty; current user message retained  

### Integration
- Mock `/llm-config` with known `context_window`  
- Turn with measured usage → `stream_end.context.source === "measured"`  
- HMAC save stores `context_buffer` meta  
- Compact persists summary; subsequent turn history starts with summary  

### Manual / browser (ChatV2)
- Meter appears after `ready`  
- Meter updates after each agent reply  
- Force low `context_window` in staging → compact notice + continued chat  
- Model switch updates `window` on next resolve  

### Regression
- Guest WS still works  
- Vision dual-write path still works (image surcharge)  
- User↔user Pusher threads unchanged (no meter)  

---

## 14. Risks & mitigations

| Risk | Mitigation |
|---|---|
| Wrong `context_window` → premature/late compact | Curated map + LiteLLM metadata; log `window` source; never default to 0 |
| Estimator drift vs provider | Prefer measured `prompt_tokens` for meter after first turn; estimator only for pre-flight |
| Summary drops critical facts | Compact prompt checklist; keep generous verbatim tail; allow Phase C “pin” messages later |
| Double-compact loops | Max one compact attempt per user turn; then hard error with UX copy |
| Cost of summarize call | Use same model or cheaper catalog sibling; record usage |
| UI shows billing totals as context | Document: meter = **prompt fill**, not lifetime conversation token sum |
| REST path diverges from WS | Share `prepare_turn` helper; Phase C parity required |

---

## 15. Success metrics

- **Hard context failures** (gateway context length errors) drop on agent WS chats  
- Median session length (messages) before user-reported “forgot earlier context” increases  
- Compact rate observable in logs/meta without user support tickets  
- Meter visible and trusted (spot-check: measured ratio within ~10% of estimator after warm turn)

---

## 16. Handoff to later roadmap items

| Later item | How Context Buffer helps |
|---|---|
| **#2 Full multimodal** | Image surcharges + compact prevent vision history from blowing the window |
| **#5 Long-context memory** | `context-summary` becomes the seed artifact for durable thread memory / knowledge promotion |
| **#9 HITL approvals** | Long approval threads stay within window via compact |
| **#12 Background jobs** | Job result injections can trigger the same budget checks |

Do **not** fold #5 into this project — only leave the summary schema compatible.

---

## 17. Decision log (defaults)

1. **Meter tracks last/next prompt fill**, not cumulative conversation billing.  
2. **Relay owns compact**; WP only stores artifacts.  
3. **Ship Phase A before B** so the meter teaches us real window values before we rewrite history.  
4. **Fixed message count (20) is insufficient** as the long-term budget — Phase B moves to token budgets while keeping a safety max message count.  
5. Authenticated ChatV2 is the primary UX surface; widget/guest can inherit frames with minimal UI.

---

## 18. Open questions (resolve during Phase A)

1. Exact LiteLLM `/models` field available in our deployment for context length?  
2. Should compact summary appear in the visible transcript or only as a chip + admin meta? (Recommend: chip + optional collapsible “Summary” row.)  
3. Prefer conversation meta vs special message row for summary storage? (Recommend: conversation meta + optional synthetic system message for relay simplicity.)  
4. Per-org window overrides needed before org-scoped agents (#11)? (Likely no for v1.)

---

*When implementation starts, update this doc’s Status line and link PRs against Phase A / B exit criteria.*
