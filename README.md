# andy-blog-content

Public **Markdown mirror** of published articles from [www.andy-y.cn](https://www.andy-y.cn).  
[www.andy-y.cn](https://www.andy-y.cn) 已发布文章的公开 **Markdown 镜像备份仓**。

Site code lives in [`AndyYJF/andy-blog`](https://github.com/AndyYJF/andy-blog). This repo only keeps the words.  
站点代码在 [`AndyYJF/andy-blog`](https://github.com/AndyYJF/andy-blog)；本仓库只放正文。

> Canonical reading experience: [www.andy-y.cn](https://www.andy-y.cn)  
> 请以线上站点为准；这里方便备份、检索与 diff。

---

## English

### What this is

Automated backup of **published** posts, pages, and moments from the Typecho → Astro pipeline. Files are plain Markdown with YAML frontmatter (same shape the Astro content collections use).

### Layout

| Path | Content |
|------|---------|
| `posts/` | Blog posts (`kind: post`) |
| `pages/` | Standalone pages such as About / DN42 (`kind: page`) |
| `moments/` | Short “moments” feed items (`kind: moment`), when published |

Filenames are the public slug, e.g. `posts/grafana-bird-status.md` → `https://www.andy-y.cn/posts/grafana-bird-status/`.

### Frontmatter (posts / pages)

Typical fields:

```yaml
slug: grafana-bird-status
kind: post
title: …
legacyCid: 30
canonicalPath: /posts/grafana-bird-status/
commentKey: /posts/grafana-bird-status/
feedGuid: https://www.andy-y.cn/index.php/archives/30/
allowComment: true
allowFeed: true
pubDate: '2026-02-19T14:39:00.000Z'
updatedDate: '2026-07-02T07:34:48.000Z'
categories: […]
tags: […]
sourceFormat: markdown   # or html
description: …
cover: https://…         # optional
```

Moments use a similar block plus `images` / `topics` where applicable.

### What is not here

- Drafts, password-protected, or unpublished entries  
- Database dumps, CMS config, secrets, `.env`  
- Comment threads / visitor PII  
- Site theme, Astro/Typecho source, or build scripts  
- A full local copy of every image (bodies keep remote URLs such as the cover CDN)

### Sync policy

Content is exported from production on a **daily systemd timer** (`blog-content-backup.timer`, ~04:30 Asia/Shanghai): Typecho snapshot → Markdown → this repository. History is append-only git commits; force-pushes are not part of the normal flow.

If a post is deleted or withdrawn on the live site, the next sync removes or updates the corresponding file here so the mirror stays truthful.

### License

**Text and original figures** in this repository: © Andy Yan (`AndyYJF`). All rights reserved unless a file says otherwise.

You may fork this repo for personal backup or offline reading. Please do **not** republish the articles as your own, scrape them into training corpora without permission, or strip attribution.

The [andy-blog](https://github.com/AndyYJF/andy-blog) site/code repository has its own license terms; this content mirror is separate.

### Related

- Live site: https://www.andy-y.cn  
- Site source: https://github.com/AndyYJF/andy-blog  
- Build status (operators): https://build2.fei.cx  

---

## 中文

### 这是什么

从 Typecho → Astro 管线自动备份的 **已发布** 文章、独立页面与瞬间（moments）。文件是带 YAML frontmatter 的 Markdown，字段形态与 Astro Content Collections 一致。

### 目录

| 路径 | 内容 |
|------|------|
| `posts/` | 博文（`kind: post`） |
| `pages/` | 独立页，如关于 / DN42（`kind: page`） |
| `moments/` | 已公开的「瞬间」短动态（`kind: moment`） |

文件名即公开 slug，例如 `posts/grafana-bird-status.md` 对应 `https://www.andy-y.cn/posts/grafana-bird-status/`。

### Frontmatter

见上方 English 示例。阅读请以 [www.andy-y.cn](https://www.andy-y.cn) 为准；本仓便于备份、全文检索与变更 diff。

### 不会出现在这里的内容

- 草稿、加密文、未发布条目  
- 数据库、CMS 配置、密钥、`.env`  
- 评论与访客隐私数据  
- 主题 / Astro·Typecho 源码 / 构建脚本  
- 图片资源的完整本地镜像（正文保留线上图片 URL）

### 同步策略

生产机每天定时（`blog-content-backup.timer`，约北京时间 04:30）导出 Typecho 快照 → 生成 Markdown → 推送到本仓库。正常流程为追加式 commit，不以强推为常态。线上删除或撤回的内容，会在下次同步时从本仓移除或更新，以保持镜像真实。

### 许可

本仓库中的 **文字与原创配图**：© Andy Yan（`AndyYJF`），保留所有权利（文件另有声明除外）。

允许 fork 供个人备份或离线阅读。请勿将文章署名为己有、未经许可批量抓取用于模型训练，或去掉署名后转载。

站点代码仓 [andy-blog](https://github.com/AndyYJF/andy-blog) 的许可与本内容镜像仓相互独立。

### 相关链接

- 线上站点：https://www.andy-y.cn  
- 站点源码：https://github.com/AndyYJF/andy-blog  
- 构建状态（运维）：https://build2.fei.cx  
