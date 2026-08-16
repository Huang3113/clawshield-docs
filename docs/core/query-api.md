# 查询与图谱 API

查询 API 是 Web 与 PostgreSQL 之间的稳定边界。除明确说明外，结果按最近时间优先。

## 会话

### `GET /v1/audit/sessions?filter=all`

`filter` 只能是 `all` 或 `risk`，最多返回 200 条。

`risk` 只保留最新 Run 为 high/critical，或最终动作为 BLOCK 的 session。

```json
{
  "sessions": [
    {
      "session_id": "session-doc-001",
      "first_seen_at": "2026-08-16T01:00:00.000Z",
      "last_seen_at": "2026-08-16T01:02:00.000Z",
      "run_count": 1,
      "message_count": 2,
      "tool_call_count": 1,
      "latest_run_id": "run-doc-001",
      "latest_run_started_at": "2026-08-16T01:00:00.000Z",
      "risk_level": "low",
      "final_decision": "ALLOW",
      "latest_event_type": "agent_end"
    }
  ],
  "filter": "all"
}
```

### `GET /v1/audit/sessions/:sessionId/runs`

session 不存在返回 404。成功返回该 session 的全部 runs，当前没有分页。

## 最近工具审计

### `GET /v1/audit/events?limit=50`

名字是 events，但当前结果实际以 `tool_calls` 为主，并关联每个工具最新 decision。`limit` 默认 50，范围 1–200。

返回工具 ID、session/run/trace、name/kind、step、correlation、resource/risk hint、status、开始时间，以及最新动作、风险、理由、规则与决策时间。

## 工具详情

### `GET /v1/tool-calls/:toolCallId`

只返回安全查询字段，不返回 `raw_params`：

```json
{
  "tool_call": {
    "tool_call_id": "tool-doc-001",
    "request_id": "audit-doc-001",
    "session_id": "session-doc-001",
    "run_id": "run-doc-001",
    "trace_id": "trace-doc-001",
    "tool_name": "read_file",
    "tool_kind": "file_read",
    "step_seq": 1,
    "correlation_source": "openclaw_id",
    "param_summary": {},
    "resource_hint": "README.md",
    "risk_hint": "file_read",
    "status": "completed",
    "started_at": "...",
    "updated_at": "..."
  }
}
```

不存在返回 `404 tool_call_not_found`。

### `GET /v1/tool-calls/:toolCallId/decision`

返回 tool、最新 decision 和该 decision 的全部 rule hits。可用于 Inspector 展示策略理由、审批和证据引用。

## 证据路径

### `GET /v1/runs/:runId/evidence-path`

Run 不存在返回 404。按 evidence 创建时间和 step order 正序返回，字段包含 evidence/step ID、tool、类型、标题、detail、decision 和 risk。

详见[证据链](evidence.md)。

## 风险图

### `GET /v1/runs/:runId/risk-graph`

选择逻辑：

1. 验证 Run 存在；
2. 查询最新成功 Method graph snapshot；
3. 存在则 `graph_source=method`；
4. 不存在则从工具与决策生成 `legacy_linear` 图。

Method 响应：

```json
{
  "run_id": "run-doc-001",
  "graph_source": "method",
  "method_evaluation_id": "...",
  "nodes": [],
  "edges": []
}
```

legacy 图以 User Request 为起点，按工具顺序形成线性 `flow`。工具节点包含 decision、risk、reason、rules 和 evidence steps。

## 对话摘要

### `GET /v1/runs/:runId/conversation-summary`

最多返回 200 条 message 投影，按发生时间与创建时间正序。只包含：

- message row ID；
- event ID/type；
- role；
- summary；
- occurred_at。

不返回 raw payload，也不返回 `content_preview`。Web 会从 summary 中选择 `preview/value/keys` 展示。

## Dashboard 统计

### `GET /v1/dashboard/runtime-status`

```json
{
  "tool_calls_24h": 4,
  "blocked_24h": 1,
  "high_risk_24h": 1,
  "policy_hits_24h": 4,
  "tool_calls_total": 29,
  "blocked_total": 1,
  "high_risk_total": 2,
  "policy_hits_total": 29,
  "demo_seed_calls_24h": 4,
  "demo_seed_calls_total": 28,
  "latest_tool_call_at": "2026-08-16T01:00:00.000Z",
  "metric_window": "24h"
}
```

口径：

- tool calls 按 `tool_calls.started_at`；
- blocked/high risk 按 `audit_decisions.created_at`；
- policy hits 按 `audit_rule_hits.created_at`；
- high risk 包含 high 和 critical；
- 演示种子按 `correlation_source=long_chain_seed` 或 `session_id LIKE 'demo-%'` 识别。

最近 24 小时无工具调用但历史存在时，`metric_window=history`；24h 字段仍诚实为零，Web 使用 total 展示并明确标记历史累计。

## 错误与分页现状

当前主要查询使用固定上限，不提供游标分页。生产化时需要为 session、runs、events、messages 和 evaluations 设计稳定游标，避免 offset 在实时写入时漂移。

