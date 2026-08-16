# 模型配置

## 默认值

| 环境变量 | 默认值 | 合法范围/说明 |
| --- | --- | --- |
| `TRACESHIELD_ASSISTANT_HOST` | `127.0.0.1` | 非空 |
| `TRACESHIELD_ASSISTANT_PORT` | `8790` | 1–65535 |
| `TRACESHIELD_ASSISTANT_ALLOWED_ORIGINS` | localhost/127.0.0.1:5173 | CSV；可使用 `*`，不推荐公网 |
| `TRACESHIELD_ASSISTANT_BASE_URL` | `https://api.deepseek.com` | 绝对 http(s) URL |
| `TRACESHIELD_ASSISTANT_MODEL` | `deepseek-v4-flash` | 非空模型名 |
| `TRACESHIELD_ASSISTANT_TIMEOUT_MS` | `60000` | 1000–300000 |
| `TRACESHIELD_ASSISTANT_MAX_TOKENS` | `1200` | 1–32768 |
| `TRACESHIELD_ASSISTANT_TEMPERATURE` | `0.2` | 0–2 |
| `TRACESHIELD_ASSISTANT_THINKING_ENABLED` | `false` | true/false |
| `TRACESHIELD_ASSISTANT_MAX_BODY_BYTES` | `262144` | 1024–4 MiB |
| `TRACESHIELD_ASSISTANT_MAX_MESSAGE_RUNES` | `12000` | 1–100000 |
| `TRACESHIELD_ASSISTANT_MAX_HISTORY_ITEMS` | `40` | 0–200 |
| `TRACESHIELD_ASSISTANT_MAX_HISTORY_RUNES` | `48000` | 0–500000 |
| `TRACESHIELD_ASSISTANT_MAX_CONTEXT_BYTES` | `32768` | 0–512 KiB |

## API key

优先级：

1. `TRACESHIELD_ASSISTANT_API_KEY`
2. `DEEPSEEK_API_KEY`
3. `TRACESHIELD_ASSISTANT_API_KEY_FILE`

推荐文件方式：

```bash
export TRACESHIELD_ASSISTANT_API_KEY_FILE=/absolute/path/outside-git/model-key
```

key 文件会 trim 空白；文件不存在或内容为空时启动失败。服务不会在健康响应或正常日志中输出 key。

Base URL 与模型也支持 `DEEPSEEK_BASE_URL/DEEPSEEK_MODEL` 别名，但 TraceShield 前缀优先。

## 快速演示配置

当前没有单独的“fast mode”布尔开关。快速体验由模型选择和 thinking 共同决定：

```dotenv
TRACESHIELD_ASSISTANT_MODEL=deepseek-v4-flash
TRACESHIELD_ASSISTANT_THINKING_ENABLED=false
TRACESHIELD_ASSISTANT_MAX_TOKENS=1200
TRACESHIELD_ASSISTANT_TEMPERATURE=0.2
```

thinking disabled 时模型适配器才传 temperature。thinking enabled 时应根据供应商语义重新评估延迟和参数兼容。

## CORS

Assistant 默认只允许本机 Vite 来源。正常架构由 Core 代理，因此 Assistant 不需要面向局域网浏览器开放。即使 Core 当前返回 `*`，也不意味着 Assistant 要同步扩大来源。

## systemd

仓库 unit 使用本机绝对路径，需要在新机器按实际 checkout 目录修改。构建并安装：

```bash
cd assistant-eino
go test ./...
go build -o bin/traceshield-assistant ./cmd/server

systemctl --user link \
  /absolute/path/to/TraceShield/deploy/systemd/traceshield-assistant.service
systemctl --user daemon-reload
systemctl --user enable --now traceshield-assistant
```

检查：

```bash
systemctl --user status traceshield-assistant --no-pager
journalctl --user -u traceshield-assistant -n 100 --no-pager
curl -s http://127.0.0.1:8790/health | jq
```

## 超时协调

Assistant、Core proxy 和反向代理的超时应满足：

```text
模型合理响应时长
  < Assistant timeout
  <= Core assistant timeout
  < 外层代理总超时
```

当前 Assistant 与 Core 都默认 60 秒，Core 的计时覆盖整条流。需要长回答时同时调整，并限制输出 token，避免仅拉长超时掩盖模型异常。

## 配置验证

`config.Load()` 对数值范围、布尔值、URL 和 allowed origin 逐项验证。无效配置会阻止进程启动，而不是默默回退。更新服务环境后必须重启，并同时验证 direct health、Core proxy health 和一次实际 chat stream。

