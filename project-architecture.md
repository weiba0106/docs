# MindBridge Python 项目架构梳理

本文档用于帮助快速理解 `mindbridge-py` 项目的整体结构、启动过程、核心业务链路和关键代码位置。

当前项目是一个基于 FastAPI 的校园心理陪伴与风险预警系统。它提供学生端聊天、管理端报告查看、知识库维护、短期记忆、心理风险评估、RAG 检索、多 Agent 编排，以及高风险后续工具任务处理。

## 1. 一句话总览

MindBridge 的主链路是：

学生通过前端发起 SSE 聊天请求，后端创建或复用会话，读取 Redis/MySQL 记忆，经 Agent 运行时判断意图和风险，必要时检索知识库并生成心理支持回复，再把完整聊天写入 MySQL、短期上下文写入 Redis，并为咨询或风险场景生成心理报告和后台工具任务。

```mermaid
flowchart LR
    A["学生前端 student.html"] --> B["POST /api/chat/stream"]
    B --> C["ChatService"]
    C --> D["MindBridgeAgentHarness"]
    D --> E["Agent Runtime"]
    E --> F["AiClient 流式生成"]
    F --> G["SSE token 返回前端"]
    D --> H["MySQL: 会话/消息/报告/trace"]
    D --> I["Redis: 短期记忆"]
    D --> J["ToolQueue: Excel/Case/Alert"]
    E --> K["KnowledgeService: RAG 检索"]
```

## 2. 顶层目录

```text
app/
  api/             FastAPI 路由
  agents/          Agent 编排、黑板协议、运行时
  core/            配置、数据库、安全、启动初始化
  knowledge/       内置心理知识库 Markdown
  mcp_tools/       MCP 工具服务端
  models/          SQLAlchemy 数据表实体
  rag_eval/        RAG 评测数据和 runner
  schemas/         Pydantic DTO
  services/        业务服务层
  static/          原生 HTML/CSS/JS 前端

skills/            本项目自定义心理支持 skill
models/            Ollama 微调模型描述文件
scripts/           本地启动、模型创建、打包脚本
tests/             单元测试
docs/              项目说明文档
```

核心入口文件：

- `app/main.py`：FastAPI 应用创建、启动初始化、静态资源挂载。
- `app/api/routes.py`：所有 HTTP API。
- `app/agents/harness.py`：单轮对话的业务编排门面。
- `app/agents/factory.py`：选择具体 Agent 运行时。
- `app/services/chat.py`：SSE 聊天流入口。
- `app/core/config.py`：读取 `.env` 配置。

## 3. 当前本地配置

运行时配置来自 `pydantic-settings`，配置类在 `app/core/config.py`，并通过：

```python
SettingsConfigDict(env_file=".env", env_file_encoding="utf-8", extra="ignore")
```

读取项目根目录 `.env`。

当前 `.env` 已配置为：

```env
DATABASE_URL=mysql+pymysql://root:123456@127.0.0.1:3306/dy_live_auction?charset=utf8mb4
REDIS_URL=redis://:123456@124.222.245.166:6379/0
REDIS_MEMORY_TTL_SECONDS=86400
REDIS_MEMORY_MAX_MESSAGES=40
REDIS_SOCKET_TIMEOUT_SECONDS=2
```

含义：

- MySQL 在本机：`127.0.0.1:3306`
- 数据库：`dy_live_auction`
- MySQL 用户：`root`
- MySQL 密码：`123456`
- Redis 在远程服务器：`124.222.245.166:6379`
- Redis 密码：`123456`
- Redis 只保存每个会话最近 40 条短期上下文，TTL 为 86400 秒。

注意：`.env` 在 `.gitignore` 中，一般不会提交。这里包含明文数据库密码和 Redis 密码，不建议上传到代码仓库。

## 4. 本机启动方式

你说不用 Docker，所以推荐使用本机虚拟环境启动。

```powershell
cd E:\agent\mindbridge-py

py -m venv .venv
.\.venv\Scripts\Activate.ps1

python -m pip install -U pip
python -m pip install -r requirements.txt
```

确保本机 MySQL 有数据库：

```powershell
mysql -uroot -p123456 -e "CREATE DATABASE IF NOT EXISTS dy_live_auction DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

启动后端：

```powershell
python -m uvicorn app.main:app --host 127.0.0.1 --port 8080
```

访问：

```text
http://127.0.0.1:8080
```

默认账号在启动时自动种子创建：

```text
学生：student / student123
管理员：admin / admin123
```

种子数据逻辑在 `app/core/bootstrap.py`，只有当 `user_accounts` 表为空时才创建默认账号。

## 5. 启动生命周期

`app/main.py` 的启动逻辑如下：

1. 创建 FastAPI 应用。
2. 注册 no-cache 中间件，避免前端 HTML/JS/CSS 缓存导致调试混乱。
3. startup 时执行 `create_schema()`。
4. `create_schema()` 调用 `Base.metadata.create_all(bind=engine)` 自动建表。
5. 创建数据库 session。
6. `seed_data(db)` 初始化默认用户和内置知识库。
7. 启动 `ToolQueueWorker` 后台线程。
8. 挂载 `app/static` 到 `/`，所以根路径就是前端页面。

对应代码：

- `app/main.py`
- `app/core/bootstrap.py`
- `app/core/database.py`

重要细节：

- 当前没有 Alembic 迁移。`create_all` 只会创建不存在的表，不会自动修改已有表结构。
- 如果实体字段变化，而数据库旧表已存在，需要手动迁移或重建表。
- 工具队列是进程内后台线程，不是独立 worker 服务。

## 6. Web 层和接口

所有 API 在 `app/api/routes.py`。

### 公共接口

```text
GET /actuator/health
GET /api/profile
GET /api/agent/status
```

`/api/profile` 根据 Basic Auth 返回当前用户信息和角色。

### 学生端接口

```text
POST /api/chat/stream
GET  /api/reports/me
```

`/api/chat/stream` 是核心聊天接口，返回 `text/event-stream`。

前端读取事件类型：

- `meta`：返回 sessionId。
- `token`：返回模型流式 token。
- `done`：本轮结束。
- `error`：前端有处理逻辑，但后端当前主要是异常抛出，不是所有异常都会包装成 SSE error。

管理员账号不能发起学生聊天，路由里有明确限制。

### 管理端接口

```text
GET  /api/admin/reports
GET  /api/admin/excel-records
GET  /api/admin/alerts
GET  /api/admin/cases
GET  /api/admin/cases/{case_id}/notes
GET  /api/admin/tool-jobs
GET  /api/admin/dead-letters
GET  /api/admin/agent-traces
GET  /api/admin/tool-audits
GET  /api/admin/conversations/{session_id}
POST /api/admin/knowledge
GET  /api/admin/knowledge/status
POST /api/admin/knowledge/rebuild-vector
POST /api/admin/knowledge/backup
POST /api/admin/knowledge/file
```

管理端前端在：

- `app/static/admin.html`
- `app/static/admin.js`

学生端前端在：

- `app/static/student.html`
- `app/static/student.js`

登录页在：

- `app/static/index.html`
- `app/static/app.js`

前端认证方式很简单：登录页把 `username:password` 做 Base64 后存在 `sessionStorage`，后续请求带：

```http
Authorization: Basic base64(username:password)
```

后端认证逻辑在 `app/core/security.py`。

## 7. 数据库模型

实体定义在 `app/models/entities.py`。下面按业务分组理解。

### 用户和会话

```text
user_accounts
chat_sessions
chat_messages
```

- `UserAccount`：用户表，包含用户名、展示名、密码哈希、角色。
- `ChatSession`：一次连续会话，有 `public_id` 给前端传递。
- `ChatMessage`：完整聊天记录，用户消息和助手消息都会写入。

密码哈希是 SHA-256，代码在 `app/core/security.py`。这适合演示项目，不适合真实生产系统，生产建议换 bcrypt/argon2。

### 知识库

```text
knowledge_chunks
```

`KnowledgeChunk` 保存知识库切片：

- `source`：来源文件或上传文件名。
- `source_index`：该来源内第几个 chunk。
- `content`：切片文本。
- `embedding_json`：可选的向量缓存。

启动时会把 `app/knowledge/*.md` 写入此表。

### 心理报告和风险个案

```text
psychological_reports
risk_cases
case_notes
```

- `PsychologicalReport`：咨询或风险对话生成的后台报告。
- `RiskCase`：中高风险时创建的管理端个案。
- `CaseNote`：个案跟进备注。

普通 `CHAT` 不生成心理报告，只有 `CONSULT` 或 `RISK` 会生成。

### 工具结果和队列

```text
excel_records
alert_records
tool_jobs
dead_letter_records
tool_audit_records
```

- `ExcelRecord`：写 Excel 台账的结果。
- `AlertRecord`：发送或记录预警通知的结果。
- `ToolJob`：后台工具队列任务。
- `DeadLetterRecord`：重试耗尽后的死信。
- `ToolAuditRecord`：工具治理审计记录。

### Agent 运行轨迹

```text
agent_run_traces
```

`AgentRunTrace` 保存每轮 Agent 运行的审计信息：

- 原始输入。
- 脱敏输入。
- 记忆摘要。
- Agent steps。
- 检索知识。
- 回复 messages。
- 评估 JSON。
- 事件驱动运行时的黑板事件、任务、产物。

保存逻辑在 `app/services/trace.py`。

## 8. 单轮聊天主流程

核心调用栈：

```text
student.js
  -> POST /api/chat/stream
    -> ChatService.stream_chat()
      -> MindBridgeAgentHarness.run()
        -> create_agent_runtime().run()
        -> 保存用户消息
        -> 生成报告
        -> 保存 trace
      -> AiClient.stream()
      -> 保存助手消息
      -> dispatch_tools()
      -> SSE done
```

关键文件：

- `app/services/chat.py`
- `app/agents/harness.py`
- `app/agents/factory.py`
- `app/services/ai.py`

### 8.1 Harness 做什么

`MindBridgeAgentHarness` 是业务编排门面，它把 HTTP 层和 Agent 层隔开。

它负责：

1. 清洗用户输入。
2. 创建或复用 `ChatSession`。
3. 调用 Agent runtime。
4. 保存用户消息到 MySQL 和 Redis。
5. 如果本轮需要报告，则创建 `PsychologicalReport`。
6. 保存 `AgentRunTrace`。
7. 生成工具执行计划。
8. 在模型流式回复完成后保存助手消息。
9. 调度 Excel/Case/Alert 等工具任务。

注意一个顺序细节：当前代码是先运行 Agent，再保存本轮用户消息。因此 Agent 读取历史时拿到的是旧历史加当前 `model_input`，不是从数据库里读到刚保存的当前消息。

### 8.2 SSE 输出

`ChatService.stream_chat()` 先发一个 `meta`：

```json
{"type":"meta","sessionId":"..."}
```

然后把 `AiClient.stream(outcome.response_messages)` 的 token 一段段发给前端：

```json
{"type":"token","content":"..."}
```

最后发：

```json
{"type":"done"}
```

## 9. AI Provider

AI 调用统一在 `app/services/ai.py`。

支持三种模式：

```text
AI_PROVIDER=ollama
AI_PROVIDER=openai
AI_PROVIDER=mock
```

### Ollama

配置项：

```env
AI_PROVIDER=ollama
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_MODEL=mindbridge-qwen2.5-7b-ft:latest
```

同步调用：

```text
POST {OLLAMA_BASE_URL}/api/chat
```

流式调用也是 Ollama `/api/chat`，读取逐行 JSON。

### OpenAI-compatible

配置项：

```env
AI_PROVIDER=openai
OPENAI_BASE_URL=https://api.openai.com/v1
OPENAI_API_KEY=...
OPENAI_MODEL=gpt-4o-mini
```

聊天接口：

```text
POST {OPENAI_BASE_URL}/chat/completions
```

向量接口：

```text
POST {OPENAI_BASE_URL}/embeddings
```

### Mock

`mock` 模式不调用外部模型，适合演示和单元测试。它根据关键词返回固定文本或固定 JSON。

## 10. 意图和风险

枚举在 `app/core/enums.py`：

```text
IntentType: CHAT, CONSULT, RISK
RiskLevel: LOW, MEDIUM, HIGH
EmotionLabel: NORMAL, ANXIETY, DEPRESSED, HIGH_RISK
```

意图含义：

- `CHAT`：普通聊天、学习、编程、校园事务。
- `CONSULT`：压力、焦虑、失眠、低落等心理支持诉求。
- `RISK`：自伤、自杀、伤人或即时危险信号。

风险评估在 `app/services/assessment.py`。

评估策略：

1. 先跑高风险关键词硬规则，命中直接返回 HIGH，且不调用模型。
2. 未命中时调用模型，让模型返回严格 JSON。
3. 如果模型失败，走 heuristic 兜底。
4. 如果情绪分数推导出的风险高于模型风险，则按更高风险覆盖。

测试覆盖在 `tests/test_privacy_and_assessment.py`，其中明确验证了高风险硬规则优先于模型调用。

## 11. 记忆系统

记忆服务在 `app/services/memory.py`。

项目分两类记忆：

### 完整长期记录

完整聊天记录写入 MySQL：

```text
chat_sessions
chat_messages
```

这些记录用于审计、管理端查看、Redis 缺失时回填。

### 短期上下文

短期记忆写入 Redis，key 格式：

```text
mindbridge:short-term-memory:{session_public_id}
```

每条 Redis 消息是 JSON：

```json
{
  "role": "user",
  "content": "...",
  "createdAt": "..."
}
```

写入时会：

1. 对手机号、邮箱、身份证做脱敏。
2. `RPUSH` 到 Redis list。
3. `LTRIM` 保留最近 `REDIS_MEMORY_MAX_MESSAGES` 条。
4. 设置 TTL。

如果 Redis 连接失败，系统会打 warning，然后退化为空记忆，不会直接让聊天失败。

### 记忆压缩

`compact_history_for_prompt()` 会把长历史压缩为：

- 一条内部 system 摘要。
- 最近若干条原始消息。

配置：

```env
MEMORY_COMPACTION_ENABLED=true
MEMORY_COMPACTION_RECENT_MESSAGES=8
MEMORY_SUMMARY_MAX_CHARS=500
```

这个摘要只给模型上下文使用，不应该展示给学生。

## 12. 隐私脱敏

脱敏类在 `app/services/privacy.py`。

当前会替换：

- 中国大陆手机号。
- 邮箱地址。
- 18 位身份证号。

替换成统一占位：

```text
[已脱敏]
```

由于当前代码文件里部分中文发生了编码错乱，占位符在源码输出里可能显示成 mojibake，例如 `[宸茶劚鏁廬`。业务意图仍然是脱敏。

## 13. RAG 知识库

知识库服务在 `app/services/knowledge.py`，向量存储在 `app/services/vector_store.py`。

### 知识来源

内置知识文件在：

```text
app/knowledge/*.md
```

启动时 `seed_data()` 会遍历这些 Markdown 文件并调用：

```python
KnowledgeService.ensure_source(file.name, file.read_text(...))
```

如果数据库已有同源同切片内容，则不重复写入；如果内容变化，会删除旧切片并重新入库。

管理端也可以上传文件：

```text
POST /api/admin/knowledge/file
```

支持：

- `.pdf`：用 `pypdf` 提取文本。
- 其他文本文件：按 UTF-8 解码，错误忽略。

### 切片策略

`chunk_text(content, size, overlap)`：

- 默认 chunk size：512。
- 默认 overlap：64。
- 先把空白规整成单空格。
- 按固定窗口切片。

### 检索策略

主路径：

```text
Chroma vector + BM25 hybrid + local reranker
```

兜底路径：

```text
local BM25 + hybrid_score reranker
```

主路径需要：

- `KNOWLEDGE_VECTOR_ENABLED=true`
- 安装 `chromadb`
- 配置 `OPENAI_API_KEY`
- OpenAI-compatible embeddings 接口可用

如果没有 `OPENAI_API_KEY`，且 `KNOWLEDGE_VECTOR_REQUIRED=false`，系统会自动退回本地 BM25。

### 混合召回流程

```mermaid
flowchart TD
    A["用户输入或改写 query"] --> B["向量召回 Chroma"]
    A --> C["BM25 关键词召回 MySQL chunks"]
    B --> D["分数归一化"]
    C --> D
    D --> E["按 vector_weight / bm25_weight 融合"]
    E --> F["local rerank"]
    F --> G["取 top_k"]
    G --> H["扩展最佳 chunk 的相邻片段"]
```

相关配置：

```env
KNOWLEDGE_TOP_K=4
KNOWLEDGE_CANDIDATE_K=16
KNOWLEDGE_CHUNK_SIZE=512
KNOWLEDGE_CHUNK_OVERLAP=64
KNOWLEDGE_HYBRID_VECTOR_WEIGHT=0.65
KNOWLEDGE_HYBRID_BM25_WEIGHT=0.35
KNOWLEDGE_RERANK_ENABLED=true
CHROMA_PERSIST_DIR=data/chroma
CHROMA_COLLECTION_NAME=mindbridge_knowledge
CHROMA_SNAPSHOT_DIR=data/chroma-snapshots
```

## 14. Skill 机制

项目自己的 skill 在根目录 `skills/`。

当前包括：

```text
supportive_response_baseline
high_risk_safety_plan
referral_resource_guidance
anxiety_grounding_support
sleep_routine_support
academic_stress_planning
counselor_handoff_summary
```

加载逻辑在 `app/services/skills.py`。

每个 skill 是一个 `SKILL.md`，要求有 YAML frontmatter：

```yaml
---
name: ...
description: ...
---
```

运行时用途：

- 对 `CHAT` 不注入心理支持 skill。
- 对 `HIGH` 风险注入 `supportive_response_baseline` 和 `high_risk_safety_plan`。
- 对普通咨询注入基础支持、转介资源，再根据关键词追加焦虑、睡眠、学业压力等 skill。
- `counselor_handoff_summary` 用于生成管理端个案交接摘要。

管理端 `/api/agent/status` 会返回 skill 状态，包括 READY/WARN/FAILED。

## 15. Agent 运行时

Agent runtime 选择在 `app/agents/factory.py`。

根据 `AGENT_FRAMEWORK` 选择：

```text
event_driven_multi_agent  默认事件驱动多 Agent
langgraph                 LangGraph 有安装时启用
custom                    自研顺序兜底
```

当前 `Settings` 默认：

```python
agent_framework = "event_driven_multi_agent"
```

所以默认走事件驱动多 Agent。

### 15.1 事件驱动多 Agent

主要文件：

- `app/agents/event_driven_runtime.py`
- `app/agents/autonomous.py`
- `app/agents/coordinator.py`
- `app/agents/events.py`
- `app/agents/registry.py`

这一套不是简单固定流水线，而是黑板协作：

```mermaid
flowchart TD
    A["CollaborationBlackboard"] --> B["CoordinatorAgent 创建任务"]
    B --> C["AgentRegistry 找可认领 Agent"]
    C --> D["Agent 根据能力和置信度 claim 任务"]
    D --> E["Agent 产出 Artifact/Message/Event"]
    E --> A
    A --> F["SafetyAgent 审核回复"]
    F --> G["CoordinatorAgent 接受 final artifact"]
```

黑板里的核心概念：

- `AgentTask`：待办任务。
- `AgentMessage`：Agent 间消息。
- `AgentArtifact`：Agent 产物，例如 intent、risk、context、response_proposal。
- `AgentEvent`：事件日志，例如任务创建、任务认领、安全覆盖、最终采纳。
- `CollaborationBlackboard`：追加式共享状态。

默认参与的 Agent：

```text
CoordinatorAgent
UnderstandingAgent
SafetyAgent
ContextAgent
ResponseAgent
```

#### CoordinatorAgent

职责：

- 创建 root task。
- 根据缺失产物派生任务。
- 控制最大轮数和每轮认领数。
- 要求最终回复必须通过 SafetyAgent 审核。
- 接受最终 `response_proposal`。

关键配置：

```env
AGENT_MAX_ROUNDS=8
AGENT_MAX_CLAIMS_PER_ROUND=4
AGENT_MAX_CLAIMS_PER_AGENT=3
AGENT_FINAL_ACCEPTANCE_MIN_CONFIDENCE=0.6
```

#### UnderstandingAgent

职责：

- 判断 `CHAT / CONSULT / RISK`。
- 输出 `intent` artifact。
- 对明显高风险先走硬规则。
- 对明显普通技术/学习问题优先归类为 `CHAT`。

#### SafetyAgent

职责：

- 独立做心理风险评估。
- 输出 `risk` artifact。
- 如果 HIGH，发布 `SAFETY_OVERRIDE`。
- 审核 `response_proposal` 是否满足安全约束。
- 不通过时发布 critique 和 revision task。

#### ContextAgent

职责：

- 读取 Redis 短期记忆。
- Redis 无数据时从 MySQL 最近消息回填。
- 压缩历史为模型上下文和 memory brief。
- 在咨询或风险场景执行 RAG 检索。
- 按意图和风险加载 skill 上下文。
- 输出 `context` artifact。

#### ResponseAgent

职责：

- 根据 intent、risk、context、skill 生成候选回复 messages。
- 普通聊天走 normal_chat mode。
- 咨询或风险走 support mode。
- 输出 `response_proposal`，等待 SafetyAgent 审核。

### 15.2 LangGraph 运行时

文件：`app/agents/langgraph_runtime.py`

这套是较早的图式流程：

```text
controller
  -> memory
  -> supervisor
  -> knowledge
  -> risk_guardian
  -> companion 或 counselor
  -> end
```

选择条件：

```env
AGENT_FRAMEWORK=langgraph
```

并且 Python 环境里有 `langgraph`。

### 15.3 Custom 顺序兜底

文件：`app/agents/runtime.py`

这套不依赖 LangGraph，也不使用黑板协议。它按固定顺序尝试：

```text
MemoryAgent
SupervisorAgent
KnowledgeAgent
RiskGuardianAgent
CompanionAgent
CounselorAgent
```

普通 `CHAT` 会跳过知识检索和风险评估，直接由 CompanionAgent 生成普通回复 prompt。

咨询或风险会检索知识库、评估风险，再由 CounselorAgent 生成支持性回复 prompt。

## 16. Agent 输出不等于最终文本

需要特别理解一点：Agent runtime 输出的是 `response_messages`，它是给模型用的 prompt messages，不是已经生成好的最终回复。

真正生成文本的是：

```python
async for token in self.ai.stream(outcome.response_messages):
```

也就是说：

1. Agent 负责判断、检索、组装 prompt。
2. AiClient 根据 prompt 调用 Ollama/OpenAI/mock。
3. 模型输出才通过 SSE 发给前端。

## 17. 报告生成规则

`AgentRunResult.requires_report`：

```python
return self.intent != IntentType.CHAT
```

所以：

- `CHAT`：只保存聊天，不生成心理报告，不触发工具队列。
- `CONSULT`：生成心理报告，写 Excel，若 MEDIUM/HIGH 创建 case。
- `RISK`：生成心理报告，写 Excel，创建 case，HIGH 时发送或记录 alert。

报告创建在：

```text
app/agents/harness.py::_create_report()
```

报告字段来自 Agent 评估结果：

- intent
- emotion
- emotion_score
- risk_level
- confidence
- summary

## 18. 工具队列和高风险闭环

工具队列在 `app/services/tool_queue.py`。

如果：

```env
TOOL_QUEUE_ENABLED=true
```

则 `MindBridgeAgentHarness.dispatch_tools()` 不直接调用 MCP，而是写 `tool_jobs` 表，由后台 `ToolQueueWorker` 轮询执行。

### 18.1 入队规则

`ToolQueueService.enqueue_report(report_id, risk_level)`：

```text
所有报告：
  EXCEL_REPORT

MEDIUM 或 HIGH：
  CASE_CREATE

HIGH：
  ALERT_SEND，依赖 CASE_CREATE
```

### 18.2 Worker 执行规则

`ToolQueueWorker`：

- 每隔 `TOOL_QUEUE_POLL_INTERVAL_SECONDS` 秒扫描 pending job。
- 每批最多 `TOOL_QUEUE_BATCH_SIZE`。
- Excel 和 Case 用 `excel_executor`。
- Email/Alert 用 `email_executor`。
- 高风险邮件有每分钟限流。
- 失败后按 attempts 重试。
- 达到最大次数后写入 `dead_letter_records`。
- 服务重启时会把残留 RUNNING job 恢复为 PENDING。

### 18.3 工具动作

工具实现集中在 `app/services/tools.py`。

#### EXCEL_REPORT

写入：

```text
data/mindbridge-risk-ledger.xlsx
```

字段：

```text
reportId, riskLevel, emotion, confidence, summary, createdAt
```

#### CASE_CREATE

创建 `risk_cases`，并用 `counselor_handoff_summary` skill 生成交接摘要。

#### ALERT_SEND

先确保 case 存在，再调用 `notify()`。

通知模式：

```env
ALERT_EMAIL_DELIVERY_MODE=log
```

默认 `log`，也就是只写 `alert_records`，不真正发邮件。

如果要发 SMTP：

```env
ALERT_EMAIL_DELIVERY_MODE=smtp
SMTP_HOST=...
SMTP_PORT=587
SMTP_USERNAME=...
SMTP_PASSWORD=...
SMTP_USE_TLS=true
ALERT_EMAIL_FROM=...
ALERT_EMAIL_TO=...
```

### 18.4 MCP 兼容路径

如果：

```env
TOOL_QUEUE_ENABLED=false
```

则 `dispatch_tools()` 会走 `MindBridgeMcpToolClient`，启动 `app.mcp_tools.server` 并通过 MCP 调用：

```text
mindbridge_excel_report
mindbridge_case_create
mindbridge_alert_send
```

相关文件：

- `app/services/mcp_client.py`
- `app/mcp_tools/server.py`

## 19. 前端结构

项目没有 React/Vue，使用原生 HTML/CSS/JS。

### 登录页

```text
app/static/index.html
app/static/app.js
```

功能：

- 检查服务健康。
- 检查 Agent 状态。
- 输入用户名密码。
- 保存 Basic Auth token。
- 根据角色跳转学生端或管理端。

### 学生端

```text
app/static/student.html
app/static/student.js
```

功能：

- 显示服务状态和模型状态。
- 新建会话。
- 发送消息。
- 读取 SSE token 流并逐步显示。

### 管理端

```text
app/static/admin.html
app/static/admin.js
```

功能：

- 报告数量、高风险数量、case 数、Excel 记录数、alert 数。
- 展示风险个案。
- 展示心理报告。
- 点击报告查看会话档案。
- 上传知识文件。
- 重建或备份向量索引。

## 20. 测试覆盖

测试在 `tests/`。

当前主要覆盖：

- `test_event_driven_multi_agent.py`
  - 黑板 append-only。
  - artifact 不覆盖。
  - AgentRegistry 按 claim confidence 排序。
  - Coordinator 通过 claim 选择任务。
  - AgentModelRegistry 可按 agent 覆盖模型。

- `test_memory_compaction.py`
  - 长历史压缩。
  - 历史摘要长度限制。
  - 禁用压缩时保留原历史。

- `test_privacy_and_assessment.py`
  - 手机、邮箱、身份证脱敏。
  - Redis 序列化前脱敏。
  - 高风险关键词硬规则不调用模型。

- `test_tool_governance.py`
  - 高风险 alert 权限。
  - medium case_create 权限。
  - 未知工具阻断。

- `test_skills.py`
  - skill 加载和校验。

## 21. 运行时常用排查命令

启动服务：

```powershell
.\.venv\Scripts\Activate.ps1
python -m uvicorn app.main:app --host 127.0.0.1 --port 8080
```

健康检查：

```powershell
curl http://127.0.0.1:8080/actuator/health
```

学生账号看状态：

```powershell
curl -u student:student123 http://127.0.0.1:8080/api/agent/status
```

管理账号看报告：

```powershell
curl -u admin:admin123 http://127.0.0.1:8080/api/admin/reports
```

知识库状态：

```powershell
curl -u admin:admin123 http://127.0.0.1:8080/api/admin/knowledge/status
```

重建向量索引：

```powershell
curl -u admin:admin123 -X POST http://127.0.0.1:8080/api/admin/knowledge/rebuild-vector
```

如果没有配置 `OPENAI_API_KEY`，重建向量索引会失败或不可用，但本地 BM25 检索仍可工作。

## 22. 当前代码注意事项

这些是阅读项目时发现的实现细节或潜在问题，建议后续修。

### 22.1 多处中文出现 mojibake

很多 Python 和前端文件里的中文显示为类似：

```text
鏍″洯蹇冪悊
绠＄悊鍛
```

这通常是 UTF-8 内容曾被错误编码读取或保存导致。它会影响：

- 页面显示文案。
- prompt 质量。
- 中文关键词匹配。
- 测试可读性。

如果业务要真实运行，建议统一修复编码。

### 22.2 `ReportService` 方法缩进疑似错误

`app/services/report.py` 中：

- `agent_run_traces` 在类外。
- `tool_audits` 在类外。
- `conversation` 看起来缩进在 `tool_audits` 内部。
- `_report_response` 也可能没有按预期挂到 `ReportService` 上。

这会导致以下路由潜在失败：

```text
GET /api/admin/agent-traces
GET /api/admin/tool-audits
GET /api/admin/conversations/{session_id}
```

以及任何调用 `ReportService._report_response()` 的方法。

我用 `py_compile` 检查时语法能通过，但语法通过不代表类方法绑定正确。建议单独修这个文件的缩进。

### 22.3 ToolGovernanceService 当前没有真正接入 Worker

`app/services/tool_queue.py` 引入了：

```python
from app.services.tool_governance import ToolGovernanceService
```

但 Worker 执行 job 时没有调用 `ToolGovernanceService.start_job()`、`require_allowed()` 或 `finish()`。

结果是：

- `ToolPolicyRegistry` 有测试。
- `tool_audit_records` 表也存在。
- 但队列执行时未实际写审计记录，也未按 policy 阻断工具。

如果要让“工具治理”真正生效，需要在 `_run_job()` 或 `_execute()` 前后接入 governance。

### 22.4 启动自动建表不是迁移系统

`Base.metadata.create_all()` 适合 demo 和首次启动，但不适合长期演进数据库结构。字段变更、索引变更、表重命名不会自动安全迁移。

### 22.5 Basic Auth 和 SHA-256 密码仅适合演示

当前认证实现简单，适合本地演示。生产建议：

- 密码用 bcrypt/argon2。
- 使用 session 或 JWT。
- 加 HTTPS。
- 增加登录失败限制。
- 管理接口加更严格权限和审计。

### 22.6 Redis 连接失败会静默降级

`RedisShortTermMemoryStore` 连接失败时会 warning 并返回空记忆。优点是服务不容易整体挂，缺点是你可能以为系统有记忆，实际 Redis 没连上。

建议启动时或管理端增加 Redis 连接状态展示。

## 23. 最推荐的阅读顺序

如果你想自己继续读代码，建议按这个顺序：

1. `app/main.py`
2. `app/core/config.py`
3. `app/core/database.py`
4. `app/models/entities.py`
5. `app/api/routes.py`
6. `app/services/chat.py`
7. `app/agents/harness.py`
8. `app/agents/factory.py`
9. `app/agents/event_driven_runtime.py`
10. `app/agents/coordinator.py`
11. `app/agents/autonomous.py`
12. `app/services/memory.py`
13. `app/services/knowledge.py`
14. `app/services/ai.py`
15. `app/services/tool_queue.py`
16. `app/services/tools.py`
17. `app/static/student.js`
18. `app/static/admin.js`

读完这些，项目主干基本就通了。

## 24. 心智模型

可以把这个项目理解成四层：

```text
前端层：
  登录页、学生聊天页、管理后台页

HTTP 层：
  FastAPI routes，负责认证、参数、响应类型

业务编排层：
  ChatService + MindBridgeAgentHarness
  负责会话、消息、报告、trace、工具计划

智能能力层：
  Agent Runtime + AiClient + KnowledgeService + SkillLibrary + MemoryStore
  负责意图、风险、上下文、检索、prompt、模型调用

持久化和后台层：
  MySQL + Redis + Chroma + Excel + ToolQueueWorker
  负责记录、短期记忆、向量索引、报告台账和预警闭环
```

最核心的一句话：

MindBridge 不是直接把学生消息丢给模型，而是在模型前面放了一层可审计的 Agent 编排，先判断意图和风险，再决定是否检索知识库、注入心理支持 skill、生成报告和触发后续工具任务。
