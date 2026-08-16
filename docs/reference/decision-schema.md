# 决策结构

## AuditDecision

```ts
interface AuditDecision {
  decision: "ALLOW" | "WARN" | "ASK" | "BLOCK"
  risk_level: "low" | "medium" | "high" | "critical"
  reason: string
  matched_rules: string[]
  policy_version?: string
  evidence_refs?: string[]
  modified_params?: Record<string, unknown> | null
  approval?: AuditApproval | null
  fallback_used?: boolean
  engine?: "legacy" | "method"
  engine_version?: string
  method_evaluation_id?: string
}
```

Core 正常响应会给 policy/evidence/modified params/approval/fallback/engine 字段；插件 parser 至少严格验证 action、risk、reason 和 rules。

## 动作语义

### ALLOW

没有策略阻止执行。它不保证工具本身成功，只允许宿主继续调用。

### WARN

允许执行并记录风险。当前 OpenClaw 适配不保证 UI 可见 warning。

### ASK

必须附 approval，否则插件转为 block。真正的用户选择由宿主完成。

### BLOCK

工具不应执行。插件生成带稳定前缀的 block reason，并通过中间件强调未执行。

## 风险与动作独立

动作表示控制行为，risk 表示严重度。它们相关但不是同一个枚举，例如外部网络是 ASK/medium，危险磁盘命令是 ASK/critical。

Run 聚合分别取最高动作和最高风险：

```text
ALLOW < WARN < ASK < BLOCK
low < medium < high < critical
```

## AuditApproval

```ts
interface AuditApproval {
  approval_id: string
  title: string
  description: string
  default_action: "ALLOW" | "BLOCK"
  timeout_ms: number
}
```

TraceShield 当前规则使用 30 秒、默认 BLOCK。approval ID 用于宿主关联，不代表 Core 已保存用户 resolution。

## modified_params

存在时插件把它映射为新的工具 params。当前 Core 基础和 Method 实时路径都未实际签发参数改写，因此通常为 null。

未来实现时必须：

- 明确修改前后差异；
- 对改写后的参数重新应用安全底线；
- 在 evidence 中保存安全摘要；
- 防止删掉关键宿主字段；
- 测试 OpenClaw 对 params replacement 的版本兼容。

## fallback_used

Core 正常决策当前为 false。插件本地 fallback 生成 true。它不表示 Method enforce 失败回退 legacy；后者通过 `engine=legacy` 识别。

## engine

- `legacy/v1`：基础 TypeScript 规则产生最终决定；
- `method/<method version>`：enforce 模式 Method 参与并通过 safety floor 合并。

shadow 模式同步响应仍为 legacy，Method suggestion 在 evaluation 查询中单独查看。

## 规则与证据

`matched_rules` 是稳定机器 ID，不应放完整敏感内容。`evidence_refs` 指向 Core evidence UUID；调用方通过 Run evidence API 查询步骤，而不是把完整证据塞回同步响应。

## 错误处理

插件收到以下响应时视为审计失败并进入 fallback：

- action 不在枚举；
- risk 不在枚举；
- reason 非字符串；
- rules 非 string array；
- HTTP 非 2xx；
- 超时/网络错误。

这种严格解析防止损坏或被中间层篡改的响应意外放行。

