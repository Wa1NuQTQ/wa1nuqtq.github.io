# Walnut

Walnut 的个人网站，记录计算机视觉研究、项目与随笔。

## 本地开发

使用 Node.js 和项目锁定的 pnpm 10.33.0。在 PowerShell 中安装依赖并启动预览。

    npx.cmd --yes pnpm@10.33.0 install --frozen-lockfile
    npx.cmd --yes pnpm@10.33.0 dev

正式构建命令如下，产物位于 `dist/`。

    npx.cmd --yes pnpm@10.33.0 build

## 写文章

- 文章放在 [src/content/posts](src/content/posts)，关于页在 [src/content/about](src/content/about)。
- 文章文件头必填 `title` 和 `published`。需要自定义摘要或标签时填写 `description` 和 `tags`，另有更新日期时填写 `updated`。
- 图片和视频放在 `public/images/` 和 `public/videos/`，文章中用 `/images/` 和 `/videos/` 开头的地址。
- 设置 `draft: true` 后，可以在开发预览中查看草稿。正式构建仅生成已公开的文章。

网站使用 Astro 和 [Retypeset](https://github.com/radishzzz/astro-theme-retypeset)。主题许可见 [LICENSE](LICENSE)。
