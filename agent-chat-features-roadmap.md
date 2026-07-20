# Agent Chat Features Roadmap

Top priority features to enhance the Cub Agent Chat system.

Open items only in the numbered list (priority order). Shipped work moves to **Completed** at the bottom (with date + notes). Feature strategy docs hold phase detail.

---

## 1. Agent-to-Agent Collaboration

Allow an owner agent with multi-agent collaboration capability to delegate work to other agents in a shared conversation. Humans see the full back-and-forth, can step in for review, and remain the visibility boundary for which sub-agents can be called.

## 2. Full Multimodal Turns

Lift the current vision constraints so image understanding can run in the same turn as tools, and prior image turns can be replayed in conversation history. That makes visual workflows (receipts, screenshots, product photos) first-class instead of one-off text-only replies.

## 3. Relay-Backed Voice Mode

Route Voice Mode through `cub-agent-chat-relay` (or an equivalent agent runtime path) so spoken turns use the same agent config, knowledge/RAG, integrations, and tools as text chat. xAI Voice can remain the speech transport, but the agent brain should no longer be a disconnected voice-only model without Cub tools or knowledge.

## 4. Per-Message Agent Response Latency

Record how long each AI agent turn takes to complete, in milliseconds, and persist it on the assistant message in `messagemeta` (alongside `token_usage` / `context_buffer`). Measure from the start of the relay turn (WS `chat` / stream start) through final assistant content so tool rounds are included. Expose the value on message APIs and optionally in ChatV2 / admin analytics so latency trends are inspectable per conversation and model. Ship early so historical timing data starts accruing.

## 5. Chat Context Buffer — Auto-Compact (Phase B)

When session fill hits ~95%, automatically summarize older turns, keep a recent verbatim tail, persist the summary, and continue so long tool / multimodal threads do not hard-fail. Builds on the shipped Phase A meter. See `agent-chat-context-buffer.md`.

## 6. Long-Context Memory & Thread Summaries

Build durable, cross-session memory on top of compaction so agents recall decisions, preferences, and prior tool outcomes beyond a single context window. Thread summaries become reusable knowledge, not only an in-session emergency shrink.

## 7. Localized Knowledge via QDrant

Add QDrant as a first-class vector store for agent and org knowledge so RAG can run alongside LiteLLM and local models without depending solely on Pinecone. Support self-hosted / on-prem namespaces for agent docs, tool-knowledge, and Cub Organization corpora while keeping the same chat and relay retrieval paths.

## 8. Knowledge Tree-Splitter

Replace naive fixed-size chunking with a tree-structured splitter that keeps related passages together (by headings, sections, and semantic boundaries) instead of cutting mid-concept. Parent/child chunks let retrieval pull a precise leaf while still carrying the surrounding section context, which improves RAG quality for agent and org knowledge alike.

## 9. Knowledge Re-ranker

After vector search (and whenever new knowledge is uploaded), run a re-ranker that scores passages by true relevance—not just embedding proximity. New uploads get relevance-scored against the existing corpus so duplicates and weak chunks sink, high-signal material rises, and chat retrieval consistently prefers the best evidence for the agent’s question.

## 10. Human-in-the-Loop Tool Approvals

Surface high-impact tool calls (payments, sends, writes, deletes) as explicit confirm/deny steps in chat before execution. Builds on long-running task approval patterns so users keep control while agents still orchestrate the work.

## 11. Warm-Pool Cloud Coding Environments

Give Cub Cloud agents a way to claim short-lived coding environments from a warm pool — sandboxes with file management, terminals, and specialized code agents they can call as tools. That is the missing substrate for cloud coding agents: not just chat planning, but an actual workspace the agent can edit, run, and hand back results from without cold-starting every job.

## 12. Org-Scoped Agents & Shared Knowledge

Extend agent ownership and knowledge beyond single-user scope so teams can share agents, integrations, and RAG corpora at the organization level. Aligns chat with CubDen org APIs and unlocks multi-user agent workspaces without leaking credentials across tenants — including corpora stored in localized QDrant.

## 13. Background Agent Jobs & Inbox Delivery

Elevate long-running and scheduled agent work into a first-class chat product: agents continue after the live turn, claim warm-pool environments or tools as needed, and deliver results into the user's inbox with clear status, resume, and failure recovery. That closes the gap between interactive WebSocket chat and agents that actually work for you while you're away.

---

## Completed

| # | Feature | Completed | Notes |
|---|---|---|---|
| — | Chat Context Buffer — Phase A (meter) | 2026-07-19 | Context % meter in ChatV2; `context_window` on `/llm-config`; relay `context` on ready/stream_end; `context_buffer` messagemeta. Phase B remains open as #5. Detail: `agent-chat-context-buffer.md` |
