# boke

用 **Hugo** 写、**GitHub Pages** 发布的静态博客。

- 线上地址：https://monsterhzh.github.io/boke/
- 主题：[PaperMod](https://github.com/adityatelange/hugo-PaperMod)（以 git submodule 挂在 `themes/PaperMod`）

## 写一篇新文章

```bash
hugo new content posts/我的文章.md     # 按 archetypes/posts.md 生成 frontmatter
```

frontmatter 四个字段：

```yaml
---
title: "标题"
date: 2026-09-16T16:10:00+08:00
tags: ["标签1", "标签2"]
author: "monsterhzh"
---
```

正文用 Markdown。本地预览（含草稿）：

```bash
hugo server -D          # http://localhost:1313/boke/
```

## 发布流程

```bash
git checkout -b post/我的文章
git add content/posts/我的文章.md
git commit -m "post: 我的文章"
git push origin post/我的文章
# 在 GitHub 上开 PR → 合并到 main → Actions 自动构建并部署
```

推送 `main` 会触发 `.github/workflows/deploy.yml`：构建 `public/` → 上传 artifact → 部署到 GitHub Pages。**不需要**把 `public/` 提交进仓库（已在 `.gitignore` 里）。

也可以手动触发：Actions 页面 → Deploy Hugo site → Run workflow。

## 改这个仓库时容易踩的坑

| 坑 | 说明 |
|---|---|
| `baseURL` 必须带 `/boke/` | 这是**项目页**（`用户名.github.io/仓库名/`），不是用户页。少了路径前缀，线上 CSS/JS 全部 404，页面裸奔。改仓库名时必须同步改 `hugo.toml`。 |
| 主题是子模块 | clone 时用 `git clone --recurse-submodules`，否则 `themes/PaperMod` 是空目录、构建报错。workflow 里已设 `submodules: recursive`。 |
| 推 workflow 文件需要 `workflow` scope | PAT 权限不够时，改 `.github/workflows/` 会被 GitHub 拒绝。 |
| `draft: true` | archetype 里**没有**默认加 draft（按约定只保留 title/date/tags/author 四字段）。写完没发布时记得自己加 `draft: true`，或给 archetype 补上这一行。 |
| Hugo 版本 | 本地与 CI 都是 **0.166.0 extended**（`theme.toml` 要求 ≥ 0.146.0）。本地用非 extended 版会构建失败。 |
| 主题自带弃用警告 | 构建日志里 `.Language.LanguageDirection` / `.Language.LanguageCode` 两条 WARN 来自 PaperMod 模板，非本站配置问题，忽略即可。 |

## 本地环境

`hugo`（portable，在 `~/.local/bin`）、`gh`（portable，同目录）都已装好。
