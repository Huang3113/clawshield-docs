# 术语表

| 术语 | 定义 |
| --- | --- |
| TraceShield | Agent 工具调用运行时安全审计系统 |
| OpenClaw | 当前接入的 Agent/Gateway 宿主 |
| Plugin | 随 OpenClaw Gateway 加载的 TraceShield 执行门与采集器 |
| Core | Fastify 审计服务，负责决策、持久化、查询和 Worker 管理 |
| Method Engine | Python 子进程，对运行轨迹做语义与链路风险分析 |
| Eino Assistant | 基于 Eino ChatModel 的只读流式解释服务 |
| Session | 一段持续的 Agent/用户会话 |
| Run | Session 中一次具体 Agent 执行过程 |
| Trace | 串联消息、工具、结果和决策的关联链 |
| Tool Call | Agent 准备执行的一次工具操作 |
| Step Sequence | 同一 Run 中工具步骤的正整数顺序 |
| Audit Request | 工具执行前发给 Core 的同步请求 |
| Audit Decision | Core 或 fallback 返回的动作、风险、理由和规则 |
| Trace Event | 异步记录消息、工具和生命周期的 v1 信封 |
| ALLOW | 允许继续执行 |
| WARN | 允许执行并记录告警 |
| ASK | 请求人工审批，超时按默认动作处理 |
| BLOCK | 在执行前阻止工具 |
| Risk Level | low/medium/high/critical 四级严重度 |
| Legacy Policy | 当前 TypeScript 确定性基础策略，名称表示基础执行路径 |
| Shadow | Method 后台评估，不改变当次工具决定 |
| Enforce | Method 参与同步裁决并受 safety floor 限制 |
| Safety Floor | enforce 模式下不可被 Method 下调的最低规则 |
| Fallback | 插件无法获得 Core 决策时的本地保守策略 |
| Fail-closed | 故障时拒绝或要求审批，而不是默认放行 |
| Evidence Item | 与一个决策关联的证据项 |
| Evidence Step | 证据项内有序的工具/规则/决策步骤 |
| Rule Hit | 一条 decision 命中的稳定规则 ID 与 detail |
| Observation | 工具执行后的返回内容，用于注入检测和轨迹补全 |
| Risk Graph | 用户意图、工具、资源、违规与 sink 的关系图 |
| Provenance | 数据/内容来自哪些资源和步骤的来源关系 |
| Provenance Barrier | 明确脱敏、匿名、仅聚合或仅公开来源造成的传播屏障 |
| Prompt Injection Drift | 工具输出中的注入信号诱导后续高风险行为 |
| Sensitive-to-External Chain | 敏感源经多步转换到达外部 sink 的路径 |
| SSE | Server-Sent Events，Core 到 Web 或 Assistant 到 Web 的流式协议 |
| Polling | HTTPS 展示入口每 10 秒读取快照的更新方式 |
| Mock | 前端或旧 Core 的模拟数据/服务，不是权威审计事实 |
| Demo Seed | 通过真实 API 写入、明确标记的演示记录 |
| Baseline Profile | 双实例演示中显式禁用 TraceShield 的 OpenClaw 对照组 |
| Protected Profile | 加载并启用 TraceShield 插件的 OpenClaw 实例 |

## 容易混淆

### Core fallback 与 Method 回退

- 插件 fallback：Core 请求失败，插件本地决定，生成 fallback event。
- Method 回退：enforce 内 Worker 失败，Core 改用 legacy；插件仍成功调用了 Core。

### 实时流在线与插件在线

Web EventSource connected 只证明 Web ↔ Core SSE 正常，不证明 OpenClaw 插件刚发送数据。

### 真实模型与真实上下文

Assistant 回复可以是真实模型流，但当前页面附加案件数据仍可为演示摘要。二者必须分别描述。

### policies 表与实际策略

表记录是展示目录；当前基础引擎硬编码，没有动态读取 enabled/config。

