# New Essential Functions - 垂直 Agent 技术架构选型决策

**定位**: 独立开发者构建垂直/通用 Agent 的技术选型与 MVP 方案指南

**视角**: 不是产品功能脑暴，而是**技术架构复用决策**

---

## 分析方法论

### 输入
- 15 个参考项目源码 (`../reference-source-code/`)
- 16 篇现有分析文档 (`../essential-functions/`)

### 处理框架
对每个技术点，按以下结构输出：

1. **已有实现** — openclaw/copaw/opencode/kimi-cli 等项目的实现方式
2. **值得直接复用** — 哪些代码/设计可以直接拿来用
3. **需要垂直特化** — 哪些需要针对特定场景改造
4. **推荐数据结构/接口/模块边界** — 具体设计建议
5. **MVP 最小实现方案** — 第一版怎么做
6. **后续扩展路线** — 迭代路径

### 输出
- 可执行的架构决策
- 可复用的代码结构建议
- 针对垂直场景的特化方案

---

## 14 个技术分析文档

| 编号 | 技术点 | 状态 | 优先级 |
|------|--------|------|--------|
| [01](01-agent-runtime-decision.md) | Agent Runtime 主循环 | ✅ 已完成 | P0 |
| [02](02-context-assembly-decision.md) | Context Assembly Pipeline | ✅ 已完成 | P0 |
| [03](03-context-compression-decision.md) | 上下文压缩/摘要/保留区策略 | ✅ 已完成 | P0 |
| [04](04-tool-registry-decision.md) | Tool Registry 与结果后处理 | ✅ 已完成 | P0 |
| [05](05-skills-system-decision.md) | Skills 体系设计 | ✅ 已完成 | P1 |
| [06](06-model-provider-decision.md) | Model Provider 抽象与路由 | ✅ 已完成 | P0 |
| [07](07-memory-boundary-decision.md) | Session/Workflow/Memory 边界 | ✅ 已完成 | P0 |
| [08](08-rag-orchestration-decision.md) | RAG Orchestration 与引用机制 | ✅ 已完成 | P1 |
| [09](09-document-ingestion-decision.md) | Document Ingestion Pipeline | ✅ 已完成 | P1 |
| [10](10-channel-adapter-decision.md) | Channel Adapter (飞书/钉钉/QQ/Web) | ✅ 已完成 | P1 |
| [11](11-workflow-engine-decision.md) | Workflow/Task Engine | ✅ 已完成 | P1 |
| [12](12-observability-decision.md) | Observability/Tracing/Replay | ✅ 已完成 | P2 |
| [13](13-plugin-arch-decision.md) | Plugin Architecture | ✅ 已完成 | P1 |
| [14](14-storage-schema-decision.md) | Storage Schema 与核心对象模型 | ✅ 已完成 | P0 |

---

## 参考项目索引

### 通用个人智能体 (8 个)

| 项目 | 语言 | 特点 | 参考位置 |
|------|------|------|----------|
| **openclaw** | TypeScript | 多通道网关，20+ 通道 | `../reference-source-code/personal-general-ai/openclaw/` |
| **nanobot** | Python | 事件驱动，轻量 | `../reference-source-code/personal-general-ai/nanobot/` |
| **nanoclaw** | Python | 最小化设计 | `../reference-source-code/personal-general-ai/nanoclaw/` |
| **zeroclaw** | Rust | 零依赖，类型安全 | `../reference-source-code/personal-general-ai/zeroclaw/` |
| **ironclaw** | Rust | 企业级，LoopDelegate 模式 | `../reference-source-code/personal-general-ai/ironclaw/` |
| **mimiclaw** | C | 嵌入式 ESP32 | `../reference-source-code/personal-general-ai/mimiclaw/` |
| **picoclaw** | Go | 超轻量，worker pool | `../reference-source-code/personal-general-ai/picoclaw/` |
| **CoPaw** | Python | AgentScope 框架 | `../reference-source-code/personal-general-ai/CoPaw/` |

### 编码智能体 (7 个)

| 项目 | 语言 | 特点 | 参考位置 |
|------|------|------|----------|
| **kimi-cli** | TypeScript | Kimi 模型 CLI | `../reference-source-code/coding-agents/kimi-cli/` |
| **pi-mono** | Python | 图分析 | `../reference-source-code/coding-agents/pi-mono/` |
| **opencode** | TypeScript | 代码智能 | `../reference-source-code/coding-agents/opencode/` |
| **gemini-cli** | TypeScript | Gemini CLI | `../reference-source-code/coding-agents/gemini-cli/` |
| **qwen-code** | TypeScript | 通义千问 CLI | `../reference-source-code/coding-agents/qwen-code/` |
| **codex** | Rust | OpenAI Codex CLI | `../reference-source-code/coding-agents/codex/` |
| **aider** | Python | 结对编程 | `../reference-source-code/coding-agents/aider/` |

---

## 使用指南

### 按开发阶段

| 阶段 | 推荐阅读 |
|------|----------|
| **技术选型** | 01 → 06 → 07 → 14 |
| **MVP 开发** | 01 → 04 → 06 → 02 → 03 |
| **生产部署** | 10 → 11 → 12 → 13 |
| **高级功能** | 05 → 08 → 09 |

### 按技术栈偏好

| 偏好 | 重点参考项目 |
|------|-------------|
| **TypeScript** | openclaw, kimi-cli, gemini-cli, qwen-code |
| **Rust** | ironclaw, zeroclaw, codex |
| **Python** | CoPaw, nanobot, aider, pi-mono |
| **Go** | picoclaw |

---

## 更新日志

| 日期 | 更新内容 |
|------|----------|
| 2026-03-22 | 14 篇技术决策文档全部完成 ✅ |
| 2026-03-21 | 创建文档框架，建立 15 项目源码库 |

---

*最后更新：2026-03-22*
*总计：14 篇技术分析文档（全部完成）*
*总字数：~4,800 行*
