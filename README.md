# Giraffe 的博客

使用 Hexo 和 Butterfly，源码推送到 main 分支后由 GitHub Actions 自动生成并发布到 GitHub Pages。

## 本地预览

在博客目录运行：

```powershell
npm run server
```

## 写文章与发布

```powershell
npx hexo new "文章标题"
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

部署方式参考：[Hexo 官方 GitHub Pages 文档](https://hexo.io/docs/github-pages)。
