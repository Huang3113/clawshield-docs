# systemd 持久运行

仓库提供三个用户级 unit：Assistant、Core、Web。OpenClaw Gateway unit 由 OpenClaw CLI 安装。

## 先构建

```bash
npm --prefix core run build
npm --prefix web run build
npm --prefix openclaw-plugin run build

cd assistant-eino
go build -o bin/traceshield-assistant ./cmd/server
cd ..
```

## 路径检查

仓库 unit 当前包含创建环境时的绝对 `WorkingDirectory` 和 `ExecStart`。在新机器必须先复制或修改为实际 checkout 路径，再 link。不要直接假设文件可移植。

```bash
rg -n 'WorkingDirectory|ExecStart|API_KEY_FILE' deploy/systemd/*.service
```

## 链接与启用

```bash
REPO=/absolute/path/to/TraceShield

systemctl --user link "$REPO/deploy/systemd/traceshield-assistant.service"
systemctl --user link "$REPO/deploy/systemd/traceshield-core.service"
systemctl --user link "$REPO/deploy/systemd/traceshield-web.service"
systemctl --user daemon-reload
systemctl --user enable --now \
  traceshield-assistant \
  traceshield-core \
  traceshield-web
```

用户未登录也运行：

```bash
sudo loginctl enable-linger "$USER"
loginctl show-user "$USER" -p Linger
```

预期 `Linger=yes`。

## 依赖关系

| 服务 | After/Wants | Restart |
| --- | --- | --- |
| Assistant | network-online | on-failure，3 秒 |
| Core | network-online + Assistant | on-failure，3 秒 |
| Web | Core | on-failure，3 秒 |
| OpenClaw Gateway | network | 由 OpenClaw unit 管理，当前通常 always |

这些 systemd 依赖只影响启动顺序，不代表依赖健康。Core 等待 Assistant 不代表 Assistant 模型可用；Web 等待 Core 不代表数据库已连接。

## PostgreSQL 的关键差异

Compose 当前没有 `restart:`，Core unit 也没有 Docker/PostgreSQL systemd 依赖。虚拟机重启后可能出现：

```text
Core unit active
PostgreSQL 未运行
/v1/health HTTP 200 但 ok=false
Method unavailable
插件审计进入 fallback
```

每次启动或录制前执行：

```bash
cd /absolute/path/to/TraceShield
docker compose up -d --wait postgres
npm --prefix core run db:migrate
systemctl --user restart traceshield-core traceshield-web
```

如果需要无人值守恢复，应单独设计受管理的 Compose systemd unit 或容器 restart policy，并验证 Docker 启动顺序；当前仓库没有完成这一层。

## OpenClaw unit

```bash
openclaw gateway install --port 18789
systemctl --user enable --now openclaw-gateway.service
```

插件源码变更：

```bash
npm --prefix openclaw-plugin run build
systemctl --user restart openclaw-gateway
```

OpenClaw 升级后建议重新 `gateway install --force`，因为 unit 可能包含安装目录的绝对入口。

## 查看状态

```bash
systemctl --user is-active \
  traceshield-assistant \
  traceshield-core \
  traceshield-web \
  openclaw-gateway

systemctl --user status traceshield-core --no-pager
journalctl --user -u traceshield-core -n 150 --no-pager
```

## 正确重启顺序

```bash
docker compose up -d --wait postgres
npm --prefix core run db:migrate
systemctl --user restart \
  traceshield-assistant \
  traceshield-core \
  traceshield-web \
  openclaw-gateway
```

然后按健康端点逐层验收，不只看 unit active。

## 发布更新

```bash
git pull --ff-only
npm --prefix core ci
npm --prefix web ci
npm --prefix openclaw-plugin ci

npm --prefix core run build
npm --prefix web run build
npm --prefix openclaw-plugin run build
(cd assistant-eino && go build -o bin/traceshield-assistant ./cmd/server)

npm --prefix core run db:migrate
systemctl --user restart \
  traceshield-assistant traceshield-core traceshield-web openclaw-gateway
```

生产部署还应在重启前完成全测试与备份，并支持回滚到上一构建。

