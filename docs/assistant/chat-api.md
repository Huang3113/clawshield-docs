# 对话与 SSE

## 直接服务请求

```http
POST /v1/chat/stream HTTP/1.1
Content-Type: application/json
Accept: text/event-stream
```

```json
{
  "conversation_id": "traceshield-investigation-001",
  "message": "解释这条 BLOCK 决策。",
  "history": [
    {
      "role": "user",
      "content": "请先总结风险路径。"
    },
    {
      "role": "assistant",
      "content": "路径包含敏感资源读取与外部发送。"
    }
  ],
  "context": {
    "decision": "BLOCK",
    "risk_level": "critical",
    "matched_rules": ["deny_secret_file_read"]
  }
}
```

正常浏览器应调用 Core 的 `/v1/assistant/chat/stream`，而不是直连此路由。

## 服务端校验

- Content-Type 为空或 `application/json`；
- 拒绝未知 JSON 字段；
- Body 默认最多 256 KiB；
- conversation ID 最长 128，字符必须匹配安全 ID 规则；
- message 非空、有效 UTF-8，默认最多 12000 Unicode rune；
- history 默认最多 40；
- history 角色只能 user/assistant，内容非空；
- history 总量默认最多 48000 rune；
- context 序列化后默认最多 32 KiB；
- 请求体只能包含一个 JSON 值。

未提供 conversation ID 时生成 `conv_` 加 24 个十六进制字符。服务不保存此 ID 对应的历史，每次请求仍需带 history。

## 消息顺序

发给模型的消息严格为：

1. TraceShield 只读系统提示；
2. 可选的、被标记为不可信证据的 JSON context system message；
3. user/assistant history；
4. 当前 user message。

## SSE 事件

```text
event: start
data: {"conversation_id":"...","model":"deepseek-v4-flash"}

event: delta
data: {"content":"..."}

event: done
data: {"conversation_id":"...","finish_reason":"stop"}
```

失败：

```text
event: error
data: {"code":"upstream_error","message":"assistant model request failed"}
```

Assistant SSE 没有 heartbeat、event ID 或 retry。Core 代理不重写正常事件内容。

## 建流前重试

模型 stream 建立前最多 2 次尝试，即失败后重试一次，不额外 sleep。可重试：

- HTTP 408、409、425、429；
- HTTP 5xx；
- status 0、网络错误或未分类适配错误。

不重试：

- 普通永久 4xx；
- context canceled/deadline exceeded；
- 已返回 stream 后的中断。

只有在下游尚未看到 `start` 时才重试，避免用户收到重复开头。一旦 start 已发，后续错误只发送 error event。

## 超时与取消

请求使用 `context.WithTimeout`，默认 60 秒。浏览器断开会沿 Core 代理取消 Assistant 请求，再取消模型流。客户端主动点击停止不会在服务端保存半截回答。

## Web 解析

聊天是 POST，浏览器不能使用原生 EventSource，因此 `web/src/api/assistant.ts` 用 fetch + ReadableStream 手动解析：

- 支持 LF 与 CRLF 空行边界；
- 忽略 SSE 注释；
- 合并多行 data；
- 支持 start/delta/done/error；
- 兼容 `[DONE]`；
- 识别 content/delta/text 增量字段；
- 流结束前没收到 done 时抛 `INCOMPLETE_STREAM`；
- done 但没有文本时抛 `EMPTY_RESPONSE`。

## 生命周期 UI

Assistant 右栏的四步状态来自真实前端事件：请求提交、start、首个 delta、done。停止、切换案例、新建调查和页面卸载都会 Abort 当前请求；request serial 防止旧回调污染新会话。

## curl 调试

```bash
curl --no-buffer \
  --request POST http://127.0.0.1:8787/v1/assistant/chat/stream \
  --header 'Content-Type: application/json' \
  --data '{
    "conversation_id":"docs-check-001",
    "message":"只回答：流式接口正常。",
    "history":[],
    "context":{"source":"developer-guide-check"}
  }'
```

这会产生真实模型费用，只用于有 key 的测试环境。

