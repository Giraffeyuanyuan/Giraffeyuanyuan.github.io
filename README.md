# Giraffe 的博客

使用 Hexo 和 Butterfly，源码推送到 main 分支后由 GitHub Actions 自动生成并发布到 GitHub Pages。

## 本地预览

在博客目录运行：

```powershell
npm.cmd ci
npm.cmd run server
```

打开 `http://localhost:4000/`。Windows PowerShell 若限制运行 `.ps1` 脚本，可直接使用上面的 `npm.cmd` 和下面的 `npx.cmd`。

## 写文章与发布

```powershell
npx.cmd hexo new "文章标题"
```

编辑 `source/_posts/` 中的 Markdown 文件，然后运行：

```powershell
git add source _config.yml _config.butterfly.yml
git commit -m "更新博客"
git push
```

GitHub 仓库的 Actions 页面可以查看发布进度。

## 配置

- `_config.yml`：博客标题、作者、语言、网址。
- `_config.butterfly.yml`：主题设置；将需要自定义的主题选项放在这个文件。
- `themes/butterfly`：通过 Git 子模块保存的主题版本。
- `.github/workflows/pages.yml`：自动发布流程。

再次下载此仓库时，使用 `git clone --recurse-submodules`，保证主题也被下载。

`node_modules/`、`public/` 和 `db.json` 不上传。GitHub 会安装依赖并生成网页。

## 已配置的基础功能

- 导航：首页、时间轴、标签、分类、常用链接、关于；导航栏固定在顶部。
- 标签页使用 `type: tags`，分类页使用 `type: categories`，常用链接页使用 `type: link`。
- 本地全文搜索：`hexo-generator-searchdb` 生成 `public/search.json`，主题使用 `local_search`，点击搜索时再加载索引，无需申请账号。
- 字数与阅读时长：`hexo-wordcount` 提供文章字数、预计阅读时间和全站总字数。
- 首页显示已有副标题，摘要优先使用文章的 `description`，否则自动截取 180 字。
- 文章目录、代码复制、版权声明、相关文章、阅读模式和返回顶部按钮沿用主题功能。
- 深色模式跟随系统，也可以通过右下角按钮手动切换；返回顶部按钮显示阅读进度。
- 图片使用浏览器原生懒加载，正文图片支持点击放大；404 页面已启用中文提示。
- 侧栏 GitHub 按钮和社交图标已填写为 `Giraffeyuanyuan`，常用网站收藏保存在 `source/_data/link.yml`。

## 文章标签与分类

新文章模板已包含 `tags`、`categories`、`description`。例如：

```yaml
---
title: 我的学习笔记
date: 2026-10-15 12:00:00
tags:
  - Hexo
  - 博客
categories:
  - 学习笔记
description: 这篇文章介绍博客的搭建与配置。
---
```

给文章填写后，标签和分类页面会自动汇总。目前示例文章未设置标签或分类，因此这两个页面暂时为空。原菜单中的 `/archives/learning-notes/` 等地址没有对应页面，已移除；归档路径按日期生成，文章分类应从分类页进入。

头像和网站图标仍使用主题自带文件。替换时把图片放入 `source/img/`，再修改 `_config.butterfly.yml` 的 `avatar.img` 和 `favicon`。首页背景可填写 `index_img`，默认文章封面可填写 `cover.default_cover`。

评论、打赏、第三方分析和数学公式保持未启用；如需使用，按实际账号、服务地址或文章内容填写，不能直接复制文档中的示例凭据。现有不蒜子访问统计保持开启。

修改完成后，可执行 `npm.cmd run clean` 和 `npm.cmd run build` 检查生成结果。提交插件变更时，需要一起提交 `package.json` 和 `package-lock.json`。

配置依据：[Butterfly 主題頁面](https://butterfly.js.org/posts/dc584b87/)、[Butterfly 主題配置](https://butterfly.js.org/posts/4aa8abbe/)。主题安装版本为 5.7.0，配置字段同时参考当前主题自带的 `_config.yml`。

部署方式参考：[Hexo 官方 GitHub Pages 文档](https://hexo.io/docs/github-pages)。
