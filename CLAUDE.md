# CLAUDE.md - claude-mem

**Version:** 9.0.12
**Repo:** josem4pro/claude-mem-ws (privado) -- fork of thedotmack/claude-mem
**Remotes:** origin=josem4pro/claude-mem-ws (privado), fork=publico (limpio), upstream=thedotmack
**License:** AGPL-3.0
**Branch:** main

---

## Description

Persistent memory compression system for Claude Code, built as a plugin. Captures tool usage observations during sessions, compresses them via AI (Claude Agent SDK, Gemini, OpenRouter), stores them in SQLite with FTS5, and injects relevant context into future sessions via lifecycle hooks. Features hybrid search (Chroma semantic + SQLite FTS5 keyword), a React web viewer at localhost:37777, and multi-IDE support (Claude Code + Cursor).

**Fork relationship:** Private fork of thedotmack/claude-mem. Keep fork-separation pattern: private repo for custom mods, public fork stays clean with upstream. Sync upstream: `git fetch upstream && git merge upstream/main`.

---

## Commands

### Build & Dev
```bash
npm run dev                   # Full dev workflow: build, sync, restart worker
npm run build                 # Compile TypeScript hooks via esbuild
npm run build-and-sync        # Build, sync to marketplace, restart worker
npm run sync-marketplace      # Copy plugin to ~/.claude/plugins/marketplaces/thedotmack/
```

### Worker Management
```bash
npm run worker:start          # Start Bun-managed worker service
npm run worker:stop           # Stop worker service
npm run worker:restart        # Restart worker service
npm run worker:status         # Check worker status
npm run worker:logs           # View last 50 lines of today's log
npm run worker:tail           # Follow today's log in real-time
```

### Testing (Bun native test runner)
```bash
npm test                      # Run all tests
npm run test:sqlite           # Database layer tests
npm run test:agents           # SDK/Gemini/OpenRouter agent tests
npm run test:search           # Search endpoint tests
npm run test:context          # Context injection tests
npm run test:infra            # Infrastructure/process tests
npm run test:server           # HTTP route tests
bun test <file.test.ts>       # Run single test file
```

### Utilities
```bash
npm run queue                 # Check pending async queue
npm run queue:process         # Process pending queue items
npm run claude-md:regenerate  # Auto-generate CLAUDE.md context files
npm run cursor:install        # Install Cursor IDE integration
```

---

## Key Metrics

| Metric | Value |
|--------|-------|
| Primary Language | TypeScript (97%), JavaScript (3%) |
| Production Code | 164 files, ~27,905 LOC |
| Test Suite | 45 test files, ~12,392 LOC |
| Scripts/Tooling | ~5,600 LOC |
| Total Codebase | ~89,072 LOC |
| Runtime | Node.js >= 18 + Bun >= 1.0 |
| Build System | esbuild (TypeScript to ESM) |
| CI/CD | GitHub Actions (4 workflows) |

---

## Structure

```
claude-mem/
+-- src/                             # Source (TypeScript)
|   +-- cli/handlers/                # Hook CLI handlers (6: context, session-init, observation, summarize, file-edit, user-message)
|   +-- sdk/                         # Claude Agent SDK integration (parser, prompts)
|   +-- servers/mcp-server.ts        # MCP protocol server (thin HTTP wrapper)
|   +-- services/
|   |   +-- worker-service.ts        # Main orchestrator (633 LOC, Express port 37777)
|   |   +-- context/                 # Context injection (ContextBuilder, ObservationCompiler, TokenCalculator)
|   |   +-- infrastructure/          # ProcessManager, HealthMonitor, GracefulShutdown
|   |   +-- sqlite/                  # Database layer
|   |   |   +-- SessionStore.ts      # Session CRUD (2,124 LOC -- largest file)
|   |   |   +-- SessionSearch.ts     # Search (577 LOC)
|   |   |   +-- migrations.ts        # Schema migrations
|   |   +-- sync/ChromaSync.ts       # Chroma vector DB embeddings (976 LOC)
|   |   +-- worker/                  # Business logic
|   |       +-- SearchManager.ts     # Search orchestration (1,849 LOC)
|   |       +-- SessionManager.ts    # Session lifecycle (414 LOC)
|   |       +-- SDKAgent.ts          # Claude Agent SDK processor
|   |       +-- GeminiAgent.ts       # Gemini AI agent
|   |       +-- OpenRouterAgent.ts   # OpenRouter agent
|   |       +-- search/              # Search strategies (Hybrid, Chroma, SQLite)
|   |       +-- http/routes/         # HTTP endpoints (Session, Data, Search, Settings, Viewer, Logs)
|   +-- ui/viewer/                   # React web viewer (App.tsx, components, hooks)
|   +-- utils/                       # Logging, privacy tags, CLAUDE.md utils
+-- plugin/                          # Built/distributable plugin
|   +-- hooks/hooks.json             # 4 lifecycle hook definitions
|   +-- scripts/                     # Compiled hook scripts (CJS)
|   +-- modes/                       # Language modes (30 files, 28 languages)
|   +-- skills/mem-search/           # Memory search skill
|   +-- ui/                          # Compiled viewer (HTML + JS bundle)
+-- tests/                           # 45 test files (sqlite, context, infra, integration, server, worker)
+-- scripts/                         # Dev/ops (build, changelog, sync, translate)
+-- docs/                            # Mintlify docs (MDX), i18n (28 languages), reports
+-- ragtime/                         # RAG sub-project (PolyForm Noncommercial)
```

---

## Architecture

### Data Flow
```
Claude Code Session
  -> Lifecycle Hook (SessionStart/UserPromptSubmit/PostToolUse/Stop)
  -> CLI Handler (src/cli/handlers/) -- strips <private> tags
  -> Worker HTTP API (localhost:37777)
     +-> SessionManager (session init/close)
     +-> SDKAgent/GeminiAgent/OpenRouterAgent (AI processing)
     +-> SessionStore (SQLite persistence)
     +-> ChromaSync (vector embeddings)
     +-> SSEBroadcaster (real-time UI updates)
  -> Context Injection (next session)
     +-> ContextBuilder -> ObservationCompiler -> TokenCalculator
     +-> MarkdownFormatter / ColorFormatter
  -> SessionStart hook returns context to Claude Code
```

### 5 Lifecycle Hooks
SessionStart, UserPromptSubmit, PostToolUse, Stop, SessionEnd

### Key Patterns
- **Session Architecture:** Dual ID system -- `content_session_id` (Claude Code) + `memory_session_id` (internal tracking)
- **Search Architecture:** Hybrid search (Chroma semantic + SQLite FTS5 keyword). 3-layer progressive disclosure: search -> timeline -> get_observations
- **Context Injection:** SessionStart hook calls `/api/context`, returns JSON with mode-specific memory
- **Privacy Tags:** `<private>content</private>` stripped at hook layer (edge) before data reaches worker/database. See `src/utils/tag-stripping.ts`
- **Endless Mode (beta):** Biomimetic memory architecture for extended sessions

### High-Connectivity Modules
| Module | Fan-out |
|--------|---------|
| `src/utils/logger.ts` | 91 importers |
| `src/services/worker-service.ts` | 26 imports (hub) |
| `src/services/worker/SearchManager.ts` | 12 imports |

### Exit Code Strategy
- **Exit 0:** Success or graceful shutdown (Windows Terminal tab-close safe)
- **Exit 1:** Non-blocking error (stderr shown, continues)
- **Exit 2:** Blocking error (stderr fed to Claude for processing)

---

## Dependencies

| Tech | Usage |
|------|-------|
| TypeScript 5.3+ | Primary language |
| Bun >= 1.0 | Runtime, test runner, process manager |
| Node.js >= 18 | Required runtime |
| Express 4.x | HTTP server |
| better-sqlite3 | SQLite persistence with FTS5 |
| ChromaDB | Vector embeddings (semantic search) |
| Claude Agent SDK | AI observation processing |
| MCP SDK | Model Context Protocol server |
| React 18 | Web viewer UI |
| esbuild | Build/bundler |
| uv | Auto-installed for Python (Chroma) |

---

## File Locations

- **Source:** `src/`
- **Built Plugin:** `plugin/`
- **Installed Plugin:** `~/.claude/plugins/marketplaces/thedotmack/`
- **Database:** `~/.claude-mem/claude-mem.db`
- **Chroma:** `~/.claude-mem/chroma/`
- **Settings:** `~/.claude-mem/settings.json`
- **Worker Logs:** `~/.claude-mem/logs/worker-YYYY-MM-DD.log`
- **Public Docs:** https://docs.claude-mem.ai (Mintlify, auto-deploys from main)

---

## Risks / Known Issues

| Risk | Severity | Details |
|------|----------|---------|
| SessionStore concentration | HIGH | 2,124 LOC in single class. All SQLite session ops. High change frequency and bug density risk. |
| SearchManager concentration | HIGH | 1,849 LOC. Complex search orchestration duplicates logic already in `worker/search/`. |
| worker-service.ts coupling | MEDIUM | Imports 26 modules. High fan-out despite refactoring from monolith. |
| Logger ripple risk | MEDIUM | Imported by 91 files. Any interface change ripples everywhere. |
| Missing tests | MEDIUM | No dedicated tests for ChromaSync, SessionManager, ContextBuilder. Test ratio ~45% by LOC. |
| Stale README badge | LOW | Version badge shows 6.5.0, package.json at 9.0.12. |
| Empty placeholder dirs | LOW | `folder-index/`, `process/`, `worker/` under src/ contain only CLAUDE.md files. |

---

## Pro Features Architecture

Open-source core: All worker API endpoints on localhost:37777 remain fully open. Pro features are headless -- no proprietary UI in this codebase. License gating via settings, not code modification.

---

## Important

Changelog is auto-generated -- never edit it manually.
