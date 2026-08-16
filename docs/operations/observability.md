# 监控与健康检查

## 分层健康

不要用单一绿色点代表整个系统。

| 层 | 检查 | 成功条件 |
| --- | --- | --- |
| PostgreSQL | `docker compose ps postgres` | healthy |
| Core/DB | `/v1/health` | `.ok and .db_connected` |
| Method | `/v1/method/status` | mode 符合预期，available=true（非 legacy） |
| Assistant 配置 | `/v1/assistant/health` | ok/configured=true |
| Assistant 上游 | 最小真实 chat stream | start+delta+done |
| Web | `/runtime` HTTP + 资源 | 页面可加载 |
| OpenClaw | `openclaw health` | Gateway 健康 |
| 插件 | inspect + Gateway 日志 | loaded + registered |
| 事件队列 | 内存 status + 磁盘文件数 | 无持续增长 |

## 一次性检查脚本思路

```bash
set -euo pipefail

docker compose ps postgres
curl -s http://127.0.0.1:8787/v1/health \
  | jq -e '.ok == true and .db_connected == true'
curl -s http://127.0.0.1:8787/v1/method/status | jq -e '.available == true'
curl -s http://127.0.0.1:8787/v1/assistant/health \
  | jq -e '.ok == true and .configured == true'
curl -fsSI http://127.0.0.1:5173/runtime >/dev/null
openclaw health
```

不要把包含 key/token 的配置输出进 CI log。

## 日志

```bash
journalctl --user -u traceshield-assistant -n 150 --no-pager
journalctl --user -u traceshield-core -n 150 --no-pager
journalctl --user -u traceshield-web -n 150 --no-pager
journalctl --user -u openclaw-gateway -n 200 --no-pager
docker compose logs --tail 150 postgres
```

实时跟随单服务：

```bash
journalctl --user -u traceshield-core -f
```

## 指标

Dashboard 提供：

- 24h/total tool calls；
- blocked；
- high/critical；
- rule hits；
- demo seed calls；
- latest call；
- metric window。

它不是完整运维 metrics。当前缺少审计延迟、HTTP 错误率、fallback 次数、flush 成功率、磁盘队列大小、Method queue wait 和 Assistant token/latency 等时间序列。

## Method 报告

```bash
npm --prefix core run method:report
```

汇总 evaluation 状态、超时/错误、延迟分位、diff、violations、完整性和 tool kinds。该命令写文件，应在受控工作树或 CI artifact 中执行。

## SSE 诊断

```bash
timeout 10 curl --no-buffer --silent \
  http://127.0.0.1:8787/v1/stream/audit-events
```

应看到 connected 和约每两秒一次注释。持续反复建立连接才说明代理/CORS/网络问题；HTTPS UI 显示 polling 是当前设计。

## 队列

`traceshield_status` 的 queued events 只统计内存，并且 status 自身会先入队。磁盘队列单独检查：

```bash
QUEUE_DIR=/absolute/path/to/traceshield-events
find "$QUEUE_DIR" -maxdepth 1 -type f -name '*.json' | wc -l
```

持续增长通常表示 Core/DB 故障或 events endpoint 失败。不要通过删除文件让数字归零。

## 告警建议

生产化至少告警：

- Core health ok=false；
- Method unavailable/queue 接近 256；
- 插件 fallback 连续触发；
- 磁盘队列增长或占满；
- 审计 P95 接近插件 timeout；
- Assistant 429/5xx/timeout；
- PostgreSQL 容量、连接与备份失败；
- 24h 完全无事件但 Gateway 有活跃请求。

