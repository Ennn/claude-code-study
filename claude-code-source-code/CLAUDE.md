# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **Claude Code source code** (v2.1.88), extracted from the npm package `@anthropic-ai/claude-code`. The source is TypeScript (~1,884 files, 512K LOC) that was compiled with Bun's bundler.

## Build Commands

```bash
cd claude-code-source-code/

npm run build    # Transform + esbuild bundle (best-effort, ~95% complete)
npm run check    # TypeScript type check (requires prepare-src first)
npm start        # Run the built CLI: node dist/cli.js
```

**Build Notes:**
- The source uses Bun compile-time intrinsics (`feature()`, `MACRO.X`, `bun:bundle`) that cannot be fully replicated with esbuild
- `feature('FLAG')` → `false` (dead code eliminated at compile time)
- `MACRO.VERSION` → `'2.1.88'` (injected at build time)
- 108 feature-gated modules are missing from this source (internal to Anthropic)
- See `QUICKSTART.md` for detailed build instructions and known issues

## Architecture Overview

```
cli.tsx (entry) → main.tsx → REPL (interactive) or QueryEngine (headless)

QueryEngine: AsyncGenerator<SDKMessage> pipeline
  ├── fetchSystemPromptParts() → assemble system prompt
  ├── processUserInput() → handle /commands
  └── query() → main agent loop (in query.ts, ~785KB)
        ├── StreamingToolExecutor → parallel tool execution
        └── runTools() → tool orchestration
```

### Core Patterns

- **Streaming**: `AsyncGenerator` chains from Claude API → tools → consumer
- **Tool Interface**: `buildTool(definition)` factory produces `Tool<Input, Output, Progress>`
- **Feature Flags**: `feature('FLAG')` gates internal code (DCE'd from published builds)
- **Branded Types**: `SystemPrompt`, `asSystemPrompt()` prevent string confusion
- **Discriminated Unions**: `Message` types for type-safe message handling
- **Dynamic Imports**: Fast-path CLI flags use lazy loading to minimize module eval

### Key Files

| File | Purpose |
|------|---------|
| `src/query.ts` | Main agent loop (~785KB, largest file) |
| `src/Tool.ts` | Tool interface + buildTool factory |
| `src/tools.ts` | Tool registry, presets, filtering |
| `src/commands.ts` | Slash command definitions (~80 commands) |
| `src/main.tsx` | REPL bootstrap (~4,683 lines) |
| `src/QueryEngine.ts` | SDK/headless query lifecycle engine |
| `src/bridge/bridgeMain.ts` | Claude Desktop / remote bridge session manager |
| `src/services/compact/` | Context compression (autoCompact, snipCompact) |

### Directory Structure

```
src/
├── entrypoints/       # Application entry points (cli.tsx, sdk/, mcp.ts)
├── cli/              # CLI infrastructure, handlers, transports
├── bridge/           # Claude Desktop / remote bridge
├── commands/         # ~80 slash commands
├── components/       # React/Ink terminal UI
├── hooks/            # React hooks (permissions, state)
├── services/         # Business logic (api, compact, mcp, analytics)
├── state/            # AppState store + React context
├── tasks/            # Task implementations (bash, agent, dream)
├── tools/            # 40+ tool implementations
├── types/            # Type definitions
├── utils/            # Utilities (30+ groups)
└── vendor/           # Native module stubs
```

## Session Persistence

Sessions stored in `~/.claude/projects/<hash>/sessions/<session-id>.jsonl`:
- Append-only log with message types: `user`, `assistant`, `progress`, `system`
- Resume via `--continue`, `--resume <id>`, or `--fork-session`

## Feature Flags

Feature flags use random word pairs (e.g., `tengu_frond_boric`) to obscure purpose. Key flags observed:
- `DAEMON` - Background worker processes
- `KAIROS` - Push notifications, autonomous agent mode
- `COORDINATOR_MODE` - Multi-agent coordinator
- `HISTORY_SNIP` - Aggressive history trimming
- `CONTEXT_COLLAPSE` - Context restructuring
- `BRIDGE_MODE` - Remote bridge connections
- `TRANSCRIPT_CLASSIFIER` - Session transcript service

Internal features gated by `process.env.USER_TYPE === 'ant'` are not available in this source.

## Missing Modules

108 modules referenced by `feature()`-gated branches are **not included** in this source. They exist only in Anthropic's internal monorepo and were dead-code-eliminated at compile time. These cannot be recovered.

## Key Design Patterns

| Pattern | Usage |
|---------|-------|
| AsyncGenerator streaming | QueryEngine, query() for API streaming |
| Builder + Factory | buildTool() for safe tool defaults |
| Snapshot State | FileHistoryState for undo/redo |
| Fire-and-Forget Write | recordTranscript() non-blocking persistence |
| Lazy Schema | lazySchema() for deferred Zod evaluation |
| Context Isolation | AsyncLocalStorage for per-agent context |

## Documentation

- `README.md` - Full architecture documentation, tool inventory, data flow diagrams
- `QUICKSTART.md` - Build instructions and known issues
- `docs/` - Deep analysis reports (telemetry, hidden features, undercover mode, remote control, roadmap)
