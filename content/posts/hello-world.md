---
title: "第一篇文章"
date: 2026-09-16T16:10:00+08:00
tags: ["Hugo", "GitHub Pages"]
author: "monsterhzh"
---

这是一篇占位文章，用来验证整条链路能跑通：本地 Hugo 构建 → 推送 main → GitHub Actions 自动构建 → GitHub Pages 发布。

## 怎么发新文章

```bash
hugo new posts/我的文章.md     # 按 archetypes/posts.md 生成 frontmatter
# 写内容，本地预览：hugo server -D
git checkout -b post/我的文章
git add . && git commit -m "post: 我的文章"
git push origin post/我的文章   # 然后开 PR，合并到 main 即自动部署
```

删掉这篇文章不会影响构建，随时可以删。
