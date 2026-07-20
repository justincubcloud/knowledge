# Agent Chat System — Current Architecture

> Source of truth analyzed from: `cub-chat-wp` (0.3.2), `cub-agents-wp` (0.1.7), `cubcloud.ai` (Next.js 16.2), `cub-agent-chat-relay` (1.0.0)  
> Related consumers: `cubden-web`, `cubden-mobile` (same WP APIs + agent WebSocket)  
> Related docs: `agent-chat-features-roadmap.md`, `agent-to-agent-chat-strategy.md` (planned), `xai-image-understanding.md`  
> Last verified against code: **2026-07-19**

---

## Overview

CubCloud Agent Chat is a four-layer system. WordPress owns identity and MySQL persistence. When an MCP relay URL is configured, **interactive agent intelligence (LLM + tools + RAG streaming) runs on `cub-agent-chat-relay`**. The Next.js frontend has no chat database — it talks to WordPress REST for CRUD and to the relay WebSocket for agent turns.

| Layer | Project | Role |
|---|---|---|
| Frontend | `cubcloud.ai` (Next.js 16.2 / React 19.2) | UI, auth cookies, conversation CRUD via WP REST; primary agent UX streams over WebSocket |
| Agent runtime | `cub-agent-chat-relay` (Node 22 / TypeScript) | Owns interactive agent turns: MCP server, multi-turn LLM + tools, RAG, guest quotas, `WS /ws`, `POST /relay/chat` |
| Chat Backend | `cub-chat-wp` (WordPress plugin 0.3.2) | Conversations/messages, LLM catalog, guest quotas, Pusher + optional simple ws-relay pub/sub, HMAC relay save endpoints, local LLM fallback |
| Agent Backend | `cub-agents-wp` (WordPress plugin 0.1.7) | Agent definitions, integrations, knowledge/RAG, capabilities, MCP proxy into the relay. **Requires** `cub-chat-wp` |

**Mental model**

```
Browser (cubcloud.ai)
  ├─ REST CRUD / auth / inbox ──────────────► WordPress (cub-chat-wp + cub-agents-wp)
  └─ Agent turns (stream + tools) ──────────► cub-agent-chat-relay /ws
                                                 │
                                                 ├─ GET /llm-config, tools, chat-context ◄─ WP
                                                 ├─ LLM gateway (Bifrost / LiteLLM / xAI / …)
                                                 ├─ Tool exec (creds from WP effective-settings)
                                                 └─ HMAC POST /relay/save-* ──────────────► WP MySQL
```

`cub-agents-wp` hard-depends on `cub-chat-wp` (`Requires Plugins: cub-chat-wp`; activation aborts if chat is inactive). The relay never stores integration credentials; it fetches effective settings from WordPress at runtime.

---

## Tech Stack

### cubcloud.ai
- **Framework**: Next.js 16.2 App Router, React 19.2, React Compiler (`babel-plugin-react-compiler`)
- **Language**: TypeScript 5
- **UI**: Tailwind CSS v4, shadcn/ui, Radix, Framer Motion
- **AI SDK**: `ai` 6.x + `@ai-sdk/openai` 3.x — used for **site assistant** routes (`/api/chat/web`), not the agent tool loop
- **Real-time**: Pusher JS 8.5 (user↔user / inbox) + agent WebSocket (`useViresAgentWs`)
- **Data fetching**: SWR 2.x for some lists
- **3D**: three.js, React Three Fiber, Drei
- **Deployment**: Vercel

### cub-chat-wp
- **Framework**: WordPress plugin (PHP), version **0.3.2**
- **Composer**: `openai-php/client`, `pusher/pusher-php-server`, Guzzle, Symfony HTTP client
- **DB**: WordPress MySQL via `dbDelta` (custom `wp_cub_chat_*` tables)
- **Real-time**: Pusher + optional bundled Node `ws-relay` pub/sub
- **LLM**: Multi-provider catalog (`openai_gateway` \| `bifrost` \| `xai` \| `litellm`) via `cub_chat_resolve_llm()`

### cub-agents-wp
- **Framework**: WordPress plugin (PHP 7.4+), version **0.1.7**, **requires cub-chat-wp**
- **DB**: WordPress MySQL (`wp_cub_agents*`, knowledge, tasks, teams, kanban)
- **Bridge**: `includes/mcp-proxy.php` → relay `POST /relay/chat` and `POST /execute`

### cub-agent-chat-relay
- **Runtime**: Node.js 22 (Docker `node:22-alpine`), TypeScript → `dist/`
- **Deps**: `@modelcontextprotocol/sdk`, `express`, `ws`, `@pinecone-database/pinecone`, `zod`
- **Transports**: stdio MCP (default) or HTTP (`MCP_TRANSPORT=http`) with SSE + `/relay/chat` + `/execute` + `WS /ws`
- **Deployment**: Docker Compose (port **3100**), optional PM2 + nginx

---

## System Architecture Diagram

```
┌──────────────────────────────────────────┐   MCP Clients
│         cubcloud.ai (Vercel)             │   (Claude Desktop / Code / Cursor / etc.)
│                                          │          │
│  /chat (auth → ChatV2*)                  │          │ stdio or HTTP/SSE
│  /chat (anon → ChatInterfaceWebsocket)   │          │
│  ChatWidget (home, authenticated)        │          ▼
│  ┌──────────┐  ┌──────────┐  ┌────────┐ │  ┌──────────────────────────────────────────┐
│  │ chat-v2  │  │  Agent   │  │  Auth  │ │  │      cub-agent-chat-relay (Node 22)       │
│  │ + Agent  │  │  Builder │  │ cookie │ │  │                                          │
│  │   WS     │  └─────┬────┘  └───┬────┘ │  │  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  └────┬─────┘        │           │      │  │  │ MCP Tools│  │ /relay/  │  │  /ws   │ │
│       │ WebSocket    │           │      │  │  │ (stdio + │  │  chat    │  │ Agent  │ │
│       │ /ws          │           │      │  │  │  HTTP/   │  │ (HTTP)   │  │ stream │ │
│  ┌────▼──────────────▼───────────▼────┐ │  │  │  SSE)    │  │          │  │        │ │
│  │  lib/api/chat + WP REST (CRUD)     │ │  │  └──────────┘  └──────────┘  └────────┘ │
│  └─────────────────┬──────────────────┘ │  │  - Multi-turn LLM + tools (max 5 rounds) │
└────────────────────┼────────────────────┘  │  - Live integrations (per-agent creds)   │
                     │  REST (Basic Auth)    │  - Pinecone RAG + tool-knowledge         │
                     │◀──────────────────────│  - Guest mode + IP/session quotas        │
                     │                       │  - HMAC-signed history persistence       │
                     │                       │  - Per-turn model via WP /llm-config     │
                     │                       └──────────────────┬───────────────────────┘
                     │                                          │  REST (Basic Auth /
                     │                                          │  HMAC / X-Cub-Secret)
                     ▼                                          ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              WordPress Backend                                          │
│  ┌───────────────────────────────────────────┐                                         │
│  │              cub-chat-wp 0.3.2            │                                         │
│  │  /wp-json/cubchat/v1/...                  │                                         │
│  │  - Conversations & Messages CRUD          │                                         │
│  │  - LLM catalog + /llm-config resolution   │                                         │
│  │  - Provider fallback (when no MCP URL)    │                                         │
│  │  - Guest sessions & rate limiting         │                                         │
│  │  - Pusher / simple ws-relay publish       │                                         │
│  │  - LRT (long-running tasks)               │                                         │
│  └──────────────────┬────────────────────────┘                                         │
│                     │  WordPress filters/hooks + mcp-proxy                              │
│  ┌──────────────────▼────────────────────────┐                                         │
│  │             cub-agents-wp 0.1.7           │                                         │
│  │  /wp-json/cubagents/v1/...                │                                         │
│  │  - Agent CRUD & metadata                  │                                         │
│  │  - Integration catalog + effective settings│                                        │
│  │  - Knowledge base & RAG (Pinecone)        │                                         │
│  │  - Tool-knowledge guidance vectors        │                                         │
│  │  - Capabilities, guards, teams, kanban    │                                         │
│  │  - MCP proxy → relay /relay/chat|/execute │                                         │
│  │  - Scheduled task queue & execution       │                                         │
│  └───────────────────────────────────────────┘                                         │
└────────────────────────────────────────────────────────────────────────────────────────┘
              │              │
     ┌────────▼───┐   ┌──────▼─────────────────────────────────────────┐
     │ LLM Gateway│   │  Real-time / External APIs                      │
     │ Bifrost    │   │  Pusher (managed), agent relay WS, optional     │
     │ LiteLLM    │   │  simple ws-relay pub/sub, Pinecone RAG          │
     │ OpenAI GW  │   │  WooCommerce, Klaviyo, DoorDash, Uber Eats     │
     │ xAI/Grok   │   │  Microsoft 365, NewsAPI, Stock/Crypto APIs     │
     └────────────┘   └────────────────────────────────────────────────┘
```

\* `chat-v2` is a **component family** (`components/chat-v2/*`), not an `app/chat-v2` route. Authenticated `/chat` renders `ChatV2PageContainer`.

---

## Primary Message Flows

### A. Authenticated agent chat — WebSocket-first (production path)

```
User (/chat → ChatV2Interface)
  → useViresAgentWs → wss://…/ws
  → auth { type: "auth", token: "Basic …", agentId, conversationId? }
  → chat { message, tempId?, images?, skipUserSave?, model? }
       ↓
cub-agent-chat-relay (ws-server → relay_chat_stream)
  1. Load agent + chat-context from cub-agents-wp
  2. Resolve LLM via GET /wp-json/cubchat/v1/llm-config?agent_id=&model=
     (X-Cub-Secret; cached ~60s)
  3. Assemble system prompt (base + date + RAG + tool-knowledge + tool guidance)
  4. Load tools from WP GET /tools?agent_id=…
  5. Multi-round LLM + tools (MAX_ROUNDS = 5); stream frames to client
  6. HMAC POST → /wp-json/cubchat/v1/relay/save-message
       ↓
WordPress MySQL (messages + token_usage in messagemeta)
```

REST remains used for conversation CRUD, attachment upload, inbox lists, search, and non-agent (user↔user) paths.

### B. Vision (authenticated) — dual-write

1. `POST …/conversations/{id}/messages` with attachment + `skip_ai=1` (persist thumbnail only).
2. Agent WebSocket `chat` frame with `images: string[]` (HTTPS attachment URLs preferred) + `skipUserSave: true`.
3. Relay builds OpenAI-compatible multimodal user content (`text` + `image_url` parts), calls the gateway **without tools**, truncates history to the last **2** text turns, streams the reply, then HMAC-saves the assistant text.

Constraints: JPEG/PNG (and data-URL equivalents), ≤20 MiB/image; no vision+tools in one round-trip; no multimodal history replay. Image **generation** (`/imagine`, Manifest, xAI Images) is a separate client path via `/api/manifest/*`.

### C. REST / WP-driven AI (fallback, DMs with @AI, scheduled tasks, WS off)

```
POST /cubchat/v1/conversations/{id}/messages
  → cub_chat_send_message_to_cub_ai_action
  → if cub_mcp_relay_chat + non-empty cub_mcp_get_relay_url()
       and not guest REST / not cub_bypass_relay
       → cub_mcp_relay_chat → POST relay /relay/chat
  → else provider path via cub_chat_resolve_llm()
       (openai_gateway | bifrost | xai | litellm)
  → Pusher and/or simple ws-relay publish for UI updates
```

Guest REST completions always stay local (`cub_guest_completions_rest`). Long-running / headless tasks may set `cub_bypass_relay`.

### D. Guest agent chat

```
Browser (no auth) — /chat anon, command Chat, or widgets
  → useViresAgentWs with empty token
  → guest_auth { type: "guest_auth", agentId }   # frontend currently omits sessionId;
                                                  # relay generates UUID if missing
  → chat { message, model? }
       ↓
relay: public+active agent only → quota (WP guest/voice/quota + in-memory IP tracker)
  → LLM + tools (guest tools via WP guest/agents/{id}/invoke-tool)
  → HMAC POST /relay/save-guest-message
```

HTTP guest path (widgets / legacy): `POST /api/chat/guest` or direct `POST /cubchat/v1/guest/chat/completions` — **does not** use the MCP relay tool loop; built-in Settings tools only.

### E. MCP clients

stdio or HTTP `GET /sse` + `POST /messages` → same tool surface; optional chat orchestration through `/relay/chat`.

---

## Frontend Chat Surface (`cubcloud.ai`)

| Surface | Route / mount | Component | Transport |
|---|---|---|---|
| Primary authenticated chat | `/chat` (logged in) | `ChatV2PageContainer` → `ChatV2Interface` | Agent WS + WP REST |
| Anonymous / guest chat | `/chat` (logged out), `/chat-v1` | `ChatPageContainer` → `ChatInterfaceWebsocket` | Guest agent WS |
| Floating widget | Home page only (`app/page.tsx`) | `ChatWidget` → `ChatWidgetChatPanel` | Auth agent WS (requires sign-in) |
| Command / cockpit | `/command` | `components/command/Chat.tsx` | Guest agent WS |
| Research / CubRoar widgets | marketing pages | `ResearchChatWidget`, `CubRoarChatWidget` | HTTP guest completions |
| Site “Cub” assistant | `/api/chat/web` | Vercel AI SDK → Bifrost | **Not** agent/tools/relay |
| Agent builder | `/build` | builder wizard + `/api/build-assistant` | WP cub-agents REST |

**Simple pub/sub relay client** (`useVireschatRelay` + `NEXT_PUBLIC_VIRESCHAT_RELAY`) is implemented in `hooks/useVireschatRelay.ts` but has **no active consumer** in cubcloud.ai today. Live realtime paths are: agent WebSocket (agent threads) and Pusher (user↔user / inbox).

---

## REST API Reference

### cub-chat-wp — `/wp-json/cubchat/v1/`

Core chat (most used by agent UI):

| Route | Method | Purpose |
|---|---|---|
| `/conversations` | GET/POST | List / create conversations |
| `/conversations/ai` | POST | Create AI conversation (optional `agent_id`) |
| `/conversations/{id}` | GET/PUT/DELETE | Detail / rename / soft-delete |
| `/conversations/{id}/messages` | GET/POST | Messages; POST supports `skip_ai`, attachments |
| `/conversations/{id}/messages/ai-image` | POST | Persist assistant image message |
| `/v2/conversations` | GET | Enhanced conversation list (namespace `cubchat/v2`) |
| `/search` | GET | Conversation / message search |
| `/latest-conversation` | GET | Current user’s latest conversation |
| `/conversations/{id}/users` | GET/POST | Participants |
| `/conversations/{id}/project` | PUT | Assign project |

LLM / relay:

| Route | Method | Purpose |
|---|---|---|
| `/llm-config` | GET | Relay-facing gateway URL/token/model (`agent_id`, `model`); token only with `X-Cub-Secret` |
| `/model-providers` | GET | Enabled model/provider catalog for UI |
| `/bifrost-models` | GET | **Deprecated alias** of `/model-providers` |
| `/relay/save-message` | POST | Relay → WP message persistence (HMAC) |
| `/relay/save-guest-message` | POST | Relay → WP guest message persistence (HMAC) |

Guest / voice / realtime:

| Route | Method | Purpose |
|---|---|---|
| `/guest/chat/completions` | POST | Guest AI completions (OpenAI-compatible; **local providers only**) |
| `/guest/voice/quota` | GET | Guest IP quota (relay may pass `relay_ip` + `X-Cub-Secret`) |
| `/guest/voice/turn` | POST | Log guest voice message + quota |
| `/guest/record-turn` | POST | Relay-authenticated quota increment |
| `/anon-conversation` | POST | Create anonymous WP user + AI conversation (legacy) |
| `/realtime/ws-token` | GET | JWT for **simple** ws-relay (not agent `/ws`) |
| `/realtime/relay-message` | POST | Client → simple relay publish helper (HMAC) |

Also registered: projects, stars, users/picker, feedback, report, OneSignal device register, chat analytics / token-usage, long-running-task (`/long-running-task/...` start/approve/pause/resume/stop/status/log).

### cub-agents-wp — `/wp-json/cubagents/v1/`

| Route | Method | Purpose |
|---|---|---|
| `/whoami` | GET | Auth diagnostic (used by relay WS auth) |
| `/agents` | GET/POST | List / create agents |
| `/agents/{id}` | GET/PUT/PATCH/DELETE | Agent CRUD (soft-delete) |
| `/agents/{id}/personality` | GET | Presentation, voice, model, starters |
| `/agents/{id}/meta` | GET/POST/DELETE | Expertise, capabilities, integrations, knowledge, guards, model |
| `/agents/{id}/chat-context` | GET | Merged system prompt + model + RAG flags (relay) |
| `/agents/{id}/integrations` | GET/POST/PUT/PATCH | Per-agent integration bindings |
| `/agents/build-assistant` | POST | Streaming LLM guidance for agent builder |
| `/my-agents/{id}/media` | POST | Upload agent image / 3D model |
| `/agents/{id}/knowledge/*` | GET/POST/DELETE | Settings, sync, retrieve, documents, files upload/process, scrape |
| `/tool-knowledge/status` | GET | Tool-knowledge catalog status (admin) |
| `/tool-knowledge/save` | POST | Persist + embed tool guidance |
| `/tool-knowledge/sync-all` | POST | Re-embed all enabled guidance |
| `/tool-knowledge/retrieve` | POST | Query-relevant tool guidance (relay; `X-Cub-Secret`) |
| `/tools` | GET | Tool definitions (`agent_id` filter) — relay loads these for LLM |
| `/integrations` | GET | Integration catalog |
| `/integrations/{id}/effective-settings` | GET | Site → user → agent merged settings (creds stay server-side) |
| `/capabilities` | GET | Capability catalog |
| `/tasks` | GET/POST | Schedule tasks for agents |
| `/kanban/tasks/{id}/run\|review` | POST | Kanban run / review |
| `/guest/agents/{id}/invoke-tool` | POST | Guest tool execution (relay secret; creds never leave WP) |
| `/guest/agents/{id}/rag-config` | GET | Guest RAG config (relay secret) |
| `/mcp` | GET/PUT/PATCH | Global MCP URL (admin) |

Also: teams, admin agents, content-sync, file-creator, my-integrations, analytics.

> **Note:** Knowledge file APIs are under `/agents/{id}/knowledge/files/...`, not a top-level `/knowledge-files` route.

### cubcloud.ai Next.js API Routes — `/api/`

| Route | Method | Purpose |
|---|---|---|
| `/api/chat/web` | POST | Standalone Bifrost “Cub” site assistant — **not** agent/tools/relay |
| `/api/chat/guest` | POST | Proxy guest completions to cub-chat-wp |
| `/api/build-assistant` | POST | Proxy to cub-agents-wp build-assistant |
| `/api/voice/tts` | POST | xAI text-to-speech |
| `/api/voice/stt` | POST | xAI speech-to-text |
| `/api/voice/realtime-token` | POST | Guest voice quota check + short-lived xAI Realtime token |
| `/api/manifest/generate` | POST | Image generation/editing via xAI |
| `/api/manifest/proxy` | GET | Allowlisted image proxy |
| `/api/cubcast/ingest` | POST | RSS podcast ingest (cron-safe) |
| `/api/audio-proxy` | GET/POST | Audio proxy helper |

### cub-agent-chat-relay

| Endpoint | Purpose |
|---|---|
| `GET /sse`, `POST /messages` | Remote MCP transport (`MCP_TRANSPORT=http`) |
| `POST /relay/chat` | Full chat turn (auth or guest); may include `images`, `model` — **JSON response** (non-streaming) |
| `POST /execute` | Single tool execution (requires `Authorization`) |
| `WS /ws` | Authenticated + guest **streaming** agent sessions |

---

## Database Schema

All tables use the WordPress table prefix (typically `wp_`).

### cub-chat-wp Tables

```
wp_cub_chat_conversations
  id, name, type ('user_to_user' | 'ai_to_user' | …),
  owner_user_id, agent_id (→ wp_cub_agents),
  project_id (→ wp_cub_chat_conversation_projects),
  guest_session_id, guest_ip, guest_user_agent,
  deleted_at

wp_cub_chat_conversation_users
  conversation_id, user_id

wp_cub_chat_messages
  id, conversation_id, user_id, agent_id,
  message, attachment_id, reactions,
  ip_address, user_agent, deleted_at

wp_cub_chat_messagemeta
  message_id, meta_key, meta_value   # e.g. token_usage

wp_cub_chat_messages_read
  message_id, user_id, read_at, conversation_id

wp_cub_chat_conversation_projects
  id, user_id, name, description, color, deleted_at

wp_cub_chat_ai_feedback
  message_id, user_id, source, category, comment, vote

wp_cub_chat_reports
  reporter/reported user, conversation/message, reason, status

wp_cub_chat_lrt_tasks          # Long-running tasks (current)
wp_cub_chat_lrt_logs
```

> Legacy helpers may still mention `cub_chat_tasks` / `cub_chat_tasks_logs`; **created** task tables are `cub_chat_lrt_*`.

### cub-agents-wp Tables

```
wp_cub_agents
  id, name, slug, description, system_prompt,
  image_id, model_3d_id, status, visibility,
  created_by, wp_user_id, deleted_at

wp_cub_agentmeta
  agent_id, meta_key, meta_value
  Keys: expertise, custom_expertise, capabilities, integrations,
        knowledge, guards, model, voice, conversation_starters, …

wp_cub_agent_integrations      # site-wide catalog (not per-agent junction)
wp_cub_agent_capabilities
wp_cub_agent_teams
wp_cub_agent_team_members
wp_cub_agent_knowledge_files
wp_cub_agent_knowledge_vectors
wp_cub_agent_tasks
wp_cub_agent_tasks_logs
(+ kanban run storage from kanban module)
```

### Relationships

```
wp_users
  ├─ wp_cub_agents (created_by, wp_user_id)
  │    ├─ wp_cub_agentmeta          # integrations / knowledge / model live here
  │    ├─ wp_cub_agent_knowledge_files
  │    ├─ wp_cub_agent_knowledge_vectors
  │    └─ wp_cub_agent_tasks → wp_cub_agent_tasks_logs
  └─ wp_cub_chat_conversations (owner_user_id)
       ├─ project_id → wp_cub_chat_conversation_projects
       └─ wp_cub_chat_messages (agent_id → wp_cub_agents)
            └─ wp_cub_chat_messagemeta
```

`agent_to_agent` conversation type is planned only (`agent-to-agent-chat-strategy.md`) — **not shipped**.

---

## Authentication & Authorization

### User Auth (Frontend → WordPress)
1. Login flow issues `auth_token` (httpOnly) + `user_info` cookies (`contexts/User.tsx`, `app/actions/auth.ts`)
2. Clients build `Authorization: Basic base64(user_login:token)` (application-password style)
3. Server routes use `getCurrentUserAuthToken()` the same way

### Agent WebSocket Auth (Frontend → Full Relay `/ws`)
- Auth: `{ type: "auth", token: "Basic …", agentId, conversationId? }` — relay verifies via WP `/whoami`
- Guest: `{ type: "guest_auth", agentId, sessionId? }` — relay requires public + active agent; generates `sessionId` UUID if omitted
- **Not** the JWT from `/realtime/ws-token` — that JWT is only for the simple pub/sub relay

### Relay → WordPress
| Mechanism | Header(s) | Used for |
|---|---|---|
| HMAC-SHA256 | `X-Cub-Timestamp` + `X-Cub-Signature` over `timestamp\nbody` (±120s) | `/relay/save-message`, `/relay/save-guest-message` |
| Shared secret | `X-Cub-Secret` = `cub_chat_relay_secret` | `/llm-config` (full token), guest quota/RAG/tool invoke, tool-knowledge retrieve |
| Basic Auth | User application password (“Cub MCP”) | Relay MCP tools / `/execute` / WP proxy calls |

Shared secret env on relay: `WP_RELAY_SECRET` (deprecated alias `VIRES_RELAY_SECRET`).

### Agent Ownership
- Creates require authenticated user (`created_by` auto-set)
- Updates/deletes require ownership OR `manage_options`
- Public agent reads filtered by `visibility`

### Multi-tenancy
Chat plugins are **user/agent-scoped**, not org-scoped. Organization concepts live mainly under adjacent `cubden/v1` APIs (projects, media) used by CubCloud / mobile — not the core chat tables. Org-scoped agents are on the roadmap.

---

## LLM Routing & Model Selection

### Providers (cub-chat-wp)

Multiple providers can be enabled via `cub_llm_providers` (fallback: legacy `llm_provider`):

| Provider | Role |
|---|---|
| `bifrost` | Bifrost gateway (manual model rows in `cub_bifrost_models`) |
| `litellm` | LiteLLM proxy (live `/models` + 5‑min cache + optional curation) |
| `openai_gateway` | Generic OpenAI-compatible gateway |
| `xai` | Direct xAI (`https://api.x.ai/v1`) |

Resolution always goes through **`cub_chat_resolve_llm({ agent_id, model })`**:

1. Explicit request `model` if present in catalog  
2. Agent `agentmeta.model`  
3. `cub_litellm_default_model`  
4. Primary provider fallback  

Catalog keys look like `provider:model` (e.g. `litellm:Qwen3.6-…`). UI: `ChatModelSelector` → `GET /model-providers`. Empty selection = agent default.

The relay never applies a client model alone; it re-resolves via `GET /llm-config?agent_id=&model=` with `X-Cub-Secret`. Env `LLM_GATEWAY_*` on the relay is **fallback only** when WP config/secret is unavailable.

When MCP relay URL is set, WordPress empties the local tool list for normal turns — the relay owns the tool loop. Guest REST and `cub_bypass_relay` paths stay on WP providers.

### Tool Pipeline

```
1. Relay loads tools: GET /cubagents/v1/tools?agent_id=
   (WP builds catalog: ai-tools + agent integrations via filters)
         ↓
2. LLM responds with tool_call(s)
         ↓
3. Relay dispatcher executes (auth: WP tools / integration handlers;
   guest: WP guest/agents/{id}/invoke-tool)
         ↓
4. Result injected; loop continues (max 5 rounds)
```

WP-local fallback still uses:

```
apply_filters('cub_tools_catalog', …)
apply_filters('cub_tools_handle_call', …)
```

### System Prompt Assembly (relay turn)
- Base from agent `system_prompt` (+ expertise / capabilities / guards / short knowledge via chat-context)
- UTC date injected
- Pinecone RAG snippets (top-k; `auto_rag` gate)
- Tool-knowledge guidance for the query (authenticated)
- Static + dynamic tool usage guidance

---

## Real-time Communication

Three mechanisms — do not conflate them:

### 1. Full Agent WebSocket (`cub-agent-chat-relay` `/ws`) — primary agent streaming
- **Hook**: `useViresAgentWs` in `hooks/useVireschatRelay.ts`
- **Auth**: Basic token or `guest_auth` in first WS message
- **Outbound**: `auth` \| `guest_auth` \| `chat` \| `ping`
- **Inbound**: `ready`, `stream_start`, `thinking`, `tool_call`, `stream_delta`, `stream_end`, `error`, `pong`
- **Chat payload**: `message`, optional `images`, `skipUserSave`, `model`, `tempId`, `conversationId`
- **Config**: `NEXT_PUBLIC_VIRES_AGENT_WS_URL` (normalized to `/ws`, `wss://` except local)
- **Consumers**: `ChatV2Interface`, `ChatInterfaceWebsocket`, `ChatWidgetChatPanel`, `command/Chat`

### 2. Simple WebSocket Relay (`cub-chat-wp/ws-relay`) — pub/sub fan-out
- Standalone Node server; WordPress publishes, relay fans out to browsers
- **Auth**: JWT from `/realtime/ws-token`
- **Rooms**: `conversation-{id}`
- **Hook**: `useVireschatRelay` — **defined but unused** in cubcloud.ai today
- **Config**: `NEXT_PUBLIC_VIRESCHAT_RELAY`, WP `cub_chat_relay_*` options
- Port default **3810** — distinct from agent relay **3100**
- Lightweight only — **not** the LLM/tool runtime

### 3. Pusher (Managed)
- **Channels** (frontend): `user-{userId}`, `conversation-{id}`
- **Events** (hyphenated): `new-message`, `message-updated`, `conversation-created`, (+ server may emit `thinking` / `assistant-delta`)
- **Hook**: `useChatInboxPusher`
- **Config**: `NEXT_PUBLIC_PUSHER_KEY`, `NEXT_PUBLIC_PUSHER_CLUSTER`
- Used for user↔user threads and inbox; agent threads prefer agent WS

---

## cub-agent-chat-relay

Standalone Node.js (TypeScript) service — MCP server **and** chat application layer. Distinct from `cub-chat-wp/ws-relay/index.js`.

### Transports

| Transport | Endpoint | Use case |
|---|---|---|
| stdio | — | Local MCP (Claude Desktop, Claude Code, Cursor); needs `WP_AUTH_TOKEN` |
| HTTP/SSE | `/sse` + `/messages` | Remote MCP clients (`MCP_TRANSPORT=http`) |
| HTTP POST | `/relay/chat` | WP mcp-proxy / non-streaming chat turns |
| HTTP POST | `/execute` | Single tool execution (auth required) |
| WebSocket | `/ws` | Authenticated + guest real-time **streaming** sessions |

### Chat Relay Flow

```
MCP Client / Next.js / WS frontend
         ↓
cub-agent-chat-relay /relay/chat or /ws
  1. Fetch agent config + chat-context + tools from cub-agents-wp
  2. Assemble system prompt (base + date + RAG + tool-knowledge + tool guidance)
  3. Fetch LLM config from WP /llm-config (60s cache) or fall back to .env
  4. POST to OpenAI-compatible LLM gateway (multi-turn, max 5 tool rounds)
  5. Execute tool calls with per-agent effective credentials (never stored in relay)
  6. Persist via HMAC-signed POST to cub-chat-wp (skipped if WP_RELAY_SECRET unset)
  7. Stream (WS) or return JSON (HTTP) to caller
         ↓
WordPress (cub-chat-wp) — final persistence
```

### MCP Resources (authenticated server)

| Resource | URI |
|---|---|
| Agents catalog | `cubagents://agents` |
| Integrations catalog | `cubagents://integrations` |
| Capabilities catalog | `cubagents://capabilities` |

Guest MCP exposes only `getGuestQuota`.

### MCP / Relay Tool Categories (implemented)

| Category | Key Tools |
|---|---|
| Agent Management | `listAgents`, `getAgent`, `createAgent`, `updateAgent`, `deleteAgent`, meta, analytics, agent integrations |
| Integrations & Capabilities | `listIntegrations`, `getIntegration`, `getIntegrationSchema`, `updateIntegrationSettings`, `testIntegrationConnection`, `listCapabilities` |
| MCP Settings | `getMcpSettings`, `updateGlobalMcpUrl`, `listIntegrationMcpUrls` |
| WooCommerce | `woocommerceRestRequest` |
| Klaviyo | lists, profile, track event, list membership |
| DoorDash | quote / create / get / cancel delivery |
| Uber Eats | orders, accept/cancel, store status |
| Microsoft 365 | mail, calendar, Teams message |
| Knowledge / RAG | `searchKnowledgeBase`, `upsertKnowledge`, `deleteKnowledge` |
| WordPress Files | `wpCreateFile`, `wpListFiles` |
| Long-running Tasks | `startLongRunningTask`, `getLongRunningTaskStatus`, `approveLongRunningTask` |
| Builtins | `searchNews`, `getStockPrice`, `getCryptoPrice`, `searchWeb`, `searchWebsite` |
| Scraping | `agentScraperScrapeUrl` |
| Guest | `getGuestQuota` (guest MCP server only) |

Tools are filtered per-agent based on enabled integrations in WordPress. Some dispatchable tools may appear only via WP `/tools` definitions without a matching MCP registration.

### Guest Mode

- Only `visibility=public` + `status=active` agents
- Quotas via cub-chat-wp transients (IP-based) + relay in-memory IP tracking across reconnects
- Tool execution through WP `guest/agents/{id}/invoke-tool` — credentials never leave the server
- Guest RAG uses `guest/agents/{id}/rag-config` (agent + shared namespaces only)

### Credential Security

```
GET /wp-json/cubagents/v1/integrations/{id}/effective-settings?agent_id=...
```

Precedence: **site catalog → current user overrides → agent overrides** (nonempty agent fields win).

### Environment Variables

| Variable | Required | Notes |
|---|---|---|
| `WP_BASE_URL` | Yes | WordPress base URL. Deprecated alias: `VIRES_WP_BASE_URL` |
| `MCP_TRANSPORT` | No | Exact `http` enables HTTP; anything else → stdio |
| `MCP_PORT` | No | Default 3100 in Docker Compose |
| `WP_AUTH_TOKEN` | Stdio | `Basic <base64(user:app_password)>` |
| `WP_RELAY_SECRET` | Relay features | Must match WP `cub_chat_relay_secret`. Alias: `VIRES_RELAY_SECRET` |
| `LLM_GATEWAY_URL` / `TOKEN` / `LLM_MODEL` | Fallback | Used only if WP `/llm-config` unavailable |
| `ENABLE_LOGGING` | No | Verbose stderr logs |
| `WP_AGENTS_NAMESPACE` | No | Default `/wp-json/cubagents/v1` |
| `WP_CHAT_NAMESPACE` | No | Default `/wp-json/cubchat/v1` |
| `WP_LLM_CONFIG_PATH` | No | Default → cubchat `/llm-config` |
| `WP_RELAY_MESSAGE_PATH` | No | Default → `/relay/save-message` |
| `WP_RELAY_GUEST_MESSAGE_PATH` | No | Default → `/relay/save-guest-message` |

`WP_USERNAME`, `WP_APP_PASSWORD`, and `MCP_SECRET` are **not** read by this service (stale in some relay docs).

### Deployment

```bash
npm run dev    # tsx, no build
npm run build  # tsc → dist/
npm start      # node dist/index.js

# Production (Docker) — MCP_TRANSPORT=http, port 3100
docker compose up -d --build
# See docs/droplet-deployment.md (verify port 3100 + WP_RELAY_SECRET; some doc sections are stale)
```

Logs go to stderr (safe for stdio MCP).

---

## Agent Lifecycle

```
1. Draft creation
   POST /wp-json/cubagents/v1/agents → status='draft'

2. Builder wizard (cubcloud.ai /build)
   Personality → Expertise → Integrations → Knowledge → Launch
   PATCH /agents/{id} or POST /agents/{id}/meta

3. Activate
   PATCH /agents/{id} { status: 'active' }

4. Chat (primary)
   WS /ws auth + chat frames with agentId
   → relay loads config/tools/knowledge; streams reply
   → HMAC saves to cub-chat-wp

   Fallback / REST:
   POST /conversations/{id}/messages { agent_id }
   → cub-chat-wp → mcp-proxy /relay/chat or WP provider path

5. Scheduled tasks
   POST /cubagents/v1/tasks
   → minutely WP-Cron → Cub Chat send pipeline → logs → optional email/push

6. Soft delete
   DELETE /agents/{id} → deleted_at
```

---

## Integration Catalog

Site catalog lives in `wp_cub_agent_integrations`. Per-agent enablement/settings live in `wp_cub_agentmeta` key `integrations`:

```json
{
  "id": "web_search",
  "enabled": true,
  "permissions": ["read"],
  "connection_settings": { "api_key": "..." },
  "allowed_tools": ["search_web"],
  "fallback": false
}
```

Effective credentials: site → user (`cub_agent_user_integration_{slug}`) → agent via `/integrations/{id}/effective-settings`.

**Categories (seeded catalog includes 200+):** Data (web/news/stock/crypto/X), Productivity (M365, Slack, Google), E‑commerce (Woo, Shopify, Uber Eats, DoorDash), Finance, Marketing (Klaviyo), Events, Custom REST.

---

## Knowledge Base / RAG

```
1. Upload / scrape via /agents/{id}/knowledge/files/* or documents
         ↓
2. File stored under uploads/cub-agents-kb/{agentId}/ (protected)
         ↓
3. Extract / chunk / (files: gpt-4o-mini fact extraction)
         ↓
4. Embeddings → Pinecone; ledger in wp_cub_agent_knowledge_vectors
         ↓
5. On chat: top-k search (agent + shared + optional user namespace)
         ↓
6. Snippets injected into system prompt (relay RAG modules)
```

Structured `knowledge` meta also syncs to Pinecone. Defaults: `text-embedding-3-small`, top_k≈5, score threshold≈0.7, namespace `agent_{id}` (+ `agent_{id}_user_{userId}` for private).

### Tool-knowledge

Separate Pinecone namespace (`tool_knowledge`) for **tool usage guidance**:
- Admin UI + `/tool-knowledge/*` on cub-agents-wp
- Relay retrieves per query (`src/knowledge/tool-knowledge.ts`)
- WP filters results to tools enabled for the agent

---

## Scheduled Tasks

| Feature | cub-chat-wp LRT | cub-agents-wp tasks |
|---|---|---|
| Tables | `wp_cub_chat_lrt_tasks` / `_logs` | `wp_cub_agent_tasks` / `_logs` |
| Trigger | REST + Heartbeat | Minutely WP-Cron (`cub_agents_tasks_minutely`) |
| Execution | LRT start/approve/pause/resume/stop | `cub_agents_process_scheduled_tasks` via Cub Chat send |
| UI / API | `/cubchat/v1/long-running-task/...` | `/cubagents/v1/tasks` + kanban runner |
| Notification | — | Email / push with continue link |

Kanban auto-run is separate (5‑minute cron, opt-in via `cub_agents_kanban_auto_run_enabled`).

---

## Guest Chat Flow

```
Browser (no auth)
         ↓
┌─ Agent WS path ──────────────────────────────────┐
│ guest_auth → relay quotas + tools + HMAC         │
│ save-guest-message                               │
└──────────────────────────────────────────────────┘
         or
┌─ HTTP path ──────────────────────────────────────┐
│ /api/chat/guest or WP /guest/chat/completions    │
│ local providers only; IP transient quota         │
│ headers: x-guest-limit / remaining / used        │
└──────────────────────────────────────────────────┘
```

Default quota options: `cub_guest_max_messages` (10), `cub_guest_window_hours` (24). Agent id: `NEXT_PUBLIC_GUEST_AGENT_ID`.

---

## Key File Reference

| File | Purpose |
|---|---|
| `cub-chat-wp/cub-chat-wp.php` | Plugin bootstrap (v0.3.2) |
| `cub-chat-wp/includes/llm-provider.php` | LLM send paths, MCP delegation, tool dispatch |
| `cub-chat-wp/includes/llm-catalog.php` | Multi-provider catalog + `cub_chat_resolve_llm()` |
| `cub-chat-wp/includes/ai-tools.php` | Base tool definitions |
| `cub-chat-wp/includes/realtime.php` | Pusher / ws-relay publish + JWT |
| `cub-chat-wp/includes/database.php` | Schema + LRT tables |
| `cub-chat-wp/public/api/messages.php` | Core chat REST |
| `cub-chat-wp/public/api/llm-config.php` | Relay-facing LLM resolution |
| `cub-chat-wp/public/api/llm-models.php` | `/model-providers` |
| `cub-chat-wp/public/api/relay-chat-save.php` | HMAC `/relay/save-message` |
| `cub-chat-wp/public/api/guest.php` | Guest completions / voice / quota |
| `cub-chat-wp/ws-relay/index.js` | Lightweight pub/sub WebSocket relay |
| `cub-agents-wp/cub-agents-wp.php` | Plugin bootstrap (v0.1.7) |
| `cub-agents-wp/includes/mcp-proxy.php` | WP → relay `/relay/chat` and `/execute` |
| `cub-agents-wp/includes/agent-tools.php` | Integrations → tool pipeline |
| `cub-agents-wp/includes/chat-context.php` | Merged agent system prompt |
| `cub-agents-wp/includes/pinecone.php` | RAG vector DB |
| `cub-agents-wp/includes/tool-knowledge.php` | Tool-guidance embeddings |
| `cub-agents-wp/includes/tasks.php` | Scheduled agent tasks |
| `cub-agents-wp/public/api/agents.php` | Agent CRUD REST |
| `cubcloud.ai/app/chat/page.tsx` | Auth → V2 / anon → websocket guest |
| `cubcloud.ai/components/chat-v2/ChatV2Interface.tsx` | Primary authenticated agent UX |
| `cubcloud.ai/components/chat/ChatInterfaceWebsocket.tsx` | Guest agent WS UX |
| `cubcloud.ai/hooks/useVireschatRelay.ts` | `useViresAgentWs` (+ unused simple relay hook) |
| `cubcloud.ai/lib/vireschatRelay.ts` | WS URL helpers / types |
| `cubcloud.ai/components/chat/ChatModelSelector.tsx` | Model catalog UI |
| `cubcloud.ai/lib/api/chat.ts` | Chat REST client (`skip_ai`, attachments) |
| `cubcloud.ai/app/api/chat/web/route.ts` | Standalone Bifrost Cub assistant |
| `cubcloud.ai/contexts/User.tsx` | Session / `useUser()` |
| `cub-agent-chat-relay/src/index.ts` | Transport selection, HTTP routes |
| `cub-agent-chat-relay/src/relay.ts` | LLM loop, tools, streaming (`MAX_ROUNDS=5`) |
| `cub-agent-chat-relay/src/ws-server.ts` | WS sessions (auth + guest + images + model) |
| `cub-agent-chat-relay/src/vision.ts` | Vision helpers |
| `cub-agent-chat-relay/src/knowledge/rag.ts` | Agent knowledge RAG |
| `cub-agent-chat-relay/src/knowledge/tool-knowledge.ts` | Tool-guidance retrieve |
| `cub-agent-chat-relay/src/tools/execution/dispatcher.ts` | Tool dispatch |
| `cub-agent-chat-relay/src/config.ts` | Env contract |
| `cub-agent-chat-relay/docs/SETUP.md` | Local MCP setup (verify against `config.ts`) |
| `cub-agent-chat-relay/docs/droplet-deployment.md` | Production Docker/nginx (verify port/secret names) |

---

## Environment Variables

### cub-chat-wp (WordPress Options)
```
cub_chat_pusher_key / _secret / _app_id / _cluster
cub_chat_relay_enabled
cub_chat_relay_url / cub_chat_relay_ws_public_url
cub_chat_relay_secret
cub_chat_relay_dual_publish
cub_chat_onesignal_* / cub_chat_enable_notifications
cub_user_id
cub_gateway_url / cub_auth_key
llm_provider                    # legacy single
cub_llm_providers               # openai_gateway | bifrost | xai | litellm
cub_bifrost_api_url / _api_key / cub_bifrost_models
cub_chat_xai_api_key
cub_litellm_api_url / _secret_key / _default_model / _curated_models
cub_guest_enabled / _max_messages / _window_hours / _user_id / _system_prompt
```

### cub-agents-wp (WordPress Options / settings)
```
cub_mcp_relay_url / cub_mcp_execute_url
cub_agents_global_mcp_url / cub_agents_global_mcp_headers
cub_chat_relay_secret           # shared with cub-chat-wp
cub_agents_tool_knowledge
cub_agents_content_sync_config  # site Pinecone/OpenAI for tool-knowledge + content sync
cub_agents_kanban_auto_run_enabled
# Pinecone primarily via integration settings (site → user → agent), not lone options
```

### cubcloud.ai (.env / Vercel)
```
# Public (browser)
NEXT_PUBLIC_API_BASE_URL
NEXT_PUBLIC_SITE_URL
NEXT_PUBLIC_PUSHER_KEY
NEXT_PUBLIC_PUSHER_CLUSTER
NEXT_PUBLIC_VIRES_AGENT_WS_URL    # Full agent WebSocket relay (…/ws)
NEXT_PUBLIC_VIRES_AGENT_ID
NEXT_PUBLIC_GUEST_AGENT_ID
NEXT_PUBLIC_VIRESCHAT_RELAY       # Simple pub/sub client (unused in UI today)

# Server-only
NEXT_BIFROST_API_URL / NEXT_BIFROST_API_KEY / NEXT_BIFROST_VIRTUAL_KEY
BIFROST_MODEL
XAI_API_KEY
CUBCAST_CRON_SECRET
```

### cub-agent-chat-relay
See [Environment Variables](#environment-variables) under the relay section above.

---

## Deployment

### WordPress (cub-chat-wp + cub-agents-wp)
1. Install MySQL + WordPress  
2. Activate **cub-chat-wp first**, then cub-agents-wp  
3. Configure: Pusher, `cub_chat_relay_secret`, LLM providers, MCP relay URL (`cub_mcp_relay_url` / global MCP)  
4. Create WordPress application password for API / MCP auth  

### Full Agent Relay (required for production agent chat)
```bash
cd cub-agent-chat-relay
# WP_BASE_URL, WP_RELAY_SECRET, MCP_TRANSPORT=http, MCP_PORT=3100
docker compose up -d --build
```

### Optional Simple WS Relay (pub/sub only)
```bash
cd cub-chat-wp/ws-relay
docker-compose up -d
# CUB_RELAY_SECRET, CUB_WP_BASE_URL — port 3810
```

### cubcloud.ai (Vercel)
```bash
# NEXT_PUBLIC_API_BASE_URL → live WordPress
# NEXT_PUBLIC_VIRES_AGENT_WS_URL → agent relay /ws
vercel deploy
```

---

## Extending the System (developer checklist)

Use this as the starting map when adding Agent Chat features (see also `agent-chat-features-roadmap.md`):

| You want to… | Start here |
|---|---|
| Change authenticated chat UX / streaming UI | `cubcloud.ai/components/chat-v2/*`, `hooks/useVireschatRelay.ts` |
| Add a WS frame or stream event | Relay `ws-server.ts` + frontend `useViresAgentWs` inbound switch |
| Change tool-loop / max rounds / vision rules | `cub-agent-chat-relay/src/relay.ts`, `vision.ts` |
| Add an integration tool | Seed/catalog in cub-agents-wp + handler in relay `tools/execution/*` + WP `agent-tools.php` |
| Change model catalog / resolution | `cub-chat-wp/includes/llm-catalog.php`, `ChatModelSelector.tsx` |
| Persist new message metadata | `wp_cub_chat_messagemeta` + relay save payloads + `relay-chat-save.php` |
| Improve RAG / tool-knowledge | `cub-agents-wp/includes/pinecone.php`, `tool-knowledge.php`, relay `src/knowledge/*` |
| Guest quotas / guest tools | `cub-chat-wp/public/api/guest.php`, relay `guest-chat.ts`, WP `guest/agents/*/invoke-tool` |
| Scheduled / background agent work | `cub-agents-wp/includes/tasks.php` and/or chat LRT APIs |
| Voice with full agent brain | Today: xAI voice routes under `/api/voice/*` (not relay tool loop). Roadmap: relay-backed voice |
| Agent-to-agent | Not shipped — see strategy doc; conversation type not in schema |

**Invariant to preserve:** WordPress remains the persistence and credential boundary; the relay is the interactive intelligence plane; the Next app is the UX plane.

---

## Related Consumers

| Project | Relationship |
|---|---|
| `cubden-web` | Parallel Next app using same cub-chat-wp APIs and agent WebSocket patterns |
| `cubden-mobile` | Mobile client over same WP REST + agent WS concepts |
| `cubden-admin` | WP theme / API docs UI for `cubagents/v1` (not chat runtime) |
| `cubchat-widget` / WP widget | Embeddable chat surfaces (separate packaging) |
| Knowledge docs | This file + roadmap / strategy / vision notes |

---

*Last updated: 2026-07-19 — verified against cub-chat-wp 0.3.2, cub-agents-wp 0.1.7, cubcloud.ai Next 16.2 / React 19.2, cub-agent-chat-relay 1.0.0 (Node 22). Corrections: LRT table names, knowledge route paths, Pusher channel/event names, dormant simple-relay client, guest_auth sessionId behavior, chat-v2 as component family under `/chat`, fuller REST surfaces, relay env contract.*
