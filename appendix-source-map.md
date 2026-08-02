# 附录：源码证据地图

本文是“模块 → 源码 → 数据 → 配置 → 测试”的快速索引。详细原理请进入对应专题文档。

## 1. 入口与生命周期

| 主题 | 核心符号 | 作用 |
|---|---|---|
| FastAPI 创建 | `app/main.py::create_app` | 中间件、startup/shutdown、路由和静态资源 |
| 建表 | `app/core/bootstrap.py::create_schema` | 调用 SQLAlchemy `Base.metadata.create_all` |
| 种子数据 | `app/core/bootstrap.py::seed_data` | 默认账号、内置知识文档同步 |
| 配置 | `app/core/config.py::Settings` | 环境变量与 `.env` 映射 |
| DB Session | `app/core/database.py` | Engine、`SessionLocal`、依赖注入 |
| HTTP 路由 | `app/api/routes.py` | 学生聊天、报告、知识库和状态接口 |

## 2. 一轮聊天主链路

| 顺序 | 符号 | 输入 | 输出/副作用 |
|---|---|---|---|
| 1 | `routes.chat_stream` | `ChatRequest`、用户、DB Session | `StreamingResponse` |
| 2 | `ChatService.stream_chat` | 用户消息 | 调用 Runtime Harness |
| 3 | `MindBridgeAgentHarness.run` | 原始消息、可选 sessionId | `AgentHarnessOutcome` |
| 4 | `PrivacySanitizer.sanitize` | 原始文本 | 模型用脱敏文本 |
| 5 | `MindBridgeAgentHarness._resolve_session` | sessionId/首轮文本 | `ChatSession` |
| 6 | `create_agent_runtime` | 配置 | 三种 Runtime 之一 |
| 7 | `runtime.run` | 用户、会话、原文、脱敏文本 | `AgentRunResult` |
| 8 | `save_message` | 用户原文 | MySQL + Redis |
| 9 | `_create_report` | 非 CHAT 的 Agent 结果 | `PsychologicalReport` |
| 10 | `AgentTraceService.save_run` | 运行结果 | `AgentRunTrace` |
| 11 | `AiClient.stream` | `response_messages` prompt | token 流 |
| 12 | `save_assistant_message` | 拼接后的最终文本 | MySQL + Redis |
| 13 | `dispatch_tools` | `AgentToolPlan` | 队列入库或 MCP 调用 |

## 3. Runtime 选择

| 配置/条件 | Runtime | 证据 |
|---|---|---|
| `event_driven_multi_agent`、`multi_agent`、`actors` | `EventDrivenAgentRuntimeService` | `app/agents/factory.py::wants_event_driven` |
| `langgraph` 且 `find_spec("langgraph")` 成功 | `LangGraphAgentRuntimeService` | `langgraph_available` |
| 其他值；或 LangGraph 不可导入 | `AgentRuntimeService` | `create_agent_runtime` 最后分支 |

环境差异：

- `Settings` 代码默认：`event_driven_multi_agent`。
- `docker-compose.yml` 容器默认：`langgraph`。
- 最终以环境变量/`.env` 覆盖后的 `Settings.agent_framework` 为准。

## 4. 事件驱动多 Agent

| 能力 | 文件/符号 |
|---|---|
| 协作协议类型 | `app/agents/events.py` |
| Agent Profile、Capability、Claim 决策 | `app/agents/registry.py` |
| Coordinator 调度和采纳策略 | `app/agents/coordinator.py::EventDrivenCoordinator` |
| Agent 实现 | `app/agents/autonomous.py` |
| Runtime 适配 | `app/agents/event_driven_runtime.py` |
| Agent 独立模型 | `app/services/agent_models.py::AgentModelRegistry` |
| 私有 Redis 记忆 | `app/agents/autonomous.py::AgentPrivateMemory` |

### 主要 Agent

| Agent | Capability | 产物 | 允许工具（当前仅元数据） |
|---|---|---|---|
| UnderstandingAgent | UNDERSTANDING | `intent` artifact | `llm.intent` |
| SafetyAgent | SAFETY | `risk`、`safety_review` 或 `critique` | `llm.risk`、规则、回复审查 |
| ContextAgent | CONTEXT | `context` artifact | Redis、MySQL、RAG、Skills |
| ResponseAgent | RESPONSE | `response_proposal` artifact | `llm.response_plan` |
| CoordinatorAgent | COORDINATION | root task、最终采纳 | taskboard、blackboard |

注意：`tool_permissions` 当前没有运行时 enforcement，仅用于 Profile 描述。

## 5. 记忆与上下文

| 类型 | Key/表/对象 | 写入 | 读取 |
|---|---|---|---|
| 完整消息归档 | `chat_messages` | `MindBridgeAgentHarness.save_message` | Redis miss 时回填、后台会话查看 |
| 短期记忆 | `mindbridge:short-term-memory:{session}` | 同上 | MemoryAgent / ContextAgent |
| Agent 私有记忆 | `mindbridge:short-term-memory:agent:{agent}:{session}` | `BaseAutonomousAgent.remember` | 对应 Agent 自己 |
| 工作记忆 | `AgentContext` | Custom/LangGraph 节点原地更新 | 同一轮后续节点 |
| 协作工作记忆 | `CollaborationBlackboard` | 复制式追加 Task/Event/Artifact/Message | Coordinator 与全部 Agent |
| 本轮摘要 trace | `agent_run_traces.memory_brief` | `AgentTraceService` | 管理 trace 接口（当前方法绑定有问题） |

关键配置：

- `redis_memory_ttl_seconds`
- `redis_memory_max_messages`
- `memory_compaction_enabled`
- `memory_compaction_recent_messages`
- `memory_summary_max_chars`
- `chat_history_limit`

## 6. RAG

| 阶段 | 核心符号 |
|---|---|
| 内置文档同步 | `seed_data -> KnowledgeService.ensure_source` |
| 文件入库 | `KnowledgeService.ingest_file` |
| 切片 | `chunk_text` |
| MySQL 存储 | `KnowledgeChunk` |
| Chroma 包装 | `ChromaKnowledgeStore` |
| Embedding | `ChromaKnowledgeStore._embed` |
| BM25 | `bm25_scores` |
| 候选融合 | `KnowledgeService._fuse_and_rerank` |
| 本地重排 | `rerank_score` |
| 邻块扩展 | `KnowledgeService._expand_best` |
| 降级 | `_handle_vector_error` 和空向量候选 |
| 评测 | `app/rag_eval/runner.py` |

关键配置：

- `knowledge_chunk_size=512`
- `knowledge_chunk_overlap=64`
- `knowledge_top_k=4`
- `knowledge_candidate_k=16`
- `knowledge_hybrid_vector_weight=0.65`
- `knowledge_hybrid_bm25_weight=0.35`
- `knowledge_rerank_enabled=true`
- `knowledge_vector_required=false`

## 7. Skill

| 阶段 | 核心符号 |
|---|---|
| 扫描 | `MindBridgeSkillRegistry.list_skills` |
| Frontmatter 解析 | `_split_frontmatter` |
| 结构校验 | `MindBridgeSkill.validation_issues` |
| 规则选择 | `MindBridgeSkillLibrary.response_skill_names` |
| Prompt 注入 | `response_skill_context` |
| 模板提取 | `template_for` |
| Staff handoff 渲染 | `counselor_handoff_summary` |

Skill 文件位于 `skills/<skill-name>/SKILL.md`，当前共 7 个。

## 8. 工具与后台任务

| 能力 | 核心符号 | 持久化 |
|---|---|---|
| 工具业务实现 | `ToolOrchestrationService` | ExcelRecord、RiskCase、AlertRecord、CaseNote |
| 工具入队 | `ToolQueueService.enqueue_report` | ToolJob |
| 调度与线程池 | `ToolQueueWorker` | ToolJob 状态 |
| 失败重试 | `_fail_or_dead_letter` | ToolJob + DeadLetterRecord |
| 限流 | `RateLimiter` | 进程内 deque，不持久化 |
| MCP 注册 | `app/mcp_tools/server.py` 的 `@mcp.tool()` | 工具业务表 |
| MCP client | `MindBridgeMcpToolClient` | 返回字符串；业务副作用由 server 保存 |
| 工具策略 | `ToolPolicyRegistry` | 无 |
| 工具审计结构 | `ToolGovernanceService` | ToolAuditRecord |

接线事实：`ToolGovernanceService` 被 Queue 模块 import，但没有被执行路径调用。

## 9. 持久化实体

| 表 | 主要用途 | 主要写入方 |
|---|---|---|
| `user_accounts` | 账号与角色 | `seed_data` |
| `chat_sessions` | 会话元数据 | `MindBridgeAgentHarness` |
| `chat_messages` | 用户/助手完整消息 | `MindBridgeAgentHarness.save_message` |
| `knowledge_chunks` | RAG 切片与 embedding JSON 缓存 | `KnowledgeService` |
| `psychological_reports` | 非 CHAT 后台评估报告 | `MindBridgeAgentHarness._create_report` |
| `risk_cases` | 中高风险人工跟进个案 | `ToolOrchestrationService.create_case` |
| `case_notes` | 个案跟进记录 | `add_case_note / acknowledge_case` |
| `alert_records` | log/SMTP 通知结果 | `notify` |
| `excel_records` | Excel 台账写入结果 | `write_excel` |
| `tool_jobs` | 后台任务状态与依赖 | `ToolQueueService/Worker` |
| `dead_letter_records` | 超过最大尝试的失败任务 | `_fail_or_dead_letter` |
| `agent_run_traces` | 单轮 Agent 运行快照 | `AgentTraceService` |
| `tool_audit_records` | 预期工具授权审计 | 当前主链路未写入 |

## 10. 测试和运行证据

| 能力 | 单元测试 | Engineering Harness |
|---|---|---|
| Blackboard 复制式更新 | `test_event_driven_multi_agent.py` | 未直接覆盖 |
| Claim 置信度排序 | 同上 | 未运行默认事件驱动链路 |
| 记忆压缩与脱敏 | `test_memory_compaction.py` | 间接覆盖 |
| 高风险硬规则 | `test_privacy_and_assessment.py` | Risk Safety PASS |
| Skill parser | `test_skills.py` | Skills 因路径断言 FAIL |
| Tool policy 纯函数 | `test_tool_governance.py` | 没有验证治理接线 |
| Custom Runtime 路由 | 无独立单测 | Agent Routing PASS |
| RAG 指标 | 无独立单测 | RAG PASS，60 条 |
| Tool Queue 幂等/依赖/死信 | 无独立单测 | Tool Queue PASS |
| API | 无独立单测 | 因 Skill 路径断言提前 FAIL |

## 11. 已确认但容易误讲的边界

1. `AgentRunTrace` 持久化的是运行快照，不是可恢复的 LangGraph checkpoint。
2. `embedding_json` 缓存在 MySQL，同时向量索引存在 Chroma；这不是两个独立语义源，MySQL 值主要用于避免重复 embedding。
3. `response_messages_json` 保存的是最终文本生成之前的 prompt messages，不是学生最终看到的 assistant 文本。
4. 工具 Queue 是数据库任务表 + 单进程线程池，不是 Celery/Kafka/RabbitMQ。
5. Blackboard 在代码上采用 frozen dataclass 和 tuple 追加，但其中 `tasks`/`payload` 仍包含可变 dict，所以是“约定上的追加式”，不是强不可变事件存储。
6. LangGraph Runtime 没有 checkpoint、持久化 thread state、interrupt 或 human-in-the-loop。
7. Skill 系统没有模型自主发现工具，也没有外部引用按需加载。

