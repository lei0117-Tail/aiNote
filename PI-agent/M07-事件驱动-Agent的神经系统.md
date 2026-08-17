---
title: M07 事件驱动 —— Agent 的神经系统
date: 2026-08-17
source: https://dg-ai-notes.pages.dev/modules/ch07-event-driven/
version: Pi v0.80.2
module: M07
tags:
  - pi-agent
  - agent
  - coding-agent
  - 架构
  - 学习笔记
  - ai
aliases:
  - Pi 事件驱动
  - Pi AgentEvent
  - Pi 同步屏障
  - Pi processEvents
  - Pi 发布订阅
---

# M07 事件驱动 —— Agent 的神经系统

> [!abstract] 核心结论
> 事件系统实现 ==发布-订阅模式==,把"发生了什么"和"谁关心什么"彻底分离——Agent 只管发事件,不知道也不关心谁在听。三个关键设计决策:① emit 是==同步屏障==(await 所有监听器);② 监听器异常==直接暴露==不加 try-catch;③ 两层事件(内核 10 种 + Session 7 种)。

本章起为进阶章节。承接 [[M06-消息系统-Agent的记忆组织传递]]——事件到处携带 AgentMessage,但事件怎么传到外部?

---

## 一、为什么需要事件系统?

### 外卖追踪类比

美团点外卖后推送一连串状态:"商家已接单"→"骑手已取餐"→"骑手距你 500 米"→"已送达"。每个状态更新就是一个事件,你不用一直盯着骑手位置,收到事件时看一眼。Pi-Agent 事件同义:Agent 运行中不断产生"发生了某事"的快照推给关心它的人。

### 不用事件会怎样?

给 Agent 加"工具调用日志"功能:

> [!warning] 不用事件:改 Agent 源码
> 在 `tool.execute()` 前后各加 `console.log`。Pi 更新时 merge 上游代码冲突——你加的日志和上游新增逻辑撞一起。手动解冲突,下周又更新又冲突……

> [!success] 用事件:六行代码不碰源码
> ```js
> session.subscribe((event) => {
>     if (event.type === "tool_execution_end") {
>         console.log(`[LOG] 调用了 ${event.toolName},结果:${event.isError?"失败":"成功"}`);
>     }
> });
> ```
> Agent 更新只需 `npm update`,日志逻辑不受影响。

==事件驱动最核心的价值:把"发生了什么"和"谁关心什么"彻底分离==。Agent 只管发事件,不知道也不关心谁在听。

### 发布-订阅 vs 直接调用

```
直接调用(打电话):                 发布-订阅(广播):
  Agent ──调用──→ 终端渲染           Agent ──emit事件──→ 📡 事件总线
       ──调用──→ 文件存储                                   ├──→ 终端渲染(订阅了)
       ──调用──→ 日志记录                                   ├──→ 文件存储(订阅了)
  Agent 需知道所有消费者存在                                ├──→ 日志记录(订阅了)
  每加新功能就要改 Agent                                    └──→ (新功能只需订阅,Agent 不需知道)
```

直接调用是"我亲自找你";发布-订阅是=="我对着空气喊了一声,谁听到算谁的"==。Pi 里 emit 是"喊",subscribe 是"听"。

---

## 二、10 种事件,4 层嵌套

```ts
export type AgentEvent =
  // 第1层:Agent 生命周期(整个运行)
  | { type: "agent_start" }
  | { type: "agent_end"; messages: AgentMessage[] }
  // 第2层:Turn 生命周期(一轮模型调用 + 工具执行)
  | { type: "turn_start" }
  | { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
  // 第3层:Message 生命周期(一条消息)
  | { type: "message_start"; message: AgentMessage }
  | { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
  | { type: "message_end"; message: AgentMessage }
  // 第4层:Tool Execution 生命周期(一次工具执行)
  | { type: "tool_execution_start"; toolCallId: string; toolName: string; args: any }
  | { type: "tool_execution_update"; toolCallId: string; toolName: string; args: any; partialResult: any }
  | { type: "tool_execution_end"; toolCallId: string; toolName: string; result: any; isError: boolean };
```

```
Agent 运行
├── agent_start ─── Agent 开始
├── Turn 1(第3章:一次模型调用 + 它触发的工具执行)
│   ├── turn_start
│   ├── Message(LLM 响应): message_start → message_update×N → message_end
│   ├── Tool Execution: tool_execution_start → update×N → tool_execution_end
│   └── turn_end
├── Turn 2 ...
└── agent_end
```

> [!important] 为什么要 4 层?
> 不同消费者关心不同粒度。TUI 需逐 token 渲染文字 → 订阅 `message_update`;Session 管理器只关心一轮对话结束没 → 只看 `turn_end`。==4 层嵌套让每种消费者都能在刚刚好的粒度上响应==。每层都是"开始→更新→结束"配对(2+2+3+3=10 种)。

---

## 三、emit 不是"通知",是"同步屏障"(最重要设计决策)

### 每次 emit 都带 await

```ts
export type AgentEventSink = (event: AgentEvent) => Promise<void> | void;
```

emit 返回 `Promise<void>`,可以是异步的。Node.js 的 `EventEmitter` 是同步 fire-and-forget,但 Pi 的 Agent Loop 每次调用 emit 都带 `await`:

```js
await emit({ type: "agent_start" });
await emit({ type: "turn_start" });
await emit({ type: "message_start", ... });
await emit({ type: "message_update", ... });
await emit({ type: "message_end", ... });
```

每个 `await` 都在说:"等这个事件被完全处理完,再继续。"

### processEvents:先更新状态,再等监听器

```ts
private async processEvents(event: AgentEvent): Promise<void> {
    // 第一步:根据事件类型更新内部状态
    switch (event.type) {
        case "message_start":  this._state.streamingMessage = event.message; break;
        case "message_update": this._state.streamingMessage = event.message; break;
        case "message_end":
            this._state.streamingMessage = undefined;
            this._state.messages.push(event.message); break;
        // ... tool_execution_start/end 更新 pendingToolCalls
    }
    const signal = this.activeRun?.abortController.signal;
    // 第三步:同步等待所有监听器完成
    for (const listener of this.listeners) {
        await listener(event, signal);   // ← 一个一个等!
    }
}
```

> [!note] 这跟"循环里直接调用函数"有什么区别?
> 区别在 Agent 的 `this.listeners` 是外部 Set,不知道里面是谁。Agent 只负责"遍历并等待",谁在 Set 里完全由外部 `subscribe()` 控制。==Agent 内核代码里没有任何一行 `updateTerminal()` 或 `appendToFile()`==——它甚至不知道 TUI 和文件存储的存在。

### 为什么非要 await?

```
假设 emit 不 await(fire-and-forget):
  Agent Loop:  emit(start)  emit(update)  emit(end)
  TUI 监听器:   [开始渲染...]  [还没处理完 start...]  [三个事件堆一起了]
  问题:TUI 还没处理完 message_start,message_update 就来了 → UI 状态不一致

实际设计(await 同步屏障):
  Agent Loop:  emit(start)──await──→  emit(update)──await──→  emit(end)──await──→
  TUI 监听器:    [处理完毕,返回]       [处理完毕,返回]          [处理完毕,返回]
  保证:Agent 在监听器返回前不会发出下一个事件
```

> [!abstract] await 不是为了"通知",而是为了"同步协商"
> 确保所有消费者都跟上了,Agent 才走下一步。==代价是性能(必须等最慢的消费者),换来的是正确性(状态永远一致)==。

### 一个例外:tool_execution_update 不等

工具执行可能输出大量进度(Bash 每一行输出),每次都 await 太慢。Pi 对高频事件特殊处理——==先收集后批量等待==:

```js
const updateEvents: Promise<void>[] = [];
let acceptingUpdates = true;
const result = await tool.execute(id, args, signal, (partialResult) => {
    if (!acceptingUpdates) return;   // 工具已结束,丢弃迟到 update
    updateEvents.push(emit({ type: "tool_execution_update", ... }));  // 不 await!先收集
});
acceptingUpdates = false;
await Promise.all(updateEvents);      // 一次性等所有 update 处理完
```

> [!tip] 同步屏障规则不松,但对进度更新开口子
> 进度更新是"高频、低价值、可合并"——多发一条少发一条不影响最终状态。生命周期事件(start/end)是"低频、高价值"——错过 message_start 就没机会了。

`acceptingUpdates` 闸门防止迟到的 `partialResult` 在 `tool_execution_end` 后又发出 update,造成"工具结束了却还在更新"的错乱序列。

---

## 四、错误处理:监听器异常直接冒泡

```ts
for (const listener of this.listeners) {
    await listener(event, signal);   // 没有 try-catch!
}
```

监听器抛异常会一路冒泡到 `runWithLifecycle`,触发整个 Agent 运行失败。一个 UI 渲染 bug 能把 Agent 干掉。

> [!question] 为什么不包 try-catch?
> Pi 设计哲学:==监听器出错,运行就停下来,问题立刻可见==。静默吞掉异常,Agent 看起来"正常"但 UI 已乱套——调试时根本找不到问题。像电路保险丝——烧断了立刻知道哪里出问题。

> [!warning] 实践建议
> 基于 Pi 写自己的 UI/扩展监听器,务必在 listener 里自己 try-catch——==Agent 不会替你兜底==。
>
> 例外:扩展系统。第三方扩展回调由框架自己做 try-catch 隔离,单个扩展崩溃不会拖垮 session。原则:对内层受信任的监听器(自己写的代码)让异常直接暴露;对外层不受信任的监听器(第三方扩展)由框架做隔离。

---

## 五、你能用事件系统做什么?

| 场景 | 做法 |
|---|---|
| 实时观测 Agent 在干什么 | 订阅 `tool_execution_start`/`end` 打日志 |
| 工具调用拦截 | 扩展的 `tool_call` 事件返回 `{block:true, reason}` → 第5章 beforeToolCall |
| 上下文预处理 | 扩展在 LLM 调用前修改消息列表 → 第6章 `transformContext` 钩子 |
| 流式转发到 Web 前端 | 订阅 `message_update` 通过 SSE 推浏览器 → Web 集成核心 |

> [!success] 共同特点:新增任何功能都不需修改 Agent 内核
> 只需 `subscribe` 在回调里做事。==事件驱动架构的真正威力不是"通知机制",而是开放扩展机制==。

---

## 六、案例:追踪一个 text_delta 的完整旅程

LLM 生成"你好",一个"你"字从产生到显示经历 5 步:

```
触发端:LLM SSE 网络流
  │  data: {"type":"text_delta","delta":"你",...}
  ▼ 中转1:AI 层 EventStream.push()  ← 第4章12种事件之一
  ▼ 中转2:Agent Loop 事件转换  ← AI 层 text_delta → Agent 层 message_update(透传 assistantMessageEvent)
  ▼ 中转3:Agent.processEvents()(同步屏障)  ← 更新 streamingMessage + await 所有 listeners
  ▼ 中转4:AgentSession._handleAgentEvent()  ← 通知扩展系统→分发 Session 监听器→持久化
  ▼ 终点:TUI 监听器  ← 提取 delta "你" 渲染到终端
```

> [!info] 关键数据变换
> AI 层的多种 delta 事件(text_delta/thinking_delta/toolcall_delta)被统一映射为 Agent 层的 `message_update`。==原始事件通过 `assistantMessageEvent` 字段附在 message_update 上透传==。Agent Loop 不关心 delta 具体类型(只关心"消息更新了"),但消费者可能关心,所以原始事件被保留而非丢弃。

整条链路每层只关心自己的事,没有任何一层直接调用另一层内部方法,==唯一通信协议就是"事件"==。

---

## 七、Session 层扩展了什么

Agent 内核 10 种事件之上,还有 `AgentSession`(产品会话层)。Session 处理的事比内核多:上下文压缩、自动重试、队列状态管理——就像 Linux 内核里找不到"蓝牙已连接"通知,蓝牙是上层的事。

### 内核 10 种 + Session 7 种

```
AgentSessionEvent =
    基础 10 种(agent_end 被重载,增加 willRetry 字段)
  + Session 新增 7 种:
      queue_update           ← steering/followUp 队列变化
      compaction_start/end   ← 上下文压缩(第9章)
      auto_retry_start/end   ← LLM 调用失败自动重试
      session_info_changed   ← 会话名称变更
      thinking_level_changed ← 思考深度切换
```

> [!important] 两层事件设计思路
> 内核只管内核的事(生命周期),产品在上面按需扩展(用户体验)。==判断标准:如果去掉某个事件后内核还能正常运行,它就属于外层==。

---

## 八、总结:三个设计决策

| 决策 | 内容 | 代价/缓解 |
|---|---|---|
| **同步屏障** | `processEvents` 中 `await` 所有监听器,不用 fire-and-forget | 代价是性能;`tool_execution_update` 的"先收集后批量等待"缓解高频事件性能 |
| **异常直接暴露** | 监听器循环不加 try-catch,出错→运行失败→问题立刻可见 | 内层受信任监听器不加保护;外层第三方扩展由框架做隔离 |
| **两层事件** | 内核 4 层生命周期 10 种;Session 联合类型扩展 7 种产品级 | 去掉某事件后内核还能运行→属外层 |

---

## 关键源码索引

| 位置 | 内容 |
|---|---|
| `agent/types.ts:413-428` | 10 种 `AgentEvent` 定义 |
| `agent-loop.ts:25` | `AgentEventSink` 类型(emit 签名) |
| `agent.ts:509-556` | `processEvents`(同步屏障实现) |
| `agent.ts:168,231-233` | `subscribe` 和 `listeners` |
| `agent-loop.ts:628-669` | `executePreparedToolCall`(update 特殊处理) |
| `coding-agent/agent-session.ts:126-150` | `AgentSessionEvent`(17 种事件) |

下一章:[[M08-上下文工程-有限窗口装无限对话]] —— `transformContext` 在 `convertToLlm` 之前执行,当对话越来越长超出窗口,它要做更激进的事——压缩。

> [!note]- 版本说明
> 基于 Pi **v0.80.2**,代码分析以 earendil-works/pi 仓库实际源码为准。原文:[dg-ai-notes.pages.dev/modules/ch07-event-driven](https://dg-ai-notes.pages.dev/modules/ch07-event-driven/)
