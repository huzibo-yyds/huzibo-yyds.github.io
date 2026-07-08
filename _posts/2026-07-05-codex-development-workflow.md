---
title: "Codex 开发工作流：从 AGENTS.md 到 Skills、MCP 与 Plugins"
date: 2026-07-05
permalink: /posts/2026/07/codex-development-workflow/
tags:
  - Codex
  - AI Agent
  - Workflow
  - MCP
  - Skills
  - Plugins
excerpt: "整理 Codex 桌面端、CLI、项目上下文、AGENTS.md、MCP、Skills、Subagents、Hooks 与 Plugins 的使用笔记，形成一套可复用的 AI Agent 开发工作流。"
image: /images/blog/codex-development/agents-md.png
toc_label: "目录"
toc_items:
  - title: "读完需要了解什么"
    id: "overview"
  - title: "认识 Codex 与环境准备"
    id: "codex-environment"
  - title: "CodeGraph 插件"
    id: "codegraph"
    level: 3
  - title: "基础交互与真实开发流程"
    id: "development-loop"
  - title: "Codex 工作流"
    id: "codex-loop"
    level: 3
  - title: "运行终端命令"
    id: "terminal-commands"
    level: 3
  - title: "权限模式"
    id: "permissions"
    level: 3
  - title: "交付闭环"
    id: "delivery-loop"
    level: 3
  - title: "上下文、多模态与会话管理"
    id: "context-session"
  - title: "项目上下文持久化"
    id: "persistent-context"
    level: 3
  - title: "Resume 与上下文压缩"
    id: "resume-and-compact"
    level: 3
  - title: "AGENTS.md"
    id: "agents-md"
  - title: "AGENTS.md 解决的问题"
    id: "agents-problem"
    level: 3
  - title: "AGENTS.md 应该写什么"
    id: "agents-content"
    level: 3
  - title: "模板"
    id: "agents-template"
    level: 3
  - title: "编码原则"
    id: "coding-principles"
    level: 3
  - title: "MCP：外部工具和上下文接口"
    id: "mcp"
  - title: "MCP、Skill、Hook、Plugin 的区别"
    id: "mcp-skill-hook-plugin"
    level: 3
  - title: "MCP Server 包含什么"
    id: "mcp-server"
    level: 3
  - title: "Skills：把重复流程沉淀下来"
    id: "skills"
  - title: "Skill 目录结构"
    id: "skill-structure"
    level: 3
  - title: "如何写好 description"
    id: "skill-description"
    level: 3
  - title: "blog-update 示例"
    id: "blog-update-example"
    level: 3
  - title: "Subagents：并行探索，写入收敛"
    id: "subagents"
  - title: "Hooks：把规则变成自动检查"
    id: "hooks"
  - title: "Plugins：能力合集与分发"
    id: "plugins"
  - title: "总结"
    id: "summary"
---

这篇文章整理一次 Codex 开发工作流学习笔记。重点不是罗列功能，而是回答一个实践问题：如何把 Codex 从“能聊天写代码”用成一个可持续工作的 AI coding agent。

核心路线可以压缩成一句话：

```text
先有项目规则，再沉淀复用流程，最后打包分发能力。
```

## 读完需要了解什么 {#overview}

读完这篇文章，需要建立以下概念：

- Codex 的基本形态：桌面端、CLI、IDE 插件等入口。
- 使用 Codex 做真实开发的闭环：读代码、计划、修改、运行命令、验证。
- `AGENTS.md` 是面向 AI 的项目级 README。
- MCP 是 Codex 连接外部工具和上下文的标准接口。
- Skill 用于把重复流程沉淀成可复用能力。
- Subagents 适合并行探索和多角度分析。
- Hooks 可以把规则变成自动化检查。
- Plugins 是 Skills、MCP、Hooks 等能力的打包和分发单元。

参考视频：

[Codex 开发程序 一次全部讲明白 - bilibili](https://www.bilibili.com/video/BV15zjF6eEji/?share_source=copy_web&vd_source=38745476c36ec7c4f674100353e0e3f9)

## 认识 Codex 与环境准备 {#codex-environment}

Codex 可以理解为面向软件开发的 coding agent。它不只是生成代码，还可以读项目、执行命令、修改文件、跑测试、检查 diff，并在一轮开发中持续推进。

常见入口包括：

- CLI
- IDE 插件
- 桌面端
- 移动端

首次使用时，应先完成几件事：

1. 安装 Codex 桌面客户端或 CLI。
2. 新建或打开一个项目。
3. 让 Codex 读取项目结构。
4. 明确项目命令、测试方式和工作约束。
5. 对大型项目，可以初始化 CodeGraph 之类的语义索引工具。

### CodeGraph 插件 {#codegraph}

CodeGraph 是本地代码语义索引工具。初始化命令示例：

```bash
codegraph init -i
```

它会把项目源码解析成 `.codegraph/` 索引，记录符号、函数/方法、调用关系和文件结构。Codex 接入后，可以通过一次 MCP 调用拿到相关代码片段和调用链，而不是反复 `rg`、`find`、打开文件逐段阅读。

适用场景：

- 大型代码库理解。
- 查找函数调用链。
- 分析修改影响范围。
- 减少主线程上下文污染。

## 基础交互与真实开发流程 {#development-loop}

把 Codex 用在真实项目里，关键不是让它“一次写完”，而是让它进入工程闭环。

### Codex 工作流 {#codex-loop}

一个可靠的工作流通常是：

1. 读代码：先理解目录、模块、调用关系和已有约束。
2. 制定计划：拆成可执行步骤，明确改哪里、怎么验证。
3. 修改文件：在项目结构里做小而明确的代码变更。
4. 运行命令：执行构建、测试、检查和调试命令。
5. 验证结果：用证据确认结果，再交付结论。

这个流程的价值在于减少“看起来改了，实际上没验证”的交付风险。

### 运行终端命令 {#terminal-commands}

Codex 的关键提升在于它能运行终端命令。也就是说，它可以从“写代码”进化成“能验证代码”的全流程 agent。

可以这样让 Codex 建立项目命令认知：

```text
请查看这个项目有哪些可用脚本，并告诉我测试、构建、启动分别应该运行什么命令。
```

对于陌生项目，不应直接要求 Codex 改代码。先让它识别：

- 依赖安装方式。
- 本地启动命令。
- 测试命令。
- lint / format / build 命令。
- CI 或部署相关约束。

### 权限模式 {#permissions}

Codex 的文件读写、命令执行和网络访问通常受 sandbox 与 approval 控制。权限模式的核心目标是降低误操作风险，尤其是：

- 删除文件。
- 修改系统目录。
- 安装依赖。
- 访问网络。
- 推送远端分支。
- 运行可能有副作用的命令。

![Codex 权限模式](/images/blog/codex-development/permissions.png)

实际使用中，应把权限申请视为风险边界：如果命令会写入系统目录、改 git 状态、访问网络或执行潜在破坏操作，需要明确审批。

### 交付闭环 {#delivery-loop}

完成修改后，不应只给“已完成”的结论。至少应检查：

1. 查看 diff，确认主要改动和是否有无关改动。
2. 运行项目已有测试。
3. 如果没有测试，运行构建或 lint。
4. 如果测试失败，先分析原因，不要盲目继续改。
5. 总结修改文件、原因、验证命令和结果。

可以直接要求 Codex：

```text
请查看这次改动的 diff，告诉我主要修改了哪些地方，有没有无关改动。
请运行项目已有的测试命令，验证这次修改是否通过。
请总结这次修改：改了哪些文件、每个文件为什么要改、运行了什么验证命令、结果是什么。
```

这些要求适合写进 `AGENTS.md`，让每个新会话自动遵守。

## 上下文、多模态与会话管理 {#context-session}

Codex 可以处理文本、截图、图片输入，也可以在上下文不足时进行压缩。但上下文不是无限的，且压缩不是无损的。因此，项目级持久化比“指望会话一直记住”更可靠。

![上下文管理](/images/blog/codex-development/context.png)

### 项目上下文持久化 {#persistent-context}

当新开 Codex 页面时，如果上下文没有带过去，推荐把关键信息落到项目文件里。

常用组合：

```text
AGENTS.md
docs/codex-handoff.md
```

`AGENTS.md` 放项目级说明，例如：

- 项目使用什么环境。
- 如何安装依赖。
- 如何启动、构建、测试。
- 代码风格偏好。
- 不要碰哪些文件。
- 部署方式。

`docs/codex-handoff.md` 放交接记录，例如：

- 最近做了什么。
- 修改了哪些文件。
- 每个修改的作用。
- 当前还有什么未完成。
- 需要注意的坑。

推荐工作流：

```text
每次做完一轮修改：
1. 让 Codex 更新 docs/codex-handoff.md。
2. 记录“改了什么 / 为什么改 / 下一步”。
3. 重要节点提交 git commit。
4. 新页面开始时直接说：先读 AGENTS.md 和 docs/codex-handoff.md。
```

这个仓库已经采用类似机制：`AGENTS.md` 是入口，里面要求 Codex 读取并维护 `docs/codex-handoff.md`。

### Resume 与上下文压缩 {#resume-and-compact}

恢复历史会话后，不要立刻继续开发。更稳妥的提示是：

```text
继续上次的任务。请先回顾当前进度，然后告诉我下一步准备做什么。
```

当上下文容量接近上限时，可以主动要求阶段总结：

```text
请总结目前任务目标、已完成内容、关键决策、剩余问题，方便后面继续。
```

这比完全依赖自动压缩更可控。

## AGENTS.md {#agents-md}

`AGENTS.md` 可以理解为面向 AI 的 README。人类读 `README.md` 理解项目，AI agent 读 `AGENTS.md` 理解项目规则。

![AGENTS.md 示例](/images/blog/codex-development/agents-md.png)

它通常包含构建命令、编码规范、测试要求、安全注意事项、项目结构说明，以及 AI 修改代码时必须遵守的流程。

### AGENTS.md 解决的问题 {#agents-problem}

很多项目的知识和规范只存在于人的脑子里：

- 哪些命令能跑。
- 哪些文件不能碰。
- 哪些目录是生成物。
- 修改后应该怎么验证。
- 项目里有哪些历史坑。

AGENTS.md 的作用是把这些信息写成 AI 能读懂的格式，放在仓库里。目标是形成“打开即理解，改完可验证”的开发体验。

### AGENTS.md 应该写什么 {#agents-content}

核心理念是：

```text
Map, not Manual
```

AGENTS.md 应该像导航地图，不应该写成冗长手册。详细内容可以链接到其他文档。

适合写进去的信息：

- 技术栈、仓库结构、核心模块、分层架构。
- 构建、启动、测试、格式化、质量检查命令。
- 违反会直接出问题的硬性规则。
- 修改文件前后的工作流程。
- 涉及删除文件、安装依赖、修改配置时的审批规则。
- 详细文档索引。

示例结构：

```text
AGENTS.md
  -> docs/architecture.md
  -> docs/development.md
  -> docs/design-docs/ref-*.md
  -> docs/design-docs/*-patterns.md
```

### 模板 {#agents-template}

通用模板：

```markdown
# AGENTS.md

## 1. 项目概述
一段话说清楚项目是什么、技术栈、仓库结构。

## 2. 快速命令
构建、启动、格式化、质量检查的命令速查表。

## 3. 后端架构
包结构树和核心子系统说明。

## 4. 前端架构
技术栈、路由方案、API 层约定、组件库规范。

## 5. 关键约定
5-10 条硬性编码规则。

## 6. 本地开发及验证流程
“改 -> 构建 -> 启动 -> 验证”的完整闭环。

## 7. 质量检查
lint、format、build、test 命令矩阵。

## 8. 参考项目约定
参考项目列表和优先级规则。

## 9. 文档导航
详细文档索引。
```

最小模板：

```markdown
# AGENTS.md

## Project Rules
- 修改前先说明计划。
- 保持最小改动，不做无关重构。
- 修改后运行测试或构建命令。
- 总结修改文件、原因和验证结果。
```

### 编码原则 {#coding-principles}

可以把以下原则写入项目规范，约束 AI 生成代码的方式。

1. 编码前先思考。

   遇到歧义时，要求 Codex 列出假设、提供多种解读，或请求澄清。不要让它默默按错误方向执行。

2. 简洁优先。

   阻止过度工程，不增加无用抽象层，不为了“看起来完整”添加不会发生的边界情况。

3. 精准修改。

   只修改与请求直接相关的代码，不顺手重构、不改无关注释、不调整无关格式。

4. 目标驱动执行。

   给成功标准，而不是模糊命令。例如，不说“修复 bug”，而说“写一个重现该 bug 的测试，然后让它通过”。

## MCP：外部工具和上下文接口 {#mcp}

MCP 是 Model Context Protocol，可以理解为让 Codex 连接外部工具和上下文的标准接口。

一个 MCP server 可以把 GitHub、浏览器、Figma、Sentry、文档搜索、数据库、内部系统等能力暴露给 Codex。Codex 在任务中就可以调用这些工具。

一句话定义：

```text
MCP 是 AI 工具连接外部工具和外部上下文的标准接口。
```

![MCP 示例](/images/blog/codex-development/mcp-example.png)

![已安装 MCP 示例](/images/blog/codex-development/mcp-installed.png)

### MCP、Skill、Hook、Plugin 的区别 {#mcp-skill-hook-plugin}

几个概念可以这样区分：

| 概念 | 作用 |
|---|---|
| Skill | 教 Codex 怎么做某类任务 |
| Hook | 在 Codex 生命周期节点自动跑脚本 |
| MCP | 给 Codex 接入外部工具、数据和动作能力 |
| Plugin | 打包 Skills、MCP servers、Hooks 等能力 |

### MCP Server 包含什么 {#mcp-server}

MCP 通常包含：

```text
Server: 服务本体
Tools: 可调用函数
Resources: 可读资源
Resource Templates: 参数化资源
Prompts: 可复用提示词
Auth: 认证方式
Transport: 通信方式
Config: 客户端配置
```

最关键的是三件事：

```text
server 怎么启动/连接
tools 有哪些
每个 tool 的输入输出 schema 是什么
```

参考：

- [What is the Model Context Protocol?](https://modelcontextprotocol.io/docs/getting-started/intro)
- [Model Context Protocol - Codex - OpenAI Developers](https://developers.openai.com/codex/mcp)

## Skills：把重复流程沉淀下来 {#skills}

Skill 的价值在于把重复任务从一次性提示词变成可维护的流程文件。

它适合解决这类问题：

- 长提示词占上下文，且每次粘贴都不稳定。
- 一个流程会反复使用，但每次输入变量不同。
- 想把个人经验沉淀为可复用能力。
- 想让多个 agent 或多个会话共享同一套流程。

Skill 的四个优势：

1. 减轻上下文容量压力，按需加载。
2. 利用文件系统做持久化。
3. 支持协同工作流。
4. 可以持续维护，积累经验。

参考：

- [吃透 Skill 实战：从新手入门到自动写作实战 - bilibili](https://www.bilibili.com/video/BV1no7p65E6r/?spm_id_from=333.1387.homepage.video_card.click&vd_source=b8e4081a996779a4645bea84fdcd2a69)
- [一篇吃透 Skill 实战：从新手入门到自动写作实战 - 技术胖](https://jspang.com/article/47)

### Skill 目录结构 {#skill-structure}

最小结构：

```text
skill-name/
└── SKILL.md
```

完整结构可以包含：

```text
skill-name/
├── SKILL.md
├── scripts/
├── references/
└── assets/
```

![Skill 目录结构](/images/blog/codex-development/skill-structure.png)

各部分职责：

- `SKILL.md`：核心说明文件，包含名称、描述、触发条件、执行步骤和标准。
- `scripts/`：放可执行脚本，用于稳定执行重复逻辑。
- `references/`：放模板、API 说明或详细规则，按需读取。
- `assets/`：放静态资源，如模板文件、图片、音乐、CSS 等。

### 如何写好 description {#skill-description}

`description` 是 Skill 是否能被正确触发的关键。推荐公式：

```text
description = 功能定义 + 触发场景 / 触发词
```

示例：

```yaml
name: blog-update
description: 当用户要求把新的 Markdown 内容整理成个人主页博客、预览本地效果、确认后提交并推送到远端主分支时使用。适用于当前 Jekyll / GitHub Pages 学术主页仓库。
```

这个描述同时说明了：

- 做什么：更新博客。
- 什么时候触发：用户要求把 Markdown 整理成博客。
- 作用范围：固定主页仓库。
- 工作流：预览、确认、提交、推送。

### blog-update 示例 {#blog-update-example}

我自己的 `blog-update` Skill 存储在：

```text
~/.codex/skills/blog-update/SKILL.md
```

它是用户级全局 Skill，但内容针对当前博客仓库定制。

使用方式：

```text
使用 blog-update，把 /path/to/article.md 整理成博客。
```

或自然描述：

```text
把这份 md 整理成我的主页博客，本地预览确认后再推送。
```

效果示例：

![blog-update Skill 示例](/images/blog/codex-development/blog-update-skill.png)

## Subagents：并行探索，写入收敛 {#subagents}

Subagents 是 Codex 的子代理机制。主 Codex 线程可以按明确要求启动多个专门 agent 并行工作，再汇总结果。

![Subagents 示例](/images/blog/codex-development/subagents.png)

适合场景：

- 代码审查。
- 日志分析。
- 文档总结。
- 项目理解。
- 大代码库定位。
- 多方向 PR review。

不适合场景：

- 很小的单点修改。
- 需要严格顺序推理的任务。
- token 成本敏感的任务。
- 还没明确每个 agent 负责什么。

原则是：

```text
并行探索，写入收敛。
```

示例 prompt：

```text
请为当前 PR 做审查。启动一个 agent 负责安全问题，一个 agent 负责 bug，一个 agent 负责测试缺口，一个 agent 负责可维护性。等待所有 agent 完成后汇总结果。
```

Codex 内置几类 agent：

- `default`：通用 fallback。
- `worker`：偏执行、实现、修复。
- `explorer`：偏只读探索、代码路径梳理。

也可以在项目中自定义：

```text
.codex/agents/reviewer.toml
```

参考：

[Subagents - Codex - OpenAI Developers](https://developers.openai.com/codex/subagents)

## Hooks：把规则变成自动检查 {#hooks}

Hooks 是 Codex 的生命周期扩展机制。它可以在关键时刻自动运行脚本，例如用户提交 prompt 前、工具调用前后、请求权限时、会话开始时、上下文压缩前后、subagent 启停时等。

它不是 Git hooks，而是 Codex agent loop 里的 hooks。

常见用途：

- 在 `PreToolUse` 阶段拦截危险命令，比如 `rm -rf`。
- 在 `PermissionRequest` 阶段自动批准或拒绝某类权限请求。
- 在 `PostToolUse` 阶段检查命令输出。
- 在 `UserPromptSubmit` 阶段扫描用户 prompt，防止粘贴 API key。
- 在 `SessionStart` 阶段注入项目约定或上下文。
- 在 `Stop` 阶段要求 Codex 再跑一轮检查。

配置位置：

```text
~/.codex/hooks.json
~/.codex/config.toml
<repo>/.codex/hooks.json
<repo>/.codex/config.toml
```

项目级 hooks 适合放在仓库 `.codex/` 下；个人通用 hooks 适合放在 `~/.codex/` 下。

最小示例：

```toml
[[hooks.PreToolUse]]
matcher = "^Bash$"

[[hooks.PreToolUse.hooks]]
type = "command"
command = '/usr/bin/python3 "$(git rev-parse --show-toplevel)/.codex/hooks/pre_tool_use_policy.py"'
timeout = 30
statusMessage = "Checking Bash command"
```

对应脚本会从 `stdin` 收到 JSON，包含当前事件名、工作目录、工具名、工具输入等。脚本可以返回 JSON 来允许、拒绝、改写或补充上下文。

重要限制：

- 当前真正执行的是 `type = "command"`。
- 多个匹配 hooks 可能一起运行，不能假设一个 hook 能阻止另一个 hook。
- `PostToolUse` 是事后检查，不能撤销已发生副作用。
- `PreToolUse` 是 guardrail，不是完整安全边界。

参考：

[Hooks - Codex - OpenAI Developers](https://developers.openai.com/codex/hooks)

## Plugins：能力合集与分发 {#plugins}

Plugin 可以理解为 Codex 能力合集。相比单个 Skill，Plugin 更适合分发一整套工作流。

它可以打包：

- Skills
- MCP servers
- Hooks
- 其他配置和资源

可以简化理解为：

```text
Plugin = Skill + MCP + Hook + 分发元信息
```

示例结构：

```text
team-code-review/
└── .codex-plugin/
    ├── plugin.json
    └── skills/
        └── code-review/
            └── SKILL.md
```

如果团队已经有稳定流程，例如代码审查、CI 修复、发布检查，就适合做成 Plugin，方便安装、复用和标准化。

## 总结 {#summary}

一个新项目使用 Codex 开发，可以按下面顺序建设：

1. `AGENTS.md`：解决项目规则。
2. `docs/codex-handoff.md`：解决跨会话交接。
3. Skills：解决流程复用，把常用步骤沉淀成方法。
4. MCP：解决外部工具和上下文接入。
5. Subagents：解决并行探索和多角度分析。
6. Hooks：解决自动化检查和安全约束。
7. Plugins：解决能力分发，把工具和流程打包给生态。

最终目标不是“让 AI 一次性生成更多代码”，而是建立一套可验证、可复用、可交接的 AI agent 开发流程。
