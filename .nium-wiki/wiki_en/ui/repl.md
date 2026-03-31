# REPL Interface

## Overview

REPL (Read-Eval-Print Loop) interface is the primary user interaction interface for Claude Code, providing an interactive command-line experience. The interface uses Ink for rendering, supporting real-time input, output display, and command completion.

The core implementation is in [screens/REPL.tsx](/restored-src/src/screens/REPL.tsx).

## Architecture Position

```mermaid
flowchart TB
    subgraph UL ["UI Layer"]
        REPL[REPL Interface]
        Components[Component Library]
        State[State Management]
    end
    REPL --> Components
    REPL --> State
```

## Features

| Feature | Description | Related Files |
|---------|-------------|---------------|
| Command Input | User input handling | [components/TextInput.tsx](/restored-src/src/components/TextInput.tsx) |
| Message Display | Conversation message rendering | [components/Message.tsx](/restored-src/src/components/Message.tsx) |
| Streaming Output | Real-time streaming display | [components/Messages.tsx](/restored-src/src/components/Messages.tsx) |
| Command Completion | Smart command completion | [hooks/useTypeahead.tsx](/restored-src/src/hooks/useTypeahead.tsx) |

## File Structure

```
restored-src/src/
├── screens/
│   └── REPL.tsx              # REPL main interface
├── components/                # React components
│   ├── Message.tsx           # Message component
│   ├── MessageRow.tsx       # Message row
│   ├── Messages.tsx         # Message list
│   ├── TextInput.tsx        # Text input
│   ├── SearchBox.tsx        # Search box
│   ├── Onboarding.tsx      # Onboarding screen
│   └── ...
├── hooks/                    # React Hooks
│   ├── useInputBuffer.ts   # Input buffer
│   ├── useTextInput.ts     # Text input
│   ├── useCommandQueue.ts  # Command queue
│   └── ...
└── ink/                      # Ink rendering engine
```

## Core Workflow

```mermaid
sequenceDiagram
    participant User
    participant REPL
    participant Engine as QueryEngine
    participant Model as Claude

    User->>REPL: Enter message
    REPL->>REPL: Process input
    REPL->>Engine: Send query
    Engine->>Model: API request
    Model-->>Engine: Streaming response
    Engine-->>REPL: Real-time output
    REPL-->>User: Display output
```

## Interface Components

```mermaid
classDiagram
    class REPL {
        +messages: Message[]
        +inputValue: string
        +render() ReactNode
    }
    class Message {
        +id: string
        +role: string
        +content: string
        +timestamp: Date
    }
    class TextInput {
        +value: string
        +onChange() void
        +onSubmit() void
    }
    REPL --> Message
    REPL --> TextInput
```

## State Management

| State | Description | Management |
|-------|-------------|------------|
| Message List | Conversation history | React State |
| Input Value | Current input | React State |
| Send Status | Sending/complete | React State |
| Error State | Error messages | React State |
| Configuration | User configuration | AppState Store |

## Input Processing

### Input Buffer

```typescript
interface InputBuffer {
  value: string           // Current input value
  cursor: number         // Cursor position
  history: string[]      // History records
  historyIndex: number   // History index
}
```

### Command Parsing

| Type | Prefix | Description |
|------|--------|-------------|
| Normal Message | None | Sent directly to model |
| Slash Command | `/` | Execute built-in command |
| Special Command | `--` | CLI options |

## Streaming Output

REPL supports real-time streaming output:

```mermaid
sequenceDiagram
    participant Model
    participant Stream as Streaming Response
    participant REPL

    Model->>Stream: Send token
    Stream-->>REPL: Update display
    Note over REPL: Real-time rendering
    Model->>Stream: Send complete
    Stream-->>REPL: Complete
```

## User Experience

### Command Completion

| Feature | Trigger | Description |
|---------|---------|-------------|
| Slash Commands | Type `/` | Show available commands |
| Argument Completion | Space after command | Show argument options |
| File Path | Type path | Auto-complete files |

### Keyboard Shortcuts

| Shortcut | Function |
|----------|---------|
| `Ctrl+C` | Cancel current input |
| `Ctrl+L` | Clear screen |
| `Ctrl+R` | Search history |
| `↑/↓` | Browse history |
| `Tab` | Complete |
| `Esc` | Cancel |

## Rendering Optimization

| Optimization | Description |
|--------------|-------------|
| Virtual List | Only render visible messages |
| Incremental Rendering | Streaming updates instead of full refresh |
| Debouncing | Input debouncing to reduce rendering |
| Memoization | Cache components to avoid re-rendering |

## Source References

- [REPL.tsx](/restored-src/src/screens/REPL.tsx)
- [components/Message.tsx](/restored-src/src/components/Message.tsx)
- [components/TextInput.tsx](/restored-src/src/components/TextInput.tsx)
- [hooks/useInputBuffer.ts](/restored-src/src/hooks/useInputBuffer.ts)

## Related Documents

- [Ink Engine](ink.md)
- [Architecture Docs](../architecture.md)

---

*Generated by [Nium-Wiki v0.0.0](https://github.com/niuma996/nium-wiki) | 2026-03-31*
