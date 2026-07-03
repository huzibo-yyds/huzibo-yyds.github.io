# Codex 项目指令

这个仓库是基于 Academic Pages 的 Jekyll / GitHub Pages 个人学术主页。

## 持久上下文工作流

用户希望项目上下文能在新的 Codex 页面中持续可用，不需要每次重复说明。对这个仓库里的任何非琐碎工作，都按以下流程执行：

1. 开始前读取：
   - `LOCAL_USAGE.md`
   - `docs/codex-handoff.md`
   - `git status --short`
2. 如果任务会影响已有工作，编辑前用 `git diff` 或 `git log` 检查相关近期改动。
3. 结束前，如果本轮修改了文件、形成了决策、发现了命令或留下了后续事项，就更新 `docs/codex-handoff.md`。
4. 最终回复中说明本轮改了什么，并说明是否已更新交接文件。

把 `docs/codex-handoff.md` 当作这个项目的长期记忆，用来记录：

- 当前项目环境和常用命令。
- 最近修改过哪些文件，以及修改原因。
- 未完成事项、待决问题和风险。
- 会影响后续工作的用户偏好。

## 本地环境

- 项目路径：`/Users/huzi/Documents/Code/huzibo-yyds.github.io`
- 项目类型：Jekyll / GitHub Pages
- 本地预览地址：`http://localhost:4000/`
- 推荐 Ruby / Bundler 命令前缀：`/opt/homebrew/opt/ruby/bin/bundle`
- 推荐预览命令：

```bash
/opt/homebrew/opt/ruby/bin/bundle exec jekyll serve -H localhost
```

- 依赖应通过 Bundler 安装到项目本地路径，不要写入系统 Ruby 目录。
- 完整本地环境说明见 `LOCAL_USAGE.md`。

## 常用内容入口

- 站点配置和个人资料：`_config.yml`
- 首页内容：`_pages/about.md`
- CV 页面：`_pages/cv.md`
- 导航菜单：`_data/navigation.yml`
- 博客文章：`_posts/`
- 论文列表：`_publications/`
- 报告 / 演讲：`_talks/`
- 教学经历：`_teaching/`
- 图片：`images/`
- PDF 和公开文件：`files/`

## 工作方式

- 优先做小而聚焦的修改。
- 除非任务明确需要，否则不要重写上游模板文件。
- 保留工作区中用户已有的改动。
- 搜索优先使用 `rg` / `rg --files`。
- 如果涉及视觉或布局变化，在可行时运行或检查本地 Jekyll 站点。
