# 实时更新

## HTTP 模式：EventSource

Runtime 初始化成功后调用：

```text
GET <coreBaseUrl>/v1/stream/audit-events
```

状态：

```text
connecting → connected → disconnected
```

原生 EventSource 负责按服务端 retry 自动重连。已连接后发生错误，前端保留健康状态 8 秒；若仍未恢复才显示离线，避免网络切换瞬时闪烁。

## 客户端监听

当前注册：

- `connected`
- `audit_event`
- `trace_event`
- `metric_update`
- `heartbeat`

当前 Core 实际发布 audit、trace 和 Method lifecycle；心跳是注释，不触发 heartbeat callback；Core 也没有发布 metric_update。开发时需要区分“客户端预留类型”和“服务端当前事件”。

## audit_event 处理

前端会：

- upsert session 的时间、风险和 unread；
- 为未知 session/run 创建临时条目；
- 创建 ToolCall UI 对象；
- 增加当前内存 metrics；
- 若是 active session，加入 timeline。

实时节点不会自动重新拉完整风险图；Method 图或复杂关系需要手动切换/刷新或增加增量同步逻辑。

## trace_event 处理

更新 session 最新事件；active session 时增加 timeline；message_received/llm_output/message_sending 还会加入 conversation 摘要。

## connected 的语义

当前 Store 在 connected/heartbeat 时把 `pluginLastSeen` 标为 just now。严格说 connected 只证明浏览器连上 Core SSE，不证明插件刚发心跳。因此 UI 文字应更偏向“Core 实时流在线”，插件活跃度需要独立后端指标。

## HTTPS 模式：轮询

当页面协议为 HTTPS，Store 不连接 EventSource：

```text
streamState = polling
每 10 秒刷新 metrics + health + sessions/runs/tool calls
```

原因是公共展示网关对 `/v1/stream/*` 返回 204，并严格限制写请求。

当前轮询不会刷新：

- 当前 Run 风险图；
- evidence path；
- conversation summary。

因此公网演示新工具调用后，列表约 10 秒出现，但已打开图可能需要重新选择会话或刷新页面。

## 页面卸载与 timer

连接函数返回 stop closure，负责 close EventSource、清除 disconnect timer 并更新状态。切换到 HTTPS polling 时也会清理已有流与旧 interval，避免重复连接。

## 排查在线/离线闪烁

1. 直接用 `curl --no-buffer` 保持 Core SSE 10 秒；
2. 确认响应含 `Access-Control-Allow-Origin`；
3. 确认每 2 秒有 heartbeat 注释；
4. 检查浏览器页面是 HTTP 还是 HTTPS；
5. 查看 network 是否不断创建新的 stream；
6. 检查中间代理 idle timeout/response buffering；
7. 不要把 HTTPS polling 标签当成离线错误。

```bash
timeout 10 curl --no-buffer --silent \
  http://127.0.0.1:8787/v1/stream/audit-events
```

## 扩展 Method 实时 UI

要展示 evaluation queued/completed/failed：

1. 扩展 `StreamEventName`；
2. 注册 listener；
3. 在 Store 保存 evaluation 状态；
4. completed 后重新请求 method graph/decision diff；
5. 避免每个 delta 都全量刷新；
6. 断线重连后通过查询 API校准。

## 自动化现状

Core 测试覆盖 SSE CORS 与短心跳；公共网关测试覆盖 Assistant SSE 不缓冲。Web 尚无 EventSource 组件级测试，建议用可注入 EventSource factory 和假时钟补齐 reconnect/grace/polling 测试。

