# 页面地图

## 路由

| 地址 | 页面 | 定位 |
| --- | --- | --- |
| `/overview` | ExecutiveOverview | 安全总览与演示叙事 |
| `/runtime` | RuntimeAudit | 实时审计工作台 |
| `/sessions` | Sessions | 会话列表 |
| `/tool-calls` | ToolCalls | 工具调用筛选与详情入口 |
| `/policies` | PolicyCenter | 策略目录与演示开关 |
| `/risk-intelligence` | RiskIntelligence | 风险情报展示 |
| `/evidence` | EvidenceVault | 证据仓库展示 |
| `/reports` | ComplianceReports | 报告展示 |
| `/core` | CoreStatus | Core/数据库状态摘要 |
| `/assistant` | SecurityAssistant | Eino 只读安全分析 |
| `/settings` | Settings | 控制台设置展示 |

`/` 重定向 overview，未知路径也回到 overview。`ComingSoon.vue` 存在但没有路由。

## 数据真实性矩阵

| 页面 | 数据来源 | 当前边界 |
| --- | --- | --- |
| Runtime | Core + HTTP SSE/HTTPS 轮询；初始化失败保留 mock | 真实数据最完整 |
| Sessions | Runtime Store | 成功时 Core，失败时 mock |
| Tool Calls | Runtime Store | 成功时 Core；真实延迟当前映射 0ms |
| Core | 部分 Core | health/版本真实；队列、策略版本等部分为展示值 |
| Assistant | 对话流真实；案件上下文演示 | 真实模型不等于真实案件数据 |
| Overview | 混合 | 多项数字有演示下限或固定值 |
| Policies | mock + localStorage | 开关不改变 Core 决策 |
| Risk Intelligence | 静态演示 | 分数、趋势、热图和链为常量 |
| Evidence | 静态演示 | 案件、哈希和时间线为常量 |
| Reports | 静态演示 | 生成/下载交互不产生真实报告 |
| Settings | 页面内状态 | 保存与连接测试为演示 |

## Runtime

Runtime 是核心开发页面：

- 顶部四张指标卡与数据窗口标记；
- 左侧 session/run 选择；
- 风险路径、时间线、工具调用、对话摘要四个 tab；
- 中央 Vue Flow 图；
- 右侧 decision/evidence/assistant Inspector；
- 底部证据路径；
- 主导航、会话栏、Inspector 均可折叠。

真实 Core 初始化成功后选择第一条 session 与其 run，并并行加载图谱、证据和 conversation summary。

## Overview

Overview 面向演示和管理叙事，不是严格的实时仪表盘。当前至少包括这些展示逻辑：

- 已防护调用最少 1284；
- 已阻断操作最少 23；
- 7/30 天数据使用固定倍率；
- 覆盖率以固定 99.72% 为基础；
- 威胁构成、活动、风险链和审批等包含固定示例。

新增真实指标时，应移除对应演示下限并标明数据库窗口。

## Policy Center

策略开关先更新前端，写入：

```text
traceshield.policy-center.demo-state.v1
```

然后尝试调用 Core policy API。当前 Core 路由不存在，返回 404 后 UI 提示本地保存/待同步。刷新仍可恢复本地演示状态，但真实引擎规则不变。

## Assistant

四个案例和四个快捷调查来自前端常量。点击案例会取消当前流、重置任务日志、生成新 conversation ID，并构造该案例的演示上下文。

对话回复是真实 Eino/DeepSeek SSE；右侧任务四步由真实 start/delta/done 回调驱动。输出目前按普通段落渲染，不解析 Markdown 或链接。

## Core Status

Core 和数据库连接来自 `/v1/health`，事件数借用工具调用指标；方法和 Assistant 需要分别查看专用 health 才能得到完整状态。不要把卡片上的所有字段都视为 Core 动态返回。

## 新页面检查清单

- [ ] 在 router 中懒加载并加入 SideRail。
- [ ] 决定使用普通或 immersive 布局。
- [ ] 明确数据是 Core、localStorage 还是静态演示。
- [ ] 为 loading、empty、error、stale 设计状态。
- [ ] 使用 token 与 Lucide 图标，避免引入孤立视觉语言。
- [ ] 支持主导航折叠和 reduced motion。
- [ ] 添加到 smoke 路由检查。

