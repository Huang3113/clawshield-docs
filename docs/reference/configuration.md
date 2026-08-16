# 环境变量参考

## Core

| 变量 | 默认 | 必需 | 说明 |
| --- | --- | --- | --- |
| `TRACESHIELD_DATABASE_URL` | 无 | 是 | PostgreSQL URL |
| `TRACESHIELD_CORE_PORT` | `8787` | 否 | 1–65535 |
| `TRACESHIELD_SAVE_RAW_PAYLOAD` | `false` | 否 | 保存 trace event 原 payload |
| `TRACESHIELD_SAVE_RAW_PARAMS` | `false` | 否 | 保存同步工具原参数 |
| `TRACESHIELD_SAVE_RAW_RESULT` | `false` | 否 | 保存工具原结果 |
| `TRACESHIELD_ENGINE_MODE` | `shadow` | 否 | `legacy/shadow/enforce` |
| `TRACESHIELD_METHOD_PROFILE` | `balanced` | 否 | Method profile 名 |
| `TRACESHIELD_METHOD_PROFILE_VERSION` | `balanced-v1` | 否 | profile 版本 |
| `TRACESHIELD_METHOD_VERSION` | `phase0-baseline` | 否 | 记录的方法版本 |
| `TRACESHIELD_METHOD_TIMEOUT_MS` | `120` | 否 | 正整数，同步 Method timeout |
| `TRACESHIELD_METHOD_QUEUE_LIMIT` | `256` | 否 | shadow 内存队列上限 |
| `TRACESHIELD_METHOD_PYTHON` | `./method-engine/.venv/bin/python` | 否 | Python executable |
| `TRACESHIELD_METHOD_WORKER` | `./method-engine/python/traceshield_method/worker.py` | 否 | Worker 文件 |
| `TRACESHIELD_ASSISTANT_BASE_URL` | `http://127.0.0.1:8790` | 否 | Eino 地址 |
| `TRACESHIELD_ASSISTANT_TIMEOUT_MS` | `60000` | 否 | Assistant 代理总 timeout |

Core boolean 只接受 `true/false`。配置由 Zod 在启动时校验。

## Web

| 变量 | 默认行为 | 说明 |
| --- | --- | --- |
| `VITE_TRACESHIELD_CORE_BASE_URL` | HTTP: 当前 hostname:8787；HTTPS: 同源 | build-time Core URL |
| `VITE_USE_MOCK_DATA` | 非 `false` 都使用 mock | 只有精确字符串 `false` 接真实 Core |

修改后重新 `npm --prefix web run build`。

## OpenClaw 插件

| 变量 | 默认 |
| --- | --- |
| `TRACESHIELD_PLUGIN_ID` | `traceshield-security-plugin` |
| `TRACESHIELD_GATEWAY_ID` | 未设置 |
| `TRACESHIELD_CORE_BASE_URL` | `http://127.0.0.1:8787` |
| `TRACESHIELD_AUDIT_TIMEOUT_MS` | `400` |
| `TRACESHIELD_EVENT_FLUSH_TIMEOUT_MS` | `1000` |
| `TRACESHIELD_EVENT_FLUSH_INTERVAL_MS` | `2000` |
| `TRACESHIELD_MODE` | `development` |
| `TRACESHIELD_FALLBACK_ENABLED` | `true` |
| `TRACESHIELD_DEBUG_FULL_PAYLOAD` | `false` |
| `TRACESHIELD_DISK_QUEUE_DIR` | `.traceshield/events` |
| `TRACESHIELD_MEMORY_QUEUE_MAX_EVENTS` | `1000` |
| `TRACESHIELD_LOCAL_ALLOW_TOOL_KINDS` | `file_read` |
| `TRACESHIELD_HIGH_RISK_TOOL_KINDS` | 7 类高风险种类 |

OpenClaw plugin config 优先于环境变量；列表用逗号。插件布尔 loader 只识别小写 `true/false`。

## Eino Assistant

| 变量 | 默认 |
| --- | --- |
| `TRACESHIELD_ASSISTANT_HOST` | `127.0.0.1` |
| `TRACESHIELD_ASSISTANT_PORT` | `8790` |
| `TRACESHIELD_ASSISTANT_ALLOWED_ORIGINS` | localhost 与 127.0.0.1:5173 |
| `TRACESHIELD_ASSISTANT_API_KEY` | 空 |
| `TRACESHIELD_ASSISTANT_API_KEY_FILE` | 空 |
| `TRACESHIELD_ASSISTANT_BASE_URL` | `https://api.deepseek.com` |
| `TRACESHIELD_ASSISTANT_MODEL` | `deepseek-v4-flash` |
| `TRACESHIELD_ASSISTANT_TIMEOUT_MS` | `60000` |
| `TRACESHIELD_ASSISTANT_MAX_TOKENS` | `1200` |
| `TRACESHIELD_ASSISTANT_TEMPERATURE` | `0.2` |
| `TRACESHIELD_ASSISTANT_THINKING_ENABLED` | `false` |
| `TRACESHIELD_ASSISTANT_MAX_BODY_BYTES` | `262144` |
| `TRACESHIELD_ASSISTANT_MAX_MESSAGE_RUNES` | `12000` |
| `TRACESHIELD_ASSISTANT_MAX_HISTORY_ITEMS` | `40` |
| `TRACESHIELD_ASSISTANT_MAX_HISTORY_RUNES` | `48000` |
| `TRACESHIELD_ASSISTANT_MAX_CONTEXT_BYTES` | `32768` |

兼容别名：`DEEPSEEK_API_KEY/BASE_URL/MODEL`，TraceShield 前缀优先。

## 公共 Web 网关

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `TRACESHIELD_PUBLIC_GATEWAY_HOST` | `127.0.0.1` | 网关监听 |
| `TRACESHIELD_PUBLIC_GATEWAY_PORT` | `5180` | 网关端口 |
| `TRACESHIELD_WEB_ORIGIN` | `http://127.0.0.1:5173` | Web 上游 |
| `TRACESHIELD_CORE_ORIGIN` | `http://127.0.0.1:8787` | Core 上游 |

公共网关允许 GET/HEAD/OPTIONS，唯一允许的 Core 写请求是 Assistant chat stream；Runtime stream 返回 204。

## 变量命名冲突

Core 与 Assistant 都使用 `TRACESHIELD_ASSISTANT_TIMEOUT_MS`，含义不同但默认相同：

- Core 进程中：等待 Assistant 上游的代理总 timeout；
- Assistant 进程中：模型请求 context timeout。

systemd 分服务设置时互不影响；同一 shell 启动多个进程时要明确作用范围。

## 秘密管理

不能提交：

- API key 文件；
- `.env`；
- OpenClaw token/config 全文；
- 生产数据库 URL；
- public tunnel credentials；
- raw 审计导出。

环境变量参考只写变量名与假值，不复制运行机器实际配置。

