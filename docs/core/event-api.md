# 事件写入 API

`POST /v1/events/batch` 接收消息、模型、工具和运行生命周期事件。它属于异步证据链，不直接决定工具是否执行。

## 信封结构

```json
{
  "events": [
    {
      "event_id": "event-doc-001",
      "schema_version": "v1",
      "type": "message_received",
      "timestamp": 1786800000000,
      "plugin_id": "traceshield-security-plugin",
      "gateway_id": "gateway-doc",
      "session_id": "session-doc-001",
      "run_id": "run-doc-001",
      "trace_id": "trace-doc-001",
      "mode": "async",
      "payload": {
        "message_id": "message-doc-001",
        "role": "user",
        "content_hash": "sha256:example",
        "summary": {
          "preview": "读取项目说明"
        }
      }
    }
  ]
}
```

批次最多 1000 项，空数组有效。`timestamp` 是非负整数毫秒；标识符最长 300；`gateway_id` 可选。

## 事件类型

| type | 常见 payload | Core 投影 |
| --- | --- | --- |
| `message_received` | message ID、role、hash、summary | `messages` |
| `llm_input` | role、输入摘要 | `messages` |
| `llm_output` | role、输出摘要 | `messages` |
| `message_sending` | 发送前摘要 | `messages` |
| `before_tool_call` | 工具、顺序、参数摘要 | 保留 trace event；同步主记录来自 Audit API |
| `after_tool_call` | tool ID、结果摘要/hash/error/duration | `tool_results`，更新 tool 状态 |
| `agent_end` | 结束摘要/metadata | `messages`，完成 Run |
| `fallback_decision` | 本地动作、理由、规则 | 保留 trace event，供追溯 |

## after_tool_call 示例

```json
{
  "event_id": "event-result-001",
  "schema_version": "v1",
  "type": "after_tool_call",
  "timestamp": 1786800000500,
  "plugin_id": "traceshield-security-plugin",
  "session_id": "session-doc-001",
  "run_id": "run-doc-001",
  "trace_id": "trace-doc-001",
  "mode": "async",
  "payload": {
    "tool_call_id": "tool-doc-001",
    "step_seq": 1,
    "tool_name": "read_file",
    "tool_kind": "file_read",
    "result_preview": "# TraceShield",
    "result_hash": "sha256:example-result",
    "result_summary": {
      "type": "text",
      "length": 13
    },
    "duration_ms": 42
  }
}
```

有 `error` 时工具状态变为 `error`，否则变为 `completed`。原始结果只有在 Core raw result 开关开启时保存。

## 响应

```json
{
  "ok": true,
  "inserted": 1,
  "duplicated": 0,
  "message_extracted": 0,
  "tool_result_extracted": 1
}
```

`event_id` 冲突计入 `duplicated`，不会再次创建 message/result 投影，也不会再次完成 Run。

## 事务语义

整个 batch 使用一个数据库事务：

- 所有新事件与投影全部成功后一起提交；
- 其中一条触发数据库异常，整个 batch 回滚；
- 正常的重复 event 不算异常，可与新事件同批处理。

插件失败后会把事件写入磁盘并重试，因此消费者必须依赖 event ID 幂等，而不能假设网络只发送一次。

## 乱序处理

如果 `after_tool_call` 早于同步 Audit API，Core 会建立 placeholder tool call，状态为 `unknown`。这是为了保留结果而不是因外键失败丢弃，但当前极端乱序可能让 placeholder 状态继续保持 unknown。正常插件路径应保证 before audit 先发生。

## agent_end

新插入 `agent_end` 后：

```text
status = completed
ended_at = 事件时间
end_reason = agent_end
```

只有状态仍为 active 的 Run 会被更新。当前没有自动 Run 超时扫描。

## SSE 发布

批次提交后，每个新插入事件发布一个 `trace_event`；重复事件不发布。SSE payload 只包含安全摘要字段，不等于完整 trace event。

