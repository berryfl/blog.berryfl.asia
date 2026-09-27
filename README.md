# 小双的踩坑日记

线上博客：https://blog.berryfl.asia

基于 GitHub Pages + [jekyll-theme-minimal](https://github.com/pages-themes/minimal)。

## 写文章

把 Markdown 放进 `_posts/`，命名格式 `YYYY-MM-DD-title.md`，开头写好 front matter：

```markdown
---
layout: post
title: "标题"
date: 2026-09-27
---
```

图片等静态资源放到 `assets/`，正文里用 `/assets/xxx.png` 引用。

## 发布

推送到 `main` 分支后，GitHub Pages 会自动构建并更新 https://blog.berryfl.asia 。
