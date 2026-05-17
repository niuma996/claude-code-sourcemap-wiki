# 工具系统

## 概述

工具系统是 Claude Code 的执行单元，允许 AI 代理通过工具与外部世界交互。系统支持多种工具类型，包括 Bash 执行、文件编辑、搜索、MCP 集成等。

工具系统采用统一的 TypeScript `Tool` interface 设计。核心实现在 [Tool.ts](/restored-src/src/Tool.ts) 文件中。

> **注意**：`Tool` 是 TypeScript `interface` 而非 class 基类。实际方法为 `call()`（执行）和 `description()`（生成描述），而非 `invoke()`/`getMetadata()`。

**核心源码位置**
- [Tool.ts](/restored-src/src/Tool.ts#L1) - 工具基类接口
- [tools.ts](/restored-src/src/tools.ts#L1) - 工具注册
- [tools/BashTool/](/restored-src/src/tools/BashTool/) - Bash 工具实现
- [tools/FileEditTool/](/restored-src/src/tools/FileEditTool/) - 文件编辑工具

## 架构位置

```mermaid
flowchart TB
    subgraph TL ["工具系统"]
        Tool[Tool 基类]
        Bash[BashTool]
        FileEdit[FileEditTool]
        Grep[GrepTool]
        MCP[MCPTool]
        LSP[LSPTool]
        Agent[AgentTool]
    end
    Tool --> Bash
    Tool --> FileEdit
    Tool --> Grep
    Tool --> MCP
    Tool --> LSP
    Tool --> Agent
```

## 功能特性

| 功能 | 说明 | 相关文件 |
|------|------|----------|
| Bash 执行 | 在终端执行命令 | [BashTool/](/restored-src/src/tools/BashTool/) |
| 文件编辑 | 读取、写入、编辑文件 | [FileEditTool/](/restored-src/src/tools/FileEditTool/) |
| 文件搜索 | 模式匹配搜索文件 | [GlobTool/](/restored-src/src/tools/GlobTool/) |
| 代码搜索 | Grep 模式搜索 | [GrepTool/](/restored-src/src/tools/GrepTool/) |
| MCP 工具 | MCP 服务器提供的工具 | [MCPTool/](/restored-src/src/tools/MCPTool/) |
| LSP 工具 | Language Server Protocol | [LSPTool/](/restored-src/src/tools/LSPTool/) |
| Agent 工具 | 子代理执行 | [AgentTool/](/restored-src/src/tools/AgentTool/) |

## 文件结构

```
restored-src/src/
├── Tool.ts                 # 工具基类接口
├── tools.ts                # 工具注册表
├── tools/                  # 工具实现目录
│   ├── BashTool/          # Bash 执行工具
│   │   ├── BashTool.ts    # Bash 工具主实现
│   │   ├── UI.tsx        # Bash 工具 UI
│   │   ├── prompt.ts     # 提示词模板
│   │   └── utils.ts      # 工具函数
│   ├── FileEditTool/     # 文件编辑工具
│   │   ├── FileEditTool.ts
│   │   ├── UI.tsx
│   │   └── prompt.ts
│   ├── FileReadTool/     # 文件读取工具
│   ├── GlobTool/         # 文件搜索工具
│   │   ├── GlobTool.ts
│   │   ├── UI.tsx
│   │   └── prompt.ts
│   ├── GrepTool/         # 代码搜索工具
│   │   ├── GrepTool.ts
│   │   ├── UI.tsx
│   │   └── prompt.ts
│   ├── MCPTool/          # MCP 工具
│   │   ├── MCPTool.ts
│   │   ├── UI.tsx
│   │   └── prompt.ts
│   ├── LSPTool/          # LSP 工具
│   │   ├── LSPTool.ts
│   │   ├── UI.tsx
│   │   ├── prompt.ts
│   │   └── schemas.ts
│   ├── AgentTool/        # Agent 工具
│   │   ├── AgentTool.ts
│   │   ├── UI.tsx
│   │   └── prompt.ts
│   ├── WebFetchTool/     # Web 获取工具
│   ├── WebSearchTool/    # Web 搜索工具
│   ├── TaskStopTool/     # 任务停止工具
│   ├── SleepTool/        # 睡眠工具
│   ├── ConfigTool/       # 配置工具
│   ├── SkillTool/        # 技能工具
│   └── BriefTool/        # 简要工具
```

**源码映射**
- [Tool.ts](/restored-src/src/Tool.ts) - 工具基类接口
- [tools.ts](/restored-src/src/tools.ts) - 工具注册逻辑
- [tools/BashTool/BashTool.ts](/restored-src/src/tools/BashTool/) - Bash 工具实现
- [tools/GlobTool/GlobTool.ts](/restored-src/src/tools/GlobTool/) - Glob 工具实现
- [tools/GrepTool/GrepTool.ts](/restored-src/src/tools/GrepTool/) - Grep 工具实现

## 核心工作流

```mermaid
sequenceDiagram
    participant Query as QueryEngine
    participant Tools as 工具系统
    participant Tool as 具体工具
    participant External as 外部系统

    Query->>Tools: 请求工具调用
    Tools->>Tool: 创建工具实例
    Tool->>Tool: 验证参数
    Tool->>External: 执行操作
    External-->>Tool: 返回结果
    Tool-->>Tools: 返回输出
    Tools-->>Query: 返回工具结果
```

## 工具接口

```mermaid
classDiagram
    class Tool {
        <<interface>>
        +call(args, context, canUseTool, parentMessage, onProgress?): Promise~ToolResult~
        +description(input, options): Promise~string~
        +inputSchema: z.ZodType
    }
    class BashTool {
        +name: "Bash"
        +call(args, context, canUseTool, ...): Promise~ToolResult~
    }
    class FileEditTool {
        +name: "Edit"
        +call(args, context, canUseTool, ...): Promise~ToolResult~
    }
    Tool <|.. BashTool
    Tool <|.. FileEditTool
```

> **方法签名说明**：`call(args, context, canUseTool, parentMessage, onProgress?)` 的参数包含 `canUseTool`（权限检查）和 `onProgress`（进度回调），不是简单的 `invoke(input)`。

## API 摘要

| 工具 | 说明 | 主要方法 | 源码位置 |
|------|------|----------|----------|
| BashTool | 执行 shell 命令 | `call(input, context, ...)` | [BashTool.ts](/restored-src/src/tools/BashTool/BashTool.ts) |
| FileEditTool | 编辑文件内容 | `call(input, context, ...)` | [FileEditTool.ts](/restored-src/src/tools/FileEditTool/FileEditTool.ts) |
| FileReadTool | 读取文件内容 | `call(input, context, ...)` | [FileReadTool.ts](/restored-src/src/tools/FileReadTool/) |
| GlobTool | 模式匹配文件 | `call(input, context, ...)` | [GlobTool.ts](/restored-src/src/tools/GlobTool/GlobTool.ts) |
| GrepTool | 代码搜索 | `call(input, context, ...)` | [GrepTool.ts](/restored-src/src/tools/GrepTool/GrepTool.ts) |
| MCPTool | MCP 工具调用 | `call(input, context, ...)` | [MCPTool.ts](/restored-src/src/tools/MCPTool/MCPTool.ts) |
| LSPTool | LSP 操作 | `call(input, context, ...)` | [LSPTool.ts](/restored-src/src/tools/LSPTool/LSPTool.ts) |
| AgentTool | 子代理执行 | `call(input, context, ...)` | [AgentTool.ts](/restored-src/src/tools/AgentTool/AgentTool.ts) |
| WebFetchTool | 网页获取 | `call(input, context, ...)` | [WebFetchTool.ts](/restored-src/src/tools/WebFetchTool/) |
| WebSearchTool | 网络搜索 | `call(input, context, ...)` | [WebSearchTool.ts](/restored-src/src/tools/WebSearchTool/) |

## BashTool 详细说明

BashTool 是最常用的工具，允许执行 shell 命令。以下为示意性输入/输出结构（非源码中的实际类型名，具体实现见 [BashTool.ts](/restored-src/src/tools/BashTool/BashTool.ts)）：

```typescript
// 示意性结构（非源码实际类型名）
// 输入：command 为必填，timeout 可选（毫秒）
{
  command: string
  timeout?: number
}

// 输出：stdout/stderr 文本 + 退出码
{
  stdout: string
  stderr: string
  exitCode: number
}
```

## 文件编辑工具

文件编辑工具支持以下操作（操作名称以源码实际参数为准，见 [FileEditTool.ts](/restored-src/src/tools/FileEditTool/FileEditTool.ts)）：

| 操作 | 说明 | 参数 |
|------|------|------|
| `create` | 创建新文件 | `file_path`, `content` |
| `str_replace` | 替换文件中的字符串 | `file_path`, `old_string`, `new_string` |

## MCP 工具集成

MCP 工具通过 [MCPTool](/restored-src/src/tools/MCPTool/) 集成：

```typescript
interface MCPInput {
  server: string           // MCP 服务器名称
  tool: string             // 工具名称
  arguments: Record<string, unknown>  // 工具参数
}
```

## 最佳实践

### 工具调用原则

| 原则 | 说明 |
|------|------|
| 参数验证 | 在 `call()` 前验证所有参数 |
| 错误处理 | 捕获并适当处理执行错误 |
| 超时控制 | 为长时间操作设置超时 |
| 结果缓存 | 缓存可重用的结果 |

### 避免的问题

| 问题 | 解决方案 |
|------|----------|
| 命令注入 | 使用参数化命令，避免字符串拼接 |
| 路径遍历 | 规范化并验证文件路径 |
| 超时失控 | 设置合理的默认超时并允许配置 |
| 资源泄露 | 确保子进程和文件句柄正确关闭 |

## 源码引用

**核心文件**
- [Tool.ts](/restored-src/src/Tool.ts) - 工具基类接口
- [tools.ts](/restored-src/src/tools.ts) - 工具注册表

**工具实现**
- [tools/BashTool/BashTool.ts](/restored-src/src/tools/BashTool/) - Bash 工具
- [tools/FileEditTool/](/restored-src/src/tools/FileEditTool/) - 文件编辑工具
- [tools/GrepTool/](/restored-src/src/tools/GrepTool/) - 搜索工具
- [tools/GlobTool/](/restored-src/src/tools/GlobTool/) - 文件匹配工具
- [tools/MCPTool/](/restored-src/src/tools/MCPTool/) - MCP 工具
- [tools/AgentTool/](/restored-src/src/tools/AgentTool/) - 代理工具

**工具 UI 和提示**
- [BashTool/UI.tsx](/restored-src/src/tools/BashTool/UI.tsx) - Bash UI
- [GlobTool/prompt.ts](/restored-src/src/tools/GlobTool/prompt.ts) - Glob 提示

## 相关文档

- [查询引擎](query.md)
- [MCP 服务](../../services/mcp.md)
- [命令系统](commands.md)
- [核心模块索引](_index.md)

---

*Generated by [Nium-Wiki v0.0.0](https://github.com/niuma996/nium-wiki) | 2026-05-15*
