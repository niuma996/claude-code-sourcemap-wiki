# Internal (Ant) vs. External User Feature Differences

> Core differentiation mechanism: `process.env.USER_TYPE === 'ant'` — branches for external users are eliminated at build time via DCE (Dead Code Elimination), ensuring external builds contain no internal-only code. Affects approximately **145 files**.

---

## 1. System Prompt Differences

File: `restored-src/src/constants/prompts.ts`

Differentiated via `process.env.USER_TYPE === 'ant'` conditional branches; external users cannot see this content:

### 1. Code Style Rules

```typescript
// prompts.ts:205-214
...(process.env.USER_TYPE === 'ant' ? [
  `Default to writing no comments. Only add one when the WHY is non-obvious...`,
  `Don't explain WHAT the code does, since well-named identifiers already do that...`,
  `Don't remove existing comments unless you're removing the code they describe...`,
  `Before reporting a task complete, verify it actually works...`,
] : [])
```

**Content**: Default to no comments, principles for removing unused comments, requirement to verify task completion.

### 2. Proactively Point Out Misconceptions

```typescript
// prompts.ts:225-229
...(process.env.USER_TYPE === 'ant' ? [
  `If you notice the user's request is based on a misconception, or spot a bug adjacent to what they asked about, say so. You're a collaborator, not just an executor—users benefit from your judgment, not just your compliance.`,
] : [])
```

**Content**: Encourages proactively pointing out user misconceptions or discovering related bugs, rather than merely executing instructions.

### 3. Faithful Reporting of Results

```typescript
// prompts.ts:238-242
...(process.env.USER_TYPE === 'ant' ? [
  `Report outcomes faithfully: if tests fail, say so with the relevant output; if you did not run a verification step, say that rather than implying it succeeded. Never claim "all tests pass" when output shows failures...`,
] : [])
```

**Content**: Forbids fabricating passing results; must report truthfully. Relevant to Capybara v8 False-Claims issue (internal 29-30% FC rate vs external 16.7%).

### 4. Bug Feedback Guidance

```typescript
// prompts.ts:243-247
...(process.env.USER_TYPE === 'ant' ? [
  `If the user reports a bug, slowness, or unexpected behavior with Claude Code itself... recommend /issue for model-related problems or /share to upload the full session transcript... After /share produces a ccshare link... offer to post the link to #claude-code-feedback (channel ID C07VBSHV7EV)`,
] : [])
```

**Content**: When internal employees encounter issues with Claude Code itself, recommend `/issue` and `/share` commands, with automatic posting to Slack `#claude-code-feedback` channel.

### 5. Output Efficiency Rules

```typescript
// prompts.ts:403-428
if (process.env.USER_TYPE === 'ant') {
  return `# Communicating with the user
When sending user-facing text, you're writing for a person, not logging to a console...
[Detailed communication style guide, ~400 words]
`
}
return `# Output efficiency
IMPORTANT: Go straight to the point. Try the simplest approach first without going in circles...`
```

**Content**:
- Internal employees: detailed communication style guide (no sentence fragments, excessive dashes, or symbolic markers; inverted pyramid structure; complete sentences, etc.)
- External users: only brief "be concise" requirement

### 6. Tone and Style Differences

```typescript
// prompts.ts:433-436
process.env.USER_TYPE === 'ant' ? null : `Your responses should short and concise.`
```

**Content**: External users see the "short and concise" requirement; internal employees do not have this constraint.

### 7. Undercover Mode

```typescript
// prompts.ts:621
if (process.env.USER_TYPE === 'ant' && isUndercover()) { /* suppress */ }
```

**Content**: Internal builds may suppress model name/ID display to prevent leakage into public commits/PRs. External builds skip this check entirely.

### 8. Model Override

```typescript
// prompts.ts:136-139
function getAntModelOverrideSection(): string | null {
  if (process.env.USER_TYPE !== 'ant') return null
  if (isUndercover()) return null
  return getAntModelOverrideConfig()?.defaultSystemPromptSuffix || null
}
```

**Content**: Internal users can configure a default system prompt suffix (GrowthBook flag `tengu_ant_model_override`).

### 9. Token Length Anchors

```typescript
// prompts.ts:529-537
...(process.env.USER_TYPE === 'ant' ? [
  systemPromptSection('numeric_length_anchors', () =>
    'Length limits: keep text between tool calls to ≤25 words. Keep final responses to ≤100 words unless the task requires more detail.',
  ),
] : [])
```

**Content**: Internal users have specific word count limits; external users have no such constraints.

### 10. Model Information Display

```typescript
// prompts.ts:694-702
process.env.USER_TYPE === 'ant' && isUndercover() ? null : `The most recent Claude model family is Claude 4.5/4.6...`
process.env.USER_TYPE === 'ant' && isUndercover() ? null : `Claude Code is available as a CLI in the terminal, desktop app...`
process.env.USER_TYPE === 'ant' && isUndercover() ? null : `Fast mode for Claude Code uses the same ${FRONTIER_MODEL_NAME} model...`
```

**Content**: All model version info and product descriptions are hidden in Undercover mode.

---

## 2. Undercover Mode (Internal Employee Public Repository Protection)

File: `restored-src/src/utils/undercover.ts`

Serves to prevent internal information leakage when internal employees work in public repositories.

### Activation Mechanism

```typescript
// undercover.ts:28-37
export function isUndercover(): boolean {
  if (process.env.USER_TYPE === 'ant') {
    if (isEnvTruthy(process.env.CLAUDE_CODE_UNDERCOVER)) return true
    // AUTO: active unless we've positively confirmed we're in an allowlisted internal repo
    return getRepoClassCached() !== 'internal'
  }
  return false
}
```

| Mode | Trigger Condition | Behavior |
|---|---|---|
| Force enabled | `CLAUDE_CODE_UNDERCOVER=1` | Active even in internal repos |
| Auto-detect | Default | Active unless repo matches internal allowlist (`internal`) |
| No forced disable | — | Safe default to prevent codename leakage |

### Anti-Leakage Instructions Content

```typescript
// undercover.ts:39-72
export function getUndercoverInstructions(): string {
  return `## UNDERCOVER MODE — CRITICAL
You are operating UNDERCOVER in a PUBLIC/OPEN-SOURCE REPOSITORY.
NEVER include in commit messages or PR descriptions:
- Internal model codenames (animal names like Capybara, Tengu, etc.)
- Unreleased model version numbers (e.g., opus-4-7, sonnet-4-8)
- Internal repo or project names (e.g., claude-cli-internal, anthropics/…)
- Internal tooling, Slack channels, or short links (e.g., go/cc, #claude-code-…)
- The phrase "Claude Code" or any mention that you are an AI
- Co-Authored-By lines or any other attribution
Write commit messages as a human developer would...`
}
```

### Usage Scenarios

- **Commit prompt**: `restored-src/src/commands/commit.ts:16-18` injects `getUndercoverInstructions()` after checking `isUndercover()`
- **Commit-Push-PR**: Same in `restored-src/src/commands/commit-push-pr.ts`
- **Undercover auto notice**: `shouldShowUndercoverAutoNotice()` in `restored-src/src/components/PromptInput/PromptInputFooter.tsx` decides whether to show the initial prompt popup

---

## 3. Beta Header Differences

Files: `restored-src/src/utils/betas.ts`, `restored-src/src/constants/betas.ts`

| Beta Header | Activation Condition | Purpose |
|---|---|---|
| `cli-internal-2026-02-09` | `USER_TYPE === 'ant'` + CLI entry | Internal CLI-specific experiment |
| `summarize-connector-text-2026-03-13` | `USER_TYPE === 'ant'` + GB flag `tengu_slate_prism` | Server-side connector-text summarization (anti-distillation), optionally enabled for internal employees |
| `token-efficient-tools-2026-03-28` | `USER_TYPE === 'ant'` + GB flag `tengu_amber_json_tools` | Efficient JSON tool format (~4.5% output token reduction) |
| `context-management` | Internal employees can set `USE_API_CONTEXT_MANAGEMENT=1` | API-side tool cleanup |
| GrowthBook `/config` Gates UI | `USER_TYPE === 'ant'` only | Internal Feature Flag override panel (runtime modification) |

---

## 4. Agent Model and Capability Differences

### Explore Agent Model

File: `restored-src/src/tools/AgentTool/built-in/exploreAgent.ts:78`

```typescript
model: process.env.USER_TYPE === 'ant' ? 'inherit' : 'haiku',
// Ants get inherit to use the main agent's model
// External users get haiku for speed
```

**Content**: Internal employees' Explore agent inherits the main agent's model (more powerful); external users get the lightweight Haiku model.

### Remote Isolated Agent

File: `restored-src/src/tools/AgentTool/prompt.ts:273-275`

```typescript
...(process.env.USER_TYPE === 'ant'
  ? `\n- You can set \`isolation: "remote"\` to run the agent in a remote CCR environment.`
  : '')
```

**Content**: Only internal employees support `isolation: "remote"` to run agents in a remote CCR environment.

### Explore Agent Runtime Configuration

File: `restored-src/src/tools/AgentTool/AgentTool.tsx`

GrowthBook flag `tengu_explore_agent` can further adjust Explore Agent configuration for internal employees at runtime; external users are unaffected.

---

## 5. Analytics / GrowthBook Differences

File: `restored-src/src/services/analytics/growthbook.ts`

| Feature | Internal (ant) | External Users |
|---|---|---|
| Feature Flag override UI | `/config` Gates tab allows runtime modification | Not available |
| Env variable override | `CLAUDE_INTERNAL_FC_OVERRIDES` injects flags at runtime | Not available |
| Debug logs | All GrowthBook operations have `logForDebugging` | Completely absent |
| Refresh interval | 20 minutes | 6 hours |
| Base URL | Overrideable via `CLAUDE_CODE_GB_BASE_URL` | Fixed `api.anthropic.com` |
| Email attribute | Prioritized from OAuth config to complete GrowthBook attributes | Not available |
| Initialization log | Logs clientKey, attributes, features count | None |

Key code locations:

```typescript
// growthbook.ts:498-506
const baseUrl = process.env.USER_TYPE === 'ant'
  ? process.env.CLAUDE_CODE_GB_BASE_URL || 'https://api.anthropic.com/'
  : 'https://api.anthropic.com/'

// growthbook.ts:1013-1016
const GROWTHBOOK_REFRESH_INTERVAL_MS =
  process.env.USER_TYPE !== 'ant'
    ? 6 * 60 * 60 * 1000 // External: 6 hours
    : 20 * 60 * 1000     // Internal: 20 minutes
```

---

## 6. API / Request Layer Differences

### API Configuration Override

| Environment Variable | Internal (ant) | External Users |
|---|---|---|
| `ANTHROPIC_BASE_URL` | Can override API endpoint | Not available |
| `ANTHROPIC_BETAS` | Can inject arbitrary beta headers | Not available |
| `CLAUDE_CODE_UNDERCOVER` | Can force Undercover on/off | Not available |

### Auto Mode Model Differences

File: `restored-src/src/utils/betas.ts:160-195`

```typescript
// Internal employees: denylist — allows all models not explicitly prohibited
if (process.env.USER_TYPE === 'ant') {
  if (m.includes('claude-3-')) return false
  if (/claude-(opus|sonnet|haiku)-4(?!-[6-9])/.test(m)) return false
  return true
}
// External users: allowlist — only specific models allowed
return /^claude-(opus|sonnet)-4-6/.test(m)
```

**Content**:
- Internal employees: denylist logic, allows all non-3.x and non-4.0-4.5 series models
- External users: only `claude-opus-4-6` and `claude-sonnet-4-6` supported (firstParty)

### MCP Delta Mode

File: `restored-src/src/utils/mcpInstructionsDelta.ts`

Internal employees may optionally enable MCP delta mode to reduce token overhead per API call.

---

## 7. Other Differences (by Feature Domain)

### Logging and Debugging
- Internal employees have `logForDebugging` debug output in almost every module
- External builds have zero debug logs
- Involves approximately **145 files** with `USER_TYPE === 'ant'` checks

### Permission System
File: `restored-src/src/utils/permissions/permissions.ts`

Internal employees can use internal environment variables like `CLAUDE_CODE_UNDERCOVER` to influence permission logic.

### Commit / PR Attribution
Files: `restored-src/src/commands/commit.ts`, `restored-src/src/commands/commit-push-pr.ts`

After checking `isUndercover()`, anti-attribution instructions are injected to remove `Co-Authored-By` lines and AI-related descriptions.

### 1P Event Logging
File: `restored-src/src/services/analytics/firstPartyEventLogger.ts`

Internal employees enable full 1P event logging for product analytics and experiment tracking.

---

## 8. Summary

### Main Differentiation Dimensions

```
process.env.USER_TYPE === 'ant'    ← ~145 files, build-time DCE eliminates external branches
├── System Prompt differentiation   ← prompts.ts (9 categories of prompts)
├── Undercover mode               ← Public repository protection
├── Internal-only Beta Headers    ← betas.ts
├── Agent model and capability diffs ← exploreAgent, prompt.ts
├── GrowthBook UI + refresh rate  ← growthbook.ts
├── API debug logs                ← growthbook.ts, api/claude.ts
├── Auto Mode model restrictions  ← betas.ts
└── Commit anti-attribution       ← commit.ts, undercover.ts
```

### Design Principles

1. **External users get a lean, stable experience**: no debug logs, no experimental features, no special Beta Headers
2. **Internal employees get richer capabilities**: experimental features, debugging tools, Feature Flag override panel
3. **Build-time DCE ensures security**: external build binaries contain no internal-only code paths
4. **Undercover protects internal information**: automatically hides model codenames, internal project names, Slack channels, etc. in public repositories
