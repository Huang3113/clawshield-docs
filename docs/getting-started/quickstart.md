# 五分钟启动

本页启动最小完整链路：PostgreSQL → Method 环境 → Core → Web。Eino Assistant 和 OpenClaw 插件作为后续可选步骤加入。

所有命令默认在仓库根目录执行。

## 1. 安装依赖

```bash
npm --prefix core ci
npm --prefix web ci
npm --prefix openclaw-plugin ci

python3 -m venv core/method-engine/.venv
core/method-engine/.venv/bin/python -m pip install \
  -r core/method-engine/requirements-runtime.lock \
  -r core/method-engine/requirements-dev.lock
```

`npm ci` 会严格使用锁文件，适合首次搭建和 CI。正在开发并需要调整依赖时才使用 `npm install`。

## 2. 启动 PostgreSQL

```bash
docker compose up -d --wait postgres
docker compose ps
```

Compose 中的用户、密码和数据库仅是本地开发默认值。生产部署必须使用独立凭据并限制 `5432` 的网络可达性。

## 3. 配置并迁移 Core

```bash
cp -n core/.env.example core/.env
npm --prefix core run db:migrate
npm --prefix core run db:check
```

默认 `.env.example` 已指向 Compose 数据库，并以 `shadow` 模式启动 Method Worker。

## 4. 启动 Core

```bash
npm --prefix core run dev
```

另开终端验证：

```bash
curl --fail --silent http://127.0.0.1:8787/v1/health
curl --fail --silent http://127.0.0.1:8787/v1/method/status
```

健康响应至少应满足：

```json
{
  "ok": true,
  "version": "0.1.0",
  "db_connected": true
}
```

Method 状态中，`mode=shadow` 且 `available=true` 表示 Worker 已建立；如果只需要基础策略，可把 `TRACESHIELD_ENGINE_MODE` 临时设为 `legacy`。

## 5. 启动真实 Web 控制台

创建或修改 `web/.env`：

```dotenv
VITE_TRACESHIELD_CORE_BASE_URL=http://127.0.0.1:8787
VITE_USE_MOCK_DATA=false
```

然后启动：

```bash
npm --prefix web run dev
```

打开 `http://127.0.0.1:5173/runtime`。如果浏览器在宿主机、服务在虚拟机中，将 Core 地址和访问地址中的主机改为虚拟机 IP；Vite 的开发脚本默认仅绑定 `127.0.0.1`，虚拟机跨机访问建议构建后使用 `npm --prefix web run preview -- --host 0.0.0.0 --port 5173`。

## 6. 写入一组真实演示数据

```bash
npm --prefix core run seed:demo
```

刷新 Runtime 页面，应看到最新会话、工具调用与非零指标。需要长风险链时：

```bash
npm --prefix core run seed:long-chain
```

长链脚本通过真实 API 写入 28 个工具步骤，并自行核验调用数、证据步骤、运行状态和 Method 路径。它不是静态前端 Mock。

## 7. 可选：启动 Eino Assistant

```bash
go -C assistant-eino test ./...
mkdir -p assistant-eino/bin
go -C assistant-eino build -o bin/traceshield-assistant ./cmd/server
```

把模型 key 放在一个不会提交的本地文件中，然后启动：

```bash
TRACESHIELD_ASSISTANT_API_KEY_FILE=/absolute/path/to/local-key \
assistant-eino/bin/traceshield-assistant
```

验证两级健康：

```bash
curl --fail --silent http://127.0.0.1:8790/health
curl --fail --silent http://127.0.0.1:8787/v1/assistant/health
```

浏览器打开 `/assistant`。该服务只读，不参与工具同步裁决。

## 8. 可选：接入 OpenClaw

```bash
npm --prefix openclaw-plugin run build
```

在 OpenClaw 配置中：

1. 将 `plugins.load.paths` 加入仓库内的 `openclaw-plugin` 绝对路径。
2. 启用 `traceshield-security-plugin`。
3. 将插件的 `core_base_url` 设为 `http://127.0.0.1:8787`。
4. 重启 OpenClaw Gateway。

验证日志应出现插件注册信息；在对话中调用 `traceshield_status` 可查看 Core URL、同步超时、fallback 和队列长度。

## 9. 最小端到端验证

在 OpenClaw 中依次测试：

- 普通公开文本读取：预期 `ALLOW`。
- 读取 `.env`：预期 `BLOCK/critical`，工具不执行。
- 明确的递归删除命令：预期 `ASK/critical`，默认超时拒绝。
- 外部 URL：预期 `ASK/medium`。

随后在 TraceShield `/runtime` 中确认新会话、工具节点、决策原因、策略命中和证据步骤均已出现。

## 停止开发环境

前台 Core/Web 使用 `Ctrl+C`。数据库保留数据停止：

```bash
docker compose stop postgres
```

不要用 `docker compose down -v` 作为普通停止命令；`-v` 会删除数据库卷。
