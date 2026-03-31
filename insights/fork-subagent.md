# Fork Subagent 技术文档

> Fork 是 Claude Code 中的一种轻量级子代理机制。通过省略 `subagent_type` 触发，子代理继承父对话的完整上下文并共享父的 prompt cache，适用于研究、并行任务和实现类工作。

---

## 一、概念与定位

### 1.1 两种子代理范式

Claude Code 实现了两套子代理机制：

| | **Fork 子代理** | **Spawned Teammate** |
|---|---|---|
| 触发方式 | 省略 `subagent_type` | 显式指定 `subagent_type` |
| 运行位置 | 同进程，共享父 prompt cache | 独立进程（tmux/iTerm2 pane）|
| 上下文传递 | 字节级精确的 API 请求前缀复用 | CLI flag + 环境变量传播 |
| 模型限制 | 必须与父相同（否则 cache 失效） | 可指定不同模型 |
| 入口文件 | `forkSubagent.ts` | `spawnMultiAgent.ts` |

### 1.2 适用场景

Fork 的核心价值在于：**用子代理的隔离上下文换取主 agent 上下文的干净**。

- **研究类任务**：探索开放性问题，父不需要中间工具输出
- **实现类任务**：多步编辑工作，工具调用噪音不值得留在主上下文
- **并行独立任务**：多个互不依赖的子任务一次性启动，共享 cache 开销极低

Fresh Agent（显式 `subagent_type`）适用于需要**独立、无偏见视角**的场景，如 code review、second opinion。

### 1.3 开关条件

```typescript
// forkSubagent.ts:32-39
export function isForkSubagentEnabled(): boolean {
  if (feature('FORK_SUBAGENT')) {
    if (isCoordinatorMode()) return false   // 与 coordinator 互斥
    if (getIsNonInteractiveSession()) return false  // 非交互会话禁用
    return true
  }
  return false
}
```

`FORK_SUBAGENT` 是一个 GrowthBook experiment flag。开启时：
- `subagent_type` 变为可选，省略即走 fork 路径
- 引导主 agent 理解 fork 语义和 directive-style prompt 写法
- `/fork <directive>` 命令行入口可用（`commands/branch/index.ts`）

---

## 二、Prompt Cache 共享机制（核心设计）

> **💡 巧思 1：用占位符常量抹平差异，让所有 fork 子代理共享同一个 cache 前缀**

### 2.1 问题背景

子代理启动时，父对话历史作为上下文传入。如果每次子代理都重新构建请求前缀，prompt cache 命中率极低。Claude Code 的 API 计费中，prompt cache 创建是重要成本来源。

### 2.2 解决方案：字节级一致的 API 请求前缀

核心逻辑在 `buildForkedMessages()`（`forkSubagent.ts:107-169`）：

```typescript
export function buildForkedMessages(
  directive: string,
  assistantMessage: AssistantMessage,
): MessageType[] {
  // 1. 克隆父 assistant message（保留所有 tool_use、thinking、text blocks）
  const fullAssistantMessage: AssistantMessage = {
    ...assistantMessage,
    uuid: randomUUID(),
    message: {
      ...assistantMessage.message,
      content: [...assistantMessage.message.content],
    },
  }

  // 2. 收集所有 tool_use blocks
  const toolUseBlocks = assistantMessage.message.content.filter(
    (block): block is BetaToolUseBlock => block.type === 'tool_use'
  )

  // 3. 为每个 tool_use 生成占位 tool_result
  //    所有占位符文本完全相同 — 这是 cache 命中的关键
  const toolResultBlocks = toolUseBlocks.map(block => ({
    type: 'tool_result' as const,
    tool_use_id: block.id,
    content: [{ type: 'text', text: FORK_PLACEHOLDER_RESULT }],
    //                    ↑ 硬编码常量: 'Fork started — processing in background'
  }))

  // 4. 合成单一 user message：[占位 tool_results..., per-child directive]
  const toolResultMessage = createUserMessage({
    content: [
      ...toolResultBlocks,
      { type: 'text', text: buildChildMessage(directive) },
    ],
  })

  return [fullAssistantMessage, toolResultMessage]
}
```

**关键洞察**：只有最后一个 text block（`buildChildMessage(directive)` 的输出）是 per-child 的。其余所有部分（system prompt、assistant message、tool_results 占位符）在所有 fork 子代理间完全一致。这意味着 API 请求前缀的 cache key 完全相同，首次调用后，后续所有 fork 子代理共享缓存。

### 2.3 系统提示的字节级精确传递

> **💡 巧思 2：`renderedSystemPrompt` 在 turn 起始冻结传递，而非重新计算 — 绕过 GrowthBook 状态漂移**

```typescript
// AgentTool.tsx:495-511
if (isForkPath) {
  if (toolUseContext.renderedSystemPrompt) {
    // 直接复用父已渲染的字节 — 字节级精确
    forkParentSystemPrompt = toolUseContext.renderedSystemPrompt
  } else {
    // Fallback: 重新计算（可能因 GrowthBook 冷→热状态变化而与缓存字节不同）
    forkParentSystemPrompt = buildEffectiveSystemPrompt({ ... })
  }
  promptMessages = buildForkedMessages(prompt, assistantMessage)
}
```

`renderedSystemPrompt` 在 turn 起始时冻结（`contentReplacementState` 在 `ToolUseContext` 中），threading 到子代理时避免了 GrowthBook feature flag 在 turn 过程中变化导致的 cache bust。注释明确写道：

> 重新调用 `getSystemPrompt()` 可能因 GrowthBook 冷→热状态变化而产生不同输出，击穿 prompt cache；直接传递已渲染的字节是字节级精确的。

### 2.4 useExactTools 继承

```typescript
// AgentTool.tsx:630-633
forkContextMessages: isForkPath ? toolUseContext.messages : undefined,
...(isForkPath && { useExactTools: true }),
```

`useExactTools: true` 让子代理继承父的精确工具定义（包含 thinking 配置等），确保 API 请求前缀完全一致。

---

## 三、动态指令注入：FORK_BOILERPLATE_TAG

> **💡 巧思 3：一个 XML tag 同时服务两个目的 — 行为规范 + 递归检测，一石二鸟**

### 3.1 构建行为规范

`buildChildMessage()`（`forkSubagent.ts:171-198`）为每个子代理动态构建行为规范：

```typescript
export function buildChildMessage(directive: string): string {
  return `<${FORK_BOILERPLATE_TAG}>
STOP. READ THIS FIRST.

You are a forked worker process. You are NOT the main agent.

RULES (non-negotiable):
1. 你的系统提示说"default to forking" — 忽略它。你就是 fork。不要再 spawn 子代理，直接执行
2. 不要对话、提问、建议下一步
3. 不要编辑性评论或元注释
4. 直接使用工具：Bash、Read、Write 等
5. 修改文件后先提交，报告中包含 commit hash
6. 工具调用之间不要输出文字，结束时一次性报告
7. 严格限定在 directive 范围内
8. 报告控制在 500 字以内
9. 响应必须以 "Scope:" 开头
10. 报告结构化事实后停止

Output format（纯文本标签，非 markdown headers）:
  Scope: <范围>
  Result: <结果>
  Key files: <文件路径>
  Files changed: <含 commit hash>
  Issues: <问题列表>
</${FORK_BOILERPLATE_TAG}>

[FORK_DIRECTIVE_PREFIX]<用户 directive>
```

### 3.2 递归防护：双层检测

> **💡 巧思 4：双层递归防护 — 第一层在 autocompact 后依然存活，fallback 层兜底**

递归保护有两层，设计非常精巧：

```typescript
// AgentTool.tsx:325-334
if (
  toolUseContext.options.querySource === `agent:builtin:${FORK_AGENT.agentType}` ||
  isInForkChild(toolUseContext.messages)
) {
  throw new Error('Fork is not available inside a forked worker.')
}
```

- **主检测**：`querySource === 'agent:builtin:fork'` — 通过 `context.options.querySource` 设置，autocompact 重写消息时**不经过** context.options，所以 survive autocompact
- **Fallback**：消息扫描 `isInForkChild()` — 扫描 `FORK_BOILERPLATE_TAG` 是否在消息历史中（autocompact 之后 fallback 触发）

这解决了一个微妙问题：autocompact 会压缩/重写消息（替换 fork-boilerplate message），所以仅靠消息扫描无法可靠检测递归 fork；但 `context.options.querySource` 在 autocompact 后依然存在，形成了 autocompact 安全的主检测层。

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

## 四、上下文精细裁剪

> **💡 巧思 5：基于生产流量数据的量化裁剪 — 不是拍脑袋，是数据驱动**

`runAgent.ts` 对子代理上下文做了精准削减，每个裁剪都有量化收益支撑：

### 4.1 CLAUDE.md 省略

```typescript
// runAgent.ts:385-398
// 节省约 5-15 Gtok/week（基于 34M+ Explore spawns 统计）
const shouldOmitClaudeMd =
  agentDefinition.omitClaudeMd &&
  !override?.userContext &&
  getFeatureValue_CACHED_MAY_BE_STALE('tengu_slim_subagent_claudemd', true)
const { claudeMd: _omitted, ...userContextNoClaudeMd } = baseUserContext
```

只读代理（Explore、Plan）的 CLAUDE.md 对主 agent 解读输出无价值，属于噪音。注释直接写明：**节省约 5-15 Gtok/week（基于 34M+ Explore spawns 统计）**，说明是生产数据分析后的决策。

### 4.2 gitStatus 省略

```typescript
// runAgent.ts:400-410
// 节省约 1-3 Gtok/week
const resolvedSystemContext =
  agentDefinition.agentType === 'Explore' || agentDefinition.agentType === 'Plan'
    ? systemContextNoGit  // 去掉最多 40KB 的 gitStatus
    : baseSystemContext
```

Explore/Plan 是只读搜索代理，父会话起始的 gitStatus（明确标注为 stale，最多 40KB）对它们是死重。注释写道：**如果需要 git 信息，子代理自己执行 `git status` 获取新鲜数据** — 这既节省了 token，又保证了数据的实时性，是双赢。

### 4.3 可选 caller override 保护

注意 `!override?.userContext` 这个条件 — 如果 caller 显式传入了 `userContext`，则保留 CLAUDE.md。这是对 API 调用方的一种尊重：知道自己在做什么的人应该能覆盖默认行为。

---

## 五、SubagentStart Hook 动态注入

> **💡 巧思 6：统一消息类型 — hook 注入与 SessionStart/UserPromptSubmit 使用完全相同的 attachment message 格式**

`runAgent.ts:530-555` 在子代理启动时执行 hook，收集额外上下文：

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

// 注入为 attachment message，与 SessionStart/UserPromptSubmit 保持同一格式
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

统一消息类型的价值在于：模型在处理来自 SessionStart、UserPromptSubmit 和 SubagentStart 的 hook 上下文时使用相同的解析逻辑，不需要为 SubagentStart 单独处理。

---

## 六、Skill 预加载机制

> **💡 巧思 7：`isMeta: true` 的 meta user message — 对模型可见但不影响对话结构**

子代理 frontmatter 可声明 `skills`，在代理启动时并发预加载：

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

- **`isMeta: true`**：作为 meta user message 注入，对模型可见但不影响对话 turn 结构
- **并发加载**：`Promise.all()` 并发所有 skill 的 prompt 获取，不串行等待
- **多策略解析**：精确匹配 → 带前缀匹配（`plugin:skill-name`）→ 后缀匹配（`:skill-name`）

---

## 七、隔离的 ToolUseContext

> **💡 巧思 8：mutation callbacks 默认 no-op — 子代理无法污染父代理状态，除非父显式授权**

`createSubagentContext()`（`forkedAgent.ts:345-462`）为子代理创建隔离上下文：

```typescript
export function createSubagentContext(
  parentContext: ToolUseContext,
  overrides?: SubagentContextOverrides,
): ToolUseContext {
  // 克隆文件状态缓存（隔离但不丢失）
  readFileState: cloneFileStateCache(parentContext.readFileState),

  // 独立 abort controller（可与父链接或独立）
  abortController: overrides?.shareAbortController
    ? parentContext.abortController
    : createChildAbortController(parentContext.abortController),

  // mutation callbacks 默认 no-op，防止子代理意外修改父状态
  setAppState: overrides?.shareSetAppState ? parentContext.setAppState : () => {},
  setResponseLength: overrides?.shareSetResponseLength
    ? parentContext.setResponseLength : () => {},

  // 深度递增：用于 analytics 和防递归
  queryTracking: {
    chainId: randomUUID(),
    depth: (parentContext.queryTracking?.depth ?? -1) + 1,
  }
}
```

关键设计原则：**零信任子代理**。子代理的 `setAppState` 和 `setResponseLength` 默认 no-op，父的状态不会被意外污染。只有 fork 子代理在显式 `overrides` 时才共享这些 callbacks。

---

## 八、MCP Server 累加模型

> **💡 巧思 9：累加而非替换 — 子代理的 MCP servers 是对父的补充，cleanup 时只清理新建的**

子代理可定义自己的 MCP 服务器，与父的 MCP 客户端**累加合并**，而非替换：

```typescript
// runAgent.ts:95-218
return {
  clients: [...parentClients, ...agentClients],  // 累加
  tools: agentTools,
  cleanup  // 仅清理代理新建的，保留父的
}
```

- **累加**：父、子 MCP 服务器并存，子代理既能用父的 MCP 工具，也能用自己的
- **选择性清理**：`cleanup` 函数只关闭子代理新建的连接，不动父的

---

## 九、CLI Flag 传播（Spawned Teammate 路径）

对于 out-of-process teammate，通过 `buildInheritedCliFlags()` 传播父会话设置：

```typescript
// spawnUtils.ts:38-89
export function buildInheritedCliFlags(options?: {
  planModeRequired?: boolean
  permissionMode?: PermissionMode
}): string {
  const flags: string[] = []

  // 权限模式传播
  if (permissionMode === 'bypassPermissions' || getSessionBypassPermissionsMode()) {
    flags.push('--dangerously-skip-permissions')
  }

  // 模型覆盖
  const modelOverride = getMainLoopModelOverride()
  if (modelOverride) flags.push(`--model ${quote([modelOverride])}`)

  // teammate mode
  flags.push(`--teammate-mode ${sessionMode}`)
}
```

---

## 十、执行流程总览

```
主 Agent 调用 AgentTool({ prompt: "...", /* 无 subagent_type */ })
         │
         ▼
    AgentTool.tsx
    ├── isForkSubagentEnabled() → true？
    │   └── 是 → isForkPath = true
    ├── 递归检测：querySource（autocompact-safe）+ FORK_BOILERPLATE_TAG scan（fallback）
    ├── 选取 FORK_AGENT 定义
    ├── 构造 forkParentSystemPrompt（复用 renderedSystemPrompt，字节级一致）
    ├── buildForkedMessages(prompt, assistantMessage)
    │   └── 克隆父 assistant message + 占位 tool_results（常量文本）+ buildChildMessage(directive)
    └── runAgent({ agentDefinition: FORK_AGENT, override: { systemPrompt }, forkContextMessages, useExactTools: true })
            │
            ▼
        runAgent.ts
        ├── 裁剪上下文（CLAUDE.md、gitStatus 按条件省略 — 数据驱动优化）
        ├── executeSubagentStartHooks() 收集额外上下文（统一 attachment message 格式）
        ├── 预加载 skills（并发 + meta user message 注入）
        ├── 初始化 agent 专属 MCP servers（与父累加）
        ├── createSubagentContext() 创建隔离上下文（mutation callbacks 默认 no-op）
        └── query() 执行子代理对话循环
                │
                ▼
            子代理响应（Scope:/Result:/Files changed:/Issues:）
                │
                ▼
            <task-notification> 推送回主 agent
            主 agent 收到结构化结果，继续工作
```

---

## 十一、关键设计原则与巧思索引

| 编号 | 巧思 | 位置 |
|---|---|---|
| 1 | **占位符常量抹平差异**：所有 fork 子代理的 tool_results 用同一个占位文本，前缀 cache key 完全一致 | `forkSubagent.ts:107-169` |
| 2 | **`renderedSystemPrompt` 冻结传递**：turn 起始冻结，避免 GrowthBook 状态漂移导致 cache bust | `AgentTool.tsx:495-511` |
| 3 | **一石二鸟的 XML tag**：`FORK_BOILERPLATE_TAG` 同时是行为规范载体和递归检测标记 | `forkSubagent.ts:171-198` |
| 4 | **双层递归防护**：querySource survive autocompact，消息扫描做 fallback | `AgentTool.tsx:325-334` |
| 5 | **数据驱动的上下文裁剪**：CLAUDE.md/gitStatus 省略有 Gtok/week 量化收益支撑 | `runAgent.ts:385-410` |
| 6 | **统一消息类型**：hook 注入与 SessionStart/UserPromptSubmit 使用相同的 attachment message 格式 | `runAgent.ts:530-555` |
| 7 | **`isMeta: true` meta message**：skill 预加载对模型可见但不影响对话 turn 结构 | `runAgent.ts:577-646` |
| 8 | **零信任隔离**：mutation callbacks（setAppState/setResponseLength）默认 no-op | `forkedAgent.ts:345-462` |
| 9 | **累加而非替换**：子代理 MCP servers 与父累加，cleanup 只清理新建的 | `runAgent.ts:95-218` |
