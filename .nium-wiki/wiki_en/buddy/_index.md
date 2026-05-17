# AI Buddy (Companion)

## Overview

AI Buddy is Claude Code's playful companion feature — a companion creature rendered as ASCII art in the terminal, with randomly generated attributes (species, appearance, personality) and independent interaction capabilities. The companion expresses reactions via speech bubbles during conversation, and users can interact via `/buddy` commands.

Buddy is not a utility feature — it is a complete end-to-end subsystem with:
- **Random generation system**: deterministic companion generation based on user ID (Bones + Soul model)
- **Rendering system**: 19 ASCII art sprites with animated frames
- **React UI components**: speech bubbles, terminal column reservation, floating bubbles
- **Prompt integration**: companion intro text injected into system prompt
- **Launch notification**: time-limited rainbow teaser (April 1-7, 2026)

## Submodules

| Module | Description | Doc |
|--------|-------------|-----|
| [Companion Roll System](companion.md) | Deterministic generation, attribute calculation, caching | [companion.md](companion.md) |
| [Sprites Rendering](sprites.md) | 19 ASCII art species, animation frames, face rendering | [sprites.md](sprites.md) |
| [CompanionSprite UI](companion-sprite-ui.md) | React component, bubbles, animation timer | [companion-sprite-ui.md](companion-sprite-ui.md) |
| [Buddy Notifications](buddy-notifications.md) | Rainbow teaser, `/buddy` trigger position detection | [buddy-notifications.md](buddy-notifications.md) |

## Architecture Position

```mermaid
flowchart TB
    subgraph Buddy["AI Buddy"]
        subgraph Types["Type System"]
            types["types.ts"]
        end
        subgraph Generation["Generation Layer"]
            companion["companion.ts"]
        end
        subgraph Rendering["Rendering Layer"]
            sprites["sprites.ts"]
            CompanionSprite["CompanionSprite.tsx"]
            CompanionBubble["CompanionFloatingBubble"]
        end
        subgraph Integration["Integration Layer"]
            prompt["prompt.ts"]
            notifications["useBuddyNotification.tsx"]
        end
    end

    types --> companion
    companion -->|CompanionBones| sprites
    companion -->|Companion| prompt
    companion -->|Companion| CompanionSprite
    CompanionSprite --> sprites
    CompanionBubble -->|companionReaction| CompanionSprite
    prompt -->|companionIntroText| AppState
    notifications -->|teaser| useNotifications
```

## Core Concepts

### Bones vs. Soul Separation Model

Buddy's persistence model splits a companion into two independent dimensions:

```mermaid
flowchart LR
    subgraph CompanionAttributes["Full Companion Attributes"]
        Bones["CompanionBones\n(Appearance)"]
        Soul["CompanionSoul\n(Spirit)"]
        Hatched["hatchedAt"]
    end

    subgraph PersistenceStrategy["Persistence Strategy"]
        Persisted["StoredCompanion\n(config.companion)"]
        Regenerated["CompanionBones\n(regenerated from hash at runtime)"]
    end

    Bones -->|regenerated from hash| Regenerated
    Soul --> Persisted
    Hatched --> Persisted
    Regenerated -->|merge| Companion["Companion (full)"]
    Persisted -->|merge| Companion
```

| Dimension | Contents | Persistence | Generation |
|-----------|---------|-------------|-----------|
| **Bones** (appearance) | rarity, species, eye, hat, shiny, stats | **Not persisted** | `hash(userId)` deterministic at runtime |
| **Soul** (spirit) | name, personality | `config.companion` | AI model generated at hatch |
| **hatchedAt** | hatch timestamp | `config.companion` | Recorded at first hatch |

**Key design**: Bones are never persisted — regenerated from `hash(userId)` on every read. This prevents:
1. Users from editing config to reroll higher rarity
2. Companions being lost when species names change in code

### Rarity System

```mermaid
stateDiagram-v2
    [*] --> Roll: mulberry32(hash(userId + SALT))
    Roll -->|60%| common: ★
    Roll -->|25%| uncommon: ★★
    Roll -->|10%| rare: ★★★
    Roll -->|4%| epic: ★★★★
    Roll -->|1%| legendary: ★★★★★

    common --> Stats: top stat 60–89 / dump stat 1–30
    uncommon --> Stats
    rare --> Stats: top stat +15
    epic --> Stats: top stat +25
    legendary --> Stats: top stat +50

    Stats --> Shiny: 1% chance
    Stats --> Hat: common companions never get hats
    Hat --> [*]: CompanionBones complete
```

| Rarity | Probability | Top Stat Range | Dump Stat Range |
|--------|------------|---------------|----------------|
| common | 60% | 60–89 | 1–30 |
| uncommon | 25% | 65–89 | 1–30 |
| rare | 10% | 70–89 | 1–30 |
| epic | 4% | 80–89 | 1–30 |
| legendary | 1% | 85–89 | 1–30 |

## Interaction Capabilities

### Speech Bubble Reactions

The companion receives reaction text via `AppState.companionReaction`, displaying in a bubble beside the ASCII sprite:

```
    /\_/\
   ( o.o )  ← ASCII sprite
    > ^ <
  ┌────────┐
  │ 哇！    │  ← bubble (fades after 10s)
  │ 好厉害！ │
  └────────┘
```

### User Commands

| Command | Effect |
|---------|--------|
| `/buddy` | Show companion info (name, rarity, personality) |
| `/buddy pet` | Trigger heart animation (floating ♥ chars for 2.5s) |
| `/buddy rename <name>` | Rename companion (modifies soul) |

### Terminal Width Adaptation

```mermaid
flowchart TD
    subgraph WidthDetection["useTerminalSize()"]
        wide["columns >= 100"]
        narrow["columns < 100"]
    end

    wide -->|full sprite| Full["5-line ASCII sprite + bubble"]
    narrow -->|compact face| Compact["single-line face + quip text"]

    Full --> CompanionSprite["CompanionSprite component"]
    Compact --> CompanionSprite

    CompanionSprite -->|reservedColumns| PromptInput["PromptInput column reservation"]
```

## Source References

- [types.ts](/restored-src/src/buddy/types.ts)
- [companion.ts](/restored-src/src/buddy/companion.ts)
- [sprites.ts](/restored-src/src/buddy/sprites.ts)
- [CompanionSprite.tsx](/restored-src/src/buddy/CompanionSprite.tsx)
- [useBuddyNotification.tsx](/restored-src/src/buddy/useBuddyNotification.tsx)
- [prompt.ts](/restored-src/src/buddy/prompt.ts)

## Related Documents

- [Companion Roll System](companion.md)
- [Sprites Rendering](sprites.md)
- [CompanionSprite UI](companion-sprite-ui.md)
- [Buddy Notifications](buddy-notifications.md)
- [Home](../index.md)
- [System Architecture](../architecture.md)

---

*Generated by [Nium-Wiki v0.0.0](https://github.com/niuma996/nium-wiki) | 2026-04-01*
