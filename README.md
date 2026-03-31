# Claude code 源码分析

## 事件背景

2026年3月31日，安全研究者 [Chaofan Shou](https://x.com/Fried_rice) 发现 Anthropic 发布到 npm 的 Claude Code 包中，官方没有删除source map 文件, 这意味着 Claude Code 的完整 TypeScript 源码，51.2万行，1903个文件，全部暴露在公网上.

## 总述

本目录中的分析文档基于 `src/` 源码静态阅读整理，目标是从“总分总”的结构回答以下核心系统设计问题：

1. 这个项目的软件架构是什么，程序从哪里启动。
2. 从用户视角看，项目收集了哪些信息（Telemetry），并如何使用这些信息。
3. 用户若希望规避或降低隐私信息收集，应如何做。
4. Agent Memory 机制体系（含多层级存储与 Session Compaction 打薄压缩逻辑）是怎么实现的。
5. Agent 极具特色的能力扩充机制（如 Skills 技能扩展、Tool Call 调用链、MCP 通信协议集成）的技术细节与运行方式分别是什么。
6. 针对大模型本地操作风险，系统是如何应对系统级逃逸并利用 Sandbox（沙盒隔离）等机制机制保护宿主机的。
7. 该项目自身的程序架构亮点有哪些。
8. 对 `src/components/` 基于 React 的 TUI 控制台中各组件族、关键子组件以及叶子组件进行专项拆解。
9. 该项目与 `Codex`、`Gemini CLI`、`Aider`、`Cursor` 等同类产品相比，有哪些显著的代差与体验差异。
10. 相关结论分别对应哪些源码证据和外部公开资料。

总判断先放在这里：

- 该项目绝不仅是一个简单的命令行聊天工具，而是一套强隔离、底层闭环的**本地代码 Agent 平台**。
- 它最突出的特征是：统一的执行内核、多态分层的 Memory 系统、以及基于 MCP / Skills 的强悍平台化外接扩展能力。
- 从架构防御侧看，它利用原生 Sandbox 隔离与细粒度 Tool Permission 拦截建立了精密的防线来保障宿主机的运行安全。
- 从用户隐私视角看，真正的风险焦点并不在于单一的数据打点 (Telemetry)，而是“庞大的模型上下文流出 + 本地持久化记忆库 + 外部组件的远程通信能力”组合后的信息发散边界。

## 一图总览

```text
+---------------------------+
| CLI / 多入口              |
| entrypoints/cli.tsx       |
| main.tsx                  |
+---------------------------+
            |
            v
+---------------------------+
| 初始化与运行环境          |
| init.ts / setup.ts        |
+---------------------------+
      |                |
      v                v
+----------------+   +---------------------------+
| 命令与控制面   |   | TUI / REPL 工作台         |
| commands.ts    |-->| App / REPL / Messages     |
| PromptInput... |   | / PromptInput             |
+----------------+   +---------------------------+
                             |
                             v
                  +---------------------------+
                  | Query / Agent 执行内核    |
                  | query.ts / QueryEngine.ts |
                  +---------------------------+
                    |           |           |
                    v           v           v
          +---------------+ +--------------------+ +----------------------+
          | Tool/Perm     | | Transcript/Memory  | | 平台扩展层           |
          | Tool.ts       | | sessionStorage     | | MCP/Plugin/Remote/   |
          | orchestration | | memdir/SessionMem  | | Swarm                |
          +---------------+ +--------------------+ +----------------------+
                    \______________   |   ______________/
                                   \  |  /
                                    \ v /
                              回流到执行内核
```

## 分章目录

### 第一部分：总体架构

- [第一章：软件架构与程序入口](./analysis/01-architecture-overview.md)

### 第二部分：用户信息与隐私

- [第二章：从用户角度看，项目收集了哪些信息，以及如何使用](./analysis/02-user-data-and-usage.md)
- [第三章：从用户角度看，如何规避或降低信息收集](./analysis/03-privacy-avoidance.md)

### 第三部分：核心机制

- [第四章：Agent Memory 机制是怎么做的](./analysis/04-agent-memory.md)
- [第五章：Skills 的技术实现细节与运行方式](./analysis/04c-skills-implementation.md)
- [第六章：Tool Call 机制实现细节](./analysis/04b-tool-call-implementation.md)
- [第七章：MCP 技术实现细节与运行机制](./analysis/04d-mcp-implementation.md)
- [第八章：Sandbox 技术实现细节与运行机制](./analysis/04e-sandbox-implementation.md)

### 第四部分：程序架构及亮点

- [第九章：程序架构及亮点](./analysis/05-differentiators-and-comparison.md)

### 第五部分：扩展分析

- [第十章：额外探索与补充发现](./analysis/06-extra-findings.md)
- [第十一章：隐藏命令、Feature Flags 与彩蛋](./analysis/11-hidden-features-and-easter-eggs.md)

### 第六部分：组件体系详解

- [组件详解（一）：组件总览、分层与依赖主干](./analysis/components/01-component-architecture-overview.md)
- [组件详解（二）：核心交互组件与消息/输入主链路](./analysis/components/02-core-interaction-components.md)
- [组件详解（三）：平台能力组件与控制面实现](./analysis/components/03-platform-components.md)
- [组件详解（四）：组件索引、长尾组件与目录映射](./analysis/components/04-component-index.md)
- [组件详解（五）：核心组件函数级实现拆解](./analysis/components/05-function-level-core-walkthrough.md)
- [组件详解（六）：平台控制面函数级实现拆解](./analysis/components/06-function-level-platform-walkthrough.md)
- [组件详解（七）：叶子组件与子函数实现拆解](./analysis/components/07-function-level-leaf-walkthrough.md)

### 第七部分：同类产品对比

- [第十二章：同类产品对比](./analysis/08-competitive-comparison.md)
- [附录A：外部对比资料](./analysis/08-reference-comparison-sources.md)

### 第八部分：证据与资料

- [第十三章：代码证据索引](./analysis/07-code-evidence-index.md)
- [附录B：src 详细文件树（含文件说明）](./analysis/10-src-file-tree.md)

### 第九部分：总结

- [第十四章：总结结论](./analysis/09-final-summary.md)
