# 实时流 API

`GET /v1/stream/audit-events` 为 Runtime 控制台提供进程内 SSE 通知。

## 连接

```bash
curl --no-buffer \
  --header 'Accept: text/event-stream' \
  http://127.0.0.1:8787/v1/stream/audit-events
```

首次输出：

```text
retry: 3000
event: connected
data: {"ok":true}
```

`retry: 3000` 告诉 EventSource 断线后三秒重连。Core 每两秒写一个 SSE 注释作为保活：

```text
: heartbeat 1786800000000
```

注释不是名为 `heartbeat` 的事件，浏览器不会触发相应 listener，但连接会保持活跃。

## audit_event

同步审计事务提交后发布：

```text
event: audit_event
id: <进程生成的 UUID>
data: {"request_id":"...","session_id":"...","run_id":"...","trace_id":"...","tool_call_id":"...","step_seq":1,"tool_name":"read_file","tool_kind":"file_read","decision":"ALLOW","risk_level":"low","reason":"...","matched_rules":["default_allow"],"engine":"legacy","engine_version":"v1"}
```

## trace_event

事件批次中的新事件提交后发布安全摘要：

```text
event: trace_event
data: {"event_id":"...","type":"message_received","timestamp":1786800000000,"plugin_id":"traceshield-security-plugin","session_id":"...","run_id":"...","trace_id":"...","role":"user","summary":{"preview":"..."}}
```

## Method 生命周期

Core 还会发布：

- `method_evaluation_queued`
- `method_evaluation_completed`
- `method_evaluation_failed`

当前 Web 尚未注册这三类 listener；开发 Method 实时 UI 时应补充客户端类型和 Store 处理。

## 实现限制

SSE 客户端列表只在当前 Core 进程内存：

- 没有持久缓冲；
- 没有断线补发；
- 不读取 `Last-Event-ID`；
- `id` 是通知 UUID，不是数据库游标；
- 多 Core 实例间不共享发布；
- 慢客户端没有独立队列治理。

因此 SSE 用于“提示界面刷新”，不是审计事实的唯一传输。客户端重连后应重新请求权威查询 API。

## CORS 与代理

路由使用 `reply.hijack()`，因此显式写入：

```text
Access-Control-Allow-Origin: *
Cache-Control: no-cache, no-transform
X-Accel-Buffering: no
Connection: keep-alive
```

心跳间隔短于常见本机代理空闲超时，减少无事件期间被断开的概率。反向代理仍需关闭缓冲并允许长连接。

## Web 连接状态

HTTP 页面使用 EventSource 原生重连。已成功连接后发生网络错误，Web 等待 8 秒才显示离线，避免短暂切网造成状态闪烁。

HTTPS 公共展示入口有意不使用此 SSE，而设为 `polling`，每 10 秒刷新统计与列表。它不是“实时流离线”，而是不同传输模式。

## 后续扩展建议

需要水平扩展与可靠补发时，可为事件分配单调游标，把通知写入共享消息流，并让客户端携带 Last-Event-ID 恢复；数据库查询仍作为最终一致性的兜底。

