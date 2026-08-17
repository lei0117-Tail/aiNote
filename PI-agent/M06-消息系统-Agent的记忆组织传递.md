---
title: M06 消息系统 —— Agent 的记忆如何组织与传递
date: 2026-08-17
source: https://dg-ai-notes.pages.dev/modules/ch06-messages/
version: Pi v0.80.2
module: M06
tags:
  - pi-agent
  - agent
  - coding-agent
  - 架构
  - 学习笔记
  - ai
aliases:
  - Pi 消息系统
  - Pi AgentMessage
  - Pi convertToLlm
  - Pi 两层消息
  - Pi CustomAgentMessages
  - Pi excludeFromContext
---

# M06 消息系统 —— Agent 的记忆如何组织与传递

> [!abstract] 核心结论
> Pi 消息系统最核心的设计是 ==两层消息==:Agent 内部用丰富的格式自由表达(`AgentMessage`,7 种),到了 LLM 边界翻译回严格的标准格式(`Message`,3 种)。翻译是最后一刻发生的、有损的、单向的——"内富外严"让 Agent 拥有无限扩展能力,同时永远不破坏 LLM 兼容性。

承接 [[M05-工具系统-Agent的手脚如何管住]]——工具结果是消息,但消息到底长什么样?

---

## 一、开场:一条 Bash 命令的消息之旅

你在 Pi 终端输入 `!ls -la`,变成 `BashExecutionMessage`——有 `command`/`output`/`exitCode` 结构化字段,UI 可漂亮渲染。但 LLM API 根本不认识它,只认三种消息:user/assistant/toolResult。这条消息怎么被 LLM 看到?中间经历了什么?

---

## 二、第一层:LLM 认识的消息只有三种

```
Message 联合类型(LLM 标准格式)
├── UserMessage        ← 用户说的/发的图片
├── AssistantMessage   ← LLM 回复(含思考、工具调用)
└── ToolResultMessage  ← 工具执行结果
```

### UserMessage(最简单)

```ts
{
    role: "user",
    content: string | (TextContent | ImageContent)[],  // 纯文本或内容块数组
    timestamp: number                                    // Unix 毫秒
}
```

### AssistantMessage(字段最多)

```ts
{
    role: "assistant",
    content: (TextContent | ThinkingContent | ToolCall)[],  // 三种内容块
    api: Api, provider: ProviderId, model: string,
    usage: Usage, stopReason: StopReason,   // 第3章讲过的5种值
    errorMessage?: string, timestamp: number
}
```

content 是内容块数组,可含三种:
```
├── TextContent       { type:"text", text:"..." }
├── ThinkingContent   { type:"thinking", thinking:"..." }    ← 模型"在想"但不告诉用户
└── ToolCall          { type:"toolCall", id:"...", name:"read", arguments:{...} }
```

> [!note] 一条助手消息可同时含文本和工具调用
> LLM 一边说"让我看看这个文件"一边发出 Read 工具调用——两者放同一个 content 数组,是同一个节点。

> [!info] *Signature 字段
> content 块里还有 `textSignature`/`thinkingSignature` 等,是某些 Provider(OpenAI/Google)要求的不透明签名 ID,下一轮必须原样回传。日常理解不需深究,知道是"Provider 间上下文连续性机制"即可。

### ToolResultMessage(五步管道终点产物)

```ts
{
    role: "toolResult",
    toolCallId: string,                       // 对应哪个 ToolCall
    toolName: string,
    content: (TextContent | ImageContent)[],
    details?: TDetails,                       // 结构化详情(给 UI 看)
    isError: boolean,                         // 第5章"永不抛出"产物
    timestamp: number
}
```

通过 `toolCallId` 和 AssistantMessage 里的 ToolCall 对应。

### 完整对话示例

```
[0] UserMessage      "帮我看看 auth.ts"
[1] AssistantMessage  content:[text:"让我看看", toolCall:{id:"tc_001",name:"read",...}]
                     stopReason:"toolUse"     ← 循环继续
[2] ToolResultMessage toolCallId:"tc_001"  content:[text:"import {auth}..."]  isError:false
[3] AssistantMessage  content:[text:"auth.ts 是认证模块..."]
                     stopReason:"stop"       ← 循环结束
```

---

## 三、矛盾:Agent 里的消息不止三种

Agent 内部除"LLM 对话",还有大量功能性数据:Bash 执行记录、压缩摘要、Git 分支切换、附件元信息……

> [!important] 两个读者需求冲突
> - **UI 端**:需要结构化字段(`command`/`output`/`exitCode`/`cancelled`/`truncated`)才能漂亮渲染
> - **LLM 端**:只需一段扁平文本("用户执行了 `ls -la`,输出是 `file1.txt\n...`")塞进 UserMessage
>
> ==冲突点==:为 LLM 提前拍扁存进 UserMessage,UI 拿不回结构化数据;只存结构化自定义消息不进 LLM 上下文,LLM 失忆。

> [!tip] Pi 的解法:两边都不妥协
> 以结构化形式存进 `context.messages`(满足 UI/持久化),在调用 LLM 边界做一次翻译(满足 LLM)。==翻译是最后一刻发生的、有损的、单向的==——损失掉的结构化字段 UI 早就用过了,无所谓。

### coding-agent 的 4 种自定义消息

```
├── BashExecutionMessage       ← Bash 命令执行记录
├── CustomMessage              ← 扩展注入的通用消息
├── BranchSummaryMessage       ← 分支切换时的摘要
└── CompactionSummaryMessage   ← 上下文压缩后的摘要
```

`BashExecutionMessage` 示例:
```ts
{
    role: "bashExecution",
    command: "ls -la", output: "...", exitCode: 0,
    cancelled: boolean, truncated: boolean,
    fullOutputPath?: string, timestamp: number,
    excludeFromContext?: boolean  // 是否排除在 LLM 上下文之外
}
```

> [!success] 自定义消息不只是"翻译中间格式",它带来三个独立能力
> 1. **UI 专用渲染**:按 `role` 分派——`bashExecution` 用终端样式、`compactionSummary` 用摘要卡片
> 2. **持久化恢复**:session 文件存完整结构化数据,重启后 UI 精确还原渲染状态
> 3. **精细化可见性控制**:自定义消息有 `role`,可在 `convertToLlm` 翻译时特殊处理——==标准消息一旦进 messages 数组就一定会翻译发给 LLM,没有"对 UI 可见但对 LLM 不可见"的余地==

---

## 四、第二层:AgentMessage —— 内富外严的双层设计

### AgentMessage 联合类型

```ts
// packages/agent/src/types.ts:314
export type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];
```

```
AgentMessage(Agent 内部使用)
├── Message(可直接发给 LLM 的标准消息)
│   ├── UserMessage / AssistantMessage / ToolResultMessage
└── CustomAgentMessages(仅 Agent 内部使用的扩展消息)
    ├── BashExecutionMessage / CustomMessage
    └── BranchSummaryMessage / CompactionSummaryMessage
```

Agent 内部 `context.messages` 是 `AgentMessage[]`,可混合存放标准和自定义消息,看 `role` 判断是哪种。

### CustomAgentMessages:默认为空的扩展点

```ts
export interface CustomAgentMessages {
    // Empty by default - apps extend via declaration merging
}
```

> [!important] 这个接口在核心包(pi-agent-core)里是空的
> 核心包完全不知道有什么 BashExecutionMessage。它只提供"插槽",让应用层往里插。

### 声明合并:类型安全的扩展魔法

```ts
declare module "@earendil-works/pi-agent-core" {
    interface CustomAgentMessages {
        bashExecution: BashExecutionMessage;
        custom: CustomMessage;
        branchSummary: BranchSummaryMessage;
        compactionSummary: CompactionSummaryMessage;
    }
}
```

编译器自动把这 4 种加入 AgentMessage 联合类型。coding-agent 项目里 `AgentMessage` 变成 7 种消息联合,TypeScript 做完整类型检查。

> [!question] 为什么不用继承或泛型?
> - 继承需修改基类——改不了 `pi-agent-core` 包
> - 泛型需到处传参数——每个用 `AgentMessage` 的函数签名都加泛型参数
> - ==声明合并:核心包零依赖(不知道扩展存在),扩展包却获完整类型安全==

不同应用可注册不同自定义消息。Web UI 就注册了 `user-with-attachments`/`artifact`。

---

## 五、转换边界:convertToLlm —— 一切自定义消息终将变成 User

每次调 LLM 前的最后一刻做翻译。时机精确:

```
context.messages: AgentMessage[]        ← Agent 内部(最多 7 种)
        │
[1] transformContext(可选)              ← AgentMessage[] → AgentMessage[]
        │                                  裁剪旧消息、注入外部上下文
[2] convertToLlm(必须)                  ← AgentMessage[] → Message[]
        │                                  自定义消息翻译成标准格式
        ▼
llmContext.messages: Message[]          ← LLM 看到的(只有 3 种)
        │
streamFunction(model, llmContext, ...)  ← 调 LLM(第4章)
```

> [!note] 顺序:先 transformContext(同层变换),再 convertToLlm(跨层翻译)

### 转换规则:所有自定义消息都变成 user

| role | 怎么处理 |
|---|---|
| `"user"`/`"assistant"`/`"toolResult"` | 直接透传 |
| `"bashExecution"` | `excludeFromContext=true`→过滤;否则→转 UserMessage |
| `"custom"` | 转 UserMessage |
| `"branchSummary"` | 转 UserMessage(XML 标签包裹) |
| `"compactionSummary"` | 转 UserMessage(XML 标签包裹) |

> [!important] 所有自定义消息都变成 user 角色
> 因为 LLM API 对角色顺序严格要求——`user→assistant→user→...` 交替,不能连续两个 assistant。自定义消息本质是"系统注入的信息",放 `user` 角色最安全。

### BashExecutionMessage 转换示例

```
Before(Agent 内部):
{ role:"bashExecution", command:"ls -la", output:"total 32\n...",
  exitCode:0, cancelled:false, truncated:false, timestamp:1748568000000 }

After(LLM 看到的):
{ role:"user",
  content:[{ type:"text", text:"Ran `ls -la`\n```\ntotal 32\n...\n```" }],
  timestamp:1748568000000 }
```

`role`:`bashExecution`→`user`;结构化字段格式化成文本;`cancelled`/`truncated` 等布尔标志融合进文本描述,不再是独立字段。

---

## 六、两阶段管道:为什么 transformContext 和 convertToLlm 分开?

| 步骤 | 处理 | 类型变化 |
|---|---|---|
| `transformContext` | AgentMessage 级操作:裁剪旧消息、注入外部上下文、触发压缩 | `AgentMessage[]` → `AgentMessage[]`(不变) |
| `convertToLlm` | 跨类型翻译:AgentMessage → Message | `AgentMessage[]` → `Message[]`(变了) |

> [!tip] 分开的好处:可只替换其中一个
> - 换上下文管理策略(删最旧→压缩成摘要)→ 只改 `transformContext`
> - 换应用类型(coding-agent→Web 客服,自定义消息变业务消息)→ 只改 `convertToLlm`
> - ==换 LLM 提供商(Claude→GPT),convertToLlm 不用动==——它输出统一 `Message[]`,往下翻译成各 Provider 私有格式是 pi-ai 层(第4章)的工作,两层独立

---

## 七、过滤机制:有些消息 LLM 不该看

### excludeFromContext:一个布尔字段的过滤力

Bash 工具 `!!` 前缀执行命令(`!!secret_cmd`)时,结果对 LLM 不可见:

```ts
case "bashExecution":
    if (m.excludeFromContext) {
        return undefined;   // 直接返回 undefined,后续被 filter 掉
    }
    // ... 否则正常转换
```

> [!info] 数据本身不删除
> `excludeFromContext=true` 的消息仍在 `context.messages`,UI 仍可见渲染。==只在调 LLM 那一刻"隐身"==。这就是"UI 能看、LLM 不能看"的机制。

### 三种消息可见性级别

| 可见性级别 | LLM 看到 | UI 看到 | 实现方式 | 典型消息 |
|---|---|---|---|---|
| 全可见 | 是 | 是 | convertToLlm 正常转换 | 普通的 BashExecution/User/Assistant |
| LLM 不可见 | 否 | 是 | `excludeFromContext=true` | `!!` 前缀的 Bash 执行 |
| 仅持久化 | 否 | 否 | UI 渲染跳过 + convertToLlm 过滤 | Web UI 的 ArtifactMessage |

---

## 八、完整数据流:从用户操作到 LLM 看到的消息

```
用户输入 !ls -la
    │
[1] 创建 BashExecutionMessage { role:"bashExecution", command:"ls -la", ... }
[2] 存入 context.messages: AgentMessage[]
[3] Agent Loop 准备调 LLM(第3章内层循环)
[4] transformContext(可选) AgentMessage[]→AgentMessage[] 裁剪/注入/压缩
[5] convertToLlm(必须) AgentMessage[]→Message[]  BashExecution→UserMessage;excludeFromContext→过滤
[6] LLM 收到 llmContext.messages = [..., {role:"user", content:"Ran `ls -la`..."}]
[7] LLM 回复 → 新 AssistantMessage → 可能触发工具 → ToolResultMessage → 回[2]循环
```

> [!abstract] 核心规律
> Agent 内部用 7 种消息类型自由表达,到 LLM 边界所有自定义消息都翻译回 3 种标准格式。==这个"内富外严"的设计让 Agent 拥有无限扩展能力,同时永远不破坏 LLM 兼容性==。

---

## 九、总结:一条主线 + 三步迁移法

### 一条主线:数据结构要同时照顾两个读者

| 层 | 关心谁 | 数据形式 | 怎么实现 |
|---|---|---|---|
| AgentMessage(内层) | 功能层 | 7 种(3 标准+4 自定义),字段丰富 | 联合类型+声明合并,核心包零依赖、应用层全栈类型安全 |
| Message(外层) | 模型 | 3 种标准,字段精简 | LLM 调用边界做一次 `convertToLlm` 翻译,有损、单向、最后一刻发生 |

### 把这条主线用到自己的项目里(三步)

1. **识别"两个读者"分别要什么**:协议规定什么(不能改)?功能层需要什么(可自定义)?列出来明确需求
2. ==以内层结构化为"源",外层翻译为"流"==:存储和功能层用原始结构化数据(不丢字段不拍扁);到协议边界再做有损翻译。不要为协议方便提前拍扁——一旦拍扁,UI 和持久化再也拿不回结构
3. **用类型系统扩展点做"核心+应用"分层**:核心包定义协议接口(封闭)、留空扩展插槽;应用包通过声明合并注入具体类型。不需要继承,不需要泛型参数污染

> [!example] 同一思想的三层递进
> 本章的 `Tool→AgentTool→ToolDefinition`(第5章)也是同样思想:每层只加自己这个层级需要的能力,不越界。识别"分层点"、给每层划清职责边界是这种设计法的核心。

---

## 关键源码索引

| 位置 | 内容 |
|---|---|
| `ai/types.ts:322-408` | Message 联合类型 + 三种消息接口 |
| `agent/types.ts:305-314` | CustomAgentMessages + AgentMessage |
| `coding-agent/messages.ts:70-77` | coding-agent 声明合并 |
| `agent-loop.ts:275-308` | 转换管道(transformContext → convertToLlm) |
| `coding-agent/messages.ts:82-195` | 自定义转换规则 |
| `coding-agent/messages.ts:38-39` | excludeFromContext 字段 |

下一章:[[M07-事件驱动-Agent的神经系统]] —— 事件怎么从 Agent 内部传到外部?为什么发完事件要"同步等待"监听器?

> [!note]- 版本说明
> 基于 Pi **v0.80.2**,代码分析以 earendil-works/pi 仓库实际源码为准。原文:[dg-ai-notes.pages.dev/modules/ch06-messages](https://dg-ai-notes.pages.dev/modules/ch06-messages/)
