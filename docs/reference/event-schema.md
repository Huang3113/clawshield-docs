# 事件结构

## TraceEvent v1

```ts
interface TraceEvent {
  event_id: string
  schema_version: "v1"
  type: TraceEventType
  timestamp: number
  plugin_id: string
  gateway_id?: string
  session_id: string
  run_id: string
  trace_id: string
  mode: "sync" | "async"
  payload: Record<string, unknown>
}
```

`timestamp` 为 Unix 毫秒。event ID 是跨网络重试的幂等键；session/run/trace 是层次关联键。

## 类型

```text
message_received
llm_input
llm_output
message_sending
before_tool_call
after_tool_call
agent_end
fallback_decision
```

## 消息类 payload

由 `normalizeMessage()` 生成：

```json
{
  "message_id": "message-001",
  "role": "user",
  "content": "脱敏并截断的预览",
  "content_hash": "短哈希",
  "summary": {
    "type": "string",
    "length": 128,
    "preview": "..."
  },
  "metadata": {}
}
```

当 `debug_full_payload=false` 时 content 使用 `redactText`；hash 对原输入计算以便关联。Core raw payload 默认不保存，但 messages 会投影 preview/hash/summary。

## before_tool_call payload

```json
{
  "tool_call_id": "tool-001",
  "step_seq": 1,
  "correlation_source": "openclaw_id",
  "tool_name": "read_file",
  "tool_kind": "file_read",
  "param_summary": {
    "path": {
      "type": "string",
      "length": 9,
      "preview": "README.md"
    }
  },
  "resource_hint": "README.md",
  "risk_hint": "file_read",
  "metadata": {}
}
```

异步 before event 不包含 raw params。同步 AuditRequest 单独发送 raw params 给 Core 判定。

## after_tool_call payload

```json
{
  "tool_call_id": "tool-001",
  "step_seq": 1,
  "correlation_source": "openclaw_id",
  "tool_name": "read_file",
  "tool_kind": "file_read",
  "param_summary": {},
  "result_preview": "...",
  "result_hash": "...",
  "result_summary": {
    "type": "string",
    "length": 120,
    "preview": "..."
  },
  "error": null,
  "duration_ms": 42,
  "risk_hint": "file_read"
}
```

## fallback_decision payload

载荷复用 AuditDecision 的核心字段：decision、risk、reason、matched rules、approval 和 `fallback_used=true`。它记录插件未取得 Core 正常响应后的本地动作。

## 工具种类推断

插件按工具名正则推断：

| 名称信号 | tool_kind |
| --- | --- |
| shell/bash/terminal/exec/command | `shell_exec` |
| write/patch/edit/save | `file_write` |
| delete/remove/unlink/rm | `file_delete` |
| http/fetch/request/curl/web | `network_request` |
| send/email/message/slack | `message_send` |
| install/plugin | `plugin_install` |
| read/open/cat | `file_read` |
| 无匹配 | `unknown` |

宿主已提供可靠 tool kind 时优先使用宿主值。

## Resource hint

依次查找参数 key：

```text
path, file, filename, url, command, cmd, to
```

取首个非空字符串并截至 300 字符。当前 resource hint 不额外走 `redactText()`，因此生产者应确保候选字段不会包含完整秘密。

## 脱敏

敏感 key 匹配 token/api key/password/passwd/secret/cookie/authorization/private key；值替换为 redacted 标记和短 hash。文本识别常见 `sk-` key、JWT 样式、PEM private key 和 secret assignment。

预览默认最多 500 字符，超出附 `[TRUNCATED:n]`。参数字符串摘要 preview 最多 120 字符。

## 版本演进

增加字段应保持 v1 消费者可忽略；改变字段含义、必填性或枚举需要新 schema version。Core 当前 route 固定只接受 `v1`，升级必须先让服务端支持双版本，再升级插件生产者。

