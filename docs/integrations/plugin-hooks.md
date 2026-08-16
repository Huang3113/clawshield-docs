# Hook 与执行门

## Hook 表

| Hook/能力 | 优先级 | 预算 | 作用 |
| --- | ---: | ---: | --- |
| `before_tool_call` | 100 | `audit_timeout_ms + 200`，默认 600 ms | 执行前同步裁决 |
| `after_tool_call` | 80 | 2000 ms | 结果留痕与 observation 检测 |
| `message_received` | 80 | 2000 ms | 采集用户输入摘要 |
| `llm_input` | 80 | 2000 ms | 采集模型输入摘要 |
| `llm_output` | 80 | 2000 ms | 采集模型输出摘要 |
| `message_sending` | 80 | 2000 ms | 采集发送前消息摘要 |
| `agent_end` | 80 | 2000 ms | 完成 Run |
| `before_prompt_build` | 60 | 1000 ms | 注入明确的阻断语义 |

## before_tool_call

```text
OpenClaw 准备工具
  → 规范化上下文与步骤
  → before_tool_call 异步事件入内存队列
  → 构造 AuditRequest
  → 最多等待 Core 400ms
  → 映射 ALLOW/WARN/ASK/BLOCK
  → OpenClaw 决定是否执行
```

Core HTTP timeout 与 Hook timeout 是两个时钟。400 ms 是 AuditClient 的等待上限；Hook 默认有 600 ms 总预算，留出映射与异常处理时间。

## 动作映射

### ALLOW

返回空 Hook 结果，宿主正常执行。

### WARN

决策会记录在插件日志和 Core。当前 `mapAuditDecision()` 虽生成 warning 元数据，但最终 Hook 适配在未改参数时返回 `undefined`，所以不能保证 OpenClaw UI 一定显示警告；运行语义是放行并留痕。

### ASK

返回 `requireApproval`，包含标题、描述、severity、timeout 和 timeout behavior。critical 映射为 critical severity，其余为 warning。approval 缺失时安全地转为 BLOCK。

### BLOCK

```json
{
  "block": true,
  "blockReason": "TraceShield BLOCKED: tool call was blocked and was not executed.\n..."
}
```

理由明确写入风险和规则，并告诉模型不能声称任务完成。

### modified params

如果 decision 含 `modified_params`，插件返回新 `params`。当前 Core 基础规则不会产生它，但契约已为参数净化保留能力。

## 阻断可见性

只有 Hook 返回 block 还不够；模型可能把工具错误概括成成功。插件做两层强化：

1. `before_prompt_build` 告诉模型，遇到 `TraceShield BLOCKED` 必须说明未执行；
2. Agent Tool Result Middleware 给阻断结果补 `status=blocked` 与可见说明。

这不是给模型新的权限，而是保持用户看到的叙述与实际执行状态一致。

## after_tool_call

工具执行后：

- Run Registry 补齐工具关联；
- 生成脱敏 result preview/hash/summary；
- 异步事件入队；
- 有 result 时非阻塞调用 observation endpoint；
- Observation 失败只记录 warning，不改变已经执行的工具结果。

## 消息 Hook

消息与模型 Hook 生成统一 `TraceEvent v1`，只入异步队列，不等待 Core。`agent_end` 还会通知 Run Registry 结束当前关联上下文。

## Security Audit Collector

插件向 OpenClaw 注册一个信息级检查，说明 TraceShield 已启用并指向哪个 Core。这是配置可见性，不是实时健康探测。

## 开发 Hook 的规则

- 同步路径不得执行长时模型调用；
- `before_tool_call` 失败必须有明确 fallback；
- 采集逻辑不能改变原工具参数，除非收到合法 modified params；
- 所有 hook 输入都视为不可信并做类型收窄；
- Hook 超时需短于宿主总工具预算；
- 阻断必须对宿主和模型都可见；
- 测试必须覆盖 OpenClaw 提供 ID 与缺失 ID 两种情况。

## 测试

插件测试覆盖决策 mapping、fallback、sanitizer、契约、配置、工具关联、共享队列和集成行为：

```bash
npm --prefix openclaw-plugin run test
```

修改 Hook 后还需真实 Gateway 重启验证，因为 TypeScript 单测无法完全覆盖宿主版本的 Hook 语义。

