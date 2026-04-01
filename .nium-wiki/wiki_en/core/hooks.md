# Hooks Module

The Hooks module is Claude Code's event-driven extension system, allowing custom logic execution at specific lifecycle events. Through hooks, developers can inject custom handling at critical moments like tool execution, session state changes, and configuration updates.

## Architecture Position

```mermaid
flowchart TB
    subgraph CLI["CLI Layer"]
        HooksCmd["hooks command<br/>/hooks"]
        ContextCmd["context command<br/>/context"]
    end

    subgraph HookSystem["Hooks System"]
        AsyncRegistry["AsyncHookRegistry<br/>Async Hook Registry"]
        HookEvents["hookEvents<br/>Event Broadcasting"]
        HookSettings["hooksSettings<br/>Config Management"]
        HookConfigMgr["hooksConfigManager<br/>Metadata Management"]
    end

    subgraph Execution["Hook Executors"]
        ExecAgent["execAgentHook<br/>Agent Hook"]
        ExecHttp["execHttpHook<br/>HTTP Hook"]
        ExecPrompt["execPromptHook<br/>Prompt Hook"]
        SessionHooks["sessionHooks<br/>Session Hooks"]
    end

    subgraph Config["Configuration Sources"]
        UserSettings["User Settings<br/>~/.claude/settings.json"]
        ProjectSettings["Project Settings<br/>.claude/settings.json"]
        LocalSettings["Local Settings<br/>.claude/settings.local.json"]
        PluginHooks["Plugin Hooks"]
        SessionHooksSrc["Session Hooks<br/>Memory"]
    end

    HooksCmd --> HookSettings
    HooksCmd --> HookConfigMgr
    HookSettings --> Config
    HookEvents --> AsyncRegistry
    AsyncRegistry --> Execution
    Execution --> SessionHooks
```

## File Structure

```
restored-src/src/
├── commands/
│   ├── hooks/
│   │   ├── index.ts           # hooks command entry
│   │   └── hooks.tsx          # HooksConfigMenu UI component
│   └── context/
│       ├── index.ts           # context command entry
│       ├── context.tsx         # ContextVisualization
│       └── context-noninteractive.ts
│
└── utils/
    └── hooks/
        ├── AsyncHookRegistry.ts    # Async hook management
        ├── hookEvents.ts           # Event system
        ├── hookHelpers.ts          # Helper functions
        ├── hooksSettings.ts        # Settings management
        ├── hooksConfigManager.ts   # Config manager
        ├── hooksConfigSnapshot.ts  # Config snapshot
        ├── sessionHooks.ts         # Session hooks
        ├── execAgentHook.ts        # Agent hook executor
        ├── execHttpHook.ts         # HTTP hook executor
        ├── execPromptHook.ts       # Prompt hook executor
        ├── execPromptHook.ts
        ├── fileChangedWatcher.ts   # File watcher
        ├── registerFrontmatterHooks.ts
        ├── registerSkillHooks.ts
        ├── skillImprovement.ts
        ├── postSamplingHooks.ts
        ├── ssrfGuard.ts
        └── apiQueryHookHelper.ts
```

## Hook Types

### Hook Event Types

The hooks system supports 26 event types covering tool execution, session lifecycle, configuration changes, and more:

```mermaid
classDiagram
    class HookEvent {
        <<enumeration>>
        PreToolUse           -- Before tool execution
        PostToolUse          -- After tool execution
        PostToolUseFailure   -- Tool execution failure
        PermissionDenied     -- Permission denied
        PermissionRequest    -- Permission request
        Notification         -- Notification sent
        UserPromptSubmit     -- User submits prompt
        SessionStart         -- Session starts
        SessionEnd           -- Session ends
        Stop                 -- Before response ends
        StopFailure          -- API error ends turn
        SubagentStart        -- Subagent starts
        SubagentStop         -- Subagent ends
        PreCompact           -- Before compaction
        PostCompact          -- After compaction
        Setup                -- Repo setup
        TeammateIdle         -- Teammate idle
        TaskCreated          -- Task created
        TaskCompleted        -- Task completed
        Elicitation          -- MCP user input request
        ElicitationResult    -- MCP input response
        ConfigChange         -- Config changed
        InstructionsLoaded   -- Instruction file loaded
        WorktreeCreate       -- Worktree created
        WorktreeRemove       -- Worktree removed
        CwdChanged           -- Working directory changed
        FileChanged          -- File changed
    }
```

### Hook Command Types

| Type | Description | Config Fields |
|------|-------------|---------------|
| `command` | Execute shell command | `command`, `shell`, `if?`, `timeout?` |
| `prompt` | Execute prompt hook, append output to context | `prompt`, `if?`, `timeout?` |
| `agent` | Use LLM Agent to verify conditions | `prompt`, `model?`, `if?`, `timeout?` |
| `http` | Send HTTP request | `url`, `method?`, `headers?`, `body?` |
| `function` | Execute TypeScript callback (session-level) | `callback`, `timeout?` |

## Core Components

### AsyncHookRegistry - Async Hook Management

Manages async hook registration, state tracking, and response collection.

```mermaid
classDiagram
    class PendingAsyncHook {
        +processId: string
        +hookId: string
        +hookName: string
        +hookEvent: HookEvent
        +toolName?: string
        +pluginId?: string
        +startTime: number
        +timeout: number
        +command: string
        +responseAttachmentSent: boolean
        +shellCommand?: ShellCommand
        +stopProgressInterval: () => void
    }

    class AsyncHookRegistry {
        -pendingHooks: Map~string, PendingAsyncHook~
        +registerPendingAsyncHook(params): void
        +getPendingAsyncHooks(): PendingAsyncHook[]
        +checkForAsyncHookResponses(): Promise~HookResponse[]~
        +removeDeliveredAsyncHooks(processIds): void
        +finalizePendingAsyncHooks(): Promise~void~
        +clearAllAsyncHooks(): void
    }
```

**Key Features:**

- **Timeout management**: Default 15s timeout, configurable
- **Progress tracking**: `startHookProgressInterval` sends stdout updates periodically
- **Response collection**: Parses JSON line output, supports `{"ok": true/false, "reason": "..."}` format

### HookEvents - Event Broadcasting System

An event system independent of the main message stream for broadcasting hook execution events.

```mermaid
sequenceDiagram
    participant Source as Hook Executor
    participant Events as hookEvents
    participant Handler as Event Handler
    participant UI as UI Components

    Source->>Events: emitHookStarted(hookId, name, event)
    Events->>Handler: HookStartedEvent {type: 'started'}
    Source->>Events: emitHookProgress({stdout, stderr})
    Events->>Handler: HookProgressEvent {type: 'progress'}
    Source->>Events: emitHookResponse({output, exitCode})
    Events->>Handler: HookResponseEvent {type: 'response'}
    Handler->>UI: Update progress display
```

**Event Types:**

```typescript
type HookStartedEvent = {
  type: 'started'
  hookId: string
  hookName: string
  hookEvent: string
}

type HookProgressEvent = {
  type: 'progress'
  hookId: string
  hookName: string
  hookEvent: string
  stdout: string
  stderr: string
  output: string
}

type HookResponseEvent = {
  type: 'response'
  hookId: string
  hookName: string
  hookEvent: string
  output: string
  stdout: string
  stderr: string
  exitCode?: number
  outcome: 'success' | 'error' | 'cancelled'
}
```

### SessionHooks - Session-Level Hooks

Session hooks are temporary, in-memory callbacks that are not persisted to config files.

```mermaid
stateDiagram-v2
    [*] --> Empty: New Session
    Empty --> HasHooks: addSessionHook()
    Empty --> HasFunctionHook: addFunctionHook()
    HasHooks --> HasBoth: addFunctionHook()
    HasBoth --> HasHooks: removeFunctionHook()
    HasHooks --> Empty: clearSessionHooks()
    HasFunctionHook --> Empty: clearSessionHooks()
```

**Exported Functions:**

| Function | Description |
|----------|-------------|
| `addSessionHook()` | Add command or prompt hook |
| `addFunctionHook()` | Add function hook, returns hook ID |
| `removeFunctionHook()` | Remove function hook by ID |
| `removeSessionHook()` | Remove specific hook |
| `getSessionHooks()` | Get all session hooks |
| `getSessionFunctionHooks()` | Get function hooks |
| `clearSessionHooks()` | Clear all session hooks |

**Function Hook Callback:**

```typescript
type FunctionHookCallback = (
  messages: Message[],
  signal?: AbortSignal
) => boolean | Promise<boolean>
```

### HookHelpers - Helper Functions

Provides hook response schema and structured output tool creation.

```typescript
// Hook response schema
const hookResponseSchema = z.object({
  ok: z.boolean().describe('Whether condition is met'),
  reason: z.string().describe('Reason if not met').optional(),
})

// Create structured output tool
const tool = createStructuredOutputTool()

// Register structured output enforcement
registerStructuredOutputEnforcement(setAppState, sessionId)
```

## Hook Execution Flow

### Agent Hook Execution Flow

```mermaid
sequenceDiagram
    participant Caller as Caller
    participant Query as query()
    participant Agent as Hook Agent
    participant Tools as Tools
    participant Output as StructuredOutput

    Caller->>Agent: execAgentHook(hook, jsonInput)
    Agent->>Agent: Create user message + system prompt
    Agent->>Query: query({messages, systemPrompt})
    loop Multiple Turns
        Query->>Agent: Process message
        Agent->>Tools: Call tools
        Tools-->>Agent: Return results
        Agent->>Output: Check structured output
    end
    Agent-->>Caller: HookResult {outcome, blockingError?}
```

**Agent Hook Features:**

- Default timeout: 60 seconds
- Max 50 conversation turns
- Auto-injects `SyntheticOutputTool`
- Filters disallowed tools (subagents, plan mode, etc.)

### HTTP Hook Execution

HTTP hooks support integration with external services, returning JSON format responses:

```typescript
type HttpHookResult = {
  hookSpecificOutput?: {
    watchPaths?: string[]  // Dynamically update file watching
  }
}
```

## Configuration Management

### Configuration Source Priority

```mermaid
flowchart LR
    subgraph Priority["Priority (High→Low)"]
        Local["Local Settings"]
        Project["Project Settings"]
        User["User Settings"]
    end

    Local -->|"override"| Project
    Project -->|"override"| User
```

### Hook Registry

```mermaid
classDiagram
    class IndividualHookConfig {
        +event: HookEvent
        +config: HookCommand
        +matcher?: string
        +source: HookSource
        +pluginName?: string
    }

    class HookSource {
        <<union>>
        userSettings
        projectSettings
        localSettings
        policySettings
        pluginHook
        sessionHook
        builtinHook
    }
```

## Usage Examples

### Basic Command Hook

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Tool: {tool_name}'",
            "shell": "bash"
          }
        ]
      }
    ]
  }
}
```

### Agent Hook Verification

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "agent",
            "prompt": "Verify that the code meets: {argument}",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

### Session Hook (Code Integration)

```typescript
import { addFunctionHook, removeFunctionHook } from './utils/hooks/sessionHooks'

// Add verification hook
const hookId = addFunctionHook(
  setAppState,
  sessionId,
  'UserPromptSubmit',
  '',
  async (messages) => {
    const lastMessage = messages[messages.length - 1]
    const content = lastMessage.content[0]?.text ?? ''
    return content.length <= 1000
  },
  'Prompt content must be less than 1000 characters'
)

// Remove hook
removeFunctionHook(setAppState, sessionId, 'UserPromptSubmit', hookId)
```

## CLI Commands

### /hooks Command

View current session's hook configurations:

```bash
/claude hooks
```

Displays:
- All hooks grouped by event
- Hook sources (user/project/local/plugin/session)
- Matcher configuration
- Hook type and command

### /context Command

Visualize current context usage:

```bash
/claude context
```

Displays:
- Message count statistics
- Token usage
- Context compression information
- Distribution by tool usage

## Best Practices

### 1. Hook Timeout Settings

```json
{
  "type": "command",
  "command": "long-running-script.sh",
  "timeout": 300
}
```

Recommendations:
- Quick checks: `timeout: 5-10`
- Medium operations: `timeout: 30-60`
- Long-running tasks: `timeout: 300+`

### 2. Conditional Execution

Use `if` field to limit hook triggering conditions:

```json
{
  "type": "command",
  "command": "check-format.sh",
  "if": "Bash(git *)"
}
```

### 3. Async Response Handling

For hooks requiring long processing, return structured JSON:

```bash
#!/bin/bash
# Return success
echo '{"ok": true}'
# Return failure with reason
echo '{"ok": false, "reason": "Code format does not comply"}'
```

### 4. Error Handling

Exit code meanings:
- `0`: Success, stdout shown to model or user
- `2`: Blocking error, stderr shown to model and blocks operation
- Other: Non-blocking error, stderr shown to user only

## Design Decisions

### Why Use Map Instead of Record?

SessionHooks uses `Map<string, SessionStore>` instead of `Record<string, SessionStore>`:

```typescript
// Map: O(1) insert, doesn't change container reference
prev.sessionHooks.set(sessionId, { hooks: newHooks })
return prev  // Reference unchanged, Object.is short-circuits

// Record: O(N) copy each time, O(N²) total complexity
prev.sessionHooks[sessionId] = { hooks: newHooks }
return { ...prev }  // Triggers all listeners
```

### Why Distinguish Sync and Async Hooks?

- **Sync hooks**: Command returns immediately, determine result via exit code and stdout
- **Async hooks**: Command may run long, collect JSON responses via periodic polling

### Structured Output Tool Design

Hook system enforces `SyntheticOutputTool` for structured results:
- Ensures consistent hook response format
- Avoids parsing natural language output
- Supports complex verification logic

## Related Documents

- [Tools System](/wiki/core/tools.md)
- [Query Engine](/wiki/core/query.md)
- [Configuration System](/wiki/core/commands.md)

---

*Generated by Nium-Wiki | 2026-04-01*
