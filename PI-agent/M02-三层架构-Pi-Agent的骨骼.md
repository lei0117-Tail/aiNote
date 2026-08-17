---
title: M02 三层架构 —— Pi-Agent 项目的骨骼
date: 2026-08-17
source: https://dg-ai-notes.pages.dev/modules/ch02-three-layer-arch/
version: Pi v0.80.2
module: M02
tags:
  - pi-agent
  - agent
  - coding-agent
  - 架构
  - 学习笔记
  - ai
aliases:
  - Pi 三层架构
  - Pi 包结构
  - Pi 依赖方向
  - Pi 类型递进
---

# M02 三层架构 —— Pi-Agent 项目的骨骼

> [!abstract] 核心结论
> Pi 是 monorepo(npm workspaces),5 个包。分层的真正规则不是"隔层调一层",而是 ==依赖方向单向向上==——底层对上层一无所知。基础类型(原子)在 `pi-ai` 统一定义,所有层跨层引用是必然,不是设计失误。

承接 [[M01-开篇-Pi-Agent总览]] 的四层架构图,本章钻进代码看包之间到底怎么依赖、类型怎么在层间流转。

---

## 一、目录结构:五个包的 monorepo

```
repo/
├── packages/
│   ├── ai/              ← @earendil-works/pi-ai
│   ├── agent/           ← @earendil-works/pi-agent-core
│   ├── coding-agent/    ← @earendil-works/pi-coding-agent
│   ├── orchestrator/    ← @earendil-works/pi-orchestrator(实验性)
│   └── tui/             ← @earendil-works/pi-tui
├── package.json         ← 根配置,"workspaces": ["packages/*"]
└── tsconfig.json
```

> [!note] 历史与外围
> - 历史上还有 `pi-web-ui`(浏览器端 Lit 组件库),已于 **2026-05-20** commit `b141e1fa` 移除
> - `pi-orchestrator` 是 v0.80.x 新增实验性包,依赖 `pi-coding-agent`,负责多 Agent 协调/RPC 进程通信/Supervisor 监控,不在核心学习主线

---

## 二、五个包,各管各的

| 包                 | 管什么             | package.json 描述                              | 关键导出                                    | 不碰什么                                   |
| ----------------- | --------------- | -------------------------------------------- | --------------------------------------- | -------------------------------------- |
| `pi-ai`           | ==调模型==         | Unified LLM API,自动模型发现                       | `Message`/`Model`/`Tool`/`streamSimple` | 无 agent/tool/loop 概念                   |
| `pi-agent-core`   | ==跑循环==         | General-purpose agent,传输抽象+状态管理              | `Agent`/`agentLoop`/`Session`/`compact` | 无 read/bash/edit 具体工具                  |
| `pi-coding-agent` | ==具体业务==        | Coding agent CLI + 7 工具 + 会话管理               | `cli.ts` 入口、扩展系统                        | ——(最厚,上百文件)                            |
| `pi-tui`          | ==显示==          | 终端 UI 库                                      | 差分渲染/Markdown/代码高亮                      | 运行时仅依赖 `marked`+`get-east-asian-width` |
| `pi-orchestrator` | 多 Agent 编排(实验性) | `supervisor.ts`/`rpc-process.ts`/`radius.ts` | 子 Agent 生命周期、RPC                        | 不实现 Agent 内核,只"编"实例                    |

> [!important] general-purpose 是关键
> `pi-agent-core` 的描述是 **general-purpose**——它不知道自己在做编程 Agent、客服 Agent 还是别的。它只管:维护状态(`AgentState`)、跑循环(`agentLoop`)、发事件(`AgentEvent`)、管会话历史与压缩(`Session`/`compact`)。

### coding-agent 的启动链路

```
你输入: pi "帮我改个 bug"
│
├── cli.ts          ← 解析命令行参数
│   └── main.ts     ← 创建会话、选运行模式(交互/打印/RPC)
│       └── AgentSession    ← 组装工具、加载扩展
│           └── Agent       ← 管理状态、跑循环
│               └── agentLoop()  ← 核心循环开始
```

---

## 三、直觉分层 vs 真相

> [!quote] 直觉画面
> 底层调模型 → 中间跑循环 → 顶层做业务,旁边一个独立 UI 包。

但打开 `packages/coding-agent/package.json` 会发现意外:

```json
"dependencies": {
    "@earendil-works/pi-agent-core": "^0.80.2",   // ← 依赖中间层,合理
    "@earendil-works/pi-ai": "^0.80.2",            // ← 也直接依赖底层?
    "@earendil-works/pi-tui": "^0.80.2"
}
```

coding-agent ==跨层直接依赖== pi-ai,而不是只通过 agent-core 间接使用。

### 答案藏在类型系统里

`packages/agent/src/types.ts` 第一行就从 pi-ai 导入大量基础类型:

```ts
import type {
    Api, AssistantMessage, AssistantMessageEvent, Context,
    ImageContent, Message, Model, SimpleStreamOptions,
    TextContent, Tool, ToolResultMessage,
} from "@earendil-works/pi-ai";
```

`Message`/`Model`/`ImageContent`/`Tool` 是整个系统的=="原子概念"==——不管在哪一层都需要。比如用户贴截图,coding-agent 要知道图片格式 → `ImageContent` 定义在 pi-ai。

> [!tip] 分层真正规则
> 关键不在于"能不能跨层引用",而在于 ==依赖方向是不是单向的==。
> 某些基础类型必须在一处统一定义,所有层都引用这一处——跨层引用类型是必然,不是失误。

---

## 四、依赖方向:所有箭头都朝上

| 包 | 依赖了谁 | 反向依赖? |
|---|---|---|
| `pi-ai` | `@anthropic-ai/sdk`/`openai`/`@google/genai` 等 SDK | ❌ 不依赖任何 pi-xxx |
| `pi-agent-core` | `pi-ai`/`typebox`/`yaml` | ❌ 只向上,不依赖 coding-agent |
| `pi-coding-agent` | `pi-ai`/`pi-agent-core`/`pi-tui` | ❌ 只向上 |
| `pi-orchestrator` | `pi-coding-agent` | ❌ 不反向渗透 |

```
pi-ai(底层)
  ↑         ↑
  │         │
  │    pi-agent-core(中间层)
  │         ↑
  │         │
  └─── pi-coding-agent(顶层)
            ↑
            │
       pi-orchestrator(实验性外围,可选)
```

> [!important] 底层永远不知道上层存在
> pi-ai 代码里没有任何 import 指向 pi-agent-core 或 pi-coding-agent。这就是分层的真正规则:**不是限制引用层级,而是控制依赖方向必须单向向上**。
>
> `pi-tui` 运行时只依赖 `marked`+`get-east-asian-width`,不依赖任何 pi-xxx,是独立终端渲染工具——coding-agent 单向依赖它,无循环。

`packages/ai/src/types.ts` 定义了 `KnownApi`(9 种 API 类型)和 `KnownProvider`(30+ 提供商),是整个系统的类型基础。

---

## 五、类型流转:原子 → 分子 → 材料

用化学类比:pi-ai 定义"原子",pi-agent-core 组合成"分子",pi-coding-agent 组合成"材料"。

### 第一层:pi-ai 定义原子

```ts
type Message = UserMessage | AssistantMessage | ToolResultMessage

interface Model<TApi> {
    id: string           // "claude-sonnet-4-6"
    name: string
    api: TApi            // "anthropic-messages"
    contextWindow: number
}

interface Tool<TSchema> {
    name: string
    description: string
    parameters: TSchema
}
```

`Message`/`Model`/`Tool` 是原子,任何和 LLM 打交道的包都要用。

### 第二层:pi-agent-core 组合成分子

```ts
// 扩展消息:联合类型扩展,不改原类型
type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages]

// 扩展工具:继承扩展
interface AgentTool<TParameters extends TSchema> extends Tool<TParameters> {
    label: string
    prepareArguments?: (args: unknown) => Static<TParameters>
    execute: (toolCallId, params, signal?, onUpdate?) => Promise<AgentToolResult>
    executionMode?: ToolExecutionMode   // "sequential" | "parallel"
}
```

- `AgentMessage` 是 `Message` 的==超集==——用联合类型 `|` 扩展(加压缩摘要、分支信息等),不修改原类型
- `AgentTool` 继承 `Tool`——底层只知道"长什么样",上层加"怎么执行、串行还是并行"

### 第三层:pi-coding-agent 组合成材料

```ts
// ToolDefinition:与 AgentTool 是"结构兼容"而非 extends 继承(types.ts:435)
interface ToolDefinition<TParams, TDetails, TState> {
    name: string
    label: string                         // UI 展示名
    description: string
    promptSnippet?: string                // 自动拼到 system prompt
    promptGuidelines?: string[]           // 工具使用守则
    parameters: TParams
    renderShell?: "default" | "self"      // 渲染模式
    execute: (toolCallId, params, signal, onUpdate, ctx: ExtensionContext) => Promise<AgentToolResult>
    // ... 渲染器、UI 组件等业务属性
}

// Extension:运行时聚合体,用 Map 而非 Record(types.ts:1585)
interface Extension {
    path: string
    handlers: Map<string, HandlerFn[]>
    tools: Map<string, RegisteredTool>
    commands: Map<string, RegisteredCommand>
    messageRenderers: Map<string, MessageRenderer>
    flags: Map<string, ExtensionFlag>
    shortcuts: Map<KeyId, ExtensionShortcut>
}
```

### 三层对照:Tool → AgentTool → ToolDefinition

```
pi-ai:        Tool        只知道"长什么样"    { name, description, parameters }
                ↓ agent-core extends
pi-agent-core: AgentTool  知道"怎么执行"      + label + execute + executionMode
                ↓ coding-agent 结构兼容
pi-coding-agent: ToolDefinition  加"怎么显示"  + 渲染器 + 权限 + promptSnippet
```

> [!info] 每层只加自己关心的字段
> - LLM 关心"长什么样" → `Tool`
> - Agent 关心"怎么执行" → `AgentTool`
> - coding-agent 关心"怎么显示" → `ToolDefinition`
>
> 底层类型从不修改——pi-ai 的 `Tool` 没有 `execute`,因为 LLM 不需要知道工具怎么执行。

---

## 六、写 Agent 真的需要三层吗?

> [!question] 三层是教条吗?
> 不是。==层数取决于复杂度,但依赖方向控制是必须的。==

| 场景 | 适合什么 | 你自己做什么 |
|---|---|---|
| 只用 `pi-ai` | 只需调 LLM、不需 Agent 循环 | 自己管状态、写循环(如需) |
| `pi-ai`+`pi-agent-core` | 需完整 Agent 能力、有独特业务场景 | 写自己的工具、自己的入口 |
| 全部三层 | 做 Pi 同类编程助手 | 直接用,或写扩展 |

- **场景 A**(不分层):循环逻辑和 OpenAI SDK 耦合,换 Claude 就得改循环代码
- **场景 B**(去 coding-agent):agent-core 不知道 read/bash 工具,只定义 `AgentTool` 接口规范,注册什么工具由你决定,甚至可纯聊天
- **场景 C**(只用 pi-ai):独立调 LLM、流式返回,但循环/状态/工具调用都得自己写——这正是 agent-core 存在的意义

> [!warning] 不可违反的规则
> 底层代码里不能出现任何对上层的引用。
> - pi-ai 不能 import pi-agent-core
> - pi-agent-core 不能 import pi-coding-agent
>
> 这条规则确保:你可以把任何一层换成自己的实现,而不影响其他层。

---

## 七、三个可带走的方法

### 方法 1:"依赖漏斗"分层法

设计包结构时先画依赖箭头:底层"不知道外面世界"、中间层"知道底层不知业务"、顶层"知道一切"。

- 找"完全不依赖外部知识"的部分 → 底层
- 找"依赖底层但不知具体业务"的部分 → 中间层
- 找"知道用户要什么"的部分 → 顶层
- **验证**:问自己"去掉上层,这一层还能跑吗?"能 → 依赖方向正确;不能 → 上层泄漏到下层

### 方法 2:"类型递进扩展"模式

底层定义最小类型接口,上层用联合类型 `|` 和继承 `extends` 扩展,==不改底层==。

- 底层定义原子(`Tool = { name, description, parameters }`)
- 中间层继承扩展(`AgentTool extends Tool`,加 `execute`)
- 顶层叠加业务属性(`ToolDefinition`,加渲染器)
- 好处:底层可独立发布复用,别人可只引用底层类型

### 方法 3:"可独立使用"测试

每层设计完做测试——去掉上层,这层还能正常工作吗?Pi 三层都能通过:

- 去掉 agent-core 和 coding-agent → pi-ai 独立调 LLM ✅
- 去掉 coding-agent → pi-ai+agent-core 跑自定义 Agent ✅
- 三层全用 → 完整编程助手 ✅

> 在 `package.json` 里暂时移除上层依赖,看底层包编译测试能否通过。报错 → 底层泄漏了对上层的依赖。

---

## 八、总结

| 要点 | 内容 |
|---|---|
| 分层 | `pi-ai`(模型)→ `pi-agent-core`(循环)→ `pi-coding-agent`(业务),+ 正交 `pi-tui` |
| 核心规则 | ==依赖方向单向向上==,底层对上层一无所知 |
| 跨层引用 | 基础类型(原子)在 pi-ai 统一定义,跨层引用是必然,非失误 |
| 类型递进 | `Tool` → `AgentTool` → `ToolDefinition`,每层只加自己关心的字段 |
| 三层必须? | 否,层数取决于复杂度;但依赖方向控制是必须的 |

下一章钻进 Agent 的心脏——[[M03-Agent-Loop-让模型转动的引擎]]:为什么需要 Loop 而不是一次调用,追踪一条用户消息从回车到 Agent 说"我完成了"的完整旅程。

---

> [!note]- 版本与阅读路径
> 本文档基于 Pi **v0.80.2**。代码分析以 earendil-works/pi 仓库实际源码为准。
> 第 4-5 节为架构进阶(依赖方向、类型流转),基于 SDK 二次开发必读;只想快速用起来可跳至第 6 节。前 6 章建议顺序通读,第 7 章起可按需跳读。
> 原文:[dg-ai-notes.pages.dev/modules/ch02-three-layer-arch](https://dg-ai-notes.pages.dev/modules/ch02-three-layer-arch/)
