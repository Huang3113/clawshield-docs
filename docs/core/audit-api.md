# 同步审计 API

`POST /v1/audit/tool-call` 是 TraceShield 的执行前控制接口。调用方必须在工具真正执行前等待它返回，或在失败时执行本地保守策略。

## 请求

```http
POST /v1/audit/tool-call HTTP/1.1
Content-Type: application/json
Accept: application/json
```

```json
{
  "request_id": "audit-demo-001",
  "schema_version": "v1",
  "session_id": "session-demo-001",
  "run_id": "run-demo-001",
  "trace_id": "trace-demo-001",
  "tool_call_id": "tool-demo-001",
  "step_seq": 1,
  "semantic_schema_version": "v1",
  "correlation_source": "openclaw_id",
  "tool_name": "read_file",
  "tool_kind": "file_read",
  "raw_params": {
    "path": "README.md"
  },
  "param_summary": {
    "path": {
      "type": "string",
      "length": 9,
      "preview": "README.md"
    }
  },
  "resource_hint": "README.md",
  "risk_hint": "file_read",
  "context": {
    "user_goal": "读取项目说明",
    "recent_message_hashes": [],
    "workspace_root": "/workspace"
  }
}
```

## 字段约束

| 字段 | 必需 | 约束与用途 |
| --- | --- | --- |
| `request_id` | 是 | 1–300 字符；同步幂等键 |
| `schema_version` | 是 | 固定 `v1` |
| `session_id/run_id/trace_id/tool_call_id` | 是 | 各 1–300 字符 |
| `step_seq` | 否 | 正整数；同 Run 不可重复 |
| `semantic_schema_version` | 否 | 当前只接受 `v1` |
| `correlation_source` | 否 | 最长 100 |
| `tool_name/tool_kind` | 是 | 各 1–300 字符 |
| `raw_params` | 是 | JSON object；仅用于当前判定，默认不落库 |
| `param_summary` | 是 | JSON object；会落库 |
| `resource_hint` | 否 | 最长 2000；路径、URL、命令目标等 |
| `risk_hint` | 否 | 最长 300 |
| `context` | 是 | object；内部字段可空 |
| `context.user_goal` | 否 | 最长 4000 |
| `context.recent_message_hashes` | 否 | 最多 100 项，每项最长 300 |
| `context.workspace_root` | 否 | 最长 2000 |

Zod 会移除未声明字段。不要依赖额外字段穿透到服务层。

## 成功响应

```json
{
  "decision": "ALLOW",
  "risk_level": "low",
  "reason": "No blocking policy matched this tool call.",
  "matched_rules": ["default_allow"],
  "policy_version": "v1",
  "evidence_refs": ["c5f2e5e2-0ee4-46e0-8ad0-eaa000000001"],
  "modified_params": null,
  "approval": null,
  "fallback_used": false,
  "engine": "legacy",
  "engine_version": "v1"
}
```

动作：`ALLOW/WARN/ASK/BLOCK`；风险：`low/medium/high/critical`。

### ASK

```json
{
  "decision": "ASK",
  "risk_level": "critical",
  "reason": "A destructive shell command requires explicit user approval.",
  "matched_rules": ["confirm_dangerous_shell_command"],
  "policy_version": "v1",
  "evidence_refs": ["..."],
  "modified_params": null,
  "approval": {
    "approval_id": "appr_...",
    "title": "Confirm destructive shell command",
    "description": "This command may recursively delete data or overwrite a filesystem. Continue only if you intended this exact operation.",
    "default_action": "BLOCK",
    "timeout_ms": 30000
  },
  "fallback_used": false,
  "engine": "legacy",
  "engine_version": "v1"
}
```

Core 只签发审批描述，不提供审批 resolution 接口；宿主必须实现展示与用户选择，且超时遵守默认 BLOCK。

## 幂等

`request_id` 已存在时，Core 直接读取并映射原 `audit_decisions` 返回，不重新执行策略、创建 evidence 或发布新的数据库记录。

调用方重试要求：

- 同一次逻辑工具调用始终复用 `request_id` 和 `tool_call_id`；
- 新工具调用必须生成新 ID；
- 不要用同一 request ID 携带不同参数，否则拿到的是第一次决策。

## 持久化结果

一次新审计成功会原子写入或更新：

- session、run；
- tool call；
- decision；
- rule hits；
- 一项 evidence 和三步 evidence；
- Run 聚合。

shadow 模式的 Method evaluation 在事务之后异步创建，因此可能比同步响应晚出现。

## 错误

### 400

```json
{
  "error": "invalid_audit_request",
  "details": {
    "fieldErrors": {},
    "formErrors": []
  }
}
```

### 500

数据库或服务异常会由统一错误处理器返回稳定的 `internal_server_error`。插件默认把超时、非 2xx 和无效响应统一视为 Core 不可用并进入 fallback。

## curl 示例

```bash
curl --silent --show-error \
  --request POST http://127.0.0.1:8787/v1/audit/tool-call \
  --header 'Content-Type: application/json' \
  --data '{
    "request_id":"audit-doc-001",
    "schema_version":"v1",
    "session_id":"session-doc-001",
    "run_id":"run-doc-001",
    "trace_id":"trace-doc-001",
    "tool_call_id":"tool-doc-001",
    "step_seq":1,
    "tool_name":"read_file",
    "tool_kind":"file_read",
    "raw_params":{"path":"README.md"},
    "param_summary":{"path":"README.md"},
    "resource_hint":"README.md",
    "risk_hint":"file_read",
    "context":{"user_goal":"阅读项目说明","workspace_root":"/workspace"}
  }' | jq
```

示例 ID 固定是为了便于重放；重复执行会返回同一结果。开发多个用例时为每个用例更换 ID。

