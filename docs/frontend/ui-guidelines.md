# UI 开发约定

## 品牌

- 产品名统一为 TraceShield。
- 主色使用现有设计 token 中的品牌红 `#C91F37`，不要在组件里散落新的相近红色。
- 使用 `web/src/assets/traceshield-logo.svg` 作为方形产品标，`traceshield-mark.svg` 用于紧凑位置。
- Logo 周围保留至少图标宽度 1/4 的留白，不拉伸、不加外部文字进图形。

## 语义颜色

动作和风险不能只靠颜色：

| 语义 | 推荐颜色倾向 | 必须同时提供 |
| --- | --- | --- |
| 正常/ALLOW | 绿色或中性色 | 文字/图标 |
| WARN/ASK | 琥珀色 | 动作标签与理由 |
| BLOCK/critical | 品牌红/风险红 | 阻断文字、规则、图标 |
| 数据等待 | 灰/蓝 | loading/empty 说明 |
| 演示数据 | 中性徽标 | 明确“演示”来源 |

## 布局与折叠

- 新页面使用现有 ProductPageLayout，不重复实现 SideRail。
- 工作台内嵌左右栏时，折叠后保留可点击的 48–54 px rail。
- 过渡使用 transform/opacity/grid-template 等可合成属性，时长 180–300 ms。
- 折叠按钮在展开/收起状态都可发现，并提供 title/aria-label。
- 尊重 reduced motion。

## 数据状态

每个真实数据区域至少设计：

- 初次 loading；
- 空数据；
- 请求失败；
- 使用 fallback/mock；
- 数据过期或轮询；
- 正常实时。

“0”不是错误，但要给出语义。例如过去 24 小时无调用时显示“实时空闲”或切到明确标记的历史累计，而不是用四个无解释的红色零。

## 文案

- 中文为主，必要技术名保留英文并首次解释。
- “实时流在线”不要写成“插件在线”，除非有真实插件 heartbeat。
- “策略演示开关”不要写成“策略已在 Core 生效”。
- “Assistant 解释”不要写成“Agent 已调查数据库”。
- `ASK` 使用“需要审批”，`BLOCK` 使用“执行前已阻断”。
- 错误给出下一步操作，不只显示 unavailable。

## 表格与图

- 表格行必须可键盘访问，选中状态不只靠背景色。
- 长 ID 默认截断，但提供复制与完整 title/Inspector。
- 风险图控制条不遮挡节点，resize 后重新 fit。
- 图、证据、Inspector 使用稳定 ID 联动，不依赖显示文本。
- 大量实时事件更新时避免破坏当前用户选择与滚动位置。

## 可访问性

- 图标按钮必须有 accessible name；
- 文本与背景满足足够对比度；
- focus ring 不移除；
- 表单 label 与输入关联；
- 动画不成为理解状态的唯一方式；
- 重要错误使用文本和图标；
- 折叠状态使用 `aria-expanded`。

## 组件结构

推荐：

```text
pages/       路由编排与页面级数据
layouts/     跨页面结构
components/  可复用展示/交互
api/         wire DTO 与转换
stores/      跨页面业务状态
types/       UI domain type
styles/      token 与全局规则
```

组件不要直接读取环境变量或拼 Core URL；通过 API client。页面不要直接操作 localStorage；通过专用 Store/utility。

## 完成定义

- [ ] 1180/1250/1380/1680 px 验证。
- [ ] 主导航和页面左右栏展开/折叠验证。
- [ ] light/dark 或项目当前主题场景验证。
- [ ] loading/empty/error/mock/core/polling 验证。
- [ ] keyboard focus 与 reduced motion 验证。
- [ ] `npm --prefix web run typecheck`。
- [ ] `npm --prefix web run build`。
- [ ] `npm --prefix web run smoke`。
- [ ] 数据来源文案与开发者指南同步。

