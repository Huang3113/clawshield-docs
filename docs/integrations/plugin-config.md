# 插件配置

## 优先级

```text
OpenClaw plugin config
  > TRACESHIELD_* 环境变量
  > 代码默认值
```

OpenClaw 配置值可通过 `api.pluginConfig` 传入；配置源也支持 `get(key)` 风格。

## 全部字段

| 字段 | 环境变量 | 默认 | 说明 |
| --- | --- | --- | --- |
| `plugin_id` | `TRACESHIELD_PLUGIN_ID` | `traceshield-security-plugin` | 事件来源 ID |
| `gateway_id` | `TRACESHIELD_GATEWAY_ID` | 无 | 多 Gateway 来源区分 |
| `core_base_url` | `TRACESHIELD_CORE_BASE_URL` | `http://127.0.0.1:8787` | Core 地址 |
| `audit_timeout_ms` | `TRACESHIELD_AUDIT_TIMEOUT_MS` | `400` | 同步 HTTP 等待上限 |
| `event_flush_timeout_ms` | `TRACESHIELD_EVENT_FLUSH_TIMEOUT_MS` | `1000` | 批次 HTTP 超时 |
| `event_flush_interval_ms` | `TRACESHIELD_EVENT_FLUSH_INTERVAL_MS` | `2000` | flush 间隔 |
| `mode` | `TRACESHIELD_MODE` | `development` | development/demo/production |
| `fallback_enabled` | `TRACESHIELD_FALLBACK_ENABLED` | `true` | Core 失败时启用保守规则 |
| `debug_full_payload` | `TRACESHIELD_DEBUG_FULL_PAYLOAD` | `false` | 本地调试完整载荷 |
| `disk_queue_dir` | `TRACESHIELD_DISK_QUEUE_DIR` | `.traceshield/events` | 补传目录 |
| `memory_queue_max_events` | `TRACESHIELD_MEMORY_QUEUE_MAX_EVENTS` | `1000` | 内存队列上限 |
| `local_allow_tool_kinds` | `TRACESHIELD_LOCAL_ALLOW_TOOL_KINDS` | `file_read` | fallback 可由缓存放行种类 |
| `high_risk_tool_kinds` | `TRACESHIELD_HIGH_RISK_TOOL_KINDS` | 见下 | fallback fail-closed 种类 |

默认 high risk：

```text
shell_exec,file_write,file_delete,network_request,message_send,plugin_install,state_change
```

## OpenClaw 配置示例

```bash
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
  'plugins.entries["traceshield-security-plugin"].config.mode' \
  demo
```

`plugin_id` 和 `gateway_id` 已被 TypeScript loader 支持，但当前 manifest 的 `additionalProperties:false` schema 未声明这两个字段。需要它们时优先使用环境变量，或先同步扩展 manifest schema。

## 环境变量格式

列表用英文逗号：

```bash
export TRACESHIELD_LOCAL_ALLOW_TOOL_KINDS=file_read,read_only
export TRACESHIELD_HIGH_RISK_TOOL_KINDS=shell_exec,file_write,file_delete
```

布尔 loader 只识别小写 `true` 和 `false`；`1`、`yes` 等不会按 true 解析。mode 只识别 development/demo/production。

## 关键安全项

### fallback_enabled

必须保持 true。当前源码中如果显式设为 false，Core 请求失败后 Hook 返回 `undefined`，相当于 fail-open。

### debug_full_payload

保持 false。production mode 只改变默认推导，并不会强制拒绝显式 true。生产治理应在配置发布与启动检查中禁止它。

### disk_queue_dir

相对路径按 Gateway 工作目录解析，前台运行与 systemd 可能不同。推荐使用不可提交、权限受控的绝对路径：

```bash
export TRACESHIELD_DISK_QUEUE_DIR=/var/lib/traceshield/openclaw-events
```

确保 Gateway 用户可写，运维监控也指向同一目录。

### core_base_url

插件和 Core 同机时使用 loopback，不要硬编码虚拟机 DHCP 地址：

```text
http://127.0.0.1:8787
```

## 变更生效

OpenClaw 配置修改或插件代码重建后，重启对应 Gateway：

```bash
systemctl --user restart openclaw-gateway
```

验证配置与真实注册：

```bash
openclaw config validate
openclaw plugins inspect traceshield-security-plugin
journalctl --user -u openclaw-gateway -n 100 --no-pager | rg TraceShield
```

