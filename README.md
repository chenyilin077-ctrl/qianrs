# 嵌入式项目知识库

基于 MkDocs Material 构建，从 Obsidian Vault 自动生成静态网站。

## 本地预览

```bash
pip install mkdocs mkdocs-material mkdocs-roamlinks-plugin mkdocs-mermaid2-plugin
mkdocs serve
```

浏览器打开 http://127.0.0.1:8000

## 部署到 GitHub Pages

1. 在 GitHub 创建仓库（如 `embedded-projects`）
2. 推送代码:
```bash
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO.git
git add -A
git commit -m "init"
git push -u origin main
```
3. GitHub Actions 自动构建并部署到 `gh-pages` 分支
4. 在仓库 Settings → Pages → Source 选 `gh-pages` 分支
5. 等待几分钟，访问 `https://YOUR_USERNAME.github.io/YOUR_REPO/`

## 项目结构

```
.
├── mkdocs.yml          # MkDocs 配置
├── .github/workflows/  # GitHub Actions 自动部署
└── site/               # 构建输出（gitignore 了，CI 里自动生成）
```
