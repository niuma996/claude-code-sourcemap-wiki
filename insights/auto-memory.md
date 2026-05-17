# Auto Memory 系统分析

> 基于 `@anthropic-ai/claude-code@2.1.88` 源码还原

## 概述

Auto Memory 是 Claude Code 的持久化记忆系统，将跨会话的上下文信息以 Markdown 文件形式存储在本地磁盘。每次对话结束后，后台子 Agent 自动提取值得保留的信息并写入记忆目录。

---

## 目录结构

```
~/.claude/
└── projects/
    └── <sanitized-git-root>/
        └── memory/
            ├── MEMORY.md          ← 索引文件，始终注入 system prompt
            ├── user_role.md       ← 各类记忆 topic 文件
            ├── feedback_testing.md
            ├── project_deadline.md
            ├── team/              ← Team Memory（TEAMMEM feature gate）
            │   └── MEMORY.md
            └── logs/              ← KAIROS 模式（assistant 长会话）
                └── YYYY/MM/YYYY-MM-DD.md
```

路径解析优先级（[paths.ts](../restored-src/src/memdir/paths.ts)）：
1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` 环境变量（Cowork 专用）
2. `settings.json` 中的 `autoMemoryDirectory`（仅 policy/local/user 级别，**不含** projectSettings 防止恶意 repo 写入 `~/.ssh`）
3. 默认：`~/.claude/projects/<sanitized-git-root>/memory/`

---

## 记忆类型

四种类型，通过 frontmatter `type:` 字段区分（[memoryTypes.ts](../restored-src/src/memdir/memoryTypes.ts)）：

| 类型 | 用途 | 何时保存 |
|------|------|----------|
| `user` | 用户角色、偏好、知识背景 | 了解到用户身份/技能时 |
| `feedback` | 用户对 AI 行为的纠正或确认 | 用户说"不要这样"或"就是这样" |
| `project` | 项目进展、目标、截止日期、事故 | 了解到谁在做什么、为什么、何时完成 |
| `reference` | 外部系统指针（Linear、Slack、Grafana） | 了解到外部资源位置时 |

**不应保存**：代码模式、架构、文件路径、git 历史、调试方案、CLAUDE.md 已有内容、临时任务状态。

### 记忆文件格式

```markdown
---
name: short-kebab-case-slug
description: 一行描述（用于相关性筛选，要具体）
type: user | feedback | project | reference
---

记忆内容正文
feedback/project 类型建议结构：规则/事实 + **Why:** + **How to apply:**
```

### MEMORY.md 索引格式

```markdown
- [Title](file.md) — 一行钩子描述（~150 字符以内）
```

- 无 frontmatter
- 超过 200 行或 25,000 字节时被截断并附加警告
- 始终注入 system prompt

---

## 核心模块

### 1. `memdir/memdir.ts` — Prompt 构建

- `buildMemoryLines()` — 构建记忆行为指令（不含 MEMORY.md 内容）
- `buildMemoryPrompt()` — 构建含 MEMORY.md 内容的完整 prompt（供 agent memory 使用）
- `loadMemoryPrompt()` — 加载 system prompt 中的记忆段落，根据功能开关分发：
  - KAIROS 模式 → `buildAssistantDailyLogPrompt()`（追加日志，不维护 MEMORY.md）
  - TEAMMEM 启用 → `buildCombinedMemoryPrompt()`（私有 + 团队双目录）
  - 默认 → `buildMemoryLines()`（单目录）
- `truncateEntrypointContent()` — 截断 MEMORY.md，先按行（200行），再按字节（25KB）
- `ensureMemoryDirExists()` — 幂等创建记忆目录，每次 session 调用一次

### 2. `memdir/paths.ts` — 路径解析

- `isAutoMemoryEnabled()` — 检查是否启用（env var > --bare > CCR > settings > 默认开启）
- `getAutoMemPath()` — 获取记忆目录路径（memoized，以 projectRoot 为 key）
- `isAutoMemPath(path)` — 判断路径是否在记忆目录内（用于权限控制）
- `getAutoMemDailyLogPath(date)` — KAIROS 模式的日志文件路径

### 3. `memdir/memoryScan.ts` — 文件扫描

- `scanMemoryFiles(memoryDir, signal)` — 递归扫描 `.md` 文件，读取 frontmatter，按 mtime 降序排列，最多返回 200 个
- `formatMemoryManifest(memories)` — 格式化为文本清单（供 extraction agent 预注入，避免浪费一轮 `ls`）

### 4. `memdir/findRelevantMemories.ts` — 相关性召回

- `findRelevantMemories(query, memoryDir, signal, recentTools, alreadySurfaced)` — 扫描记忆文件头，调用 Sonnet 模型选出最相关的最多 5 个文件
- 过滤已展示过的文件（`alreadySurfaced`）
- 对正在使用的工具，跳过其参考文档（避免噪音），但保留警告/已知问题类记忆

### 5. `services/extractMemories/extractMemories.ts` — 后台提取

核心机制：每次 query loop 结束（模型产出最终响应，无工具调用）后，在 `handleStopHooks` 中 fire-and-forget 触发。

**关键设计**：
- **forked agent 模式**：完美 fork 主对话，共享 prompt cache，不写入 transcript
- **互斥逻辑**：若主 agent 已写入记忆文件（`hasMemoryWritesSince`），跳过后台提取，推进游标
- **节流**：`tengu_bramble_lintel` feature flag 控制每 N 轮触发一次（默认每轮）
- **串行化**：同时只有一个提取在运行；新调用到来时 stash 上下文，当前完成后执行 trailing run
- **工具限制**（`createAutoMemCanUseTool`）：
  - 允许：FileRead、Grep、Glob（无限制）
  - 允许：只读 Bash（ls/find/grep/cat/stat/wc/head/tail）
  - 允许：FileEdit/FileWrite，但**仅限记忆目录内**
  - 拒绝：其他所有工具（MCP、Agent、写入 Bash 等）
- **最大轮数**：5 轮（防止验证兔子洞）
- **游标**：`lastMemoryMessageUuid` 追踪上次处理位置，每次只处理新消息

**提取流程**：
```
handleStopHooks
  └─ executeExtractMemories (fire-and-forget)
       └─ runExtraction
            ├─ hasMemoryWritesSince? → 跳过，推进游标
            ├─ scanMemoryFiles → 预注入文件清单
            ├─ buildExtractAutoOnlyPrompt / buildExtractCombinedPrompt
            └─ runForkedAgent (maxTurns=5)
                 ├─ Turn 1: 并行 FileRead 所有可能更新的文件
                 └─ Turn 2: 并行 FileWrite/FileEdit
```

### 6. `services/autoDream/autoDream.ts` — 记忆整合（KAIROS 模式）

将日志文件蒸馏为 topic 文件 + MEMORY.md 索引。触发条件（三个门全部通过）：
1. 距上次整合 >= minHours（默认 24h）
2. 触碰的 session 数 >= minSessions（默认 5）
3. 无其他进程正在整合（锁）

---

## 调用链

```
main.tsx
  └─ startBackgroundHousekeeping()          [backgroundHousekeeping.ts]
       ├─ initExtractMemories()             ← 初始化闭包状态
       └─ initAutoDream()

query loop
  └─ handleStopHooks()                      [stopHooks.ts]
       ├─ executeExtractMemories()          ← fire-and-forget
       └─ executeAutoDream()               ← fire-and-forget

session 结束
  └─ drainPendingExtraction()              [print.ts]
       └─ 等待所有 in-flight 提取完成（超时 60s）
```

---

## 功能开关（Feature Gates）

| Flag | 用途 |
|------|------|
| `EXTRACT_MEMORIES` | 后台提取 Agent |
| `KAIROS` | 日志追加模式（assistant 长会话） |
| `TEAMMEM` | 团队记忆支持 |
| `tengu_passport_quail` | 提取功能启用 |
| `tengu_herring_clock` | 团队记忆启用 |
| `tengu_bramble_lintel` | 提取节流（默认 1 = 每轮） |
| `tengu_moth_copse` | 跳过 MEMORY.md 索引（仅用日志） |
| `tengu_coral_fern` | 启用"搜索历史上下文"章节 |
| `tengu_onyx_plover` | autoDream 配置（minHours, minSessions） |
| `MEMORY_SHAPE_TELEMETRY` | 记忆召回形状遥测 |

---

## 环境变量

| 变量 | 作用 |
|------|------|
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | 完全禁用 auto memory |
| `CLAUDE_CODE_SIMPLE` / `--bare` | 禁用 auto memory |
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | 覆盖记忆基础目录 |
| `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` | 完整路径覆盖（Cowork 专用） |
| `CLAUDE_COWORK_MEMORY_EXTRA_GUIDELINES` | 注入额外记忆指导文本 |

---

## 安全设计

- **路径遍历防护**：`validateMemoryPath()` 拒绝相对路径、根路径、UNC 路径、null 字节
- **projectSettings 排除**：`autoMemoryDirectory` 不读取 `.claude/settings.json`（防止恶意 repo 重定向写入 `~/.ssh`）
- **写入权限隔离**：`isAutoMemPath()` 确保 FileEdit/FileWrite 只能写记忆目录
- **Team Memory 符号链接防护**：`teamMemPaths.ts` 验证路径不逃逸（PSR M22186）

---

## 关键文件索引

| 文件 | 职责 |
|------|------|
| [memdir/memdir.ts](../restored-src/src/memdir/memdir.ts) | Prompt 构建、截断、目录管理 |
| [memdir/paths.ts](../restored-src/src/memdir/paths.ts) | 路径解析、启用检查 |
| [memdir/memoryTypes.ts](../restored-src/src/memdir/memoryTypes.ts) | 类型定义、prompt 文本常量 |
| [memdir/memoryScan.ts](../restored-src/src/memdir/memoryScan.ts) | 文件扫描、清单格式化 |
| [memdir/findRelevantMemories.ts](../restored-src/src/memdir/findRelevantMemories.ts) | Sonnet 相关性召回 |
| [memdir/teamMemPaths.ts](../restored-src/src/memdir/teamMemPaths.ts) | 团队记忆路径、符号链接验证 |
| [services/extractMemories/extractMemories.ts](../restored-src/src/services/extractMemories/extractMemories.ts) | 后台提取 Agent 主逻辑 |
| [services/extractMemories/prompts.ts](../restored-src/src/services/extractMemories/prompts.ts) | 提取 Agent prompt 模板 |
| [services/autoDream/autoDream.ts](../restored-src/src/services/autoDream/autoDream.ts) | 记忆整合（KAIROS） |
| [query/stopHooks.ts](../restored-src/src/query/stopHooks.ts) | 触发提取的调用点 |
| [utils/backgroundHousekeeping.ts](../restored-src/src/utils/backgroundHousekeeping.ts) | 初始化入口 |
