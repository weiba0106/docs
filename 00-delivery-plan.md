# MindBridge 项目深度解读交付计划

> 本计划仅用于本地文档编写，不修改业务代码，不执行 Git 提交或远端上传。

**目标：** 基于当前工作区源码，产出一套面向 AI Agent / LLM 应用工程师面试的项目深度解读文档。

**写作原则：** 先还原端到端运行事实，再拆解各模块；每项结论必须能回到源码符号、测试或实际运行结果。设计意图、当前实现、测试覆盖和改进建议分开描述。

**图形策略：** 静态结构、时序和状态迁移优先使用 Mermaid；只有 Mermaid 难以清晰表达的密集交互关系才增加本地 HTML 可视化。

## 全局约束

- 事实基线是当前本地工作区，不把 README 或旧架构文档直接当成事实。
- 不读取或复制 `.env` 中的敏感值。
- 已实现、部分实现、未接线、合理推断和建议方案必须有明确标记。
- 第三方框架只解释项目实际依赖的机制，不扩写成通用教程。
- 文档只保存在 `docs/project-deep-dive/`。
- 不提交 Git，不推送远端，不修改业务代码。

## 输出文件

| 文件 | 职责 |
|---|---|
| `README.md` | 总目录、学习路径、状态图例和推荐阅读顺序 |
| `01-project-overview.md` | 项目边界、分层架构、启动生命周期、端到端主链路 |
| `02-context-flow.md` | 单轮上下文从 HTTP 到模型、SSE、持久化和下一轮的完整流转 |
| `03-tool-calling.md` | MCP 注册、队列/MCP 双路径、结果存储、解析失败、重试、死信和治理 |
| `04-memory.md` | 短期、长期归档、工作记忆、Agent 私有记忆、压缩与降级 |
| `05-rag.md` | 入库切片、向量/BM25 召回、融合、重排、邻块扩展、降级和评测 |
| `06-skills.md` | SKILL.md 加载、校验、路由、prompt 注入、渐进式披露判断 |
| `07-multi-agent.md` | 事件驱动协议、黑板、任务/Claim/Artifact/Event、Coordinator 和 Agent 隔离 |
| `08-runtime-comparison.md` | 事件驱动、LangGraph、自研顺序 Runtime 的选择、状态和优缺点 |
| `09-harness.md` | Runtime Harness 与 Engineering Harness 的职责、隔离手段和覆盖缺口 |
| `10-persistence-observability-safety.md` | 数据模型、trace、隐私、风险评估、认证和运维边界 |
| `11-gaps-and-roadmap.md` | 已证实问题、未接线能力、风险分级和改进路线 |
| `12-interview-handbook.md` | 面试讲法、高频追问、源码细节题和速查表 |
| `appendix-source-map.md` | 模块到源码、配置、表、测试和运行证据的索引 |

## 证据等级

| 标记 | 含义 |
|---|---|
| `✅ 已实现` | 主链路有调用，源码和运行/测试证据一致 |
| `🟡 部分实现` | 核心结构存在，但覆盖、持久化或接线不完整 |
| `🔴 未接线` | 类、表或配置存在，但当前主链路没有调用 |
| `🧪 已验证` | 本地测试或 Harness 实际执行通过 |
| `⚠️ 已复现问题` | 本地命令可稳定复现 |
| `💡 改进建议` | 不属于当前实现，单独说明收益和代价 |

## 执行任务

- [x] 盘点源码、现有文档、依赖、配置、测试与 Harness。
- [x] 运行 17 个单元测试，确认全部通过。
- [x] 运行完整 Engineering Harness，记录 4 个套件通过、2 个套件失败。
- [x] 建立完整模块证据矩阵。
- [x] 编写总览和上下文主链路。
- [x] 编写工具、记忆、RAG、Skill、多 Agent、Runtime 和 Harness 专题。
- [x] 编写持久化、安全、可观测性、问题清单与改进路线。
- [x] 编写面试手册并建立交叉链接。
- [x] 校验 Mermaid、文件链接、术语、事实标记和测试数据。

## 验证命令

```powershell
.\.venv\Scripts\python.exe -m unittest discover -s tests -v
.\.venv\Scripts\python.exe -m app.harness.runner
```

当前基线：单元测试 `17/17` 通过；Engineering Harness 的 Risk、Routing、RAG、Tool Queue 四套通过，Skills 与 API 两套因 Windows 路径分隔符断言失败。详细根因放在问题清单中。

最终文档验收（2026-08-02）：本地共生成 `15` 个 Markdown 文件、`29` 张 Mermaid 图、`3887` 行内容；文件存在性、内部 Markdown 链接、代码围栏、占位符和乱码检查均通过。该统计包含本交付计划。
