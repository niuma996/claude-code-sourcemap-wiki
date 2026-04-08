# Command System

## Overview

The command system is one of the core modules of Claude Code, responsible for managing and executing all slash commands. The system supports 80+ built-in commands and provides extension mechanisms for plugins and skills.

The command system uses a lazy loading strategy, loading command modules only when needed to ensure fast startup performance. The core implementation is in the [commands.ts](/restored-src/src/commands.ts) file.

## Architecture Position

```mermaid
flowchart TB
    subgraph CM ["Command System"]
        CommandsTS[commands.ts]
        CommandTypes[Command Types]
        SkillSystem[Skill System]
        PluginSystem[Plugin System]
    end
    CommandsTS --> CommandTypes
    CommandsTS --> SkillSystem
    CommandsTS --> PluginSystem
```

## Features

| Feature | Description | Related Files |
|---------|-------------|---------------|
| Command Registration | Unified registration of all built-in commands | [commands.ts](/restored-src/src/commands.ts) |
| Dynamic Loading | Support for skill and plugin commands | [loadSkillsDir.ts](/restored-src/src/skills/loadSkillsDir.ts) |
| Availability Checks | Command-level availability control | `meetsAvailabilityRequirement()` |
| Feature Gating | Conditional loading based on feature flags | `feature()` function |
| Remote Mode Filtering | Filter commands not safe for remote mode | `REMOTE_SAFE_COMMANDS` |

## File Structure

```
restored-src/src/
├── commands.ts              # Command system main file
├── commands/               # Command implementation directory
│   ├── help/              # /help command
│   ├── config/            # /config command
│   ├── login/             # /login command
│   ├── logout/            # /logout command
│   ├── commit/           # /commit command
│   ├── review/            # /review command
│   ├── diff/              # /diff command
│   ├── tasks/             # /tasks command
│   ├── skills/            # /skills command
│   ├── mcp/               # /mcp command
│   ├── agents/            # /agents command
│   ├── context/           # /context command
│   ├── compact/           # /compact command
│   ├── resume/            # /resume command
│   └── ...                # 70+ other commands
```

## Core Workflow

```mermaid
sequenceDiagram
    participant User
    participant CLI
    participant Commands
    participant Skills
    participant Plugins
    participant LoadAll

    User->>CLI: Input /command
    CLI->>Commands: Call getCommands()
    Commands->>LoadAll: Load all command sources
    LoadAll->>Skills: Load skill commands
    LoadAll->>Plugins: Load plugin commands
    Skills-->>Commands: Return skill list
    Plugins-->>Commands: Return plugin list
    Commands-->>CLI: Return available commands list
    CLI->>Commands: findCommand()
    Commands-->>CLI: Return matching command
    CLI->>Commands: Execute command
```

## API Summary

| Function | Description | Return Type |
|----------|-------------|-------------|
| `getCommands(cwd)` | Get all available commands | `Promise<Command[]>` |
| `findCommand(name, commands)` | Find specific command | `Command \| undefined` |
| `getCommand(name, commands)` | Get command (throws if not found) | `Command` |
| `hasCommand(name, commands)` | Check if command exists | `boolean` |
| `meetsAvailabilityRequirement(cmd)` | Check command availability | `boolean` |
| `getSkillToolCommands(cwd)` | Get skill command list | `Promise<Command[]>` |
| `filterCommandsForRemoteMode(commands)` | Filter remote-safe commands | `Command[]` |

## Command Types

| Type | Description | Example |
|------|-------------|---------|
| `prompt` | Prompt command, expanded to text | `/help`, `/skills` |
| `local` | Local command, text output | `/version`, `/compact` |
| `local-jsx` | Local JSX command, renders UI | `/config`, `/tasks` |
| `special` | Special command | Internal use |

## Command Availability

Commands support multiple availability controls:

```typescript
type Availability = 'claude-ai' | 'console'

// Example: Only available to Claude.ai subscribers
{
  name: 'voice',
  availability: ['claude-ai'],
  // ...
}
```

## Feature Gating

Use the `feature()` function for conditional compilation:

```typescript
// Only load when VOICE_MODE is enabled
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null

// Only available in KAIROS mode
const assistantCommand = feature('KAIROS')
  ? require('./commands/assistant/index.js').default
  : null
```

## Remote Mode Safety

Only safe commands can be used in remote mode:

```typescript
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([
  session, exit, clear, help, theme, color,
  vim, cost, usage, copy, btw, feedback,
  plan, keybindings, statusline, stickers, mobile,
])
```

## Source References

- [commands.ts](/restored-src/src/commands.ts#L1-L100)
- [commands/help/](/restored-src/src/commands/help/)
- [commands/config/](/restored-src/src/commands/config/)
- [commands/commit/](/restored-src/src/commands/commit/)

---

*Generated by [Nium-Wiki v0.0.0](https://github.com/niuma996/nium-wiki) | 2026-03-31*
