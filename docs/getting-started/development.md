# 开发工作流

## 推荐分支流程

```bash
git switch main
git pull --ff-only
git switch -c feat/<short-topic>
```

一次变更尽量只跨越必要模块。修改契约时允许 Core、插件和 Web 同时调整，但必须把契约测试与文档放在同一个提交中。

## 常见开发循环

### Core

```bash
npm --prefix core run typecheck
npm --prefix core run test
npm --prefix core run build
```

开发服务：

```bash
npm --prefix core run dev
```

### OpenClaw 插件

```bash
npm --prefix openclaw-plugin run format:check
npm --prefix openclaw-plugin run typecheck
npm --prefix openclaw-plugin run test
npm --prefix openclaw-plugin run build
```

插件构建完成不代表运行中的 Gateway 自动刷新。真实验证前必须重启相应 OpenClaw profile。

### Web

```bash
npm --prefix web run typecheck
npm --prefix web run build
npm --prefix web run smoke
```

### Eino Assistant

```bash
cd assistant-eino
gofmt -w ./cmd ./internal
go test ./...
go test -race ./...
go vet ./...
go build ./cmd/server
```

### Method Worker

```bash
npm --prefix core run method:test
npm --prefix core run method:health
```

修改工具映射、风险规则或状态转移后，还应运行 replay/report，比较允许、询问与阻断夹具的结果。

## 契约优先的修改顺序

当你要增加一个字段或事件类型时，按以下顺序工作：

1. 明确字段属于同步 `AuditRequest` 还是异步 `TraceEvent`。
2. 更新共享语义文档和生产者类型。
3. 更新 Core Zod 校验与 TypeScript 类型。
4. 如需持久化，增加幂等迁移和查询映射。
5. 更新插件规范化、脱敏和发送逻辑。
6. 更新 Web API 类型与转换层。
7. 增加成功、缺失、非法和重复请求测试。
8. 更新本指南对应 schema/API 页面。

不要直接让 Web 依赖数据库列名；查询 API 是前后端之间的稳定边界。

## 数据库迁移规则

- 已提交的迁移不回写，使用新的递增编号文件。
- DDL 尽量幂等：使用 `IF NOT EXISTS` 或先删除已知约束再建立。
- 新增非空字段时提供默认值或分阶段回填。
- 外键删除语义需要明确；当前会话删除会级联其运行与大部分审计数据。
- 迁移后运行 `db:check`，并用空库和已有数据的库各验证一次。

## 安全相关代码评审清单

- [ ] 工具是否在执行前完成同步审计？
- [ ] 超时、解析失败和 Core 不可用时是否 fail-closed？
- [ ] `ASK` 在没有审批能力时是否默认拒绝？
- [ ] 阻断结果是否明确告诉模型“未执行”？
- [ ] 新字段是否可能包含原始秘密？是否经过脱敏？
- [ ] raw 保存开关默认是否仍为 `false`？
- [ ] 日志是否避免 API key、token 和完整参数？
- [ ] 幂等键是否防止重试产生重复决策或事件？
- [ ] Web 是否把演示状态与真实 Core 状态清晰区分？

## 文档同步

修改完成后：

```bash
cd developer-guide
. .venv/bin/activate
mkdocs build --strict
```

再检查旧产品名、占位用户名、敏感文件和无效路径没有进入站点：

```bash
rg -n -i 'api[_-]?key\s*=|gateway[_-]?token\s*=' docs mkdocs.yml || true
git diff --check
```

## 提交前最低验证

任何代码变更至少运行受影响模块的类型检查和测试。涉及执行门或契约的变更，必须同时运行 Core 与插件测试；涉及前端映射的变更，再加 Web 构建。发布级完整矩阵见[测试与发布检查](../operations/testing.md)。
