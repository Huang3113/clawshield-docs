# 数据库与迁移

## 开发数据库

仓库 Compose 提供 PostgreSQL 16：

```bash
docker compose up -d --wait postgres
docker compose ps
```

开发配置把宿主 `5432` 映射到容器并使用命名卷 `traceshield_pgdata`。它没有配置容器 restart policy；宿主重启后必须检查数据库是否实际运行。

```bash
docker compose ps
curl -s http://127.0.0.1:8787/v1/health | jq -e '.ok and .db_connected'
```

Core 进程 `active` 不等于数据库健康，因为服务可以在数据库不可达时保持监听并返回 `ok=false`。

## 连接配置

```dotenv
TRACESHIELD_DATABASE_URL=postgresql://<user>:<password>@127.0.0.1:5432/<database>
```

连接池：最大 10，idle timeout 30 秒，connection timeout 3 秒。数据库目前只支持 PostgreSQL 方言，并依赖 `pgcrypto` 扩展生成 UUID。

## 初始化

```bash
cp -n core/.env.example core/.env
npm --prefix core run db:migrate
npm --prefix core run db:check
```

`db:check` 验证 14 张预期业务表和至少 4 条默认 policy。

## 迁移执行

迁移器在一个事务中执行：

1. `schema.sql`；
2. 按文件名排序的 `migrations/*.sql`；
3. `seed_policies.sql`。

当前没有 migration history 表。每个脚本必须幂等，并允许服务在已迁移数据库上再次执行。

新增迁移示例：

```sql
ALTER TABLE audit_runs
  ADD COLUMN IF NOT EXISTS example_field TEXT;

CREATE INDEX IF NOT EXISTS idx_audit_runs_example
  ON audit_runs(example_field);
```

不要修改已经进入共享分支的旧迁移；创建下一编号文件。

## 数据检查

进入容器：

```bash
docker exec -it traceshield-postgres \
  psql -U traceshield -d traceshield
```

安全的只读检查：

```sql
SELECT COUNT(*) FROM audit_sessions;
SELECT status, COUNT(*) FROM audit_runs GROUP BY status ORDER BY status;
SELECT decision, COUNT(*) FROM audit_decisions GROUP BY decision ORDER BY decision;
SELECT status, COUNT(*) FROM method_evaluations GROUP BY status ORDER BY status;
```

## 清理测试数据

删除一条精确 session 会通过外键级联清理其 runs、calls、decisions、evidence 与 Method 数据。执行前先查询确认：

```sql
SELECT session_id, first_seen_at, last_seen_at
FROM audit_sessions
WHERE session_id = 'session-exact-test-id';

BEGIN;
DELETE FROM audit_sessions WHERE session_id = 'session-exact-test-id';
-- 核对影响后选择 COMMIT 或 ROLLBACK
ROLLBACK;
```

不要用模糊条件清理共享演示库，也不要把删除 volume 当作普通重置手段。

## 备份与恢复

开发备份：

```bash
docker exec traceshield-postgres \
  pg_dump -U traceshield -d traceshield -Fc \
  > traceshield-development.dump
```

备份文件可能包含审计摘要和资源路径，不应提交 Git。恢复前使用隔离数据库验证迁移兼容性。

## 当前缺口

- 无 migration history/version 表；
- 无自动备份、保留、TTL 或归档；
- 无多租户隔离；
- 无数据库级 row-level security；
- Compose 使用开发凭据并映射宿主端口；
- 部分预留 Method 列尚未被 live 代码写入。

