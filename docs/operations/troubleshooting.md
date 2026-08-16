# 故障排查

## 先做分层检查

```bash
ss -ltnp | rg ':(5173|5432|8787|8790|18789|18790)\b'
docker compose ps postgres
systemctl --user is-active \
  traceshield-assistant traceshield-core traceshield-web openclaw-gateway
curl -s http://127.0.0.1:8787/v1/health | jq
curl -s http://127.0.0.1:8787/v1/method/status | jq
```

进程 active、端口 listening、HTTP 200 与业务健康是四个不同层次。

## Core active 但 health false

**症状**：systemd active，`/v1/health` 返回 200，但 `ok=false/db_connected=false`。

**原因**：PostgreSQL 未启动或连接配置不一致。Compose 当前没有 restart policy，虚拟机重启后常见。

```bash
docker compose up -d --wait postgres
npm --prefix core run db:migrate
systemctl --user restart traceshield-core
curl -s http://127.0.0.1:8787/v1/health | jq -e '.ok and .db_connected'
```

## OpenClaw 显示“审计超时 400ms”

这通常不是错误。`traceshield_status` 的 Audit timeout 表示配置的等待上限，不是最近一次发生过超时。

真正超时/fallback 看日志：

```bash
journalctl --user -u openclaw-gateway --no-pager \
  | rg 'audit request failed|fallback policy'
```

如果频繁触发，检查 Core health、数据库、Audit API 实际延迟和虚拟机负载；不要直接提高 timeout 掩盖故障。

## status 队列为 1

status 工具自己先经过 before_tool_call 并入内存队列，随后读取 queue size；flush 每两秒执行，所以 1 很常见。它不统计磁盘文件。

## Runtime 四张卡片全零

Core 指标是过去 24 小时窗口。先看：

```bash
curl -s http://127.0.0.1:8787/v1/dashboard/runtime-status | jq
```

- 24h=0、total>0：历史数据超过窗口，UI 应显示历史累计。
- total=0：数据库确实没有调用，可运行 `seed:demo`。
- 接口有值、页面无值：检查 `VITE_USE_MOCK_DATA=false`、Core URL 和是否重新 build。

## 实时流一会在线一会离线

```bash
timeout 10 curl --no-buffer --silent \
  http://127.0.0.1:8787/v1/stream/audit-events
```

确认连接只有一条且有 2 秒心跳、CORS 头存在、中间代理不缓冲。HTTP UI 使用 SSE；HTTPS UI 按设计显示 polling，不是 SSE 故障。

## 插件没有加载

```bash
npm --prefix openclaw-plugin run build
openclaw config validate
openclaw plugins inspect traceshield-security-plugin
openclaw plugins doctor
journalctl --user -u openclaw-gateway -n 150 --no-pager
```

检查 dist、load path 绝对地址、enabled、manifest schema 和 Gateway 是否在构建后重启。

## 插件走 fallback，但 Core URL 正确

status 只显示配置 URL。直接检查：

```bash
curl -s http://127.0.0.1:8787/v1/health | jq
docker compose ps postgres
journalctl --user -u traceshield-core -n 150 --no-pager
```

Core 监听但 DB 断开时同步 audit 仍会失败。

## 事件没有进入控制台

```bash
QUEUE_DIR=/absolute/path/to/traceshield-events
find "$QUEUE_DIR" -maxdepth 1 -type f -name '*.json' | wc -l
journalctl --user -u openclaw-gateway -n 200 --no-pager \
  | rg 'flush failed|persisted to disk|requeued'
curl -s http://127.0.0.1:8787/v1/health | jq
```

恢复 Core 后等待 flush；不要删除队列。公共 HTTPS 页面还需等 10 秒轮询或刷新选中 Run。

## Assistant 显示不可用

```bash
curl -s http://127.0.0.1:8790/health | jq
curl -s http://127.0.0.1:8787/v1/assistant/health | jq
systemctl --user status traceshield-assistant --no-pager
journalctl --user -u traceshield-assistant -n 150 --no-pager
```

configured=true 仍可能是模型网络、额度、模型名或限流问题；发最小 SSE 请求验证真实上游。服务会在建流前安全重试一次，但流开始后不会重试。

## Assistant 回复慢

检查：

- model 是否为低延迟配置；
- thinking 是否 false；
- history/context 是否过大；
- 上游网络/429/5xx；
- 是否在 60 秒总 timeout 附近；
- 浏览器是否很久才收到 start，还是 delta 间隔慢。

当前没有独立 fast-mode 开关，速度由模型与 thinking 等参数决定。

## 风险图为空或仍是线性

```bash
curl -s http://127.0.0.1:8787/v1/runs/<run-id>/risk-graph | jq
curl -s http://127.0.0.1:8787/v1/method/status | jq
curl -s 'http://127.0.0.1:8787/v1/method/evaluations?limit=20' | jq
```

没有成功 Method snapshot 时 `/risk-graph` 正常回退 legacy linear。等待 shadow 队列，检查 evaluation error/timeout 和 step completeness。

## 策略开关后决策不变

这是当前边界，不是同步延迟。Policy Center 使用 localStorage；Core 没有 policy API，基础规则硬编码。只有实现动态策略引擎后开关才会改变真实审计。

## 宿主无法访问虚拟机

```bash
ip -4 -brief address
ss -ltnp | rg ':(5173|18789|18790)\b'
```

确认 VM 网络模式、宿主到 VM IP 路由、防火墙、Web/Gateway bind。OpenClaw 页面能开但无法连接通常是 token、allowed origin 或设备身份问题，不是 TraceShield Core。

## 端口冲突

不要同时运行同端口 systemd 服务与开发进程：

```bash
ss -ltnp | rg ':(5173|8787|8790)\b'
systemctl --user stop traceshield-web   # 然后才开 Vite dev
```

## 收集诊断包

只收集非敏感信息：版本、health、unit status、最近脱敏日志、端口和队列数量。不要包含 `.env`、OpenClaw config 全文、key 文件、Gateway token 或原始审计 payload。

