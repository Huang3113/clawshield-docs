# 环境要求

## 必需软件

| 软件 | 用途 | 项目当前验证基线 |
| --- | --- | --- |
| Git | 获取与管理代码 | 支持现代 Git 的版本 |
| Node.js | Core、Web、插件 | OpenClaw 依赖要求 `>=22.19.0`；当前环境使用 Node 24 |
| npm | JavaScript 依赖与脚本 | 随 Node 安装 |
| Python | Method Worker 与文档站 | Method 使用 Python 3；当前环境为 Python 3.12 |
| Go | Eino Assistant | `go.mod` 指定 Go 1.22 |
| PostgreSQL | 持久化 | Compose 使用 PostgreSQL 16 |
| Docker + Compose | 推荐的数据库运行方式 | 能运行 Compose v2 即可 |

OpenClaw 只在需要真实 Agent 集成时才是必需项。插件依赖当前锁定的 OpenClaw `2026.6.6`，升级 OpenClaw 前应先运行插件全部测试并做真实 Gateway 回归。

## 资源建议

完整开发环境建议至少：

- 2 个 CPU 核心；并行构建和运行方法测试时建议 4 核。
- 4 GB 内存；同时运行浏览器、OpenClaw、数据库和构建任务时建议 8 GB。
- 5 GB 可用磁盘空间，用于 `node_modules`、Go 缓存、Python 虚拟环境、PostgreSQL 数据与构建产物。
- 可访问模型兼容接口的网络，仅 Eino Assistant 对话需要；安全审计主链不依赖模型可用性。

## 端口规划

| 端口 | 组件 | 建议暴露范围 |
| --- | --- | --- |
| `5432` | PostgreSQL | 仅本机或开发私网 |
| `8787` | Core | 本机与受控局域网 |
| `8790` | Eino Assistant | 仅 `127.0.0.1`，由 Core 代理 |
| `5173` | Web | 本机或受控局域网 |
| `18789` | OpenClaw Gateway 默认实例 | 按 OpenClaw 鉴权要求配置 |
| `8000` | 本文档本地预览 | 开发时临时使用 |

先检查占用：

```bash
ss -ltn | grep -E ':(5173|5432|8787|8790|18789|8000)\b' || true
```

## 虚拟机网络

TraceShield 可以完整运行在 Linux 虚拟机内，宿主机通过虚拟机 IP 访问 Web 和 OpenClaw。获取地址：

```bash
hostname -I
```

宿主机至少需要能访问虚拟机的 `5173/tcp`。如需直接访问 Core 健康接口，再放行 `8787/tcp`；不建议向宿主或外网开放 `8790` 和 `5432`。

!!! note "OpenClaw 控制台的安全上下文"
    `http://<VM_IP>:18789` 可以返回页面，但 OpenClaw Control UI 的设备身份依赖浏览器安全上下文。更稳妥的本机访问方式是 SSH 端口转发后打开 `http://localhost:18789`，或为虚拟机入口配置受信任 HTTPS。此要求属于 OpenClaw 控制台连接层，不影响 TraceShield Web。

SSH 转发示例：

```bash
ssh -N -L 18789:127.0.0.1:18789 <vm-user>@<vm-ip>
```

## 仓库准备

```bash
git clone git@github.com:potato-vita/TraceShield.git
cd TraceShield
```

依赖安装：

```bash
npm --prefix core ci
npm --prefix web ci
npm --prefix openclaw-plugin ci
go -C assistant-eino mod download
```

Method Worker 使用独立环境：

```bash
python3 -m venv core/method-engine/.venv
core/method-engine/.venv/bin/python -m pip install \
  -r core/method-engine/requirements-runtime.lock \
  -r core/method-engine/requirements-dev.lock
```

## 凭据与本地文件

以下内容必须保持在 Git 之外：

- 模型 API key 和 `api-key` 文件。
- `core/.env`、`web/.env` 等机器本地配置。
- OpenClaw Gateway token 与用户目录配置。
- PostgreSQL 数据目录和插件磁盘补传队列。
- 构建产物、虚拟环境、测试缓存和日志。

仓库已包含通用忽略规则；新增凭据文件类型时，应先更新 `.gitignore`，再创建文件。

## 环境验收

```bash
node --version
npm --version
python3 --version
go version
docker compose version
```

继续前，确保上述命令全部成功；数据库可以稍后由快速启动流程创建。
