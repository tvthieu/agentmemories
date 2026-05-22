# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

**agentmemory** is a persistent memory system for AI coding agents (Claude Code, Cursor, Codex CLI, Gemini CLI, etc.). It runs as a self-contained server powered by **iii-engine** (WebSocket on port 49134), with REST APIs (port 3111), a real-time viewer (port 3113), and 53 MCP tools. Data is stored in SQLite via iii-engine's StateModule at `~/.agentmemory/data/state_store.db` — no external databases required.

## Commands

```bash
# Build
npm run build          # tsdown → dist/
npm run dev            # Hot reload via tsx

# Test
npm test               # Unit tests (vitest, excludes integration)
npm run test:one test/search.test.ts  # Single test file
npm run test:watch     # Watch mode
npm run test:integration  # API integration tests (requires running server)
npm run test:all       # All tests

# Run
npm start              # Start from dist/
```

To run a single test by name pattern: `npx vitest run test/search.test.ts --reporter=verbose`

## Architecture

### Core Pipeline

Hook fires (e.g. PostToolUse) → `src/hooks/` standalone script → REST call → `observe()` → dedup/privacy filter → `compress()` → BM25 + vector + graph indexing → available for retrieval. At SessionStart, `context()` runs hybrid search (BM25 + dense embeddings + knowledge graph with RRF fusion) and injects top-K memories into the conversation.

### Key Layers

- **`src/state/`** — KV abstraction, BM25 search index, vector index, hybrid search, reranker. Schema defines 34 KV scopes in `schema.ts`.
- **`src/functions/`** — 123 iii functions registered via `sdk.registerFunction("mem::name", ...)`. Core: `observe`, `compress`, `smart-search`, `context`, `summarize`, `consolidate`, `remember`, `auto-forget`, `evict`.
- **`src/triggers/api.ts`** — 124 REST endpoints at `http://localhost:3111/agentmemory/*`. Validated request bodies are passed to `sdk.trigger()` — never pass raw `req.body`.
- **`src/mcp/`** — 53 MCP tools. `tools-registry.ts` holds the registry; `server.ts` has the dispatch switch; `standalone.ts` runs without iii-engine.
- **`src/providers/`** — LLM providers (Anthropic, OpenAI, Gemini, OpenRouter, Minimax) and embedding providers (local all-MiniLM-L6-v2, OpenAI, Voyage, Cohere, Gemini). Auto-detected from env vars.
- **`src/hooks/`** — Standalone Node.js scripts (no iii-sdk import). Use `AbortSignal.timeout()` for best-effort HTTP calls.
- **`src/cli.ts`** — CLI commands: `start`, `stop`, `connect <agent>`, `doctor`, `upgrade`, `demo`.

### Core Types (`src/types.ts`)

- **`RawObservation`** — captured from hooks (toolName, toolInput, toolOutput, hookType)
- **`CompressedObservation`** — after LLM/synthetic compression (title, facts, narrative, concepts, files, importance, confidence)
- **`Memory`** — manually saved facts (type: pattern|preference|architecture|bug|workflow|fact, supports versioning via `supersedes`/`isLatest`)
- **`Action`** — work items with DAG dependencies (blockedBy, depends_on)
- **`Lease`**, **`Signal`**, **`Routine`**, **`Sentinel`** — multi-agent coordination primitives

## Consistency Requirements (from AGENTS.md)

### Adding an MCP tool — update all of:
1. `src/mcp/tools-registry.ts` — tool definition + `getAllTools()` array
2. `src/mcp/server.ts` — handler case in `mcp::tools::call` switch
3. `src/triggers/api.ts` — corresponding REST endpoint
4. `src/index.ts` — function registration + endpoint count log
5. `test/mcp-standalone.test.ts` — tool count assertion
6. `README.md` — tool counts
7. `plugin/.claude-plugin/plugin.json` — tool count in description

### Adding a REST endpoint — update all of:
1. `src/triggers/api.ts`
2. `src/index.ts` — endpoint count
3. `README.md` — endpoint count

### Bumping version — update all of:
1. `package.json`
2. `src/version.ts`
3. `src/types.ts` — `ExportData` version union
4. `src/functions/export-import.ts` — `supportedVersions` set
5. `test/export-import.test.ts` — version assertion
6. `plugin/.claude-plugin/plugin.json`

### Adding KV scopes — update all of:
1. `src/state/schema.ts`
2. `src/types.ts` — corresponding interface

## Key Patterns

- **IDs**: `generateId()` for unique IDs (timestamp + crypto random); `fingerprintId()` for content-addressable dedup (SHA256 prefix)
- **Dedup**: 5-minute SHA256 window in `src/functions/dedup.ts`
- **Privacy**: `src/functions/privacy.ts` strips API keys, secrets, `<private>` tags before storage
- **Memory consolidation**: 4-tier (Working → Episodic → Semantic → Procedural) with Ebbinghaus decay curve. Controlled by `CONSOLIDATION_ENABLED=true`.
- **Feature flags**: Most advanced features are opt-in via env vars (see `src/config.ts`): `AGENTMEMORY_AUTO_COMPRESS`, `AGENTMEMORY_SLOTS`, `AGENTMEMORY_REFLECT`, `GRAPH_EXTRACTION_ENABLED`, `CONSOLIDATION_ENABLED`

## Configuration

Config lives at `~/.agentmemory/.env`. Key env vars:
- LLM: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`, `OPENROUTER_API_KEY`
- Embedding: `EMBEDDING_PROVIDER=local` (default: offline all-MiniLM-L6-v2), or set `VOYAGE_API_KEY` / `OPENAI_API_KEY`
- Search weights: `BM25_WEIGHT=0.4`, `VECTOR_WEIGHT=0.6`, `AGENTMEMORY_GRAPH_WEIGHT=0.3`
- Token budget for context injection: `TOKEN_BUDGET=2000`
- Auth: `AGENTMEMORY_SECRET` (bearer token for protected endpoints)
