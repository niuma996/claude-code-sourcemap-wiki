# Auto Memory System Analysis

> Reconstructed from `@anthropic-ai/claude-code@2.1.88` source maps

## Overview

Auto Memory is Claude Code's persistent memory system that stores cross-session context as Markdown files on local disk. After each conversation, a background sub-agent automatically extracts noteworthy information and writes it to the memory directory.

---

## Directory Structure

```
~/.claude/
└── projects/
    └── <sanitized-git-root>/
        └── memory/
            ├── MEMORY.md          ← Index file, always injected into system prompt
            ├── user_role.md       ← Per-topic memory files
            ├── feedback_testing.md
            ├── project_deadline.md
            ├── team/              ← Team Memory (TEAMMEM feature gate)
            │   └── MEMORY.md
            └── logs/              ← KAIROS mode (long assistant sessions)
                └── YYYY/MM/YYYY-MM-DD.md
```

Path resolution priority ([paths.ts](../restored-src/src/memdir/paths.ts)):
1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` env var (Cowork-specific)
2. `autoMemoryDirectory` in `settings.json` (policy/local/user levels only — **excludes** projectSettings to prevent malicious repos writing to `~/.ssh`)
3. Default: `~/.claude/projects/<sanitized-git-root>/memory/`

---

## Memory Types

Four types, distinguished by the `type:` frontmatter field ([memoryTypes.ts](../restored-src/src/memdir/memoryTypes.ts)):

| Type | Purpose | When to Save |
|------|---------|--------------|
| `user` | User role, preferences, knowledge background | When learning about the user's identity or skills |
| `feedback` | User corrections or confirmations of AI behavior | When user says "don't do that" or "exactly right" |
| `project` | Project progress, goals, deadlines, incidents | When learning who is doing what, why, and by when |
| `reference` | Pointers to external systems (Linear, Slack, Grafana) | When learning about external resource locations |

**Should NOT save**: code patterns, architecture, file paths, git history, debugging solutions, content already in CLAUDE.md, ephemeral task state.

### Memory File Format

```markdown
---
name: short-kebab-case-slug
description: One-line description (used for relevance filtering — be specific)
type: user | feedback | project | reference
---

Memory body content.
For feedback/project types, recommended structure: rule/fact + **Why:** + **How to apply:**
```

### MEMORY.md Index Format

```markdown
- [Title](file.md) — one-line hook description (~150 chars max)
```

- No frontmatter
- Truncated with a warning when it exceeds 200 lines or 25,000 bytes
- Always injected into the system prompt

---

## Core Modules

### 1. `memdir/memdir.ts` — Prompt Construction

- `buildMemoryLines()` — builds memory behavior instructions (without MEMORY.md content)
- `buildMemoryPrompt()` — builds the full prompt including MEMORY.md content (used by agent memory)
- `loadMemoryPrompt()` — loads the memory section of the system prompt, dispatching based on feature flags:
  - KAIROS mode → `buildAssistantDailyLogPrompt()` (appends to log, does not maintain MEMORY.md)
  - TEAMMEM enabled → `buildCombinedMemoryPrompt()` (private + team dual directories)
  - Default → `buildMemoryLines()` (single directory)
- `truncateEntrypointContent()` — truncates MEMORY.md, first by line count (200), then by bytes (25KB)
- `ensureMemoryDirExists()` — idempotently creates the memory directory, called once per session

### 2. `memdir/paths.ts` — Path Resolution

- `isAutoMemoryEnabled()` — checks whether auto memory is enabled (env var > --bare > CCR > settings > enabled by default)
- `getAutoMemPath()` — retrieves the memory directory path (memoized, keyed by projectRoot)
- `isAutoMemPath(path)` — checks whether a path is inside the memory directory (used for permission control)
- `getAutoMemDailyLogPath(date)` — log file path for KAIROS mode

### 3. `memdir/memoryScan.ts` — File Scanning

- `scanMemoryFiles(memoryDir, signal)` — recursively scans `.md` files, reads frontmatter, sorts by mtime descending, returns up to 200 files
- `formatMemoryManifest(memories)` — formats a text manifest (pre-injected for the extraction agent to avoid wasting a round-trip `ls`)

### 4. `memdir/findRelevantMemories.ts` — Relevance Recall

- `findRelevantMemories(query, memoryDir, signal, recentTools, alreadySurfaced)` — scans memory file headers, calls a Sonnet model to select up to 5 most relevant files
- Filters out already-surfaced files (`alreadySurfaced`)
- For tools currently in use, skips their reference docs (to reduce noise), but retains warning/known-issue memories

### 5. `services/extractMemories/extractMemories.ts` — Background Extraction

Core mechanism: fires fire-and-forget after each query loop ends (model produces a final response with no tool calls) via `handleStopHooks`.

**Key design decisions**:
- **Forked agent mode**: perfectly forks the main conversation, shares prompt cache, does not write to transcript
- **Mutex logic**: if the main agent has already written memory files (`hasMemoryWritesSince`), skips background extraction and advances the cursor
- **Throttling**: `tengu_bramble_lintel` feature flag controls how often extraction fires per turn (default: every turn)
- **Serialization**: only one extraction runs at a time; new calls stash context and execute a trailing run after the current one completes
- **Tool restrictions** (`createAutoMemCanUseTool`):
  - Allowed: FileRead, Grep, Glob (unrestricted)
  - Allowed: read-only Bash (ls/find/grep/cat/stat/wc/head/tail)
  - Allowed: FileEdit/FileWrite, but **only within the memory directory**
  - Denied: all other tools (MCP, Agent, write Bash, etc.)
- **Max turns**: 5 (prevents verification rabbit holes)
- **Cursor**: `lastMemoryMessageUuid` tracks the last processed position; only new messages are processed each run

**Extraction flow**:
```
handleStopHooks
  └─ executeExtractMemories (fire-and-forget)
       └─ runExtraction
            ├─ hasMemoryWritesSince? → skip, advance cursor
            ├─ scanMemoryFiles → pre-inject file manifest
            ├─ buildExtractAutoOnlyPrompt / buildExtractCombinedPrompt
            └─ runForkedAgent (maxTurns=5)
                 ├─ Turn 1: parallel FileRead of all potentially updated files
                 └─ Turn 2: parallel FileWrite/FileEdit
```

### 6. `services/autoDream/autoDream.ts` — Memory Consolidation (KAIROS Mode)

Distills log files into topic files + MEMORY.md index. Triggers when all three gates pass:
1. Time since last consolidation >= minHours (default 24h)
2. Sessions touched >= minSessions (default 5)
3. No other process is currently consolidating (lock)

---

## Call Chain

```
main.tsx
  └─ startBackgroundHousekeeping()          [backgroundHousekeeping.ts]
       ├─ initExtractMemories()             ← initializes closure state
       └─ initAutoDream()

query loop
  └─ handleStopHooks()                      [stopHooks.ts]
       ├─ executeExtractMemories()          ← fire-and-forget
       └─ executeAutoDream()               ← fire-and-forget

session end
  └─ drainPendingExtraction()              [print.ts]
       └─ waits for all in-flight extractions to complete (60s timeout)
```

---

## Feature Gates

| Flag | Purpose |
|------|---------|
| `EXTRACT_MEMORIES` | Background extraction agent |
| `KAIROS` | Log-append mode (long assistant sessions) |
| `TEAMMEM` | Team memory support |
| `tengu_passport_quail` | Extraction feature enabled |
| `tengu_herring_clock` | Team memory enabled |
| `tengu_bramble_lintel` | Extraction throttle (default 1 = every turn) |
| `tengu_moth_copse` | Skip MEMORY.md index (log-only mode) |
| `tengu_coral_fern` | Enable "search historical context" section |
| `tengu_onyx_plover` | autoDream config (minHours, minSessions) |
| `MEMORY_SHAPE_TELEMETRY` | Memory recall shape telemetry |

---

## Environment Variables

| Variable | Effect |
|----------|--------|
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | Completely disables auto memory |
| `CLAUDE_CODE_SIMPLE` / `--bare` | Disables auto memory |
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | Overrides the memory base directory |
| `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` | Full path override (Cowork-specific) |
| `CLAUDE_COWORK_MEMORY_EXTRA_GUIDELINES` | Injects additional memory guidance text |

---

## Security Design

- **Path traversal protection**: `validateMemoryPath()` rejects relative paths, root paths, UNC paths, and null bytes
- **projectSettings exclusion**: `autoMemoryDirectory` is not read from `.claude/settings.json` (prevents malicious repos redirecting writes to `~/.ssh`)
- **Write permission isolation**: `isAutoMemPath()` ensures FileEdit/FileWrite can only write within the memory directory
- **Team Memory symlink protection**: `teamMemPaths.ts` validates paths do not escape (PSR M22186)

---

## Key File Index

| File | Responsibility |
|------|---------------|
| [memdir/memdir.ts](../restored-src/src/memdir/memdir.ts) | Prompt construction, truncation, directory management |
| [memdir/paths.ts](../restored-src/src/memdir/paths.ts) | Path resolution, enable checks |
| [memdir/memoryTypes.ts](../restored-src/src/memdir/memoryTypes.ts) | Type definitions, prompt text constants |
| [memdir/memoryScan.ts](../restored-src/src/memdir/memoryScan.ts) | File scanning, manifest formatting |
| [memdir/findRelevantMemories.ts](../restored-src/src/memdir/findRelevantMemories.ts) | Sonnet-based relevance recall |
| [memdir/teamMemPaths.ts](../restored-src/src/memdir/teamMemPaths.ts) | Team memory paths, symlink validation |
| [services/extractMemories/extractMemories.ts](../restored-src/src/services/extractMemories/extractMemories.ts) | Background extraction agent main logic |
| [services/extractMemories/prompts.ts](../restored-src/src/services/extractMemories/prompts.ts) | Extraction agent prompt templates |
| [services/autoDream/autoDream.ts](../restored-src/src/services/autoDream/autoDream.ts) | Memory consolidation (KAIROS) |
| [query/stopHooks.ts](../restored-src/src/query/stopHooks.ts) | Extraction trigger call site |
| [utils/backgroundHousekeeping.ts](../restored-src/src/utils/backgroundHousekeeping.ts) | Initialization entry point |
