# Fork Subagent Technical Documentation

> Fork is a lightweight subagent mechanism in Claude Code. By omitting `subagent_type`, the forked subagent inherits the parent's full context and shares the parent's prompt cache, suitable for research, parallel tasks, and implementation work.

---

## 1. Concept and Positioning

### 1.1 Two Subagent Paradigms

Claude Code implements two subagent mechanisms:

| | **Fork Subagent** | **Spawned Teammate** |
|---|---|---|
| Trigger | Omit `subagent_type` | Explicitly specify `subagent_type` |
| Location | Same process, shares parent prompt cache | Independent process (tmux/iTerm2 pane) |
| Context transfer | Byte-exact API request prefix reuse | CLI flags + environment variable propagation |
| Model constraint | Must match parent (otherwise cache invalidates) | Can specify different model |
| Entry file | `forkSubagent.ts` | `spawnMultiAgent.ts` |

### 1.2 Use Cases

The core value of Fork is: **using a subagent's isolated context to keep the main agent's context clean**.

- **Research tasks**: Exploring open-ended questions where the parent doesn't need intermediate tool outputs
- **Implementation tasks**: Multi-step editing work where tool call noise isn't worth keeping in the main context
- **Parallel independent tasks**: Multiple independent subtasks launched at once with negligible shared cache overhead

Fresh Agent (explicit `subagent_type`) is suitable for scenarios requiring an **independent, unbiased perspective**, such as code review or second opinion.

### 1.3 Feature Flags

```typescript
// forkSubagent.ts:32-39
export function isForkSubagentEnabled(): boolean {
  if (feature('FORK_SUBAGENT')) {
    if (isCoordinatorMode()) return false   // Mutually exclusive with coordinator
    if (getIsNonInteractiveSession()) return false  // Disabled for non-interactive sessions
    return true
  }
  return false
}
```

`FORK_SUBAGENT` is a GrowthBook experiment flag. When enabled:
- `subagent_type` becomes optional — omitting it triggers the fork path
- Guides the main agent to understand fork semantics and directive-style prompt writing
- `/fork <directive>` CLI entry is available (`commands/branch/index.ts`)

---

## 2. Prompt Cache Sharing Mechanism (Core Design)

> **💡 Insight 1: Using placeholder constants to level differences, so all forked subagents share the same cache prefix**

### 2.1 Background

When a subagent starts, the parent's conversation history is passed as context. If each subagent rebuilds the request prefix from scratch, prompt cache hit rate would be extremely low. Prompt cache creation is a significant cost driver in Claude Code's API billing.

### 2.2 Solution: Byte-Exact API Request Prefix

Core logic in `buildForkedMessages()` (`forkSubagent.ts:107-169`):

```typescript
export function buildForkedMessages(
  directive: string,
  assistantMessage: AssistantMessage,
): MessageType[] {
  // 1. Clone parent assistant message (preserve all tool_use, thinking, text blocks)
  const fullAssistantMessage: AssistantMessage = {
    ...assistantMessage,
    uuid: randomUUID(),
    message: {
      ...assistantMessage.message,
      content: [...assistantMessage.message.content],
    },
  }

  // 2. Collect all tool_use blocks
  const toolUseBlocks = assistantMessage.message.content.filter(
    (block): block is BetaToolUseBlock => block.type === 'tool_use'
  )

  // 3. Generate placeholder tool_result for each tool_use
  //    All placeholder text is IDENTICAL — this is the key to cache hits
  const toolResultBlocks = toolUseBlocks.map(block => ({
    type: 'tool_result' as const,
    tool_use_id: block.id,
    content: [{ type: 'text', text: FORK_PLACEHOLDER_RESULT }],
    //                    ↑ hardcoded constant: 'Fork started — processing in background'
  }))

  // 4. Compose single user message: [placeholder tool_results..., per-child directive]
  const toolResultMessage = createUserMessage({
    content: [
      ...toolResultBlocks,
      { type: 'text', text: buildChildMessage(directive) },
    ],
  })

  return [fullAssistantMessage, toolResultMessage]
}
```

**Key insight**: Only the last text block (output of `buildChildMessage(directive)`) is per-child. Everything else (system prompt, assistant message, tool_results placeholders) is identical across all fork subagents. This means the API request prefix's cache key is exactly the same — after the first call, all subsequent fork subagents share the cache.

### 2.3 Byte-Exact System Prompt Transfer

> **💡 Insight 2: `renderedSystemPrompt` is frozen at turn start and passed through, not recalculated — bypassing GrowthBook state drift**

```typescript
// AgentTool.tsx:495-511
if (isForkPath) {
  if (toolUseContext.renderedSystemPrompt) {
    // Directly reuse parent's already-rendered bytes — byte-exact
    forkParentSystemPrompt = toolUseContext.renderedSystemPrompt
  } else {
    // Fallback: recalculate (may differ from cached bytes due to GrowthBook cold→hot state change)
    forkParentSystemPrompt = buildEffectiveSystemPrompt({ ... })
  }
  promptMessages = buildForkedMessages(prompt, assistantMessage)
}
```

`renderedSystemPrompt` is frozen at turn start (`contentReplacementState` in `ToolUseContext`), and threading to subagents avoids cache busting caused by GrowthBook feature flags changing during the turn. The comment explicitly states:

> Re-calling `getSystemPrompt()` may produce different output due to GrowthBook cold→hot state transitions, busting the prompt cache; directly passing already-rendered bytes is byte-exact.

### 2.4 useExactTools Inheritance

```typescript
// AgentTool.tsx:630-633
forkContextMessages: isForkPath ? toolUseContext.messages : undefined,
...(isForkPath && { useExactTools: true }),
```

`useExactTools: true` makes the subagent inherit the parent's exact tool definitions (including thinking configuration, etc.), ensuring the API request prefix is completely consistent.

---

## 3. Dynamic Directive Injection: FORK_BOILERPLATE_TAG

> **💡 Insight 3: One XML tag serves two purposes — behavioral spec + recursion detection, killing two birds with one stone**

### 3.1 Building Behavioral Spec

`buildChildMessage()` (`forkSubagent.ts:171-198`) dynamically constructs behavioral specs for each subagent:

```typescript
export function buildChildMessage(directive: string): string {
  return `<${FORK_BOILERPLATE_TAG}>
STOP. READ THIS FIRST.

You are a forked worker process. You are NOT the main agent.

RULES (non-negotiable):
1. Your system prompt says "default to forking" — ignore it. You ARE a fork. Do not spawn subagents, execute directly
2. No dialogue, questions, or suggested next steps
3. No editorial comments or meta-commentary
4. Use tools directly: Bash, Read, Write, etc.
5. After modifying files, commit first — include commit hash in report
6. No text output between tool calls — report everything at once at the end
7. Stay strictly within directive scope
8. Reports limited to 500 words
9. Response MUST start with "Scope:"
10. Report structured facts then stop

Output format (plain text tags, not markdown headers):
  Scope: <scope>
  Result: <result>
  Key files: <file paths>
  Files changed: <including commit hash>
  Issues: <issue list>
</${FORK_BOILERPLATE_TAG}>

[FORK_DIRECTIVE_PREFIX]<user directive>
```

### 3.2 Recursion Protection: Two-Layer Detection

> **💡 Insight 4: Two-layer recursion protection — first layer survives autocompact, fallback layer catches what it misses**

Recursion protection has two layers with clever design:

```typescript
// AgentTool.tsx:325-334
if (
  toolUseContext.options.querySource === `agent:builtin:${FORK_AGENT.agentType}` ||
  isInForkChild(toolUseContext.messages)
) {
  throw new Error('Fork is not available inside a forked worker.')
}
```

- **Primary detection**: `querySource === 'agent:builtin:fork'` — set via `context.options.querySource`, survives autocompact because autocompact rewriting messages does **not** pass through context.options
- **Fallback**: Message scan `isInForkChild()` — scans for `FORK_BOILERPLATE_TAG` in message history (triggered after autocompact)

This solves a subtle problem: autocompact compresses/rewrites messages (replacing the fork-boilerplate message), so relying on message scanning alone cannot reliably detect recursive forks; but `context.options.querySource` persists after autocompact, forming the autocompact-safe primary detection layer.

```typescript
// forkSubagent.ts:78-89
export function isInForkChild(messages: MessageType[]): boolean {
  return messages.some(m => {
    if (m.type !== 'user') return false
    const content = m.message.content
    if (!Array.isArray(content)) return false
    return content.some(
      block =>
        block.type === 'text' &&
        block.text.includes(`<${FORK_BOILERPLATE_TAG}>`),
    )
  })
}
```

---

## 4. Context Precision Trimming

> **💡 Insight 5: Quantified trimming based on production traffic data — not guesswork, data-driven**

`runAgent.ts` trims subagent context with precision, each trim backed by quantified savings:

### 4.1 CLAUDE.md Omission

```typescript
// runAgent.ts:385-398
// Saves ~5-15 Gtok/week (based on 34M+ Explore spawns statistics)
const shouldOmitClaudeMd =
  agentDefinition.omitClaudeMd &&
  !override?.userContext &&
  getFeatureValue_CACHED_MAY_BE_STALE('tengu_slim_subagent_claudemd', true)
const { claudeMd: _omitted, ...userContextNoClaudeMd } = baseUserContext
```

Read-only agents (Explore, Plan) have CLAUDE.md that's of no value to the main agent interpreting their output — it's just noise. The comment explicitly states: **saves ~5-15 Gtok/week (based on 34M+ Explore spawns)**, showing this is a production data analysis decision.

### 4.2 gitStatus Omission

```typescript
// runAgent.ts:400-410
// Saves ~1-3 Gtok/week
const resolvedSystemContext =
  agentDefinition.agentType === 'Explore' || agentDefinition.agentType === 'Plan'
    ? systemContextNoGit  // Remove up to 40KB of gitStatus
    : baseSystemContext
```

Explore/Plan are read-only search agents; the parent's session-start gitStatus (explicitly marked stale, up to 40KB) is dead weight for them. The comment states: **if subagents need git info, they can run `git status` themselves to get fresh data** — saving tokens while ensuring data freshness is a win-win.

### 4.3 Optional Caller Override Protection

Note the `!override?.userContext` condition — if the caller explicitly passes `userContext`, CLAUDE.md is preserved. This is a nod to API consumers: people who know what they're doing should be able to override defaults.

---

## 5. SubagentStart Hook Dynamic Injection

> **💡 Insight 6: Unified message type — hook injection uses the exact same attachment message format as SessionStart/UserPromptSubmit**

`runAgent.ts:530-555` executes hooks on subagent startup to collect additional context:

```typescript
for await (const hookResult of executeSubagentStartHooks(
  agentId,
  agentDefinition.agentType,
  agentAbortController.signal,
)) {
  if (hookResult.additionalContexts?.length > 0) {
    additionalContexts.push(...hookResult.additionalContexts)
  }
}

// Injected as attachment message, maintaining same format as SessionStart/UserPromptSubmit
if (additionalContexts.length > 0) {
  const contextMessage = createAttachmentMessage({
    type: 'hook_additional_context',
    content: additionalContexts,
    hookName: 'SubagentStart',
    ...
  })
  initialMessages.push(contextMessage)
}
```

The value of unified message types: the model uses the same parsing logic when processing hook context from SessionStart, UserPromptSubmit, and SubagentStart — no special handling needed for SubagentStart.

---

## 6. Skill Preloading Mechanism

> **💡 Insight 7: `isMeta: true` meta user message — visible to the model but doesn't affect conversation turn structure**

Subagent frontmatter can declare `skills`, preloaded concurrently at agent startup:

```typescript
// runAgent.ts:577-646
const skillsToPreload = agentDefinition.skills ?? []
const loaded = await Promise.all(
  validSkills.map(async ({ skillName, skill }) => ({
    skillName,
    content: await skill.getPromptForCommand('', toolUseContext),
  }))
)
for (const { skillName, content } of loaded) {
  initialMessages.push(
    createUserMessage({ content: [...content], isMeta: true })
  )
}
```

- **`isMeta: true`**: Injected as a meta user message, visible to the model but doesn't affect conversation turn structure
- **Concurrent loading**: `Promise.all()` loads all skill prompts concurrently without serial waiting
- **Multi-strategy matching**: Exact match → prefix match (`plugin:skill-name`) → suffix match (`:skill-name`)

---

## 7. Isolated ToolUseContext

> **💡 Insight 8: Mutation callbacks default to no-op — subagent cannot pollute parent state unless parent explicitly authorizes**

`createSubagentContext()` (`forkedAgent.ts:345-462`) creates an isolated context for subagents:

```typescript
export function createSubagentContext(
  parentContext: ToolUseContext,
  overrides?: SubagentContextOverrides,
): ToolUseContext {
  // Clone file state cache (isolated but not lost)
  readFileState: cloneFileStateCache(parentContext.readFileState),

  // Independent abort controller (can be linked to parent or independent)
  abortController: overrides?.shareAbortController
    ? parentContext.abortController
    : createChildAbortController(parentContext.abortController),

  // Mutation callbacks default to no-op, preventing accidental parent state modification
  setAppState: overrides?.shareSetAppState ? parentContext.setAppState : () => {},
  setResponseLength: overrides?.shareSetResponseLength
    ? parentContext.setResponseLength : () => {},

  // Depth incrementing: used for analytics and anti-recursion
  queryTracking: {
    chainId: randomUUID(),
    depth: (parentContext.queryTracking?.depth ?? -1) + 1,
  }
}
```

Key design principle: **zero trust subagents**. Subagent's `setAppState` and `setResponseLength` default to no-op — parent state won't be accidentally polluted. Only fork subagents with explicit `overrides` share these callbacks.

---

## 8. MCP Server Accumulation Model

> **💡 Insight 9: Accumulate rather than replace — subagent's MCP servers supplement the parent's, cleanup only removes newly created ones**

Subagents can define their own MCP servers, **merged cumulatively** with the parent's MCP clients, not replacing them:

```typescript
// runAgent.ts:95-218
return {
  clients: [...parentClients, ...agentClients],  // Accumulate
  tools: agentTools,
  cleanup  // Only close agent-created connections, leave parent's intact
}
```

- **Accumulation**: Parent and subagent MCP servers coexist; subagent can use both parent's and its own MCP tools
- **Selective cleanup**: `cleanup` function only closes connections the subagent created, leaving parent's untouched

---

## 9. CLI Flag Propagation (Spawned Teammate Path)

For out-of-process teammates, parent session settings are propagated via `buildInheritedCliFlags()`:

```typescript
// spawnUtils.ts:38-89
export function buildInheritedCliFlags(options?: {
  planModeRequired?: boolean
  permissionMode?: PermissionMode
}): string {
  const flags: string[] = []

  // Permission mode propagation
  if (permissionMode === 'bypassPermissions' || getSessionBypassPermissionsMode()) {
    flags.push('--dangerously-skip-permissions')
  }

  // Model override
  const modelOverride = getMainLoopModelOverride()
  if (modelOverride) flags.push(`--model ${quote([modelOverride])}`)

  // teammate mode
  flags.push(`--teammate-mode ${sessionMode}`)
}
```

---

## 10. Execution Flow Overview

```
Main Agent calls AgentTool({ prompt: "...", /* no subagent_type */ })
         │
         ▼
    AgentTool.tsx
    ├── isForkSubagentEnabled() → true?
    │   └── Yes → isForkPath = true
    ├── Recursion check: querySource (autocompact-safe) + FORK_BOILERPLATE_TAG scan (fallback)
    ├── Select FORK_AGENT definition
    ├── Construct forkParentSystemPrompt (reuse renderedSystemPrompt, byte-exact)
    ├── buildForkedMessages(prompt, assistantMessage)
    │   └── Clone parent assistant message + placeholder tool_results (constant text) + buildChildMessage(directive)
    └── runAgent({ agentDefinition: FORK_AGENT, override: { systemPrompt }, forkContextMessages, useExactTools: true })
            │
            ▼
        runAgent.ts
        ├── Trim context (CLAUDE.md, gitStatus omitted conditionally — data-driven optimization)
        ├── executeSubagentStartHooks() collect additional context (unified attachment message format)
        ├── Preload skills (concurrent + meta user message injection)
        ├── Initialize agent-specific MCP servers (cumulative with parent)
        ├── createSubagentContext() create isolated context (mutation callbacks default to no-op)
        └── query() execute subagent conversation loop
                │
                ▼
            Subagent response (Scope:/Result:/Files changed:/Issues:)
                │
                ▼
            <task-notification> pushed back to main agent
            Main agent receives structured result, continues working
```

---

## 11. Key Design Principles and Insights Index

| # | Insight | Location |
|---|---|---|
| 1 | **Placeholder constants level differences**: All fork subagents use identical placeholder text for tool_results, making prefix cache key identical | `forkSubagent.ts:107-169` |
| 2 | **`renderedSystemPrompt` frozen passing**: Frozen at turn start, avoids GrowthBook state drift causing cache bust | `AgentTool.tsx:495-511` |
| 3 | **Two-birds-one-stone XML tag**: `FORK_BOILERPLATE_TAG` serves as both behavioral spec carrier and recursion detection marker | `forkSubagent.ts:171-198` |
| 4 | **Two-layer recursion protection**: querySource survives autocompact, message scan as fallback | `AgentTool.tsx:325-334` |
| 5 | **Data-driven context trimming**: CLAUDE.md/gitStatus omission backed by Gtok/week quantified savings | `runAgent.ts:385-410` |
| 6 | **Unified message type**: Hook injection uses same attachment message format as SessionStart/UserPromptSubmit | `runAgent.ts:530-555` |
| 7 | **`isMeta: true` meta message**: Skill preloading visible to model but doesn't affect conversation turn structure | `runAgent.ts:577-646` |
| 8 | **Zero-trust isolation**: Mutation callbacks (setAppState/setResponseLength) default to no-op | `forkedAgent.ts:345-462` |
| 9 | **Accumulate rather than replace**: Subagent MCP servers accumulate with parent, cleanup only removes newly created | `runAgent.ts:95-218` |
