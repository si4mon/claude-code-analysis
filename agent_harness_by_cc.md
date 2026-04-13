# Claude Code Agent Runtime：Harness Engineering 完整性源码分析（深度映射版）

> 你提到“上版太概念化，不够代码映射”。本版按 **关键运行时链路** 做“逐段（line-range）解释”，尽量回答：
> 1) 这一段代码在做什么；
> 2) 其 Harness Engineering 作用是什么；
> 3) 若缺失会造成什么风险。

---

## 0. 分析边界与方法说明

### 0.1 边界
- 本分析基于仓库 `src/` 快照（不是 Anthropic 内部全量仓库）。
- “完整性”按 **runtime 主路径完整性** 来做：主循环、模型调用、工具执行、上下文压缩、状态持久化、stop hooks、工具注册。

### 0.2 方法
- 以 `src/query.ts` 为中枢，沿调用关系做剖析：
  - `query.ts`（主循环与恢复）
  - `services/api/claude.ts`（API 请求编排与工具 schema 暴露）
  - `Tool.ts` / `tools.ts`（工具契约与注册）
  - `services/tools/toolOrchestration.ts` / `toolExecution.ts`（并发与执行安全）
  - `history.ts` / `utils/sessionStorage.ts`（状态与持久化）
  - `query/stopHooks.ts`（停止策略）
  - `utils/model/model.ts`（模型选择）

---

## 1. Runtime 主循环逐段解剖（`src/query.ts`）

## 1.1 循环骨架：状态容器 + 无限循环

**代码段**：`queryLoop` 初始化与 `State` 装载（L241-L307）。

- `state` 把跨轮次可变项集中：`messages/toolUseContext/recovery计数/transition` 等。  
- `while (true)` 明确这是一个“多轮 agent 回合执行器”，不是单次请求函数。  
- `taskBudgetRemaining` 与 `budgetTracker` 表明预算是跨轮持有状态，不是单 request 变量。

**Harness 作用（计划与编排）**：
- 这是典型 harness 主循环框架：统一状态、统一退出路径、统一重试入口。

---

## 1.2 每轮前置并行：memory/skill prefetch

**代码段**：`startRelevantMemoryPrefetch` + `startSkillDiscoveryPrefetch`（L297-L335）。

- memory prefetch 用 `using` 生命周期管理，保证退出即释放。
- skill prefetch 每轮触发，预期在模型流式返回期间“藏延迟”。

**Harness 作用（内存与上下文）**：
- 将 IO 延迟并入模型思考时间，减少主路径阻塞。

---

## 1.3 观测链：queryTracking

**代码段**：`queryTracking = {chainId, depth}`（L346-L363）。

- 首轮创建 chainId，后续 `depth+1`。
- 后续日志、工具执行、错误统计都带这个链路 ID。

**Harness 作用（推理引擎 + 可观测性）**：
- 让“一个用户请求触发的多次模型调用/工具执行”可追踪。

---

## 1.4 上下文治理总管线（关键）

**代码段**：L369-L447 + L453-L543。

按顺序：
1. `applyToolResultBudget`（L379）
2. `snipCompactIfNeeded`（L401-L409）
3. `microcompact`（L413-L426）
4. `contextCollapse.applyCollapsesIfNeeded`（L440-L446）
5. `autocompact`（L453-L466）
6. compact 成功后 `buildPostCompactMessages` 并 yield（L528-L535）

**逐点解释**：
- `applyToolResultBudget` 先处理超大 tool result，避免后续压缩器处理“无意义大块”。
- `snip` 和 `microcompact` 都是细粒度裁剪；`contextCollapse` 是“折叠视图”；`autocompact` 是重压缩兜底。
- compact 失败会把 `consecutiveFailures` 写回 tracking（L536-L542），相当于断路器输入。

**Harness 作用（内存与上下文管理）**：
- 不是“一个摘要器”而是“多级策略链”，对稳定性远优于单一 compact。

---

## 1.5 请求调用阶段：模型、权限、工具集一次性打包

**代码段**：L570-L707。

- 运行时模型从 `getRuntimeMainLoopModel` 得到（L572）。
- `deps.callModel` 传入：
  - `thinkingConfig`
  - `tools`
  - `getToolPermissionContext`
  - `fallbackModel`
  - `agents/allowedAgentTypes`
  - `mcpTools/hasPendingMcpServers`
  - `taskBudget.remaining`（如有）

**Harness 作用（推理引擎 + 工具注册表）**：
- 每轮是“模型+权限+工具+预算”的原子组合调用，减少跨模块状态漂移。

---

## 1.6 流式阶段稳定性：orphan 清理 + 错误 withheld

**代码段**：L712-L741（fallback 清 orphan）；L788-L825（withhold recoverable errors）。

- 流式 fallback 后会 tombstone 已产生但无效的 assistant partial（L713-L718）。
- 对 413/max_output/media 错误先 withheld，不立刻向上游输出。

**Harness 作用（结构化 I/O + 恢复工程）**：
- 防止上游 SDK 看到临时错误后提前结束，会破坏后续恢复流程。

---

## 1.7 非可靠 stop_reason 的替代机制

**代码段**：L553-L557 + L829-L835。

- 代码明确写了 `stop_reason === 'tool_use'` 不可靠。
- 改用“流中真实解析到 tool_use block”作为继续条件。

**Harness 作用（计划与编排）**：
- 从“信 API 字段”转为“信运行时事实”，鲁棒性显著提升。

---

## 1.8 fallback 重试链

**代码段**：L893-L950。

- 触发 `FallbackTriggeredError` 后：
  1) 生成缺失 tool_result（L900-L903）
  2) 清空中间缓存数组（L904-L907）
  3) 重建 `StreamingToolExecutor`（L912-L918）
  4) 更新模型并剥离签名块（L922-L929）
  5) 发 warning 系统消息（L945-L948）

**Harness 作用（推理引擎）**：
- 回退不是“换模型就完了”，而是“换模型 + 清状态 + 保消息链完整”。

---

## 1.9 终止前恢复：413/media/max_output

**代码段**：L1065-L1256。

- 413：先 `contextCollapse.recoverFromOverflow`（L1094-L1115），再 `reactiveCompact`（L1120+）。
- max_output：先尝试升档 token cap（L1199-L1221），再最多 3 次 meta continuation（L1223-L1252）。

**Harness 作用（计划编排 + 上下文管理）**：
- 形成“轻恢复 -> 重恢复 -> 终止”的分级策略，而非直接失败。

---

## 1.10 Stop Hooks 接入与死循环防护

**代码段**：L1267-L1306。

- stop hook blocking 会插入 blockingErrors 并继续一轮。
- 明确保留 `hasAttemptedReactiveCompact`，注释解释可避免“compact -> hook blocking -> compact”死循环（L1292-L1297）。

**Harness 作用（计划与编排）**：
- hooks 与恢复策略的交互风险被显式处理。

---

## 1.11 工具执行后编排：队列、附件、memory 注入、refreshTools

**代码段**：L1360-L1728。

- 工具执行：streaming executor 或 `runTools`（L1380-L1383）。
- 队列消费有 agent 作用域过滤（L1560-L1578）。
- memory prefetch 仅在 settled 且未消费时注入（L1599-L1614）。
- 每轮前 refresh tools 让新接入 MCP 服务可见（L1659-L1669）。

**Harness 作用（工具注册表 + 多智能体编排）**：
- 工具池不是静态的，支持运行中演化；多 agent 下保证“谁看见谁的消息”。

---

## 2. 工具系统逐段解剖

## 2.1 工具契约层（`src/Tool.ts`）

### 2.1.1 ToolUseContext 是 runtime 总上下文
**代码段**：L158-L260。

- `options` 包含模型、工具列表、mcp、agentDefinitions、refreshTools。
- 持有 `abortController/readFileState/queryTracking/contentReplacementState`。
- 支持 REPL/SDK 差异行为（如 `handleElicitation/requestPrompt/setSDKStatus`）。

**Harness 作用**：统一上下文注入，避免“工具私自取全局变量”。

### 2.1.2 Tool 接口是行为契约
**代码段**：L362-L520。

关键字段：
- `inputSchema/outputSchema`
- `isConcurrencySafe/isReadOnly/isDestructive`
- `shouldDefer/alwaysLoad`
- `maxResultSizeChars`
- `validateInput/checkPermissions`

**Harness 作用（结构化 I/O + 工具注册表）**：
- 工具可治理能力前置到类型契约，降低后续分支复杂度。

---

## 2.2 工具注册与可见性（`src/tools.ts`）

### 2.2.1 全量工具源与环境开关
**代码段**：`getAllBaseTools`（L193-L250）。

- 通过 feature/env 控制工具集合。
- 注释强调该列表需和缓存策略配置保持同步（L191）。

### 2.2.2 权限过滤 + 模式过滤 + REPL 隐藏原语
**代码段**：L262-L327。

- `filterToolsByDenyRules` 让 deny 在“暴露前”就生效。
- REPL 模式下隐藏 `REPL_ONLY_TOOLS` 原语，避免模型直接乱调。

### 2.2.3 组合工具池并排序稳定缓存
**代码段**：`assembleToolPool`（L345-L367）。

- built-in + MCP 分区排序；注释说明这样做是为了 prompt cache 稳定。

**Harness 作用（工具注册表）**：
- 工具可见性、权限、安全和缓存性能一体化设计。

---

## 2.3 工具调度与并发（`src/services/tools/toolOrchestration.ts`）

### 2.3.1 先分批再执行
**代码段**：L19-L82 + L91-L116。

- `partitionToolCalls` 依据 `inputSchema.safeParse + isConcurrencySafe` 分批。
- 并发安全批次并行执行；不安全批次串行执行。

### 2.3.2 并发后 contextModifier 延迟应用
**代码段**：L31-L63。

- 并发批次先收集 `contextModifier`，再按 block 顺序回放修改 context。

**Harness 作用（计划与编排）**：
- 兼顾吞吐与一致性，避免并发写上下文造成竞态。

---

## 2.4 工具执行安全（`src/services/tools/toolExecution.ts`）

### 2.4.1 结构化输入校验
**代码段**：L614-L680。

- `tool.inputSchema.safeParse` 失败即返回 `tool_result(is_error=true)`。
- 如果失败原因是 deferred schema 未下发，会拼接“先 ToolSearch 再重试”的恢复提示（L578-L596, L619-L630）。

### 2.4.2 语义校验与权限前置
**代码段**：L682-L733。

- `validateInput` 失败同样结构化回传。
- 错误事件带 queryChainId/queryDepth，便于全链路诊断。

### 2.4.3 防御性输入净化
**代码段**：L756-L760 起（_simulatedSedEdit 处理）。

- 即便 schema 已防守，仍在运行时做 defense-in-depth。

**Harness 作用（结构化 I/O + 安全）**：
- “错误也结构化”，让上层 agent 可以继续编排，不是崩溃退出。

---

## 3. API 组装层：工具动态暴露与消息修复（`src/services/api/claude.ts`）

## 3.1 Tool Search 的启停与发现式加载

**代码段**：L1118-L1172。

- 先判断 `isToolSearchEnabled`。
- 若无 deferred tool 且无 pending MCP server，则关闭 tool search（L1136-L1148）。
- 开启时仅发送“已发现 deferred tools + ToolSearchTool + 非deferred tools”。

**Harness 作用（工具注册表）**：
- 控制 prompt 体积，避免把所有工具 schema 一次性塞给模型。

## 3.2 工具 schema 生成与缓存策略联动

**代码段**：L1207-L1246。

- `willDefer` 影响 tool schema 的 defer_loading。
- 全局缓存策略考虑 MCP 动态性（L1210-L1215）。

## 3.3 消息规范化与故障修复

**代码段**：L1265-L1315。

- `normalizeMessagesForAPI` 后再做模型能力相关 strip。
- `ensureToolResultPairing` 修复 resume/remote 的 tool_use-result 失配。
- `stripExcessMediaItems` 避免 >100 media 直接报错。

**Harness 作用（结构化 I/O + 状态恢复）**：
- 将“请求前修复”前置，显著降低线上随机 400。

---

## 4. 停止控制层（`src/query/stopHooks.ts`）

## 4.1 stop hooks 的统一出口

**代码段**：L65-L81 + L175-L240。

- `handleStopHooks` 返回 `{blockingErrors, preventContinuation}`。
- 消费 hook progress/attachment，收集阻断原因、输出信息、耗时。

## 4.2 主线程与子 agent 作用域治理

**代码段**：L92-L98 + L159-L173。

- 只在主会话保存 cache-safe params。
- CU lock cleanup 只在主线程执行，避免子 agent 抢释放。

**Harness 作用（计划编排 + 多智能体稳定性）**：
- stop hooks 不只是扩展点，而是被纳入主循环的“可控分支”。

---

## 5. 状态与持久化逐段解剖

## 5.1 历史持久化（`src/history.ts`）

### 5.1.1 读取侧：容错与会话去重
**代码段**：L106-L143 + L190-L217。

- 逆序读取 JSONL；解析失败仅跳过该行。
- 当前 session 优先输出，避免多终端并发时上下箭头历史互相穿插。

### 5.1.2 写入侧：锁 + 重试 + 清理
**代码段**：L291-L353。

- `lock(historyPath)` + append。
- 最多 5 次重试，每次间隔 500ms。

### 5.1.3 大文本外置
**代码段**：L371-L393。

- 小文本内联，大文本 hash 后 fire-and-forget 持久化。

### 5.1.4 语义撤销
**代码段**：L442-L464。

- `removeLastFromHistory` 能撤销“已提交但语义回滚”的最后一条输入。

**Harness 作用（状态与持久化）**：
- 为交互式 agent 提供“可回放、可撤销、可容错”的用户输入历史层。

---

## 5.2 Transcript 持久化规则（`src/utils/sessionStorage.ts`）

**代码段**：L130-L156。

- 明确 transcript message 类型与 chain participant 规则：progress 不入链。
- 注释直接指出历史 bug：progress 入链会造成 parentUuid 分叉并 orphan 真正消息。

**代码段**：L202-L229。

- transcript 路径按 sessionProjectDir/sessionId 组织；读取上限 50MB 防 OOM。

**Harness 作用（状态与持久化）**：
- 这是“可恢复会话”的结构性约束，不是仅仅写文件。

---

## 6. 推理引擎：模型选择（`src/utils/model/model.ts`）

**代码段**：L61-L98。

- 模型选择优先级：/model -> 启动参数 -> 环境变量 -> 设置 -> 默认。

**代码段**：L145-L167。

- `getRuntimeMainLoopModel` 在 plan 模式可将 haiku/opusplan 映射到更合适模型。

**Harness 作用（推理引擎）**：
- 模型选择逻辑可解释、可预测、可覆盖测试。

---

## 7. 与 Harness 六要素的“功能点 -> 代码映射总表”

| Harness 组件 | 功能点 | 关键代码映射 |
|---|---|---|
| 推理引擎 | runtime 选模 | `model.ts` L61-L98, L145-L167 |
| 推理引擎 | fallback 重试清理 | `query.ts` L893-L950 |
| 推理引擎 | task/token budget | `query.ts` L280-L291, L1308-L1354 |
| 计划编排 | 状态化主循环 | `query.ts` L241-L321 |
| 计划编排 | 多停止条件 | `query.ts` L1062-L1357, L1704-L1712 |
| 计划编排 | stop hook 回路 | `query.ts` L1267-L1306 + `stopHooks.ts` L65-L240 |
| 工具注册表 | 全量注册与模式过滤 | `tools.ts` L193-L327 |
| 工具注册表 | 组合池与缓存稳定排序 | `tools.ts` L345-L367 |
| 工具注册表 | ToolSearch 动态发现加载 | `claude.ts` L1118-L1172, L1231-L1246 |
| 内存上下文 | 多级压缩链 | `query.ts` L369-L447, L453-L543 |
| 内存上下文 | 413/media/max_output恢复 | `query.ts` L1065-L1256 |
| 内存上下文 | memory/skill prefetch | `query.ts` L297-L335, L1592-L1628 |
| 状态持久化 | history JSONL 读写与撤销 | `history.ts` L106-L143, L291-L353, L442-L464 |
| 状态持久化 | transcript 链规则 | `sessionStorage.ts` L130-L156 |
| 结构化I/O | 输入 schema 校验 | `toolExecution.ts` L614-L680 |
| 结构化I/O | 校验失败结构化回传 | `toolExecution.ts` L666-L677, L719-L730 |
| 结构化I/O | API 前 normalize + pairing repair | `claude.ts` L1265-L1302 |

---

## 8. 缺失或薄弱点（基于源码证据，不是泛泛而谈）

## 8.1 缺显式状态机定义文件（当前为“隐式状态机”）

**证据**：主循环通过 `while(true)+continue` + `transition.reason` 实现状态转移（`query.ts` L307-L321, L1099-L1115, L1207-L1220, L1283-L1305, L1715-L1726）。

**影响**：
- 功能增长后，分支交互验证成本上升。
- 需要靠集成测试覆盖“状态转移正确性”，而非可声明式校验。

## 8.2 持久化路径仍有 best-effort 分支

**证据**：
- history parse 失败直接 skip（`history.ts` L131-L134）
- 写入失败记录日志后继续（L320-L322）
- 大文本外置写入 fire-and-forget（L391-L392）

**影响**：
- 更偏可用性优先，强一致恢复语义在极端故障下可能不足。

## 8.3 主循环文件承担职责过多

**证据**：`query.ts` 超大体量（主循环+压缩+流式+恢复+hooks+工具+附件+队列+预算）。

**影响**：
- 可维护性压力高；新增策略时容易影响其他路径。

---

## 9. 综合结论（给架构落地）

1. Claude Code 的稳定性关键不是“某个算法”，而是 **主循环 + 多级恢复 + 结构化工具协议** 三位一体。  
2. 最值得复用的是：
   - **withhold-then-recover** 错误处理范式；
   - **分层上下文压缩链**；
   - **工具动态暴露（defer + search）**；
   - **tool_use/tool_result pairing repair**。  
3. 若你要在自己系统中落地，可按顺序实施：
   - 先做状态化主循环（含 transition reason）；
   - 再做工具契约与结构化错误返回；
   - 再做多级上下文治理与预算；
   - 最后做调度与长期记忆评估闭环。

---

## 10. 说明：关于“每行代码”

你要求“细致到每行代码在做什么”。对 `src/` 全量逐行展开会形成上千页文档，不利于工程消费。  
本版采用了 **关键路径逐段逐行范围（line-range）** 方式：
- 覆盖了 runtime 的核心执行链条；
- 每段都给出“代码行为 + harness 作用 + 风险影响”；
- 同时保留可继续下钻的文件/行号。

如果你下一步需要，我可以在此基础上输出“`query.ts` 单文件逐行注释版（1:1行号）”。
