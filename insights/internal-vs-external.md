# 内部员工（ant）与外部用户功能差异

> 核心区分机制：`process.env.USER_TYPE === 'ant'` — 构建时通过 DCE（Dead Code Elimination）消除外部分支，确保外部构建不含任何内部专用代码。涉及约 **145 个文件**。

---

## 一、System Prompt 提示词差异

文件：`restored-src/src/constants/prompts.ts`

通过 `process.env.USER_TYPE === 'ant'` 条件分支区分，外部用户看不到这些内容：

### 1. 代码风格规则

```typescript
// prompts.ts:205-214
...(process.env.USER_TYPE === 'ant' ? [
  `Default to writing no comments. Only add one when the WHY is non-obvious...`,
  `Don't explain WHAT the code does, since well-named identifiers already do that...`,
  `Don't remove existing comments unless you're removing the code they describe...`,
  `Before reporting a task complete, verify it actually works...`,
] : [])
```

**内容**：默认不写注释、删除无用注释的原则、必须验证任务完成的说明。

### 2. 主动指出误解

```typescript
// prompts.ts:225-229
...(process.env.USER_TYPE === 'ant' ? [
  `If you notice the user's request is based on a misconception, or spot a bug adjacent to what they asked about, say so. You're a collaborator, not just an executor—users benefit from your judgment, not just your compliance.`,
] : [])
```

**内容**：鼓励主动指出用户的误解或发现相关 Bug，而非仅仅执行指令。

### 3. 准确报告结果

```typescript
// prompts.ts:238-242
...(process.env.USER_TYPE === 'ant' ? [
  `Report outcomes faithfully: if tests fail, say so with the relevant output; if you did not run a verification step, say that rather than implying it succeeded. Never claim "all tests pass" when output shows failures...`,
] : [])
```

**内容**：禁止捏造通过结果，必须如实报告。适用于 Capybara v8 False-Claims 问题（内部 29-30% FC 率 vs 外部 16.7%）。

### 4. Bug 反馈引导

```typescript
// prompts.ts:243-247
...(process.env.USER_TYPE === 'ant' ? [
  `If the user reports a bug, slowness, or unexpected behavior with Claude Code itself... recommend /issue for model-related problems or /share to upload the full session transcript... After /share produces a ccshare link... offer to post the link to #claude-code-feedback (channel ID C07VBSHV7EV)`,
] : [])
```

**内容**：内部员工遇到 Claude Code 自身问题时，推荐 `/issue` 和 `/share` 命令，并可自动发布到 Slack `#claude-code-feedback` 频道。

### 5. 输出效率规则

```typescript
// prompts.ts:403-428
if (process.env.USER_TYPE === 'ant') {
  return `# Communicating with the user
When sending user-facing text, you're writing for a person, not logging to a console...
[详细沟通风格指南，约 400 字]
`
}
return `# Output efficiency
IMPORTANT: Go straight to the point. Try the simplest approach first without going in circles...`
```

**内容**：
- 内部员工：详细的沟通风格指南（禁止片段式句子、过多破折号、符号标记；使用倒金字塔结构；完整句子等）
- 外部用户：仅简短的"be concise"要求

### 6. 语气风格差异

```typescript
// prompts.ts:433-436
process.env.USER_TYPE === 'ant' ? null : `Your responses should short and concise.`
```

**内容**：外部用户看到"short and concise"要求；内部员工无此限制。

### 7. Undercover 模式

```typescript
// prompts.ts:621
if (process.env.USER_TYPE === 'ant' && isUndercover()) { /* suppress */ }
```

**内容**：内部构建时可隐藏模型名称/ID，防止泄露到公开 commit/PR。外部构建直接跳过此检查。

### 8. 模型覆盖

```typescript
// prompts.ts:136-139
function getAntModelOverrideSection(): string | null {
  if (process.env.USER_TYPE !== 'ant') return null
  if (isUndercover()) return null
  return getAntModelOverrideConfig()?.defaultSystemPromptSuffix || null
}
```

**内容**：内部用户可配置默认 system prompt 后缀（GrowthBook flag `tengu_ant_model_override`）。

### 9. Token 长度锚点

```typescript
// prompts.ts:529-537
...(process.env.USER_TYPE === 'ant' ? [
  systemPromptSection('numeric_length_anchors', () =>
    'Length limits: keep text between tool calls to ≤25 words. Keep final responses to ≤100 words unless the task requires more detail.',
  ),
] : [])
```

**内容**：内部用户有具体字数限制；外部用户无此约束。

### 10. 模型信息展示

```typescript
// prompts.ts:694-702
process.env.USER_TYPE === 'ant' && isUndercover() ? null : `The most recent Claude model family is Claude 4.5/4.6...`
process.env.USER_TYPE === 'ant' && isUndercover() ? null : `Claude Code is available as a CLI in the terminal, desktop app...`
process.env.USER_TYPE === 'ant' && isUndercover() ? null : `Fast mode for Claude Code uses the same ${FRONTIER_MODEL_NAME} model...`
```

**内容**：Undercover 模式下隐藏所有模型版本信息和产品描述。

---

## 二、Undercover 模式（内部员工公开仓库保护）

文件：`restored-src/src/utils/undercover.ts`

专门服务内部员工在公开仓库中工作时防止泄露内部信息。

### 激活机制

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

| 模式 | 触发条件 | 行为 |
|---|---|---|
| 强制开启 | `CLAUDE_CODE_UNDERCOVER=1` | 即使在内网仓库也开启 |
| 自动检测 | 默认 | 除非 repo 匹配内部白名单（`internal`），否则开启 |
| 无强制关闭 | — | 防止 codename 泄露的 safe default |

### 反泄露指令内容

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

### 使用场景

- **Commit 提示词**：`restored-src/src/commands/commit.ts:16-18` 检查 `isUndercover()` 后注入 `getUndercoverInstructions()`
- **Commit-Push-PR**：`restored-src/src/commands/commit-push-pr.ts` 同上
- **Undercover 自动提示**：`restored-src/src/components/PromptInput/PromptInputFooter.tsx` 中 `shouldShowUndercoverAutoNotice()` 决定是否显示初次提示弹窗

---

## 三、Beta Header 差异

文件：`restored-src/src/utils/betas.ts`、`restored-src/src/constants/betas.ts`

| Beta Header | 生效条件 | 作用 |
|---|---|---|
| `cli-internal-2026-02-09` | `USER_TYPE === 'ant'` + CLI 入口 | 内部 CLI 专用实验 |
| `summarize-connector-text-2026-03-13` | `USER_TYPE === 'ant'` + GB flag `tengu_slate_prism` | 服务端 connector-text 摘要（防蒸馏），内部员工可选开启 |
| `token-efficient-tools-2026-03-28` | `USER_TYPE === 'ant'` + GB flag `tengu_amber_json_tools` | 高效 JSON 工具格式（~4.5% output token 减少） |
| `context-management` | 内部员工可主动设置 `USE_API_CONTEXT_MANAGEMENT=1` | API 端工具清理 |
| GrowthBook `/config` Gates UI | 仅 `USER_TYPE === 'ant'` | 内部 Feature Flag 覆盖面板（运行时改写） |

---

## 四、Agent 模型与能力差异

### Explore Agent 模型

文件：`restored-src/src/tools/AgentTool/built-in/exploreAgent.ts:78`

```typescript
model: process.env.USER_TYPE === 'ant' ? 'inherit' : 'haiku',
// Ants get inherit to use the main agent's model
// External users get haiku for speed
```

**内容**：内部员工探索 agent 继承主 agent 模型（更强大）；外部用户使用轻量 Haiku 模型。

### 远程隔离 Agent

文件：`restored-src/src/tools/AgentTool/prompt.ts:273-275`

```typescript
...(process.env.USER_TYPE === 'ant'
  ? `\n- You can set \`isolation: "remote"\` to run the agent in a remote CCR environment.`
  : '')
```

**内容**：仅内部员工支持 `isolation: "remote"` 在远程 CCR 环境中运行 agent。

### Explore Agent 运行时配置

文件：`restored-src/src/tools/AgentTool/AgentTool.tsx`

GrowthBook flag `tengu_explore_agent` 在运行时可进一步调整内部员工的 Explore Agent 配置，外部用户不生效。

---

## 五、Analytics / GrowthBook 差异

文件：`restored-src/src/services/analytics/growthbook.ts`

| 功能 | 内部（ant） | 外部用户 |
|---|---|---|
| Feature Flag 覆盖 UI | `/config` Gates 标签页可运行时改写 | 不可用 |
| Env 变量覆盖 | `CLAUDE_INTERNAL_FC_OVERRIDES` 可在运行时注入 flag | 不可用 |
| 调试日志 | 所有 GrowthBook 操作均有 `logForDebugging` | 完全无日志 |
| 刷新间隔 | 20 分钟 | 6 小时 |
| Base URL | 可通过 `CLAUDE_CODE_GB_BASE_URL` 覆盖 | 固定 `api.anthropic.com` |
| Email 属性 | 优先从 OAuth 配置补全 GrowthBook 属性 | 不可用 |
| 初始化日志 | 记录 clientKey、attributes、features 数量 | 无 |

核心代码位置：

```typescript
// growthbook.ts:498-506
const baseUrl = process.env.USER_TYPE === 'ant'
  ? process.env.CLAUDE_CODE_GB_BASE_URL || 'https://api.anthropic.com/'
  : 'https://api.anthropic.com/'

// growthbook.ts:1013-1016
const GROWTHBOOK_REFRESH_INTERVAL_MS =
  process.env.USER_TYPE !== 'ant'
    ? 6 * 60 * 60 * 1000 // 外部: 6 小时
    : 20 * 60 * 1000     // 内部: 20 分钟
```

---

## 六、API / 请求层面差异

### API 配置覆盖

| 环境变量 | 内部（ant） | 外部用户 |
|---|---|---|
| `ANTHROPIC_BASE_URL` | 可覆盖 API 端点 | 不可用 |
| `ANTHROPIC_BETAS` | 可注入任意 beta header | 不可用 |
| `CLAUDE_CODE_UNDERCOVER` | 可强制开启/关闭 undercover | 不可用 |

### Auto Mode 模型差异

文件：`restored-src/src/utils/betas.ts:160-195`

```typescript
// 内部员工: denylist（黑名单）— 允许所有未明确禁止的模型
if (process.env.USER_TYPE === 'ant') {
  if (m.includes('claude-3-')) return false
  if (/claude-(opus|sonnet|haiku)-4(?!-[6-9])/.test(m)) return false
  return true
}
// 外部用户: allowlist（白名单）— 仅允许特定模型
return /^claude-(opus|sonnet)-4-6/.test(m)
```

**内容**：
- 内部员工：denylist 逻辑，允许所有非 3.x 系列和非 4.0-4.5 系列
- 外部用户：仅支持 `claude-opus-4-6` 和 `claude-sonnet-4-6`（firstParty）

### MCP Delta 模式

文件：`restored-src/src/utils/mcpInstructionsDelta.ts`

内部员工可选启用 MCP delta 模式，减少每次 API 调用的 token 开销。

---

## 七、其他差异（按功能域）

### 日志与调试
- 内部员工几乎所有模块都有 `logForDebugging` 调试输出
- 外部构建完全没有调试日志
- 涉及约 **145 个文件**的 `USER_TYPE === 'ant'` 检查

### 权限系统
文件：`restored-src/src/utils/permissions/permissions.ts`

内部员工可使用 `CLAUDE_CODE_UNDERCOVER` 等内部环境变量影响权限判断逻辑。

### Commit / PR 归因
文件：`restored-src/src/commands/commit.ts`、`restored-src/src/commands/commit-push-pr.ts`

检查 `isUndercover()` 后注入反归因指令，移除 `Co-Authored-By` 行和 AI 相关描述。

### 1P Event Logging
文件：`restored-src/src/services/analytics/firstPartyEventLogger.ts`

内部员工开启完整的 1P 事件日志记录，用于产品分析和实验追踪。

---

## 八、总结

### 主要区分维度

```
process.env.USER_TYPE === 'ant'    ← 约 145 个文件，构建时 DCE 消除外部分支
├── System Prompt 差异化            ← prompts.ts（9 类提示词）
├── Undercover 模式                ← 公开仓库保护
├── 内部专属 Beta Header           ← betas.ts
├── Agent 模型与能力差异            ← exploreAgent, prompt.ts
├── GrowthBook UI + 刷新频率        ← growthbook.ts
├── API 调试日志                   ← growthbook.ts, api/claude.ts
├── Auto Mode 模型限制             ← betas.ts
└── Commit 反归因                  ← commit.ts, undercover.ts
```

### 设计原则

1. **外部用户获得精简、稳定的体验**：无调试日志、无实验性功能、无特殊 Beta Header
2. **内部员工获得更丰富的能力**：实验性功能、调试手段、Feature Flag 覆盖面板
3. **构建时 DCE 确保安全**：外部构建的二进制不包含任何内部专用代码路径
4. **Undercover 保护内部信息**：在公开仓库中自动隐藏模型 codename、内部项目名称、Slack 渠道等信息
