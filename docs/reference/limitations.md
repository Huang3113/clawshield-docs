# 已知边界

本页记录当前版本的明确边界，避免开发和演示时过度承诺。

## Core 与安全

1. Core 没有身份认证、用户/租户隔离或角色权限。
2. CORS 当前为 `*`，没有内置 TLS。
3. Core 监听所有接口，依赖网络/代理限制访问。
4. 没有 API 速率限制和配额。
5. `policies.enabled/priority/config` 不参与实际决策。
6. 没有 policies CRUD 或真实开关 API。
7. ASK 只签发描述，没有 Core approval resolution API。
8. `modified_params` 当前始终为空。
9. Core 信任插件脱敏，没有通用二次脱敏。
10. raw 关闭后仍保存 preview/summary/resource hint。
11. event batch 整体事务，一条 DB 错误回滚全批。
12. 极端 after-before 乱序可能留下 unknown tool status。
13. health 只检查 PostgreSQL，不检查 Method/Assistant。
14. Run 只有 agent_end 自动完成，无超时/abandoned 定时任务。
15. 列表主要是固定上限，没有游标分页。
16. 仅支持 PostgreSQL。

## Method

1. 默认 shadow，不改变当次工具决定。
2. shadow queue 是内存队列，重启丢失未处理项。
3. pending observation map 在内存中。
4. live AuditRequest 不能传 `authorized_risky_calls`。
5. recent message hashes 尚未进入 live IntentFrame。
6. 数据库预留 intent/semantic 字段尚未写入。
7. decision 的 method evaluation ID 尚未完整回写/建外键。
8. runtime profile 当前只贯通项目默认 profile。
9. method version 记录存在，但 Worker 不按该值切换实现。
10. enforce 异常回退 legacy 时编排器不暴露详细错误。
11. Web 尚未监听 Method 专用 SSE 生命周期。

## SSE 与多实例

1. 客户端集合只在单 Core 进程。
2. 无持久缓冲、断线补发或 Last-Event-ID 恢复。
3. 多 Core 实例不共享通知。
4. Web 预留 metric_update，但 Core 当前不发布。
5. Core heartbeat 是注释，不是命名 heartbeat event。
6. HTTPS 入口使用 10 秒轮询，不是实时 SSE。
7. HTTPS 轮询不刷新当前图、证据和对话摘要。

## 插件

1. `fallback_enabled=false` 当前会 fail-open，部署必须保持 true。
2. 生产代码未预填 local allow cache，普通 read fallback 通常 ASK。
3. WARN 不保证 OpenClaw UI 可见。
4. status 不探测 Core/DB/Method，不统计磁盘队列。
5. 内存队列超限静默淘汰最老事件。
6. 正常停止没有 final flush。
7. 磁盘队列没有容量、TTL 和坏文件隔离。
8. 相对队列路径受 Gateway WorkingDirectory 影响。
9. manifest 尚未声明 loader 支持的 plugin_id/gateway_id 配置字段。
10. resource hint 不额外经过通用文本脱敏。

## Assistant

1. 只使用 Eino ChatModel，不是工具型安全 Agent。
2. 没有工具、RAG、服务端记忆或人工审批。
3. 不参与 ALLOW/WARN/ASK/BLOCK。
4. Web case 与 context 当前是硬编码展示摘要。
5. health 不实际探测模型上游。
6. 输出目前不解析 Markdown。
7. Core 和 Assistant 同为 60 秒，长流可能被整体中止。
8. 建流前只重试一次，建流后不重试。
9. Core Assistant API 也没有认证/用户隔离/限流。
10. Web/Core/Assistant 的长度限制未完全统一。

## Web

1. Runtime 是最完整真实页；多个其他页面仍是静态或混合演示。
2. Policy Center 开关只写 localStorage。
3. Overview 含展示下限、固定倍率与常量数据。
4. Risk Intelligence/Evidence/Reports/Settings 大量为展示状态。
5. Tool call latency 当前映射为 0，不是真实测量。
6. Core Status 部分字段为展示值。
7. 首次 Core 初始化失败后无自动重试，需刷新。
8. 全局 min-width 1180，不是移动端适配。
9. 没有正式 i18n。
10. 没有 Vue 组件单测和浏览器 E2E。
11. `graph_source` 当前未在 Store/UI 展示。
12. SSE connected 被近似用于 plugin last seen，语义不严格。

## 数据与运维

1. 没有保留周期、TTL、归档或自动清理。
2. 没有 migration history 表。
3. Compose 是单机开发形态，映射 5432。
4. Compose 未配置 PostgreSQL restart policy。
5. 仓库 systemd unit 硬编码某个 checkout 绝对路径。
6. Core/Web/Assistant user unit active 不代表数据库已运行。
7. 没有统一 metrics/exporter、集中日志和正式告警。
8. 没有自动备份与恢复演练。

## 演示时必须如实说明

- Method 当前默认是旁路评估。
- 策略开关是演示交互。
- Assistant 是只读对话边界，不执行工具。
- Overview 等页面含展示数据。
- 公网 HTTPS Runtime 是轮询更新。
- Baseline 是显式禁用插件的对照 profile，不必声称软件从未安装。

边界不是永久设计承诺。实现对应能力时，应在同一变更中删除此条并更新架构、API、测试和演示文案。

