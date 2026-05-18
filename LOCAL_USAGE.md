# 本地使用说明

这个项目是 Jekyll / GitHub Pages 站点，可以在本地启动一个预览服务，修改 Markdown、页面和模板后查看效果。

## 当前本机环境

- 项目目录：`/Users/huzi/Documents/Code/huzibo-yyds.github.io`
- 本机预览地址：`http://localhost:4000/`
- 推荐 Ruby：Homebrew Ruby `3.4.4`
- 系统自带 Ruby：`2.6.10`，不建议用于本项目，依赖会冲突
- Docker：当前 shell 中没有 `docker` 命令
- Node：当前 shell 有 `node`，但没有 `npm`

## 首次准备

如果依赖还没有安装，先进入项目目录：

```bash
cd /Users/huzi/Documents/Code/huzibo-yyds.github.io
```

设置 Bundler 把依赖安装到项目内，避免写入系统 Ruby 目录：

```bash
/opt/homebrew/opt/ruby/bin/bundle config set --local path vendor/bundle
```

安装 Jekyll / GitHub Pages 依赖：

```bash
/opt/homebrew/opt/ruby/bin/bundle install
```

## 📌启动本地预览

在项目根目录运行：

```bash
cd /Users/huzi/Documents/Code/huzibo-yyds.github.io
/opt/homebrew/opt/ruby/bin/bundle exec jekyll serve -H localhost
```

看到类似下面的输出就表示启动成功：

```text
Server address: http://localhost:4000/
Server running... press ctrl-c to stop.
```

然后用浏览器打开：

```text
http://localhost:4000/
```

停止服务时，在运行服务的终端里按 `Ctrl-C`。

## 修改后如何刷新

Jekyll 会自动监听大部分文件变更。修改 `_posts`、`_pages`、`_portfolio`、Markdown、HTML、SCSS 等文件后，等待终端重新生成完成，再刷新浏览器即可。

如果修改了 `_config.yml`，建议停止服务后重新启动。

## 常用修改入口

如果你以后主要是维护个人主页，最常改的文件通常是这些：

- 站点全局信息：`_config.yml`
- 顶部导航菜单：`_data/navigation.yml`
- 首页正文：`_pages/about.md`
- 简历页面：`_pages/cv.md`
- 博客文章：`_posts/`
- 论文列表：`_publications/`
- 报告 / 演讲：`_talks/`
- 教学经历：`_teaching/`
- 头像和图片：`images/`
- PDF、简历附件、论文附件：`files/`

## 如何更新主页

这个模板的首页就是：

```text
_pages/about.md
```

你可以直接修改这个文件里的正文内容，例如个人简介、研究方向、新闻、招生信息等。

首页顶部和左侧边栏里的个人信息，主要来自：

```text
_config.yml
```

常见可修改字段包括：

- `title`：浏览器标题和站点标题
- `name`：站点名称
- `description`：站点描述
- `author.name`：左侧边栏姓名
- `author.bio`：左侧边栏简介
- `author.location`：地点
- `author.employer`：单位
- `author.email`：邮箱
- `author.googlescholar`、`author.github`、`author.orcid` 等：社交 / 学术链接
- `author.avatar`：头像文件名，通常对应 `images/profile.png`

如果你要换头像，通常做法是：

1. 把新头像放到 `images/` 目录
2. 在 `_config.yml` 中把 `author.avatar` 改成对应文件名

如果你要调整顶部菜单，修改：

```text
_data/navigation.yml
```

例如可以删除 `Portfolio`、`Teaching`，或者调整 `Blog Posts`、`CV` 的顺序。

## 如何添加博客

博客文章放在：

```text
_posts/
```

文件名必须遵循 Jekyll 的命名格式：

```text
YYYY-MM-DD-title.md
```

例如：

```text
_posts/2026-05-18-my-first-post.md
```

一个最小可用示例如下：

```md
---
title: "我的第一篇博客"
date: 2026-05-18
permalink: /posts/my-first-post/
tags:
  - research
  - note
---

这里写正文内容。支持普通 Markdown 语法。
```

说明：

- `title`：文章标题
- `date`：发布日期
- `permalink`：页面链接，可改可不改
- `tags`：标签，可选

写完后，本地服务会自动重新生成，刷新浏览器即可。

博客列表页面默认在顶部菜单的 `Blog Posts`，对应归档页：

```text
_pages/year-archive.html
```

一般不需要改这个文件，新增 `_posts` 内容后它会自动出现。

## 如何更新简历

这个仓库默认启用的是 Markdown 版简历，也就是：

```text
_pages/cv.md
```

你可以直接在这个文件里修改：

- 教育经历
- 工作经历
- 技能
- 论文
- 演讲
- 教学
- 服务和其他信息

改完后，本地刷新就能看到 `/cv/` 页面变化。

如果你还想提供 PDF 简历下载，可以把 PDF 放到：

```text
files/
```

例如：

```text
files/cv.pdf
```

仓库里还支持 JSON 版 CV 页面：

```text
_pages/cv-json.md
_data/cv.json
```

但当前导航默认启用的是 Markdown 版 CV，`_data/navigation.yml` 里 `/cv/` 是开启的，`/cv-json/` 是注释掉的。

如果你以后想改成 JSON 版 CV，有两种方式：

1. 手工编辑 `_data/cv.json`
2. 先改 `_pages/cv.md`，再运行脚本同步生成 JSON：

```bash
./scripts/update_cv_json.sh
```

注意：这个脚本会调用 Python，并且会读取 `_config.yml` 和 `_pages/cv.md`。

## 如何添加论文、报告和教学内容

虽然你这次主要问的是博客、主页和简历，但这个模板最常见的另外三类内容也很固定：

- 论文：`_publications/`
- 报告 / 演讲：`_talks/`
- 教学：`_teaching/`

这些目录下通常是“一个条目一个 Markdown 文件”，顶部带 YAML 信息。例如论文：

```md
---
title: "论文标题"
collection: publications
permalink: /publication/paper-name
date: 2026-05-18
venue: "Conference / Journal Name"
paperurl: "/files/paper.pdf"
citation: "Your Name. (2026)..."
---

这里写论文简介、摘要、补充说明等。
```

如果有 PDF、slides、bib 文件，通常放在 `files/` 目录里，再在 front matter 里引用。

## 已知冲突和处理方式

### Ruby 版本冲突

不要直接运行系统自带的：

```bash
bundle install
bundle exec jekyll serve
```

因为当前系统 Ruby 是 `2.6.10`，而依赖中的部分 gem 需要 Ruby 3.x。请使用 Homebrew Ruby 的完整路径：

```bash
/opt/homebrew/opt/ruby/bin/bundle exec jekyll serve -H localhost
```

### 端口 4000 被占用

如果启动时报 `port is in use`，说明 `localhost:4000` 已经有别的服务在用。可以换一个端口，例如：

```bash
/opt/homebrew/opt/ruby/bin/bundle exec jekyll serve -H localhost -P 4001
```

然后打开：

```text
http://localhost:4001/
```

### LiveReload 端口问题

如果使用 `-l` 或 `--livereload` 时出现 `no acceptor`，可以先不用 LiveReload：

```bash
/opt/homebrew/opt/ruby/bin/bundle exec jekyll serve -H localhost
```

浏览器手动刷新即可。

### Docker 方式暂不可用

仓库里有 `Dockerfile` 和 `docker-compose.yaml`，但当前 shell 中没有 `docker` 命令。如果以后安装 Docker Desktop，可以尝试：

```bash
docker compose up
```

然后访问：

```text
http://localhost:4000/
```

### npm 不可用

当前 shell 中有 `node`，但没有 `npm`。本地预览网站不依赖 npm；只有需要重新构建压缩后的前端 JS 文件 `assets/js/main.min.js` 时，才需要 npm。

相关脚本在 `package.json` 中：

```bash
npm run build:js
```

如果将来要运行这个命令，需要先修复本机 npm 环境。

## 常用命令

构建并启动本地服务：

```bash
/opt/homebrew/opt/ruby/bin/bundle exec jekyll serve -H localhost
```

只构建静态文件，不启动服务：

```bash
/opt/homebrew/opt/ruby/bin/bundle exec jekyll build
```

检查当前 Ruby：

```bash
/opt/homebrew/opt/ruby/bin/ruby -v
```

检查 Bundler：

```bash
/opt/homebrew/opt/ruby/bin/bundle -v
```
