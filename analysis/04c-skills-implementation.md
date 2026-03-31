# Skills 机制实现细节与运行方式

[返回总目录](../README.md)

Claude Code 通过一整套强大的本地平台化扩展能力来增强 Agent 模型的功能，这其中的核心就是 **Skills（技能）机制**。它的重要性在于能够通过标准化前端定义（Frontmatter）和 Markdown 文档，将外部程序的执行结果和静态信息暴露为模型在运行时的知识或命令。

根据静态代码分析，主要负责加载和运行逻辑的文件为 [`src/skills/loadSkillsDir.ts`](../src/skills/loadSkillsDir.ts)、[`src/skills/bundledSkills.ts`](../src/skills/bundledSkills.ts)。以下是技能发现、加载与运行方式的技术实现细节。

## 1. Skills 的三种主要形态与来源

在架构上，技能主要分为三种来源和形态，系统会分别识别和加载：

1. **File-based Skills（文件系统技能）：**
   此类型的技能主要是基于本地目录的 Markdown 文件定义。发现路径包括：
   - 策略管理与配置目录（Policy & Managed settings）：如 `~/.claude/skills` 等全局范围目录。
   - 用户目录：全局配置相关的 `./claude/skills`。
   - 工程项目目录：项目内部 `.claude/skills` 及旧版本的 `/commands/` 目录。
   - 附加指定目录：通过启动参数 `--add-dir` 指定的第三方目录。

2. **Bundled Skills（内建打包技能）：**
   这部分技能并不暴露为明显的外部配置文件，而是存在于系统源码自身中（即硬编码与内置代码绑定），由系统通过 [`BundledSkillDefinition`](../src/skills/bundledSkills.ts) 在内存中硬核注册，例如某些系统级别的状态查看和高权限文件操作等。

3. **MCP Skills（协议映射技能）：**
   基于 MCP（Model Context Protocol）连接远程服务器提取的功能被抽象和动态构建为模型可以随时执行的类 Skill API。详见 [`src/skills/mcpSkills.ts`](../src/skills/mcpSkills.ts)。

## 2. Skills 的发现与加载工作流

对于占主导地位且开发者最常接触的 File-based Skills 而言，它的发现与加载有严密的逻辑（位于 [`src/skills/loadSkillsDir.ts`](../src/skills/loadSkillsDir.ts)）：

### (1) 目录爬取与结构约定

- 系统入口函数为 [`getSkillDirCommands`](../src/skills/loadSkillsDir.ts)，负责收集所有符合约定的路径并汇总所有技能目录下的可用命令列表。
- 所有技能必须遵循特定目录结构：每一个独立目录即为一个技能槽，且只读取其中名为 `SKILL.md` 的入口文件（在一些 Legacy 场景中支持单文件 `*.md`，但新版规范要求目录化存放）。

### (2) 解析元数据 (Frontmatter 解析)

每个加载到的核心入口文件（`SKILL.md`）使用标准的 YAML 前置语言（Frontmatter）配置相关属性。内置模块 [`parseFrontmatter`](../src/skills/loadSkillsDir.ts) 会读取这些变量。目前核心参数包括：

- `name` / `description` / `when_to_use`：这是暴露给大模型用于决策是否使用以及何时使用核心逻辑的基础。
- `allowed-tools`：这表明该技能有权调用的系统/内置 Tool。
- `model`：用于指定运行该技能时的底座模型（是否默认 fallback 或指定新模型）。
- `effort`：任务估时。
- `user-invocable`：定义该请求是否可以由用户从客户端 CLI / REPL 界面隐式执行，或是专属于大模型的背后逻辑。
- `paths`：极其重要的条件触发参数。若存在该参数，则该技能会被判定为 **Conditional Skill（条件技能）**。在用户修改甚至阅读对应目录内的该后缀/命名文件时，这部分技能被自动激活触发并提供上下文和预检操作。

### (3) 安全判重 (Deduplication)

由于可能在通过 `~/.claude` 和当前项目 `.claude` 结构时载入不同位置的文件系统和软连映射关系，该加载逻辑使用了底层操作系统级别的身份验证（通过 Node.js `fs.realpath` 来避免 inode 误判）来规避软链接或目录重复导致的重复加载问题。一旦遇到相同的路径，优先加载的作为准则。详见 [`getSkillDirCommands`](../src/skills/loadSkillsDir.ts) 中的去重集合逻辑。

## 3. Skills 的运行与变量注入机制

在 [`src/skills/loadSkillsDir.ts`](../src/skills/loadSkillsDir.ts) 内，由 [`createSkillCommand`](../src/skills/loadSkillsDir.ts) 函数负责将解析后的 Skill 定义实例化为一个可执行的闭包命令对象，并挂入全局命令系统。运行机制有着非常清晰的设计模式：

1. **变量占位符展开：**
   执行前夕，`${CLAUDE_SKILL_DIR}` 和 `${CLAUDE_SESSION_ID}` 这样的内置常量会被即时替换。通过这些常量，内置技能可以在同级目录使用子脚本/二进制运行相关任务。

2. **本地 Shell 强执行注入 (Inline Bash Compilation)：**
   非常核心的一环是一项名为"Shell Commands In Prompt"的技术封装。
   Claude Code 的 Skill 内容里可以预留 Bash 指令行代码（类似于 `! <shell command>` 的设计）。在将最终的技能 Prompt 交由大模型之前，通过内置函数 [`executeShellCommandsInPrompt`](../src/skills/loadSkillsDir.ts) 会先用本地宿主机 Bash 去执行这些脚本获取标准输出流。随后将标准输出直接置换在指令体内，使得大模型拿到的 Prompt 就直接带有当前系统上下文结果。
   - **安全切断机制：** 这一过程只对文件和内置打包来源起作用。由于 MCP Server 的安全性并不处于同一安全信任区，此机制在 MCP Skills 内强硬规避（跳过提取执行）以杜绝 RCE（Remote Code Execution）攻击层面。

3. **动态磁盘提权（Bundled 特性）：**
   对于内建打包技能（Bundled Skill）而言，存在类似 `files` 参数提取的操作。系统会在第一次成功运行此类要求时动态调用 [`extractBundledSkillFiles`](../src/skills/bundledSkills.ts) 在宿主机的应用层缓存目录内将相关内联代码脚本通过 `fs` 直接按属主权限落地成为实际文件。从而让大模型不仅能靠 Prompt 调用，还能靠 `read_file` 甚至是 `grep_search` 等默认工具对这个技能进行后续挖掘分析。

## 4. 结论

通过解析技能管理系统我们可以发现：
1. Claude Code 并不将扩展功能仅仅作为 API 并入，而是提供了一个类似轻量级 Serverless 系统的平台执行层模式。
2. 将 **变量注入 + 本地隐式 Shell 执行 + 模型预提示词组合** 三位一体进行合并。这极大降低了二次开发的门槛，开发方通过少量 Markdown 和 Bash 即可实现强大的复杂插件扩展。
3. 充分使用了条件技能（`paths`）实现精准分发的 Hook 订阅模式，避免了传统 agent 面对大规模知识库时的认知过载。
