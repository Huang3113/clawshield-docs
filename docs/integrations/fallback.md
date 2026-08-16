# 降级与补传

TraceShield 的降级目标是：中心审计短暂不可用时不把高风险工具变成默认放行，同时尽量保留稍后可补传的证据。

## 触发条件

- `audit_timeout_ms` 内 Core 没有响应；
- 连接拒绝、断网或 fetch 错误；
- Core 返回非 2xx；
- Core JSON 不是合法 AuditDecision；
- Core 进程在线但数据库错误导致审计失败。

## 本地决策

| 场景 | fallback 动作 |
| --- | --- |
| 敏感文件读取 | BLOCK/critical |
| high-risk tool kind | BLOCK/critical |
| 只读种类且命中本地 allow cache | ALLOW/low |
| 其他未知或未缓存工具 | ASK/medium，30 秒默认 BLOCK |

敏感识别还覆盖 `/etc/passwd` 和命令中明显读取 secret/token/private 目标的 `cat`/`curl`，比在线基础规则的局部表达更保守。

!!! warning "不要关闭 fallback"
    当前实现中 `fallback_enabled=false` 会在 Core 请求失败时直接返回 `undefined`，OpenClaw 继续执行工具。它是 fail-open，不是“禁用本地规则但仍阻断”。

## 在线与 fallback 差异

| 调用 | Core 在线基础策略 | Core 故障 fallback |
| --- | --- | --- |
| 敏感读取 | BLOCK | BLOCK |
| 破坏性 Shell | ASK | BLOCK |
| 普通 Shell | 通常 ALLOW | BLOCK |
| 外部网络 | ASK | BLOCK |
| 普通 file read | ALLOW | 当前通常 ASK |
| file write | 当前通常 ALLOW | BLOCK |
| unknown | WARN | ASK |

本地 allow cache 在生产代码中当前没有预填入口，因此普通 file read 故障时通常不会命中缓存。这是保守设计边界，不能承诺只读调用离线自动放行。

## fallback 证据

每次本地决策会把 `fallback_decision` 事件放入异步队列。Core 不可用时它也会等待补传。确认真实 fallback 应检查 Gateway 日志：

```bash
journalctl --user -u openclaw-gateway --no-pager \
  | rg 'audit request failed|fallback policy|fallback_decision'
```

`Fallback enabled: true` 只表示配置，不代表刚刚使用过。

## 内存队列

- 默认最多 1000 条；
- event ID 去重；
- 超限静默淘汰最老事件；
- Gateway 重启后内存内容丢失；
- 同进程重复插件注册共享队列。

## Flush Worker

默认每 2 秒执行，批次最多 100，HTTP timeout 1 秒：

1. 取内存事件；
2. 从磁盘补足批次；
3. 磁盘事件排在内存前；
4. Core 成功后删除已发磁盘文件；
5. 失败时把内存事件写到磁盘；
6. 磁盘也失败则重新入内存。

Core 以 event ID 去重，允许相同文件重复发送。

## 磁盘队列

默认 `.traceshield/events` 是相对路径，取决于 Gateway 工作目录。检查时应先确认配置和进程目录。

```bash
QUEUE_DIR=/absolute/path/to/traceshield-events
find "$QUEUE_DIR" -maxdepth 1 -type f -name '*.json' | wc -l
```

不要直接删除积压文件；它们是尚未回灌的审计证据。先恢复 Core，观察 flush 成功和文件数归零。

## 当前可靠性边界

- 正常停止只停计时器，没有 final flush；尚未落盘内存事件可能丢失。
- 磁盘队列没有容量与 TTL。
- 一个损坏 JSON 文件可能反复阻塞排序批次。
- status 只统计内存，不统计磁盘。
- 没有最近成功 flush 或最近 fallback 时间指标。

生产化方向包括优雅关闭 flush、坏文件隔离、磁盘容量上限、可观测指标与队列加密。

