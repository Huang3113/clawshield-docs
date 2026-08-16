# 双 Profile 对照演示

为了演示“同一 Agent 在无保护和有保护下的差异”，推荐同时运行两个隔离的 OpenClaw profile，而不是录制时反复开关同一实例。

## 设计

| 项目 | 受保护 | 基线 |
| --- | --- | --- |
| Profile | default | `traceshield-baseline` |
| 端口 | 18789 | 18790 |
| State/config | `~/.openclaw` | 独立 profile 目录 |
| Workspace | 同一安全演示 workspace | 同一安全演示 workspace |
| TraceShield | enabled | 显式 disabled |
| 会话/设备状态 | 独立 | 独立 |
| 模型与速度设置 | 保持相同 | 保持相同 |

这样 A/B 的主要变量是 TraceShield 插件。基线在全局注册表中仍可能“发现”插件，权威判断是 `enabled=false` 和启动日志无 TraceShield 注册，而不是要求 inspect 完全找不到。

## 1. 准备受控 workspace

两个 profile 只共享专门的演示目录，不要指向包含真实凭据或个人文件的工作区。

```bash
DEMO_WORKSPACE=/absolute/path/to/safe-demo-workspace
openclaw setup --workspace "$DEMO_WORKSPACE"
openclaw --profile traceshield-baseline setup --workspace "$DEMO_WORKSPACE"
```

用交互方式分别配置模型和 Gateway 凭据，避免 secret 进入命令历史：

```bash
openclaw configure
openclaw --profile traceshield-baseline configure
```

## 2. 受保护实例

```bash
PLUGIN_PATH=/absolute/path/to/TraceShield/openclaw-plugin

openclaw config set gateway.port 18789 --strict-json
openclaw config set gateway.bind lan
openclaw config set plugins.load.paths "[\"${PLUGIN_PATH}\"]" --strict-json
openclaw plugins enable traceshield-security-plugin
openclaw config set \
  'plugins.entries["traceshield-security-plugin"].config.core_base_url' \
  'http://127.0.0.1:8787'
openclaw config set \
  'plugins.entries["traceshield-security-plugin"].config.audit_timeout_ms' \
  400 --strict-json
openclaw config set \
  'plugins.entries["traceshield-security-plugin"].config.fallback_enabled' \
  true --strict-json
openclaw config set \
  'plugins.entries["traceshield-security-plugin"].config.mode' demo
openclaw config set tools.alsoAllow '["traceshield_status"]' --strict-json
```

## 3. 基线实例

```bash
openclaw --profile traceshield-baseline \
  config set gateway.port 18790 --strict-json
openclaw --profile traceshield-baseline \
  config set gateway.bind lan
openclaw --profile traceshield-baseline \
  plugins disable traceshield-security-plugin
openclaw --profile traceshield-baseline \
  config set tools.alsoAllow '[]' --strict-json
openclaw --profile traceshield-baseline \
  config unset plugins.load.paths
```

显式 disabled 是必要条件；只删除 load path 可能仍被全局插件发现机制加载。

## 4. 安装用户服务

```bash
openclaw gateway install --port 18789
openclaw --profile traceshield-baseline gateway install --port 18790

systemctl --user enable --now \
  openclaw-gateway.service \
  openclaw-gateway-traceshield-baseline.service
```

需要用户退出登录后仍启动：

```bash
sudo loginctl enable-linger "$USER"
loginctl show-user "$USER" -p Linger
```

## 5. 验证隔离

```bash
openclaw plugins inspect traceshield-security-plugin
openclaw --profile traceshield-baseline \
  plugins inspect traceshield-security-plugin

openclaw config get \
  'plugins.entries["traceshield-security-plugin"].enabled'
openclaw --profile traceshield-baseline config get \
  'plugins.entries["traceshield-security-plugin"].enabled'
```

预期：受保护 `loaded/true`，基线 `disabled/false`。

检查日志：

```bash
journalctl --user -u openclaw-gateway -n 120 --no-pager | rg TraceShield
journalctl --user -u openclaw-gateway-traceshield-baseline \
  -n 120 --no-pager | rg TraceShield || true
```

受保护实例应有插件注册，基线不应有。

## 6. 宿主机访问虚拟机

获取虚拟机 IP：

```bash
hostname -I
```

可访问：

```text
http://<VM_IP>:18789  受保护
http://<VM_IP>:18790  基线
```

两个 Gateway 都应使用 token 鉴权。普通 HTTP 的设备身份要求由 OpenClaw 版本和配置决定；更安全、稳定的方式是分别做 SSH 端口转发：

```bash
ssh -N \
  -L 18789:127.0.0.1:18789 \
  -L 18790:127.0.0.1:18790 \
  <vm-user>@<vm-ip>
```

宿主打开 localhost，避免为演示关闭设备身份校验。若确需直接 LAN HTTP，必须把它标为受控演示配置，并在录制后恢复安全设置。

## 7. A/B 演示规则

- 两边使用同一安全 workspace、同一模型和同一提示词。
- 演示文件只能是假数据。
- 先验证基线动作有可见但可恢复的副作用。
- 再在受保护实例执行相同输入，展示工具执行前 BLOCK/ASK。
- 控制台只会收到受保护实例的审计链；这正是对照的一部分。
- 台词应说“未启用安全插件的基线”，不要说完全没安装，因为 profile 可能仍能发现插件条目。

## 8. 重启后的状态

Profile 配置与 systemd enable 会持久化，但完整链路还依赖 PostgreSQL。当前 Compose 默认没有 restart policy，虚拟机重启后要先：

```bash
docker compose up -d --wait postgres
npm --prefix core run db:migrate
```

再验证两个 Gateway 与 Core；不能仅凭 user service active 判断审计链已恢复。

