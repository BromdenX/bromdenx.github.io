# BromdenX Blog

An engineering field notebook for AI systems, networks, and long-term learning，基于 Jekyll 与 GitHub Pages。

- 网站：https://bromdenx.github.io
- 内容：AI 与 Agent、网络与技术实验、个人成长与生活
- 发布：提交到 `master` 后由 GitHub Actions 自动构建并部署

## 发布新文章

在 `_posts/` 创建 Markdown 文件，文件名必须使用 `YYYY-MM-DD-english-slug.md`。

```yaml
---
layout: post
title: "文章标题"
subtitle: "一句话说明文章解决什么问题"
date: 2026-09-01
author: BromdenX
header-img: img/post-bg-desk.jpg
catalog: true
tags:
  - AI-Agents
  - Technical-Notes
---
```

随后用 Markdown 编写正文并提交。GitHub Actions 成功后，文章会自动出现在首页与对应主题页。

建议保持标签稳定，优先使用：`AI-Agents`、`Personal-AI-OS`、`Network-Systems`、`Technical-Notes`、`Life-and-Learning`。

## 本地预览

安装 Ruby、Jekyll 与 `jekyll-paginate` 后运行 `jekyll serve`，打开 `http://127.0.0.1:4000`。

## 致谢

主题源自 Hux Blog 与 qiubaiying.github.io，并依据 MIT License 保留许可。
