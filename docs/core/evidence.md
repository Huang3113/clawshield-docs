# 证据链

证据链把一次工具审计的输入要点、规则匹配和最终动作组织为可查询步骤，使控制台能从风险图节点定位到决定性依据。

## 创建时机

每个新的同步 audit decision 在同一数据库事务中创建：

- 1 条 `evidence_items`；
- 3 条 `evidence_steps`；
- decision 的 `evidence_refs` 回写 evidence UUID。

重复 `request_id` 直接返回旧 decision，不重复创建。

## 三个固定步骤

| step_order | step_type | title | detail |
| --- | --- | --- | --- |
| 0 | `tool_call` | Tool call received | tool name/kind |
| 1 | `policy_match` | Policy evaluation completed | matched rules |
| 2 | `decision` | `<ACTION> decision issued` | risk 与 reason |

Evidence item 的 metadata 还保存 request ID、工具名/种类和 resource hint；summary 使用决策理由。

## 查询

```http
GET /v1/runs/:runId/evidence-path
```

```json
{
  "run_id": "run-doc-001",
  "steps": [
    {
      "evidence_step_id": "...",
      "evidence_id": "...",
      "run_id": "run-doc-001",
      "tool_call_id": "tool-doc-001",
      "step_order": 0,
      "step_type": "tool_call",
      "title": "Tool call received",
      "detail": {
        "tool_name": "read_file",
        "tool_kind": "file_read"
      },
      "created_at": "...",
      "evidence_type": "policy_decision",
      "evidence_summary": "No blocking policy matched this tool call.",
      "decision": "ALLOW",
      "risk_level": "low"
    }
  ]
}
```

步骤按创建时间与 step order 正序；同一 Run 多个工具会形成多组三步证据。

## 图谱关联

legacy 风险图会把对应 evidence steps 放入工具节点。Web API 转换层再按 `tool_call_id` 或 node ID 把证据行关联到图节点：

- 点击图节点，Inspector 显示工具/决策/证据；
- 点击证据行，选中对应图节点；
- selected node 变化时证据表平滑滚动。

Method 图中的节点可能只有 `step_seq`，Web 会用同 Run 工具的 step 补关联。

## 证据与 raw 数据

证据不是原始载荷备份。默认情况下它只保存：

- 工具语义；
- 资源提示；
- 规则 ID；
- 动作、风险与理由；
- 关联标识符。

这有利于最小采集，但也意味着调查人员不能仅凭 evidence 重建原文。需要更强不可抵赖性时，应设计签名、哈希链、外部时间戳和受控原始证据仓库，而不是直接长期打开所有 raw 开关。

## 完整性边界

- 当前 evidence 只由同步 decision 创建；异步消息不会自动创建 evidence item。
- 三步模板固定，不会自动引用 Method violation。
- 没有签名或 append-only 存储保证。
- 删除 session 会级联删除证据。
- 没有导出/封存工作流。

扩展时应保持“审计事实”“分析推断”“模型解释”三类来源可区分，不能把 Assistant 文本写成权威证据。

