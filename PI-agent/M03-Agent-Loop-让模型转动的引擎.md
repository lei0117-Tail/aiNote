---
title: M03 Agent Loop —— 让模型转动起来的引擎
date: 2026-08-17
source: https://dg-ai-notes.pages.dev/modules/ch03-agent-loop/
version: Pi v0.80.2
module: M03
tags:
  - pi-agent
  - agent
  - coding-agent
  - 架构
  - 学习笔记
  - ai
aliases:
  - Pi Agent Loop
  - Pi 循环引擎
  - Pi stopReason
  - Pi steering
  - Pi followUp
---

# M03 Agent Loop —— 让模型转动起来的引擎

> [!abstract] 核心结论
> Agent Loop 的本质是 ==ReAct 循环==(Reason→Act→Observe→Reason)。整个循环由一个信号驱动:模型输出里有没有 toolCall 块且未全部 terminate。"什么时候停"不是模型的智能决策,而是人类定义的规则——==模型不再输出工具调用时,就认为本轮结束==。

承接 [[M02-三层架构-Pi-Agent的骨骼]],本章钻进 Agent 的心脏。

---

## 一、大模型的三种用法

| 维度 | 直接调用 | Workflow | Agent Loop |
|---|---|---|---|
| 决策者 | 用户 | 你的代码 | ==模型== |
| 模型调用次数 | 1 次 | N 次(代码控制) | 不确定(模型控制) |
| 核心工作 | 写提示词 | 设计流程 | 定义工具和循环 |
| 模型角色 | 执行者 | 流水线环节 | 自主决策者 |
| 典型场景 | 翻译、摘要 | 文档流水线、RAG | 编程助手、自动化任务 |

Agent 与 Workflow 的本质区别:步骤流转不再由代码写死,而是由==模型输出内容驱动==。代码只做两件事:① 把用户输入和工具结果喂给模型;② 模型输出含工具调用就执行,没有就认为任务完成。

---

## 二、两个概念:Trace 与 Turn

> [!important] 精确定义
> - **Trace**:从用户回车到 Agent 彻底停下、发 `agent_end` 的整个过程。一个 Trace 含多个 Turn。
> - **Turn**:= 一次模型调用 + 这次调用触发的所有工具执行。一个 Turn ==只有一次== 模型调用,由 `turn_start`/`turn_end` 包裹。

```
Trace(一次 agent_start → agent_end)
├── Turn 1: turn_start → 调模型 → toolUse → 执行工具(read+grep) → turn_end
├── Turn 2: turn_start → 调模型 → toolUse → 执行工具(edit)      → turn_end
└── Turn 3: turn_start → 调模型 → stop → 无工具                  → turn_end → agent_end
```

模型一口气要 3 个工具(read+grep+find),这 3 个都在同一 Turn 执行(同一次模型调用的产物);把结果喂回去再调模型的那一刻才进入下一 Turn。

> [!note] 首轮 turn_start 机制
> 首轮 Turn 的 `turn_start` 在 `runAgentLoop()` 入口就发出,然后 `runLoop()` 内用 `firstTurn` 标志跳过首圈的 turn_start,避免重复。

---

## 三、全景:一条消息的旅程

```
你按下回车:"帮我读一下 src/main.ts"
│  ① 输入变成 UserMessage
│  ② 进入循环 agentLoop —— agent_start(Trace 开始)
└── runLoop()
    │  ③ 消息转换(AgentMessage → LLM 认识的 Message)
    │  ┌── Turn 1 ──────────────────────────────────┐
    │  │  turn_start                                  │
    │  │  ④ 调 Model(每 Turn 仅一次) streamSimple()   │
    │  │     → AssistantMessage { stopReason:"toolUse"}│
    │  │  ⑤ 执行 Tool → ToolResultMessage              │
    │  │  turn_end                                     │
    │  └──────────────────────────────────────────────┘
    │  循环判断:stopReason 是 toolUse → 继续
    │  ┌── Turn 2 ──────────────────────────────────┐
    │  │  ⑥ 第二次调 Model(工具结果已追加)             │
    │  │     → AssistantMessage { stopReason:"stop"}   │
    │  │  turn_end                                     │
    │  └──────────────────────────────────────────────┘
    │  hasMoreToolCalls=false,pendingMessages 空 → 内层退出
    │  外层检查 followUp → 空 → 外层退出
    └── agent_end(一个 Trace 结束,共 2 个 Turn)
```

### stopReason —— 唯一的信号灯

> [!warning] 关键认知:模型不会说"我要停了"
> 模型只是 token 预测器,不"知道"任务做完了没有。`stopReason` 虽挂在模型返回值上,但值来自两个不同地方:

| 来源 | stopReason | 含义 |
|---|---|---|
| 模型 API 返回 | `"toolUse"` | 输出了工具调用 JSON,API 检测到 |
| 模型 API 返回 | `"stop"` | 生成自然终止(遇结束标记),无工具调用 |
| 模型 API 返回 | `"length"` | token 达 maxTokens 上限被截断 |
| 框架流式层注入 | `"error"` | 调用异常(网络断、API 报错) |
| 框架流式层注入 | `"aborted"` | 用户主动中止(AbortSignal 触发) |

代码证据(`packages/ai/src/`):catch 块执行 `output.stopReason = options?.signal?.aborted ? "aborted" : "error"`——这是框架替模型兜底,不是模型说的。

### 一条规则驱动整个循环

> [!tip] 实际驱动循环的不是 stopReason,而是 toolCalls 数组
> `hasMoreToolCalls = toolCalls.length > 0 && !terminate`
> - 即使 `stopReason==="length"`,只要 content 有 toolCall 块,循环仍执行工具
> - 即使 `stopReason==="toolUse"`,如果所有工具结果都 `terminate:true`,循环也停

```js
// 简化逻辑(agent-loop.ts:202-216)
const toolCalls = message.content.filter(c => c.type === "toolCall");
hasMoreToolCalls = false;
if (toolCalls.length > 0) {
  const executedToolBatch = await executeToolCalls(...);
  hasMoreToolCalls = !executedToolBatch.terminate;  // 任何一个工具 terminate 则停
}
```

内层循环条件:`while (hasMoreToolCalls || pendingMessages.length > 0)`

### 五种退出路径

| 退出路径 | 触发条件 | 原因 |
|---|---|---|
| 正常退出 | `stop`/`length` + 无 followUp + 无 pending | 最常见,模型没要工具 |
| 硬停止 | `error`/`aborted` | 模型调用本身失败,不检查 followUp |
| 外部钩子停 | `shouldStopAfterTurn()` 返回 true | 上下文快满、达最大 Turn 数 |
| 工具终止 | 一批工具结果==全部== `terminate:true` | `every` 不是 `some` |

> [!question] 为什么不让代码智能判断"任务完成没"?
> 这正是 Agent 和 Workflow 的本质区别。Workflow 知道流程几步可判断进度;Agent 模式下你不知道模型要读几个文件、改几处——唯一能稳定依赖的信号就是:输出里有没有工具调用。==这既是局限也是优雅==——不需要任何"任务完成度"判断逻辑。

---

## 四、源码详解:内核 + coding-agent 叠加

### 最简 Loop:所有 Agent 的最小公约数

```js
async function simpleLoop(messages, model, tools) {
    while (true) {
        const response = await callModel(model, messages, tools);
        messages.push(response);
        if (response.stopReason !== "toolUse") return messages;  // 没工具 → 结束
        for (const toolCall of response.toolCalls) {
            const result = await executeTool(toolCall);
            messages.push(result);  // 有工具 → 执行,把结果喂回去
        }
    }
}
```

十几行代码。任何 Agent 都需要这个内核。

### coding-agent 在内核上叠加了什么

| 真实需求 | 叠加设计 | 源码位置 |
|---|---|---|
| 用户工作期间又输入新指令 | steering 消息注入(紧急插队) | 内层循环开头 |
| 系统完成后想追加任务 | 外层 followUp 循环 | 内层外层 while(true) |
| 不同复杂度任务用不同模型 | prepareNextTurn 钩子 | turn_end 之后 |
| 上下文窗口快满触发压缩 | shouldStopAfterTurn 钩子 | prepareNextTurn 之后 |

> [!important] 叠加是产品选择,非通用法则
> 如果你做"一问一答带工具"的简单 Agent,上表全是多余的——你只需要最简 Loop。

### 4.1 入口 runAgentLoop()

调用链:`Agent.prompt()` → `runPromptMessages()` → `runAgentLoop()`(`agent-loop.ts:95-118`)

```ts
async function runAgentLoop(
    prompts: AgentMessage[],     // 你的消息
    context: AgentContext,       // 对话上下文快照(副本)
    config: AgentLoopConfig,     // 模型、钩子、队列回调
    emit: AgentEventSink,        // 事件发射器
    signal?: AbortSignal,        // 中止信号
    streamFn?: StreamFn,         // 流式函数(可替换)
): Promise<AgentMessage[]>
```

> [!note] context 是快照副本
> `agent.ts:414-420` 的 `createContextSnapshot()` 创建。Loop 运行期间对 context 的修改不影响 Agent 类原始状态。

config 的关键钩子==都是函数而非数据==——Loop 运行时调用它们"拉取"最新状态,让 Loop 和外部消息源彻底解耦:`convertToLlm`/`transformContext`/`getSteeringMessages`/`getFollowUpMessages`/`shouldStopAfterTurn`/`beforeToolCall`/`afterToolCall`/`prepareNextTurn`。

入口三步:① 创建 `newMessages` 数组收集本轮 Trace 产生的新消息;② prompts 追加到 `context.messages`;③ 发初始事件 `agent_start`+`turn_start`+每条 prompt 的 `message_start`/`message_end`。

### 4.2 runLoop() 骨架:内核 + 两层叠加

```js
async function runLoop(currentContext, newMessages, config, signal, emit, streamFn) {
    let pendingMessages = (await config.getSteeringMessages?.()) || [];  // ① 首次 steering
    while (true) {                                  // 叠加2:外层 followUp 循环
        let hasMoreToolCalls = true;
        let firstTurn = true;
        while (hasMoreToolCalls || pendingMessages.length > 0) {  // 内核+叠加1
            if (!firstTurn) emit({ type: "turn_start" });
            firstTurn = false;
            // A:注入 pendingMessages(steering) ← 叠加1
            // B:调 LLM streamAssistantResponse()  ← 内核
            // C:检查 stopReason                  ← 内核
            // D:执行工具                          ← 内核
            // E:emit turn_end                     ← 内核
            // F:prepareNextTurn → shouldStopAfterTurn ← 叠加(钩子)
            //   → 再次检查 steering                ← 叠加1
        }
        const followUpMessages = (await config.getFollowUpMessages?.()) || [];  // 叠加2
        if (followUpMessages.length > 0) { pendingMessages = followUpMessages; continue; }
        break;
    }
}
```

### 4.3 steering 消息注入(叠加1·步骤A)

用户在 Agent 修 bug 时突然说"也检查下测试文件"——这条指令要插队,不等当前任务跑完。每圈内层循环开头检查 steering 队列,把紧急消息注入上下文。

首次 steering 检查在 `agent-loop.ts:167`(进入内层循环之前)——因为用户等 LLM 首次响应时可能又输入了内容,不提前取出来就漏掉了。

### 4.4 streamAssistantResponse() —— 调 LLM(内核·步骤B)

四阶段:

**阶段 A**:上下文预处理(可选)`transformContext`(如压缩算法)

**阶段 B**:`convertToLlm(messages)` —— ==AgentMessage → Message 转换==,两层消息边界。默认实现是 `.filter()`,只保留三种标准消息(user/assistant/toolResult)。完整设计见 [[M06-消息系统-Agent的记忆组织传递]]。

**阶段 C**:构建 Context 调模型。`llmContext` 每圈内层循环重建一次(systemPrompt+messages+tools)。`tools` 和 `systemPrompt` 是引用赋值(字节级稳定),只有 `messages` 真的在长。

> [!tip] 每圈重建 wrapper 不会破坏 prompt cache
> Anthropic 的 prompt cache 是==内容寻址==的——看发过去的字节,不是请求 identity。只要 `system+tools` 字节不变,cache 就命中。Pi 在 `anthropic-messages.ts` 三个位置打 `cache_control: { type: "ephemeral" }`:
> - System prompt 末尾(L922/929/938)
> - 最后一个 tool(L1208)
> - 最后一条 user message(L1157,rolling cache)
>
> 第三条尤其精妙:cache breakpoint 跟着最新 user message 走,旧前缀继续命中、新内容被写入。

```
Turn 1: 写入 [system+tools] → 写入 [messages §1]
Turn 2: 命中 [system+tools] → 命中 [messages §1] → 写入 [messages §2]
Turn 3: 命中 [system+tools] → 命中 [messages §1+§2] → 写入 [messages §3]
```

**阶段 D**:流式处理响应——==原地替换==的妙用。先 push 空壳消息,再逐 token 用 `context.messages[last] = partialMessage` 覆盖,消息数量不变但最后一条内容在"长大"。为了 UI 能逐字渲染。

### 4.5 检查 stopReason(内核·步骤C)

```js
// agent-loop.ts:196-200
if (message.stopReason === "error" || message.stopReason === "aborted") {
    await emit({ type: "turn_end", message, toolResults: [] });
    await emit({ type: "agent_end", messages: newMessages });
    return;   // 直接退出整个 runLoop,不检查 followUp
}
```

`error`/`aborted` 是==硬停止==(fail fast)——立即发 turn_end+agent_end 直接 return,连工具都不执行、followUp 都不检查。`stop`/`toolUse`/`length` 继续往下走(后两者 toolCalls 为空,工具执行步骤空转)。

### 4.6 executeToolCalls() —— 执行工具(内核·步骤D)

过滤 toolCall 块 → 决定并行/串行。==一票否决==:只要有一个工具声明 `executionMode:"sequential"`,整批串行。edit 工具内部还有第二道防线 `withFileMutationQueue`(同文件编辑串行化)。

**并行模式三阶段设计**:
- 阶段1 准备(顺序):prepareArguments→validate→beforeHook——验证和权限检查不能并行
- 阶段2 执行(并行):只有 `tool.execute` 并行(`Promise.all`)
- 阶段3 事件(有序):end 按完成顺序发,result 按调用顺序发(模型收到的上下文才正确)

> [!info] terminate 机制用 every 不是 some
> 工具可在结果设 `terminate:true`。但 Loop 用 `every`——必须这批工具==全部== terminate 才退出。保守策略:只要有一个工具正常工作,Loop 就不中断。

工具执行完整五步管道详见 [[M05-工具系统-Agent的手脚如何管住]]。

### 4.7 turn_end + 钩子(内核+叠加·步骤E~F)

```js
await emit({ type: "turn_end", message, toolResults });          // ① 内核
const nextTurnSnapshot = await config.prepareNextTurn?.({...});   // ② 叠加
if (nextTurnSnapshot) {
    currentContext = nextTurnSnapshot.context ?? currentContext;
    config.model = nextTurnSnapshot.model ?? config.model;
}
if (await config.shouldStopAfterTurn?.({...})) {                  // ③ 叠加
    await emit({ type: "agent_end", messages: newMessages }); return;
}
pendingMessages = (await config.getSteeringMessages?.()) || [];   // ④ 叠加1
```

> [!example] prepareNextTurn 强大扩展点
> 每个 turn_end 后、下一轮前调用,可动态切换下一轮配置:
> - 按复杂度切模型:简单问题用 Haiku(便宜),复杂任务切 Opus(强)
> - 换 context(注入新上下文)、换 thinkingLevel(思考强度)
> - 返回 `undefined` 表示什么都不换

`shouldStopAfterTurn` 是产品层"安全阀"——检查上下文窗口是否快满(触发压缩)、是否达最大 Turn 数。最简 Loop 不需要它。

### 4.9 外层循环:followUp 续命(叠加2·步骤G)

```js
const followUpMessages = (await config.getFollowUpMessages?.()) || [];
if (followUpMessages.length > 0) { pendingMessages = followUpMessages; continue; }
break;
```

有 followUp → 塞进 pending,`continue` 回外层顶部,内层循环检测到 pending 不为空,==在同一 Trace 内==继续跑。比重新启动 `runAgentLoop()` 好在连续性:同一个 newMessages、同一个事件序列,不需额外合并。

### 4.10 steering vs followUp

| 维度 | steering(叠加1) | followUp(叠加2) |
|---|---|---|
| 触发按键 | Enter | Alt+Enter |
| 检查时机 | runLoop 开始前 + 内层每圈结尾 | 内层循环全部结束后 |
| 语义 | "紧急插队"——工具执行间隙插入 | "排队等叫号"——等当前任务告一段落 |
| 前提 | agent 忙(`isStreaming=true`) | agent 忙(`isStreaming=true`) |

> [!quote] 生活类比
> steering 是你正在开会,有人敲门递了张纸条——"紧急,先看这个"。followUp 是会议还在进行,你写了张条子放进信箱——"不急,等这议题讲完再看"。

### 4.11 三岔口:agent 忙 vs 空闲

> [!question] 双层循环到底为谁服务?
> 任务跑完两层都退出,下一轮对话本就是新循环;紧急消息内层就够;跑完后追加任务带历史上下文启动下一轮即可——那双层循环服务谁?

答案:**双层循环只活在"agent 忙"这个分支**。分流判据不是消息内容,而是==按键==。`prompt()` 在 `isStreaming=true` 时直接抛错,强制 steer/followUp 二选一;空闲时才正常启动新 Trace。

```js
// agent-session.ts:1167(基于 0.84.2)
if (this.isStreaming) {
    if (!options?.streamingBehavior)
        throw new Error("Agent is already processing. Specify streamingBehavior ('steer' or 'followUp')...");
    if (options.streamingBehavior === "followUp") await this._queueFollowUp(...);
    else await this._queueSteer(...);
    return;
}
// ↓ 空闲才走到这里:新 Trace
```

| 按键 | handler | streamingBehavior | 队列 | 消化者 | 源码(0.84.2) |
|---|---|---|---|---|---|
| Enter | `onSubmit` | `"steer"` | steeringQueue | 内层 | `interactive-mode.ts:3070` |
| Alt+Enter | `handleFollowUp` | `"followUp"` | followUpQueue | 外层 | `interactive-mode.ts:4012` |

> [!note] 空闲时 Alt+Enter 退化成 Enter
> `interactive-mode.ts:4016`:`isStreaming` 为 false 时 Alt+Enter 直接走 `onSubmit`——开新 Trace。排队语义在空闲时无对象,自动失效。

```
agent 忙:Enter→steer→内层;Alt+Enter→followUp→外层
agent 空闲:Enter/Alt+Enter 都→prompt()→新 Trace(带历史上下文)
```

"跑完后手动追加任务带历史上下文启动下一轮"走的是==空闲分支==,和双层循环无关。双层循环是 agent 忙时排队消息的==自驱动出口==:followUp 不能走内层(会退化成插队,失去"等叫号"语义),也不能等用户手动开新 Trace(塞入时 agent 正忙,`prompt()` 抛错;且语义会从"当前任务延续"错位成"新意图")。外层循环让 Loop 在内层整轮退出后主动 drain 一次,自驱动续命。

---

## 五、总结:Loop 的四条核心设计

1. **ReAct 循环模式**:Reason→Act→Observe→Reason。模型输出决定调什么工具(Agent 区别 Workflow 的核心);"什么时候停"是人类规则——不再输出工具调用时本轮结束
2. **stopReason 驱动**:实际驱动循环的是 `toolCalls.length > 0 && !terminate`。==把决策外包给模型输出模式,代码只做最简单的信号判断==
3. **内核 + 叠加**:内核十几行;coding-agent 叠加 steering/followUp/prepareNextTurn/shouldStopAfterTurn。==内核是通用法则,叠加是产品功能按需选择==。做自己的 Agent 时先搭内核,再按场景加叠加
4. **四层洋葱**:剥掉任何一层叠加,里层仍能跑——这是判断"内核是否被污染"的试金石

---

## 关键源码索引(v0.80.2)

| 位置 | 内容 |
|---|---|
| `agent-loop.ts:95-118` | `runAgentLoop()` 入口 |
| `agent-loop.ts:155-269` | `runLoop()` 双层循环完整实现 |
| `agent-loop.ts:167` | steering 首次检查 |
| `agent-loop.ts:175-179` | turn_start 首轮跳过机制 |
| `agent-loop.ts:196-200` | stopReason 硬停止判断 |
| `agent-loop.ts:275-368` | `streamAssistantResponse()` 流式响应 |
| `agent-loop.ts:313-357` | 流式处理原地替换 |
| `agent-loop.ts:373-516` | `executeToolCalls()` 并行/串行调度 |
| `agent-loop.ts:218-253` | turn_end + prepareNextTurn + shouldStopAfterTurn |
| `agent-loop.ts:253` | steering 每圈二次检查 |
| `agent.ts:118-152` | `PendingMessageQueue` 和 drain 模式 |
| `agent.ts:451-474` | `runWithLifecycle()` 生命周期管理 |

下一章:[[M04-模型调用-一行代码驾驭多模型]] —— `streamSimple()` 内部做了什么,Pi 怎么用同一套代码调 30+ 供应商。

> [!note]- 版本说明
> 基于 Pi **v0.80.2**,代码分析以 earendil-works/pi 仓库实际源码为准。原文:[dg-ai-notes.pages.dev/modules/ch03-agent-loop](https://dg-ai-notes.pages.dev/modules/ch03-agent-loop/)
>
> 4.11 节的按键映射与 `prompt()` 分流逻辑经 **v0.84.2** 源码复核,行号以 0.84.2 为准。
