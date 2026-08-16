# 文档站部署

开发者指南位于独立 `developer-guide/`，使用 MkDocs Material 构建静态站点。

## 本地环境

```bash
cd developer-guide
python3 -m venv .venv
. .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

`.venv/` 与 `site/` 已由本目录 `.gitignore` 排除。

## 预览

```bash
mkdocs serve -a 127.0.0.1:8000
```

虚拟机供宿主预览：

```bash
mkdocs serve -a 0.0.0.0:8000
```

打开 `http://<VM_IP>:8000`，并在预览结束后关闭临时端口。

## 严格构建

```bash
mkdocs build --strict
```

严格模式会把无效导航、缺失页面和部分警告当作失败。生成目录 `site/` 可交给任何静态文件服务器。

## GitHub Pages

在已配置 Pages 权限的仓库中，可从 developer-guide 目录运行：

```bash
mkdocs gh-deploy --force
```

它会生成并推送 `gh-pages` 分支。执行前确认当前 remote 和 Pages 设置，避免在错误仓库发布。团队环境更推荐用 CI 在 main 通过严格构建后部署。

## 静态服务器

构建后：

```bash
mkdocs build --strict
```

将 `site/` 同步到 Nginx、对象存储或其他静态站点。需要正确提供 HTML/CSS/JS/SVG，并让目录 URL 回退到对应 `index.html`。

## 内容更新流程

1. 修改代码或契约；
2. 同步对应文档；
3. 新页面加入 `mkdocs.yml` nav；
4. 检查站内相对链接；
5. 搜索旧名、占位用户名与敏感内容；
6. `mkdocs build --strict`；
7. 由代码评审确认技术事实；
8. main 合并后部署。

```bash
rg -n -i 'api[_-]?key\s*=|gateway[_-]?token\s*=' \
  docs mkdocs.yml || true
```

## 品牌资产

文档使用 `docs/assets/traceshield-logo.svg`，配色由 `docs/stylesheets/extra.css` 定义。修改 Logo 时同步 Web asset 与文档 asset，并验证浅色/深色标题栏、小尺寸 favicon 和移动导航。

## 文档写作边界

- 不复制第三方开发指南正文；只链接必要的依赖官方说明。
- 每个当前未实现能力明确写“边界”或“后续方向”。
- 不把某台机器的 IP、用户名、绝对路径、运行状态写成产品默认。
- 不收录 key/token、真实数据库口令和原始敏感 payload。
- API 示例用可识别的假 ID 与假数据。
