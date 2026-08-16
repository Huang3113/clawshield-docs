# Core 职责

TraceShield Core 是安全审计的权威服务端。它用 Fastify 暴露 HTTP/SSE，用 PostgreSQL 保存事实，并管理本地 Python Method Worker。

## 启动顺序

`core/src/server.ts` 的生产入口：

1. 解析并校验环境变量；
2. 构建 Fastify 实例并注册 CORS、路由和错误处理；
3. 按引擎模式启动 Method 服务；
4. 监听 `0.0.0.0:<TRACESHIELD_CORE_PORT>`；
5. 关闭时停止 Method Worker 并关闭数据库连接池。

数据库 URL 是唯一没有默认值的必填配置。解析失败会在服务监听前终止进程。

## 模块地图

| 层 | 目录 | 说明 |
| --- | --- | --- |
| HTTP | `core/src/routes/` | 参数校验、状态码、SSE 与公共响应 |
| 服务 | `core/src/services/` | 审计事务、事件投影、证据、统计、图谱、Method 生命周期 |
| 引擎 | `core/src/engine/` | 模式编排、Intent/Trace、Worker 协议与 safety floor |
| 数据库 | `core/src/db/` | 连接池、schema、迁移和策略种子 |
| 契约 | `core/src/types/` | 插件与 Method 的 TypeScript 类型 |
| 脚本 | `core/scripts/` | smoke、种子、Method health/replay/report |

## 路由总表

### 健康与统计

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| `GET` | `/health` | Core 与数据库健康 |
| `GET` | `/v1/health` | 同上 |
| `GET` | `/v1/dashboard/runtime-status` | 24 小时与历史统计 |

### 写入

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| `POST` | `/v1/audit/tool-call` | 工具执行前同步审计 |
| `POST` | `/v1/events/batch` | 异步事件批量入库 |
| `POST` | `/v1/method/observation` | 工具结果注入检测 |

### 查询

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| `GET` | `/v1/audit/sessions` | 会话聚合 |
| `GET` | `/v1/audit/sessions/:sessionId/runs` | 会话下所有运行 |
| `GET` | `/v1/audit/events` | 最近工具调用审计列表 |
| `GET` | `/v1/tool-calls/:toolCallId` | 工具详情 |
| `GET` | `/v1/tool-calls/:toolCallId/decision` | 最新决策与规则命中 |
| `GET` | `/v1/runs/:runId/evidence-path` | 运行证据步骤 |
| `GET` | `/v1/runs/:runId/risk-graph` | 风险图，优先 Method |
| `GET` | `/v1/runs/:runId/conversation-summary` | 消息摘要 |

### Method

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| `GET` | `/v1/method/status` | Worker 与队列状态 |
| `GET` | `/v1/method/evaluations` | 最近评估 |
| `GET` | `/v1/method/evaluations/:id` | 评估、违规和图快照 |
| `GET` | `/v1/runs/:runId/method-graph` | 最新成功 Method 图 |
| `GET` | `/v1/runs/:runId/decision-diff` | 基础决策与 Method 差异 |

### 流与 Assistant

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| `GET` | `/v1/stream/audit-events` | Runtime SSE |
| `GET` | `/v1/assistant/health` | 代理 Assistant 健康 |
| `POST` | `/v1/assistant/chat/stream` | 代理 Assistant SSE 对话 |

## CORS 与错误响应

Core 当前对所有来源返回：

```text
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET,POST,PATCH,OPTIONS
Access-Control-Allow-Headers: Content-Type,Accept,Last-Event-ID
```

`OPTIONS` 返回 204。未知路径返回 `404 {"error":"not_found"}`。未捕获错误由统一处理器记录，并返回：

```json
{
  "error": "internal_server_error",
  "message": "TraceShield Core could not process the request."
}
```

当前没有认证或租户隔离，部署时必须在受信网络或受控反向代理之后使用。

## 健康语义

`/v1/health` 只执行 `SELECT 1`：

```json
{
  "ok": true,
  "version": "0.1.0",
  "db_connected": true
}
```

HTTP 状态仍可能是 200 而 `ok=false`，因此自动检查必须解析 JSON，不能只依赖 `curl --fail`。它也不包含 Method 或 Assistant 状态；完整健康需要再请求相应端点。

```bash
curl -s http://127.0.0.1:8787/v1/health | jq -e '.ok and .db_connected'
curl -s http://127.0.0.1:8787/v1/method/status | jq -e '.available'
curl -s http://127.0.0.1:8787/v1/assistant/health | jq -e '.ok and .configured'
```

## 本地开发

```bash
npm --prefix core run db:migrate
npm --prefix core run db:check
npm --prefix core run dev
```

发布构建：

```bash
npm --prefix core run typecheck
npm --prefix core run test
npm --prefix core run build
npm --prefix core run start
```

