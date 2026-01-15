# Agentic-Plus

一个“可直接落地”的 Agent 服务模板：用 **FastAPI** 提供 HTTP API，用 **LangGraph** 组织 Agent 工作流，用 **PostgreSQL + pgvector** 做会话检查点与长期记忆存储，并内置 **JWT 鉴权**、**SSE 流式输出**、**Langfuse 可观测**、**Prometheus/Grafana 监控** 与 **Rate Limit**。

---

## 功能特性

- **用户体系 + JWT**：注册/登录获取 *User Token*；创建会话获取 *Session Token*（用于聊天接口）。
- **聊天接口**：普通对话 `/chatbot/chat`，流式对话（SSE）`/chatbot/chat/stream`，查询/清空消息历史。
- **Agent 工作流**：LangGraph 两节点（`chat` / `tool_call`），支持工具调用（内置 DuckDuckGo 搜索工具）。
- **长期记忆**：使用 `mem0` + `pgvector`，按 `user_id` 写入与召回相关记忆片段。
- **可观测性**：
  - Prometheus 指标：`/metrics`（含 LLM 延迟直方图）。
  - Grafana 仪表盘：已提供 LLM 延迟示例面板。
  - Langfuse：对话回调、Trace 与评测回传。
- **工程化**：环境分层（`.env.<env>[.local]`）、结构化日志（console/json）、Docker/Docker Compose 一键运行。

---

## 技术栈

- Web：FastAPI、Uvicorn
- Agent：LangGraph、LangChain
- 观测：Langfuse、Prometheus、Grafana
- 存储：PostgreSQL（pgvector 镜像）、SQLModel、LangGraph Postgres Checkpointer、mem0
- 其他：slowapi（限流）、structlog（结构化日志）

---

## 目录结构（核心）

- `src/main.py`：FastAPI 应用入口（路由、限流、CORS、异常处理、metrics）
- `src/interface/`：HTTP API
  - `auth.py`：注册/登录/会话管理
  - `interaction.py`：聊天（含 SSE 流式）
  - `router.py`：汇总路由
- `src/agent/workflow.py`：LangGraph Agent 工作流与长期记忆逻辑
- `src/agent/tools/`：工具（当前为 DuckDuckGo Search）
- `src/config/settings.py`：配置加载（按 `APP_ENV` 自动选择 `.env.*`）
- `docker-compose.yml`：Postgres + API + Prometheus + Grafana + cAdvisor
- `evals/`：基于 Langfuse Trace 的自动评测工具

---

## 环境变量与配置

项目会按 `APP_ENV` 读取以下文件（优先级从高到低）：

1. `.env.<env>.local`
2. `.env.<env>`
3. `.env.local`
4. `.env`

示例模板在 `.env.example`。第一次启动前建议拷贝一份：

```bash
cp .env.example .env.development
```

### 必填项（最低可运行集）

- `OPENAI_API_KEY`：LLM 调用必需
- `JWT_SECRET_KEY`：JWT 加密密钥（生产环境必须更换）
- `POSTGRES_*`：数据库连接（建议用 docker compose 启动 pgvector）

### 本地运行 vs Docker 运行（数据库 Host 的区别）

- **如果你在宿主机直接跑 API（`make dev`）**：通常应设置 `POSTGRES_HOST=localhost`
- **如果你在 docker compose 里跑 API**：应设置 `POSTGRES_HOST=db`（compose service 名）

推荐做法：

- `docker compose` 用 `.env.development`（`POSTGRES_HOST=db`）
- 本地启动用 `.env.development.local` 覆盖（`POSTGRES_HOST=localhost`）

---

## 快速开始（Docker Compose，一键起全家桶）

### 1) 准备配置

```bash
cp .env.example .env.development
# 然后编辑 .env.development，至少填好 OPENAI_API_KEY / JWT_SECRET_KEY，并确认 POSTGRES_* 配置
```

### 2) 启动（API + DB + Prometheus + Grafana + cAdvisor）

```bash
make docker-compose-up ENV=development
```

或不用 Makefile：

```bash
APP_ENV=development docker compose --env-file .env.development up -d --build
```

### 3) 访问地址

- API：`http://localhost:8000`
- Swagger：`http://localhost:8000/docs`
- Metrics：`http://localhost:8000/metrics`
- Prometheus：`http://localhost:9090`
- Grafana：`http://localhost:3000`（默认账号 `admin`，密码 `admin`）
- cAdvisor：`http://localhost:8080`

---

## 本地开发（uv + 宿主机运行 API）

### 1) 安装依赖

```bash
make install
```

### 2) 启动数据库（只起 DB，不起 API）

```bash
cp .env.example .env.development
# 本地跑 API 建议写到 .env.development.local：
cp .env.example .env.development.local
# 把 .env.development.local 里的 POSTGRES_HOST 改成 localhost，并填好 OPENAI_API_KEY / JWT_SECRET_KEY

APP_ENV=development docker compose --env-file .env.development up -d db
```

### 3) 启动 API（热更新）

```bash
make dev
```

---

## API 使用说明

> 下面示例假设 `API_V1_STR=/api/v1`（`.env.example` 默认如此）。如果你改了前缀，请自行替换。

### Token 说明（很重要）

- **User Token**：注册/登录后拿到，`sub` 是 `user_id`；用于创建会话、获取会话列表等。
- **Session Token**：创建会话后拿到，`sub` 是 `session_id`；用于聊天接口与消息历史接口。

两者都是 JWT，使用方式一致：`Authorization: Bearer <token>`。

下面用环境变量简化：

```bash
BASE_URL=http://localhost:8000
API_PREFIX=/api/v1
```

### 1) 注册

密码要求：长度 ≥ 8，且包含**大写/小写/数字/特殊字符**。

```bash
curl -sS -X POST "$BASE_URL$API_PREFIX/auth/register" \
  -H "Content-Type: application/json" \
  -d '{"email":"you@example.com","password":"Aa123456!"}'
```

返回中包含 `token.access_token`（User Token）。

### 2) 登录

登录接口使用 `application/x-www-form-urlencoded`（Form）字段：`username` / `password` / `grant_type=password`。

```bash
curl -sS -X POST "$BASE_URL$API_PREFIX/auth/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "username=you@example.com" \
  --data-urlencode "password=Aa123456!" \
  --data-urlencode "grant_type=password"
```

返回 `access_token`（User Token）。

### 3) 创建会话（拿到 Session Token）

```bash
USER_TOKEN="替换成上一步的 access_token"

curl -sS -X POST "$BASE_URL$API_PREFIX/auth/session" \
  -H "Authorization: Bearer $USER_TOKEN"
```

返回：

- `session_id`
- `token.access_token`（Session Token，后续聊天要用它）

### 4) 普通对话

```bash
SESSION_TOKEN="替换成会话返回的 token.access_token"

curl -sS -X POST "$BASE_URL$API_PREFIX/chatbot/chat" \
  -H "Authorization: Bearer $SESSION_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"你好，介绍一下你自己。"}]}'
```

### 5) 流式对话（SSE）

返回为 `text/event-stream`，每个事件形如：

```text
data: {"content":"...","done":false}
```

最后会发送 `done=true` 的结束事件。

```bash
curl -N -X POST "$BASE_URL$API_PREFIX/chatbot/chat/stream" \
  -H "Authorization: Bearer $SESSION_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"用三句话总结 FastAPI 的优点。"}]}'
```

### 6) 获取/清空会话消息

```bash
# 获取
curl -sS -X GET "$BASE_URL$API_PREFIX/chatbot/messages" \
  -H "Authorization: Bearer $SESSION_TOKEN"

# 清空
curl -sS -X DELETE "$BASE_URL$API_PREFIX/chatbot/messages" \
  -H "Authorization: Bearer $SESSION_TOKEN"
```

### 7) 会话列表（需要 User Token）

```bash
curl -sS -X GET "$BASE_URL$API_PREFIX/auth/sessions" \
  -H "Authorization: Bearer $USER_TOKEN"
```

### 8) 修改会话名 / 删除会话（需要 Session Token）

```bash
SESSION_ID="替换成 session_id"

# 修改名称（Form 字段 name）
curl -sS -X PATCH "$BASE_URL$API_PREFIX/auth/session/$SESSION_ID/name" \
  -H "Authorization: Bearer $SESSION_TOKEN" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "name=我的新会话名"

# 删除会话
curl -sS -X DELETE "$BASE_URL$API_PREFIX/auth/session/$SESSION_ID" \
  -H "Authorization: Bearer $SESSION_TOKEN"
```

---

## Agent 工作流说明（简版）

- 工作流定义在 `src/agent/workflow.py`：
  - `chat`：调用 LLM（带重试与模型轮换 fallback），并在 Prometheus 中记录 `llm_inference_duration_seconds`。
  - `tool_call`：当 LLM 触发工具调用时执行工具（当前为 DuckDuckGo Search），再回到 `chat`。
- 系统 Prompt：`src/agent/prompts/system.md`（支持 `long_term_memory` / 当前时间等占位符）。
- 长期记忆：
  - 召回：每次对话会根据最后一条用户消息查询相关记忆并注入到系统提示中
  - 写入：对话完成后异步写入（不阻塞响应）

---

## 监控与可观测

- Prometheus 指标：`/metrics`
  - `llm_inference_duration_seconds`：普通对话 LLM 延迟
  - `llm_stream_duration_seconds`：流式对话 LLM 延迟
- Grafana：已内置示例仪表盘 `LLM Inference Latency`（p95/avg/请求数）
- Langfuse：
  - 通过 `LANGFUSE_PUBLIC_KEY` / `LANGFUSE_SECRET_KEY` / `LANGFUSE_HOST` 配置
  - `evals/` 工具会从 Langfuse 拉取 Trace 并把评分写回 Langfuse

---

## 评测（evals）

评测会：

1. 拉取 Langfuse 过去 24 小时内 **尚未评分** 的 traces
2. 使用 `evals/metrics/prompts/` 下的 metric prompt 让 LLM 给出 0~1 分
3. 将评分回写到 Langfuse，并在本地生成 JSON 报告（可选）

常用命令：

```bash
# 交互模式
make eval ENV=development

# 快速跑默认配置
make eval-quick ENV=development

# 不生成本地报告
make eval-no-report ENV=development
```

相关配置（见 `src/config/settings.py`）：

- `EVALUATION_LLM`：评测模型（默认 `gpt-5`）
- `EVALUATION_BASE_URL`：评测 API Base URL（默认 OpenAI）
- `EVALUATION_API_KEY`：评测 API Key（默认复用 `OPENAI_API_KEY`）
- `EVALUATION_SLEEP_TIME`：每条 trace 之间 sleep 秒数（避免限流）

---

## 常见问题

- **启动后连不上数据库**：确认 `POSTGRES_HOST` 是 `localhost`（本地运行）还是 `db`（compose 内运行）。
- **Grafana 登录不上**：默认账号 `admin`，密码在 `docker-compose.yml` 里设置为 `admin`（建议上线前修改）。
- **流式接口看不到输出**：请用 `curl -N` 或前端使用 SSE/EventSource，避免缓冲。

---

