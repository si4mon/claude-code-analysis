# Claude Code Agent Runtime 的 Harness Engineering 分析（基于 `src/`）

## 分析范围与方法

- 代码范围：`https://github.com/si4mon/claude-code-analysis/tree/main/src` 对应仓库快照中的 `src/`。
- 目标：围绕 Harness Engineering（你给出的 6 个核心组件）分析 Claude Code 运行时如何提升 agent 稳定性与可控性。
- 方法：以主循环（`query.ts`）为骨架，串联工具系统（`Tool.ts`、`tools.ts`、`toolExecution.ts`、`toolOrchestration.ts`）、上下文压缩、状态持久化（`sessionStorage.ts`、`history.ts`）与停止钩子（`query/stopHooks.ts`）。

---

## 一、代码仓中的 Harness 优秀实践（按 6 个核心点映射）

## 1) 推理引擎（模型选择 / 算力分配 / 推理历史）

### 1.1 运行时模型动态选择 + 回退机制（稳定性核心）

`query.ts` 在每轮根据权限模式、token 规模、fallback 状态选择模型：

```ts
let currentModel = getRuntimeMainLoopModel({
  permissionMode,
  mainLoopModel: toolUseContext.options.mainLoopModel,
  exceeds200kTokens:
    permissionMode === 'plan' &&
    doesMostRecentAssistantMessageExceed200k(messagesForQuery),
})
```

并在 `FallbackTriggeredError` 时执行“清理中间态 -> 切模型重试 -> 输出系统提示”的恢复链，避免半成品 thinking/tool_use 污染后续轮次。

**价值点**：把“模型不稳定/拥堵”从致命错误降级为可恢复路径，降低任务中断率。

### 1.2 多重预算体系（token_budget + task_budget）

- `task_budget`：跨压缩边界跟踪 `remaining`，避免 compact 后服务器“看不到历史上下文成本”导致预算失真。
- `token_budget`：在 stop hooks 后进行继续/终止决策，必要时注入 meta nudge 继续完成任务。

**价值点**：不是“单次请求预算”，而是“agentic turn 生命周期预算”，更符合复杂任务现实。

### 1.3 推理历史与遥测链路

`queryTracking`（`chainId + depth`）在主循环内持续透传到日志事件与工具执行路径，形成跨轮、跨工具的一致观测域。

**价值点**：提升故障定位与行为归因能力，是大规模线上稳定性的基础设施。

---

## 2) 计划与编排（主循环控制 / 停止条件 / 分解 / 多智能体协调）

### 2.1 显式 query 主循环 + 状态对象化

`query.ts` 采用 `while(true)` + `State` 的方式，明确把跨迭代状态集中（messages、recovery 次数、transition 原因、pending summary 等），并通过 `continue` 切换路径。

**优秀点**：虽然不是“纯状态机 DSL”，但已经接近可测的状态机风格（`transition.reason` 明确记录继续原因）。

### 2.2 多级停止条件设计（不是单点 stop_reason）

系统停止条件覆盖：

- 无 tool_use 时正常完成
- stop hooks 阻断
- max turns 到达
- prompt too long / media error 恢复失败
- user abort（streaming/tools 两路径）
- token budget 到达策略阈值

并且对 `stop_reason === 'tool_use'` 的不可靠性做了显式规避（改为流式捕获 tool_use block）。

**价值点**：把“停止”从 API 字段依赖升级为运行时语义判断，显著提升鲁棒性。

### 2.3 多智能体/多线程编排中的作用域隔离

在队列消费中，主线程与子 agent 仅消费各自可见命令（`agentId` 过滤）；stopHooks 里也对“仅主线程执行某些清理”有明确约束。

**价值点**：减少多 agent 并发导致的串扰与锁误释放。

---

## 3) 工具注册表（可见性 / 描述 / 命名 / 暴露时机）

### 3.1 工具池分层组装 + 权限前置过滤

`tools.ts` 的 `assembleToolPool()` 将内置工具与 MCP 工具合并，并先执行 deny 规则过滤，再做去重与排序（保证 cache 稳定性）。

**价值点**：把“工具可见性”前移到注册阶段，而非执行时报错，降低模型误用概率与 token 噪音。

### 3.2 动态工具暴露（ToolSearch + defer loading）

`services/api/claude.ts` 中：

- 先判定是否启用 tool search
- deferred tool 仅在被“发现”后才真正进入 schema
- 不支持 tool search 的模型会剥离相关字段

**价值点**：控制首轮提示体积、减少 schema 负担，同时支持 MCP 动态接入。

### 3.3 工具契约规范化（Tool 接口）

`Tool.ts` 定义了输入 schema、权限检查、并发安全、可破坏性、是否 deferred、maxResultSize 等统一契约。

**价值点**：工具生态在“统一运行时规则”下扩展，避免 tool-by-tool 的行为漂移。

---

## 4) 内存与上下文管理（可见信息 / 顺序 / 密度）

### 4.1 上下文“多层压缩管线”

主循环里按顺序执行：

1. `applyToolResultBudget`（超大 tool result 替换）
2. `snip`（历史剪裁）
3. `microcompact`
4. `contextCollapse`
5. `autocompact`
6. 异常时 `reactiveCompact` 恢复

**价值点**：不是“单压缩器”，而是按场景分层治理，能在保留信息与控 token 之间动态平衡。

### 4.2 预取并行化（memory/skill prefetch）

memory prefetch 与 skill discovery prefetch 在流式响应阶段并行，后置消费。

**价值点**：把本来阻塞主路径的 IO/检索延迟隐藏到模型思考时间里，提升体感性能。

### 4.3 错误消息 withheld 机制

对可恢复错误（PTL/max_output/media）先“暂缓外显”，优先尝试恢复，再决定是否对外暴露。

**价值点**：避免上层客户端（收到 error 就终止）与底层恢复逻辑冲突。

---

## 5) 状态与持久化（持久化内容 / 形式 / 检索时机）

### 5.1 JSONL 事件流式持久化 + 会话路径体系

`sessionStorage.ts` 通过 transcript 路径、subagent 路径、chain participant 规则管理持久化粒度；并明确 progress 不进入链。

**价值点**：保证恢复时链路一致性，避免“进度消息污染 parentUuid 链”。

### 5.2 Prompt 历史的增量写入与并发锁

`history.ts` 使用 lock + append JSONL，并把大 pasted text 通过 hash 外置到 paste store，兼顾性能与体积。

**价值点**：持久化成本与可检索性兼顾，避免 history 文件无限膨胀。

### 5.3 面向恢复场景的补丁式修复

代码中可见多个“resume/remote 场景修复”策略（如 tool_use/result pairing 修复、边界/链条桥接），体现运行时对真实线上脏状态的工程化兜底。

---

## 6) 结构化输入输出（可验证、可信任、可复用）

### 6.1 强输入校验（Zod）+ 错误回传规范

`toolExecution.ts` 对 tool input 做 `safeParse`，失败时返回结构化 `tool_result(is_error=true)`，并附带可恢复提示（如 deferred tool schema 未下发时引导先 ToolSearch）。

### 6.2 前后处理一致性

- `normalizeMessagesForAPI`
- `ensureToolResultPairing`
- 模型不支持时剥离 tool_search 相关字段

**价值点**：把“消息能不能被 API 接受”变成可预处理问题，而非在线随机失败。

### 6.3 工具执行中的语义防护

工具调用前有 `validateInput`、`checkPermissions`、hooks；执行中支持 streaming executor；执行后统一产出可追踪消息。

**价值点**：I/O 结构化不仅在“格式”，更在“执行语义可验证”。

---

## 二、仓库中相对缺少的 Harness 实践（及系统影响）

> 说明：这里的“缺少”是相对企业级 agent runtime 的更高成熟度目标，不是说当前实现不可用。

### A. 缺少“显式可视化状态机/策略表”

当前主循环已经很强，但仍以 `while(true)+多个 continue 分支` 承载复杂恢复逻辑。`transition.reason` 虽然存在，但尚未上升为统一状态机定义（例如状态图、转移表、守卫条件统一声明）。

**影响**：

- 新增恢复分支时更易出现分支交互回归。
- 行为验证偏“集成回归”，缺少“状态转移级”可证明性。

### B. 持久化错误大量采用 best-effort / 日志吞错

在历史与部分会话存储路径中，多处采用失败记录日志后继续（包括 fire-and-forget）。

**影响**：

- 极端场景下会出现“用户以为已持久化，实际未落盘”的可恢复性缺口。
- 线上故障可能转化为“慢性一致性问题”，不易第一时间暴露。

### C. 多智能体编排缺少更高层调度语义

当前有 agentId 作用域隔离与队列优先级快照，但尚未看到“全局任务级公平性/抢占策略/背压治理”的统一调度层（更像进程内策略组合）。

**影响**：

- 当 agent 数量、MCP 工具数量、长任务并发继续增长时，可能出现资源竞争导致的尾延迟上升。
- 难以对“跨 agent SLA”做稳定承诺。

### D. 缺少统一的“长期记忆质量回路”指标体系

已有 memory prefetch/attachment/filter 机制，但尚未看到“记忆命中率、错误召回率、过期记忆污染率”的闭环指标与自动治理策略。

**影响**：

- 长会话中记忆质量可能随时间衰减，且问题定位需要人工排查。

---

## 三、综合评估：每个 Harness 组件的技术价值、成熟度、可吸收点

| Harness 组件 | 当前技术价值 | 成熟度判断 | 建议吸收点 |
|---|---|---|---|
| 推理引擎 | 动态选模 + fallback + 预算控制，具备生产级韧性 | 高 | 学习“恢复优先”而非“报错即终止” |
| 计划与编排 | 主循环覆盖大量异常/恢复路径，stop hooks 深度集成 | 高 | 学习 `transition reason` 的可观测编排 |
| 工具注册表 | 契约统一 + 动态暴露 + 权限前置过滤 | 高 | 学习 defer loading + ToolSearch 的规模化设计 |
| 内存与上下文管理 | 多层压缩与恢复策略组合，实践先进 | 高 | 学习“分层压缩管线”而非单摘要 |
| 状态与持久化 | JSONL + session/subagent 路径 + 恢复修复，工程经验丰富 | 中高 | 学习 transcript chain 一致性治理 |
| 结构化 I/O | 输入校验、消息修复、错误结构化回传完整 | 高 | 学习“格式+语义”双层结构化 |

### 可新增的第 7 个 Harness 维度（建议）

**7）可观测性与恢复工程（Observability & Recovery Engineering）**

Claude Code 在这个维度非常突出：

- queryChainId/queryDepth
- 大量 logEvent/telemetry 点位
- withheld error + 多级恢复
- tombstone 清理 orphan 消息

这在复杂 agent 系统中几乎与“计划编排”同等重要，建议单独作为 harness 设计维度。

---

## 四、给你这个分析任务的落地建议（如何用于你的架构设计）

1. **先复制骨架，不先复制工具数量**：优先复用 query-loop + recoveries + structured I/O，再扩展工具生态。  
2. **把“错误恢复路径”作为一等公民测试**：不仅测 happy path，还要测 413、max tokens、fallback、abort、resume。  
3. **工具注册与可见性要解耦**：让“可调用工具集合”与“初始暴露 schema”分离（defer + search）。  
4. **上下文管理做成策略管线**：snip/microcompact/collapse/autocompact/reactive 各司其职。  
5. **持久化先保链路一致性，再追求完美可用性**：宁可少存临时进度，也不要破坏可恢复主链。

