# 方法评估 API

Method API 用于健康检查、Observation 检测、评估查询和决策差异分析。

## 状态

### `GET /v1/method/status`

```json
{
  "mode": "shadow",
  "available": true,
  "queue_depth": 0,
  "pending_requests": 0,
  "profile": "balanced",
  "profile_version": "balanced-v1",
  "method_version": "phase0-baseline"
}
```

- `available` 表示 Worker 当前可用，不表示每一次评估都会在超时预算内完成。
- `queue_depth` 是 shadow 后台队列。
- `pending_requests` 是当前通过 NDJSON 等待 Worker 响应的请求。
- legacy 模式不启动 Worker，因此 available 应为 false。

## Observation 检测

### `POST /v1/method/observation`

```json
{
  "event_id": "event-observation-001",
  "session_id": "session-doc-001",
  "run_id": "run-doc-001",
  "trace_id": "trace-doc-001",
  "tool_call_id": "tool-doc-001",
  "step_seq": 1,
  "observation": "工具返回文本",
  "observation_hash": "sha256:example"
}
```

所有 ID 与正整数 step 必填，observation 可为任意 JSON 值，hash 可选。当前内部实际用 run/tool/step/observation/hash；其余字段保留关联契约。

```json
{
  "ok": true,
  "injection_detected": false,
  "injection_score": 0,
  "injection_reasons": []
}
```

检测到注入时当前 score 为 1，未检测为 0。结果尚未入库时先存进程内存，tool result 到达后应用。

## 评估列表

### `GET /v1/method/evaluations?limit=50`

limit 范围 1–200，按创建时间倒序。返回：

- evaluation/request/tool/run/step；
- profile 与版本；
- status；
- Method 原始 decision 与 runtime suggestion；
- risk、latency、diff、trace completeness；
- error code/message；
- revision 与时间。

## 评估详情

### `GET /v1/method/evaluations/:id`

成功：

```json
{
  "evaluation": {},
  "violations": [],
  "graph": {
    "nodes": [],
    "edges": []
  }
}
```

路由只校验 ID 是非空字符串；不存在的合法 UUID 返回 `404 method_evaluation_not_found`。当前没有 UUID 格式预校验，非空但格式错误的 ID 可能由 PostgreSQL 错误进入统一 500，这是后续应收紧的边界。详情通过三个数据库查询组合，供诊断方法结果使用。

## Run Method 图

### `GET /v1/runs/:runId/method-graph`

只返回最新 `status=ok` 的 graph snapshot。没有成功快照返回 `404 method_graph_not_found`。与 `/risk-graph` 不同，它不会回退 legacy 图。

## 决策差异

### `GET /v1/runs/:runId/decision-diff`

按 step 和创建时间返回 evaluations 的 status、suggestion、diff、latency。不存在的 Run 当前返回空数组，而不是 404。

常见 diff：

```text
same_action
legacy_allow_method_block
legacy_block_method_allow
legacy_ask_method_allow
legacy_allow_method_warn
risk_level_changed
```

## Shadow 状态解释

`queued/running/ok/timeout/error/unavailable` 都是有效结果。同步工具已经在 shadow evaluation 完成前按 legacy 结果处理，因此：

- 不要把 Method `BLOCK` 解释为当时已经阻断；
- 它表示方法引擎的旁路建议；
- 调查 UI 应同时展示实际 decision、Method suggestion 与 diff。

## 输入数据保护

Method evaluation 数据库存储输入 hash，不保存完整 runtime trace。Graph/violations 仍可能包含资源提示与理由，开发规则时应避免把 observation 原文复制进 metadata。
