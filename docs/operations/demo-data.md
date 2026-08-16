# 演示数据

演示数据通过真实 Core API 生成，目的是验证数据库、决策、证据、Method 与前端映射，而不是只在 Web 里显示静态卡片。

## 标准种子

```bash
npm --prefix core run seed:demo
```

脚本写入一组当前时间的典型审计场景，适合让 24 小时指标立即非零并验证 ALLOW/BLOCK/ASK 等状态。

## 长链种子

```bash
npm --prefix core run seed:long-chain
```

长链包含 28 个真实 Audit API 工具步骤、对应 after-tool 结果、消息与 agent end。核心路径从敏感读取，经多步变换，到外部网络 sink；脚本会等待 Method 队列并断言：

- 28 个 tool calls；
- step sequence 连续；
- 84 个基础 evidence steps；
- Run completed/critical/blocked；
- 存在 25 步 Method 风险路径；
- 最终敏感读取规则证据存在。

固定 demo session 已存在时，脚本生成隔离的新 ID，避免覆盖原演示。

## 指标窗口

Runtime 卡片默认统计过去 24 小时。种子超过 24 小时后，24h 数字会自然归零；这不是数据丢失。

Core 同时返回历史 total。当前 Web 在 24h 空且历史存在时显示历史累计并明确标注：

- 历史累计；
- 过去 24 小时无新调用；
- 最近调用时间；
- 包含多少演示 seed calls。

不要静默把固定 mock 数字伪装成实时数据库数据。

## 安全演示原则

- 所有凭据、customer data、`.env` 内容都使用明确假值。
- 攻击副作用必须可恢复，只作用于专门 demo workspace。
- 不对真实网络接收端发送数据。
- 不使用真实个人目录、SSH key、浏览器 cookie 或生产服务。
- 在 baseline 与 protected A/B 中使用同一提示和同一假数据。
- 录制前先人工跑通并记录预期结果。

## 单 session 清理

优先删除精确测试 session，不清空整库：

```bash
docker exec -it traceshield-postgres \
  psql -U traceshield -d traceshield
```

```sql
SELECT session_id, first_seen_at, last_seen_at
FROM audit_sessions
WHERE session_id = 'exact-demo-session-id';

BEGIN;
DELETE FROM audit_sessions
WHERE session_id = 'exact-demo-session-id';
COMMIT;
```

外键会清理下游。共享演示环境先备份，且绝不使用宽泛 `LIKE` 删除。

## 生成后的验证

```bash
curl -s http://127.0.0.1:8787/v1/dashboard/runtime-status | jq
curl -s 'http://127.0.0.1:8787/v1/audit/sessions?filter=risk' | jq
curl -s 'http://127.0.0.1:8787/v1/audit/events?limit=50' | jq
curl -s http://127.0.0.1:8787/v1/method/status | jq
```

Runtime 页面切到目标 session，检查长链布局、25/27/28 等高风险节点、右侧 decision/evidence 和底部 84 步证据。

## Mock 数据

`web/src/mock/runtimeMock.ts` 只服务前端展示与 Core 失败 fallback。`mock-core/` 是无数据库旧联调服务。二者不能证明真实持久化或执行门工作，也不要在真实 Core 已占 8787 时启动 mock-core。

