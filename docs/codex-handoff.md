# Codex 交接记录

这个文件是未来 Codex 页面使用的项目长期记忆。对非琐碎工作，开始前读取它；如果本轮有实质变更，结束前更新它。

## 当前项目快照

- 仓库：`/Users/huzi/Documents/Code/huzibo-yyds.github.io`
- 项目类型：Jekyll / GitHub Pages 学术个人主页。
- 本地预览地址：`http://localhost:4000/`
- 主要本地说明：`LOCAL_USAGE.md`
- Codex 项目级指令：`AGENTS.md`
- 当前仓库主分支：`master`，远端 `origin/HEAD` 指向 `origin/master`。

## 环境说明

- 使用 Homebrew Ruby，不使用系统 Ruby：

```bash
/opt/homebrew/opt/ruby/bin/bundle
```

- 推荐在仓库根目录启动本地预览：

```bash
/opt/homebrew/opt/ruby/bin/bundle exec jekyll serve -H localhost
```

- 如需安装依赖，Bundler 应配置为安装到项目本地的 `vendor/bundle` 路径。
- `LOCAL_USAGE.md` 记录：当前 shell 有 `node`，但可能没有 `npm`；使用 npm 脚本前需要先检查。
- `LOCAL_USAGE.md` 记录：当前 shell 中没有 Docker。
- 之前这里出现过 GitHub 推送失败：`Permission denied (publickey)`。当时原因是 SSH 认证，不是仓库权限；本地存在并发送了 `id_ed25519` key，但 GitHub 拒绝，除非匹配的公钥已添加到正确的 GitHub 账号。
- 当前 SSH 配置会把 `github.com` 通过 `ssh.github.com:443` 和本地代理命令转发；这个路径可以连到 GitHub，因此失败发生在连接建立之后的认证阶段。

## 用户偏好

- 把项目假设、环境细节和修改历史保存在仓库文件中，让新的 Codex 页面可以恢复上下文。
- 不希望每次对话后都显式提醒 Codex 读取或更新上下文。
- 关于工作流说明，优先使用简洁中文。
- `AGENTS.md` 和 `docs/codex-handoff.md` 可使用中文维护；命令、路径、文件名和代码标识符保持原样。

## 最近修改记录

### 2026-07-05：新增 Codex 开发工作流博客草稿

修改文件：

- `_posts/2026-07-05-codex-development-workflow.md`：从用户 Obsidian 笔记 `Codex开发.md` 整理出新博客，合并了其直接链接的 `MCP.md`、`Skills.md`、`Subagents.md`、`Hooks.md`、`Plugins.md` 的核心内容。
- `images/blog/codex-development/`：新增博客图片资源，避免公开页面引用本地 Obsidian 路径。
- `_pages/about.md`：更新“阅读最新文章”按钮和近期时间线，链接到新文章。
- `docs/codex-handoff.md`：记录这次草稿工作。

目的：

- 生成文章《Codex 开发工作流：从 AGENTS.md 到 Skills、MCP 与 Plugins》，路径为 `/posts/2026/07/codex-development-workflow/`。
- 按 `blog-update` Skill 流程完成本地预览，用户确认后提交并推送到 `master`。

验证：

- 成功运行 `git diff --check`。
- 成功解析 Markdown front matter。
- 验证新文章有 27 个 TOC 条目，且都有匹配的 heading anchors，没有缺失或多余 id。
- 成功运行 `/opt/homebrew/opt/ruby/bin/bundle exec jekyll build`。
- 通过本地 Jekyll 预览检查文章页面返回 HTTP 200，图片资源返回 HTTP 200。
- 用户已确认本地效果，后续提交应包含新博客、首页入口、图片资源和本交接记录。

### 2026-07-03：将 Codex 上下文文件改写为中文

修改文件：

- `AGENTS.md`：将项目级 Codex 指令从英文改写为中文，保留原有工作流、环境、常用入口和工作方式要求。
- `docs/codex-handoff.md`：将交接记录从英文改写为中文，并记录当前仓库实际主分支为 `master`。

目的：

- 让用户和后续 Codex 页面都能直接用中文阅读项目规则和交接记录。
- 保留跨会话上下文机制，不改变原有行为。

验证：

- 已确认当前远端只有 `origin/master`，`origin/HEAD` 指向 `origin/master`。

### 2026-06-11：将包名示例从 calcc 改为 hzb

修改文件：

- `_posts/2026-06-11-tvm-nbu-custom-op-te-extern-guide.md`：在 import path 冲突说明和推荐文件组织结构中，把剩余的 `calcc` 包路径示例替换为 `hzb`。
- `docs/codex-handoff.md`：记录这次后续修改。

验证：

- 确认目标文章中已没有剩余 `calcc`。
- 成功运行 `git diff --check`。
- 验证文章仍有 28 个 TOC 条目，并且都有匹配的 heading anchor。
- 成功运行 `/opt/homebrew/opt/ruby/bin/bundle exec jekyll build`。

### 2026-06-11：新增 NBU custom op te.extern 博客

修改文件：

- `_posts/2026-06-11-tvm-nbu-custom-op-te-extern-guide.md`：从 `/Users/huzi/Documents/Code/tvm/NBU_CUSTOM_OP_TE_EXTERN_GUIDE.md` 转换出的新博客文章，包含 front matter、excerpt、tags、显式 heading anchors，以及供右侧自定义 TOC 使用的 `toc_items`。
- `_pages/about.md`：更新“阅读最新文章”按钮和近期时间线，链接到新文章。
- `docs/codex-handoff.md`：记录这次发布工作。

目的：

- 发布文章《TVM 自定义算子接入：从 Relay 注册到 te.extern Runtime》，路径为 `/posts/2026/06/tvm-nbu-custom-op-te-extern-guide/`。
- 保留原始指南的实践流程：C++ Relay op 注册、Python API wrapper、FTVMCompute/FTVMSchedule、`te.extern`、runtime PackedFunc、示例和常见错误。

验证：

- 成功运行 `git diff --check`。
- 成功解析 Markdown front matter。
- 验证新文章有 28 个 TOC 条目，且都有匹配的 heading anchors，没有缺失或多余 id。
- 成功运行 `/opt/homebrew/opt/ruby/bin/bundle exec jekyll build`。
- 确认生成的 HTML 包含新文章、TOC 条目、年份归档条目和首页链接。

### 2026-05-29：为 ResNet18 博客添加完整 Relay IR 附录

修改文件：

- `_posts/2026-05-28-resnet18-relay-ir-structure.md`：新增一个可折叠附录，包含来自 `toys/resnet18/resnet18_relay.log` 的完整 `IR BEFORE Optimization` Relay IR，并添加对应 TOC 条目。
- `docs/codex-handoff.md`：记录这次更新。

目的：

- 让读者可以直接在页面内对照原始 Relay IR 理解文章内容。
- 使用 Markdown `<details>` 把较长 IR 放到主阅读流程之外。

验证：

- 移除粘贴 IR 中的行尾空格后，成功运行 `git diff --check`。
- 验证新文章有 28 个 TOC 条目，且都有匹配的 heading anchors，没有缺失或多余 id。
- 成功运行 `/opt/homebrew/opt/ruby/bin/bundle exec jekyll build`。
- 确认生成的 HTML 包含附录 TOC 条目、`<details>` summary 和 `IR BEFORE Optimization` 代码块。

### 2026-05-28：新增 ResNet18 Relay IR 结构博客

修改文件：

- `_posts/2026-05-28-resnet18-relay-ir-structure.md`：从用户的 Obsidian 笔记转换出的新博客文章，主题是通过 TVM Relay IR 阅读 ResNet18，包含 front matter、excerpt、显式 heading anchors、Mermaid 图，以及自定义文章 TOC 使用的 `toc_items`。
- `_pages/about.md`：更新“阅读最新文章”按钮和近期时间线，链接到新的 ResNet18 Relay IR 文章。
- `docs/codex-handoff.md`：记录这次发布工作。

目的：

- 发布文章《ResNet18 结构详解：如何阅读 TVM Relay IR》，路径为 `/posts/2026/05/resnet18-relay-ir-structure/`。
- 保持博客右侧大纲行为一致，确保每个 TOC 条目都映射到显式 `##` 或 `###` heading id。

验证：

- 成功运行 `git diff --check`。
- 成功解析 Markdown front matter。
- 验证新文章有 27 个 TOC 条目，且都有匹配的 heading anchors，没有缺失或多余 id。
- 成功运行 `/opt/homebrew/opt/ruby/bin/bundle exec jekyll build`。
- 确认生成的 HTML 包含新文章、首页链接和 TOC 条目。

### 2026-05-25：修复博客时间戳头部布局

修改文件：

- `_layouts/single.html`：将文章时间戳 wrapper 从主题自带的 `page__date` class 改为专用的 `post-date-meta` 组件，并在 GitHub commit 数据加载前使用文章日期作为更新时间的可见 fallback。
- `_sass/layout/_page.scss`：新增独立的时间戳布局规则，包括全宽放置、清除浮动、内部 label/time 不换行，以及移动端堆叠布局。
- `docs/codex-handoff.md`：记录这次修复。

目的：

- 修复已部署博客页面中“发布时间”和“更新时间”可能被挤到右侧窄列并竖向换行的问题。
- 避免把“自动获取中”作为持久或慢网络下可见状态；GitHub commits API 仍会在可用时替换 fallback。

验证：

- 成功运行 `/opt/homebrew/opt/ruby/bin/bundle exec jekyll build`。
- 验证生成的 `_site` HTML 使用 `.post-date-meta`，生成的 CSS 包含新规则。
- 确认 GitHub commits API 能返回 `_posts/2026-05-18-tvm-relay-resnet18-beginner-guide.md` 的 commit 数据。
- 注意：本地 Jekyll 页面目前会从配置的生产 `url` 加载 CSS，因此浏览器预览可能显示旧的已部署 CSS，直到新 CSS 被推送上线。

### 2026-05-21：新增博客更新时间显示

修改文件：

- `_layouts/single.html`：博客文章头部现在同时渲染创建时间和更新时间。客户端脚本会查询当前文章源文件路径的 GitHub commits，使用第一次提交作为创建时间、最近一次提交作为更新时间。
- `_data/ui-text.yml`：为英文、简体中文、繁体中文新增本地化的 `updated_label` 文本。
- `_includes/seo.html`：让静态 `article:modified_time` 仍只使用 `modified`；可见文章时间由运行时 GitHub commit 数据填充。
- `_sass/layout/_page.scss`：为文章发布 / 更新元信息添加紧凑响应式样式。
- `_posts/2026-05-18-tvm-relay-resnet18-beginner-guide.md`：移除手动 `last_modified_at`；不再需要时间戳字段。
- `_posts/2026-05-15-tvm-custom-operator-layernorm.md`：移除手动 `last_modified_at`；不再需要时间戳字段。
- `LOCAL_USAGE.md`：说明博客创建 / 更新时间会自动从 GitHub commit 历史读取。

目的：

- 为每篇博客显示可见的“发布时间”和“更新时间”，格式精确到小时/分钟，不显示秒。
- 避免在 Markdown front matter 中手动维护时间戳字段。
- 保持 GitHub Pages 兼容，不新增自定义 Jekyll 插件。

运行说明：

- 未来文章不需要 `last_modified_at`。可见时间戳基于远程 GitHub 中每个文章文件的 commit 历史，不基于本地文件保存时间。

### 2026-05-21：重做博客页相关文章区域

修改文件：

- `_layouts/single.html`：将旧的 `site.related_posts` 网格渲染改为相关文章区；优先选择与当前文章共享 tags 的文章，如果没有 tag 重叠，再回退到 Jekyll 默认 related posts。
- `_includes/related-post-card.html`：新增专用相关文章卡片 include，包含日期、阅读时间、标题、摘要和 tags。
- `_sass/layout/_page.scss`：重做相关文章区域样式，使其与文章正文列对齐，并使用全宽响应式卡片，而不是窄的归档网格项。
- `docs/codex-handoff.md`：记录这次修改，供后续 Codex 会话使用。

目的：

- 修复博客底部“你可能感兴趣的”区域视觉不协调的问题。
- 通过优先推荐共享 tags 的文章，让推荐行为更直观、更容易解释。

验证：

- 成功运行 `/opt/homebrew/opt/ruby/bin/bundle exec jekyll build`。
- 验证生成的 ResNet18 文章使用 `.related-post` 标记，不再使用旧的 `.grid__wrapper`，并通过共享 tags 推荐 LayerNorm 文章。
- 检查本地 `http://localhost:4000/posts/2026/05/tvm-relay-resnet18-beginner-guide/`，确认相关文章区域宽度与正文列对齐。

### 2026-05-20：拆分 tag 索引页和 tag 详情页

修改文件：

- `_layouts/archive.html`：新增可选的 `archive_class` wrapper class，允许归档页设置自己的标题样式且不影响其他页面。
- `_includes/taxonomy-cloud.html`：新增 `/tags/` 使用的 tag cloud include，按相关文章数量设置 tag 大小等级。
- `_includes/taxonomy-card-grid.html`：新增卡片网格 include，用于 category 风格的归档预览。
- `_includes/taxonomy-archive.html`：移除旧的“tag 索引 + 分组文章列表”混合 include。
- `_includes/tag-list.html`：将文章级 tag 链接更新为 `/tag/?tag=<slug>`，不再使用旧的 `/tags/#<slug>` anchor。
- `_pages/tag-archive.html`：将 `/tags/` 改为居中的“全部标签”云，每个 tag 链接到 `/tag/?tag=<slug>`。
- `_pages/tag-detail.html`：新增单 tag 详情页，读取 `tag` query 参数并按年份时间线只显示该 tag 的文章。
- `_pages/category-archive.html`：将 `/categories/` 改为卡片网格归档预览；由于现有文章未定义 `categories`，目前基于文章 tags 展示。
- `_sass/layout/_archive.scss`：新增居中归档标题、加权 tag cloud、category cards 和 tag-detail timeline 样式。

目的：

- 避免 `/tags/` 页面堆叠杂乱，把索引视图和详情视图拆开。
- 匹配参考站点中“全部标签云”和“标签详情时间线”的交互方式，同时保持静态 GitHub Pages 兼容。

验证：

- 成功运行 `/opt/homebrew/opt/ruby/bin/bundle exec jekyll build`。
- 由于沙箱内端口绑定失败，经批准在本地启动 Jekyll 预览：`http://localhost:4000/`。
- 验证 `/tags/` 只渲染加权 tags，`/tag/?tag=compiler` 只显示 Compiler 相关文章，`/categories/` 渲染卡片预览。

### 2026-05-19：新增 tag/category 归档页

修改文件：

- `_includes/taxonomy-archive.html`：新增可复用 Liquid include，按字段分组文章，渲染所有 taxonomy 值的顶部索引和文章数量，并在每个分组下列出匹配文章。
- `_pages/tag-archive.html`：将 `/tags/` 改为中文 tag 归档页，显示所有 tags、数量和分组博客文章。
- `_pages/category-archive.html`：复用相同归档 include 改造 `/categories/`，在文章没有 categories 时显示空状态。
- `_data/navigation.yml`：新增顶部导航链接 `/tags/`。
- `_sass/layout/_archive.scss`：新增 taxonomy chips、计数、分组 section 和紧凑文章行样式。

目的：

- 匹配 `albresky.cn/categories/` 和 `albresky.cn/tags/` 的参考模式：顶部收集 tags/categories 和数量，下方按 taxonomy 分组展示文章。
- 使用 Liquid 而不是 `jekyll-archives`，保持 GitHub Pages 兼容。

验证：

- 成功运行 `/opt/homebrew/opt/ruby/bin/bundle exec jekyll build`。
- 由于沙箱内端口绑定失败，经批准在本地启动 Jekyll 预览：`http://localhost:4000/`。
- 验证 `/tags/` 显示 7 个 tags 和 2 篇博客文章；`/categories/` 因当前文章未定义 `categories` 而显示空状态提示。

### 2026-05-19：新增持久化 Codex 上下文工作流

修改文件：

- `AGENTS.md`：新增项目级指令，要求 Codex 读取本地使用说明、读取本交接文件、检查 git 状态，并在有意义的工作后更新交接文件。
- `docs/codex-handoff.md`：新增长期项目记忆文件，用于记录环境说明、用户偏好、近期修改和后续事项。

目的：

- 在新的 Codex 页面中保留项目设置和工作历史。
- 减少用户重复手动提示。

## 后续事项

- 未来做功能或内容修改时，在“最近修改记录”下追加简短的日期条目，说明修改文件和目的。
- 如果本地环境发生变化，同时更新 `LOCAL_USAGE.md` 和上面的环境说明。
