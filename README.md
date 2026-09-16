# AI_BLOG

彭梓坚的技术博客 —— Hugo + GitHub Pages。

站点：https://peng2505.github.io/AI_BLOG/

## 结构

```
content/posts/     文章（Markdown，frontmatter: title / date / tags / author）
layouts/           主题模板（无外部主题依赖，纯自带 layouts + assets）
assets/css/        样式
.github/workflows/ deploy.yml —— push 到 main 后自动构建并发布到 Pages
```

## 本地预览

```bash
hugo server -D     # http://localhost:1313/AI_BLOG/
```

## 发布流程

1. 在 `content/posts/` 新增文章，frontmatter 固定四项：`title` / `date` / `tags` / `author`。
2. 提交并推送到 `main`（或通过 PR 合并）。
3. GitHub Actions 自动构建 → 发布到 GitHub Pages。

## 自动化

本仓库的日常文章由 Hermes 的多 profile 流水线产出：
`orchestrator 拆解 → researcher 调研 → writer 写作 → reviewer 审校 → publisher 发布`。
