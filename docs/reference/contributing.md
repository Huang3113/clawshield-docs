# 贡献指南

## 开始之前

1. 阅读[系统概览](../getting-started/overview.md)和相关模块章节。
2. 从 main 创建短生命周期分支。
3. 检查工作树，保留其他人的未提交变更。
4. 不复制历史项目文档或第三方指南正文。
5. 不读取、提交或展示本地 secret。

## 变更范围

| 变更 | 同步更新 |
| --- | --- |
| Audit/Event schema | 插件 type/producer、Core validator/type/service、测试、reference docs |
| 数据库列/表 | 新 migration、db type/query、测试、data model docs |
| 策略 | engine/safety floor、seed catalog、插件行为测试、policy docs |
| Method | fixture、Python test、Core mapping/diff、Method docs |
| Web route/page | router、SideRail、smoke、page/data-source docs |
| Assistant contract | Go validator、Core proxy、Web parser、三层测试、Assistant docs |
| 启动/配置 | example、unit/runbook、configuration/operations docs |

## 编码约定

### TypeScript

- 保持 strict 类型，不用无说明 `any`。
- 在边界用 Zod/类型收窄，内部使用 domain type。
- 异步调用显式处理 timeout/cancel/error。
- 不记录 raw secret。
- 数据库写入使用参数化 SQL。
- 枚举映射提供保守默认。

### Go

- `gofmt`；
- request limits 与错误正文稳定；
- context 取消沿调用链传播；
- 上游错误不原样泄露；
- streaming 一旦可见不自动重放；
- race test 通过。

### Python Method

- 输入 schema 严格；
- 规则产生可解释 violation；
- 映射/关系/图变更增加 fixture；
- 不把 observation 原文放进错误或 graph metadata；
- Worker 对坏请求恢复并继续处理下一行。

### Vue

- API DTO 与 UI type 分离；
- 真实/历史/mock/demo/fallback 来源可见；
- 使用 token 与现有布局；
- 支持折叠、键盘和 reduced motion；
- Abort 长请求并防旧回调覆盖新状态。

## 测试

先运行受影响模块，然后完整发布矩阵。安全门或契约修改最低要求：Core tests + plugin tests + build + 真实 smoke。

```bash
npm --prefix core run test
npm --prefix openclaw-plugin run test
npm --prefix web run build
(cd assistant-eino && go test ./...)
```

详见[测试与发布检查](../operations/testing.md)。

## Commit

推荐清晰的命令式主题：

```text
feat(core): add stable audit field
fix(plugin): preserve tool correlation on retry
docs: expand deployment troubleshooting
test(method): cover provenance barrier
```

一次提交不混入论文、PDF、旧文档、截图缓存、构建产物或本地配置。

## Pull Request 清单

- [ ] 说明问题、实现和安全影响。
- [ ] 列出真实运行边界是否改变。
- [ ] 包含迁移/回滚说明。
- [ ] 附测试命令和结果。
- [ ] UI 变更附无敏感信息的截图。
- [ ] 无 API key、token、`.env`、个人绝对路径。
- [ ] `git diff --check` 通过。
- [ ] 文档 `mkdocs build --strict` 通过。
- [ ] 新行为已更新 limitations 或删除过时限制。

## 安全问题

发现可能导致危险工具绕过、secret 泄露、fallback fail-open 或审计记录伪造的问题时：

- 不先制作公开攻击演示；
- 保留最小复现与受影响版本；
- 不在 issue 中粘贴真实 secret；
- 修复同时覆盖在线、fallback 与阻断可见性；
- 增加防回归测试和升级说明。

## 文档风格

- 先写结果和准确边界，再写操作步骤。
- 命令默认从仓库根执行，切目录时明确。
- 使用 `<VM_IP>`、`<vm-user>`、`/absolute/path` 等占位符。
- 所有示例 secret 使用不可误认的假值或完全省略。
- 不用“实时”“已生效”“已阻断”等词描述尚未被真实链路证明的状态。

