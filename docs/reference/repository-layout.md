# 仓库目录

```text
TraceShield/
├── assistant-eino/         # Go + Eino 只读安全助手
│   ├── cmd/server/
│   └── internal/
│       ├── chat/
│       ├── config/
│       └── httpapi/
├── core/                   # Fastify + PostgreSQL 审计 Core
│   ├── docs/
│   ├── method-engine/      # Python NDJSON Method Worker
│   ├── scripts/
│   └── src/
│       ├── db/
│       ├── engine/
│       ├── routes/
│       ├── services/
│       └── types/
├── openclaw-plugin/        # OpenClaw Runtime 插件
│   ├── docs/
│   └── src/
│       ├── client/
│       ├── events/
│       ├── hooks/
│       ├── policy/
│       ├── queue/
│       ├── register/
│       ├── runtime/
│       ├── sanitizer/
│       ├── tests/
│       ├── types/
│       └── worker/
├── web/                    # Vue 3 控制台
│   ├── scripts/
│   └── src/
│       ├── api/
│       ├── assets/
│       ├── components/
│       ├── layouts/
│       ├── mock/
│       ├── pages/
│       ├── stores/
│       ├── styles/
│       └── types/
├── deploy/systemd/         # 项目用户级服务模板
├── developer-guide/        # 本开发者指南
├── demo-lab/               # 受控演示素材
├── mock-core/              # 旧的无数据库最小联调服务
├── docker-compose.yml      # PostgreSQL 16 开发服务
└── README.md
```

## 权威文件

| 主题 | 权威实现 |
| --- | --- |
| Audit/Event wire schema | 插件 types + Core route Zod schema |
| 数据库 | `core/src/db/schema.sql` + migrations |
| 实际基础策略 | `core/src/services/policyEngine.ts` |
| Method runtime | Core engine/service + `core/method-engine/python/` |
| 插件配置默认 | `openclaw-plugin/src/types/config.ts` |
| Assistant 配置默认 | `assistant-eino/internal/config/config.go` |
| Web 路由 | `web/src/router/index.ts` |
| Runtime 状态 | `web/src/stores/runtimeStore.ts` |
| 服务启动 | `deploy/systemd/` + package scripts |
| 开发文档 | `developer-guide/` |

当旧说明与当前代码冲突时，以实现、迁移、测试和本指南最新修订为准，并提交文档修复。

## 构建产物

不提交：

```text
**/node_modules/
**/dist/
assistant-eino/bin/
core/method-engine/.venv/
developer-guide/.venv/
developer-guide/site/
.traceshield/
```

clone 后必须重新安装与构建。

## 历史与演示目录

- `mock-core/` 仅用于早期无数据库协议联调，不提供当前完整功能。
- 根目录阶段记录和历史说明不是新开发的唯一入口。
- `demo-lab/` 必须只放安全、可恢复、无真实秘密的演示素材。
- 开发者指南不收录论文、PDF、旧图片和过时产品文档。

## 新模块放置

- 新 HTTP endpoint：route + service + type/test。
- 新数据库字段：新 migration + db type + service/query + docs。
- 新 Web API：`api/` DTO/mapper + domain type + Store/page。
- 新插件 Hook：`hooks/` 逻辑，`register/` 宿主连接，`tests/` 覆盖。
- 新 Method 规则：Python method config/logic + fixtures/tests + Core mapping。
- 新开发文档：对应章节目录，并加入 mkdocs nav。

