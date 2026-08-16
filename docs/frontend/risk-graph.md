# 风险图与证据联动

Runtime 使用 Vue Flow 展示线性长链与分支图，不再把所有节点强制排成一条横向序列。

## 数据来源

Core `/v1/runs/:runId/risk-graph` 返回：

- `graph_source=method`：最新成功 Method snapshot；
- 否则 legacy 线性图。

前端规范化节点类型：

```text
user_intent
tool_call
sensitive_object
network_sink
policy_decision
```

节点风险会与关联工具调用比较，保留更高值；tool ID 缺失时尝试用 step sequence 关联。

## 线性识别

同时满足：

- edge 数等于 node 数减一；
- 只有一个入度为 0 的起点；
- 每个节点最多一条入边和一条出边。

认为是线性链。

## 蛇形长链

每行最多 6 节点，下一行反向排列：

```text
1 → 2 → 3 → 4 → 5 → 6
                      ↓
12 ← 11 ← 10 ← 9 ← 8 ← 7
↓
13 → 14 → ...
```

主要布局参数：节点宽 174 px、高 58 px，横向步距 232 px，行距 178 px。它能让 20+ 步链在中间工作区内保持可浏览，而不是产生数千像素单行。

## 分层图

非线性图执行拓扑分层：

- 按入度从起点展开；
- depth 决定横向层；
- 同层节点按 98 px 纵向步距排列；
- 层间约 226 px；
- 循环或未被拓扑排序的节点追加展示，不静默丢弃。

## Edge

真实使用 Core edges，而不是只按 nodes 数组画箭头。正常 flow、evidence 关联和 blocked_by 有不同视觉；阻断相关边红色并可动画强调，证据边使用琥珀色。

## 交互

- 鼠标/触控平移缩放；
- 放大、缩小；
- 100% 恢复；
- 全图适配；
- 定位当前节点；
- 节点选择后平滑居中；
- 图数据改变后自动 fit view；
- 点击节点驱动 Inspector 与 evidence table。

## 证据联动

Store 关联：

```text
graph node
  → tool_call_id 或 step_seq
  → ToolCall
  → EvidenceStep
  → selectedNodeId
```

证据行点击会选中图节点；节点变化时表格滚动到对应 evidence。Inspector getters 从 selected node 找到 tool call 与 evidence。

## 图调试

```bash
curl -s http://127.0.0.1:8787/v1/runs/<run-id>/risk-graph | jq '{graph_source, nodes: (.nodes|length), edges: (.edges|length)}'
curl -s http://127.0.0.1:8787/v1/runs/<run-id>/evidence-path | jq '.steps | length'
```

若图有节点但 Inspector 为空，检查：

1. Method node 是否有 tool call ID；
2. step sequence 是否与调用一致；
3. Web 是否加载了同一 run 的 tool calls；
4. evidence 是否存在对应 tool ID；
5. 当前 selected node ID 是否仍属于新图。

## 长链演示

```bash
npm --prefix core run seed:long-chain
```

脚本通过真实 API 创建 28 个工具步骤和 84 个基础 evidence steps，并验证 Method 25 步风险路径。适合回归蛇形布局、缩放、选中和 Inspector 联动。

## 后续改进

- 展示 `graph_source` 和 Method evaluation ID；
- Method SSE 完成后自动刷新图；
- 增加小地图或超长图概览；
- 为循环边、跨层边和并行分支增加测试夹具；
- 使用 E2E 验证节点选择与证据滚动；
- 将布局参数集中为可测试配置。

