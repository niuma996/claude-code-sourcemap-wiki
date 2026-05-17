# claude-code-sourcemap-wiki

> 本项目仅供学习参考

**[English](./README_en.md)** | **中文**

***

基于 [claude-code-sourcemap](https://github.com/ChinaSiro/claude-code-sourcemap) 自动生成的 Wiki 文档

借助 [nium-wiki](https://github.com/niuma996/nium-wiki) (skill)&#x20;

## 本地预览

```bash
npx nium-wiki serve
```

启动后访问 [localhost:4000](http://localhost:4000) 查看

## 目录结构

- `.nium-wiki/wiki/` — 中文 Wiki 文档
  - `core/` — 核心系统（QueryEngine、Task、Tool、命令、Hooks、技能）
  - `agent/` — Agent 工具与内置 Agent 详解
  - `cli/` — CLI 入口与快速路径
  - `coordinator/` — 协调器与多 Agent 调度
  - `buddy/` — Buddy 伴侣精灵 UI 与通知系统
  - `plugins/` — 插件系统与内置技能
  - `services/` — API、MCP、MagicDocs 等服务层
  - `remote/` — 桥接模式与远程控制
  - `ui/` — React/Ink 用户界面组件
- `.nium-wiki/wiki_en/` — 英文 Wiki 文档（结构同上）
- `insights/` — 代码走读的启发点杂记

## 效果预览

### 本地服务

![local_server](./assets/local_server.png)

### 依赖图（全局）

![graph_1](./assets/graph_1.png)

### 依赖图（局部）

![graph_2](./assets/graph_2.png)

## 相关链接

- [claude-code-sourcemap](https://github.com/ChinaSiro/claude-code-sourcemap)
- [nium-wiki](https://github.com/niuma996/nium-wiki)

