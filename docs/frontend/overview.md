# 前端架构

TraceShield Web 是 Vue 3 单页应用，面向桌面安全运营与演示场景。

## 技术栈

| 依赖 | 当前版本 | 用途 |
| --- | --- | --- |
| Vue | 3.5.13 | 组件与响应式系统 |
| Vue Router | 4.5.0 | 路由与懒加载 |
| Pinia | 2.3.0 | Runtime 与 UI 状态 |
| Vite | 6.0.7 | 开发与构建 |
| TypeScript | 5.7.2 | 类型检查 |
| Naive UI | 2.40.4 | 部分 UI 基础组件 |
| Lucide Vue | 0.468.0 | 图标 |
| Vue Flow | 1.42.5 | 风险图交互 |
| Tailwind CSS | 3.4.17 | 工具样式基础设施 |

入口链：

```text
index.html
  → src/main.ts
  → Pinia + Router + global styles
  → App.vue
  → runtimeStore.initialize()
  → lazy-loaded route component
```

HTML 使用 `zh-CN`、TraceShield 标题、主题色与 Logo。当前没有 i18n 框架，中英文文案直接位于 Vue 文件。

## 设计系统

`web/src/styles/tokens.css` 定义品牌红、背景、表面、边框、文字、风险状态、阴影和过渡变量；`globals.css` 提供全局 reset、滚动和响应行为。

页面切换采用约 150–180 ms 的淡入与轻微纵向移动。全局遵循 `prefers-reduced-motion`，用户要求减少动画时将动画压缩至接近零。

## 两套布局

### ProductPageLayout

用于 Overview、Sessions、Tool Calls、Policies、Risk Intelligence、Evidence、Reports、Core、Assistant 和 Settings。

- 顶栏 58 px；
- 主导航默认 188 px，折叠 68 px；
- 内容最大宽 1560 px；
- workspace/immersive 模式取消常规标题与内边距；
- Assistant 使用沉浸模式。

### AuditWorkspaceLayout

Runtime 使用五区域网格：顶栏、主导航、会话栏、中间工作区、右 Inspector，加中下证据区。

| 轨道 | 展开 | 折叠 |
| --- | ---: | ---: |
| 主导航 | 188 px | 68 px |
| 会话栏 | 276 px | 54 px |
| Inspector | 356 px | 54 px |
| 中央 | 最小 540 px | 自适应 |
| 证据区 | 220 px 高 | — |

全局 min-width 1180 px，当前不是手机适配；窄屏主要保持桌面结构并出现更紧凑布局。

## Assistant 三栏

- 会话栏 242 px，折叠 48 px；
- 中央对话自适应；
- Agent 工作区 318 px，折叠 48 px；
- 折叠过渡约 280 ms；
- 1380 px 以下缩窄并隐藏部分品牌文字。

## 折叠状态

`uiStore` 把以下布尔值写入 localStorage：

```text
traceshield.ui.navigation-collapsed
traceshield.ui.runtime-sessions-collapsed
traceshield.ui.runtime-inspector-collapsed
traceshield.ui.assistant-sessions-collapsed
traceshield.ui.assistant-inspector-collapsed
traceshield.ui.settings-navigation-collapsed
```

存储不可用不影响当前交互，但刷新后不保留。

## 构建与开发

```bash
npm --prefix web ci
npm --prefix web run typecheck
npm --prefix web run build
npm --prefix web run smoke
```

开发：

```bash
npm --prefix web run dev
```

生产预览：

```bash
npm --prefix web run preview -- --host 0.0.0.0 --port 5173
```

Vite 环境变量在 build 时写入，修改 `.env` 后必须重新构建。

## 开发注意

- Runtime Store 是跨页面主数据源，避免页面重复发同一查询。
- Core DTO 通过 `src/api/` 转为 UI 类型，不直接在组件里解析 snake_case。
- 演示数据必须在页面明确标识，不能和真实 Core 指标合并后当作实时值。
- 新的宽度与折叠动画应同时验证 1180、1250、1380、1680 px。
- 组件目前缺少正式单元/E2E 测试，关键交互修改要手工回归并逐步补测试。

