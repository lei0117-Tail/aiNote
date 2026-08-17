---
title: M05 工具系统 —— Agent 的手脚是怎么被管住的
date: 2026-08-17
source: https://dg-ai-notes.pages.dev/modules/ch05-tools/
version: Pi v0.80.2
module: M05
tags:
  - pi-agent
  - agent
  - coding-agent
  - 架构
  - 学习笔记
  - ai
aliases:
  - Pi 工具系统
  - Pi 五步管道
  - Pi AgentTool
  - Pi ToolDefinition
  - Pi Operations 抽象
  - Pi 错误即消息
---

# M05 工具系统 —— Agent 的手脚是怎么被管住的

> [!abstract] 核心结论
> 工具不是简单的函数调用,而是一条 ==五步管道==:参数预处理 → Schema 验证 → 权限拦截 → 工具执行 → 结果后处理。每一步出错都统一编码成 `isError:true` 的 `ToolResultMessage` 发给模型——==错误信息是给模型的反馈,不是给框架的终止信号==。

承接 [[M04-模型调用-一行代码驾驭多模型]]——模型返回 `ToolCall`,谁来执行。

---

## 一、三层类型:为什么"一个工具"要分三层?

### 第一层:Tool —— 一张"名片"(pi-ai)

```ts
// packages/ai/src/types.ts:433-437
export interface Tool<TParameters extends TSchema = TSchema> {
    name: string;            // 工具名,如 "read"、"bash"
    description: string;     // 给 LLM 看的描述
    parameters: TParameters; // 参数 JSON Schema(TypeBox 定义)
}
```

三个字段。唯一关心:怎么把工具信息告诉模型。能描述自己,但不能执行任何操作。

### 第二层:AgentTool —— 加上"执行能力"(pi-agent-core)

```ts
// packages/agent/src/types.ts:371-394
export interface AgentTool<TParameters, TDetails> extends Tool<TParameters> {
    label: string;                     // 给人看的标签(不同于给 LLM 的 description)
    prepareArguments?: (args) => Static<TParameters>;  // 兼容性垫片
    execute: (toolCallId, params, signal?, onUpdate?) => Promise<AgentToolResult>;
    executionMode?: "sequential" | "parallel";  // 执行模式
}
```

新增 4 个字段:`label`(模型看 name,UI 看 label)、`prepareArguments`(兼容层)、`execute`(真正干活)、`executionMode`(能否并行)。

### 第三层:ToolDefinition —— 产品层再加东西(pi-coding-agent)

`execute` 比 `AgentTool` ==多一个参数== `ctx: ExtensionContext`(会话上下文):
```
AgentTool.execute:     (toolCallId, params, signal, onUpdate) => ...
ToolDefinition.execute: (toolCallId, params, signal, onUpdate, ctx) => ...
```
新增 `promptSnippet`(系统提示词片段)、`renderCall`/`renderResult`(UI 渲染)等字段。

### 桥接两层:wrapToolDefinition

```ts
// tool-definition-wrapper.ts
export function wrapToolDefinition(definition, ctxFactory?) {
    return {
        name: definition.name, /* ...复制字段... */
        execute: (toolCallId, params, signal, onUpdate) =>
            definition.execute(toolCallId, params, signal, onUpdate, ctxFactory?.()),
            // ↑ 通过闭包注入 ExtensionContext,Agent Loop 永远不知道它的存在
    };
}
```

> [!question] 为什么非要分三层?
> 每层有独立依赖范围。`pi-ai` 的 `Tool` 只依赖 TypeBox `TSchema`。若加 `renderCall`(返回终端 UI 组件),`pi-ai` 就得依赖终端 UI 渲染库——但它是纯模型适配层。==三层递进本质:每层只加自己这个层级需要的能力,不越界==。

| 层 | 管 | 关心 |
|---|---|---|
| `Tool` | 能描述自己 | LLM 关心"长什么样" |
| `AgentTool` | 能被执行 | Agent 关心"怎么执行" |
| `ToolDefinition` | 能被展示和扩展 | coding-agent 关心"怎么显示" |

---

## 二、五步管道:工具调用不是"调个函数就完了"

> [!warning] 为什么不能直接调函数?
> 模型输出不总是规矩的:
> - 参数格式不对:Edit 期望 `edits` 数组,某些模型序列化成字符串 `"[{...}]"`
> - 参数类型错误:Read 的 `path` 是 string,模型可能传 number `12345`
> - 危险操作:模型要求执行 `rm -rf /`

```
LLM 输出 ToolCall
    │
┌───┴──────────────────────────────────────────────┐
│ 第 1 步:prepareArguments(参数预处理)处理 LLM 怪癖  │
│ 第 2 步:validateToolArguments(Schema 验证)TypeBox  │
│ 第 3 步:beforeToolCall(前置钩子)权限拦截可阻止    │
│ 第 4 步:tool.execute(实际执行)支持 onUpdate 流式  │
│ 第 5 步:afterToolCall(后置钩子)可修改返回值       │
└──────────────────────────────────────────────────┘
    │
    ▼
ToolResultMessage
```

### 第 1 步:prepareArguments —— 兼容性垫片

```
模型传来:{ edits: "[{\"oldText\":\"hello\",\"newText\":\"world\"}]" }  // 字符串化的数组
处理后:  { edits: [{ oldText:"hello", newText:"world" }] }
```

> [!info] 为什么不和 Schema 验证一起处理?
> `prepareArguments` 是"我知道某模型会犯什么错"的兼容层(只处理特定模型已知问题);`validateToolArguments` 是"不管谁调我都得验"的安全层(保证类型正确)。==一个是兼容性,一个是正确性==,混在一起难维护。

### 第 2 步:validateToolArguments —— Schema 验证

`path` 定义 string 但模型传 number → 验证失败报错不执行。验证错误被 `prepareToolCall` try-catch 捕获,生成错误 ToolResultMessage。工具永远收不到类型错误的参数。

### 第 3 步:beforeToolCall —— 前置钩子(可阻止)

| 返回值 | 效果 |
|---|---|
| `undefined` | 放行继续执行 |
| `{ block:true, reason:"危险命令" }` | 阻止,生成错误 ToolResultMessage |

> [!note] 即使被阻止也是正常消息
> 结果仍是 `ToolResultMessage` 只是 `isError:true`。模型看到拒绝原因,自己决定下一步(换命令或向用户解释)。==不抛异常,不打断循环==。

### 第 4 步:tool.execute —— 实际执行

```ts
execute: (toolCallId, params, signal, onUpdate) => Promise<AgentToolResult>
```

`onUpdate` 解决==长任务进度感知==——Bash 跑 30 秒命令,没有它用户只能盯加载动画。Bash 每 100ms 推送当前输出,Grep 每找到一批匹配推送一次。推送包装成 `tool_execution_update` 事件流向 UI。==没有 onUpdate 工具是黑盒,有了它可观察==。

> [!tip] acceptingUpdates 闸门
> `execute` 是 async,return 后内部可能还有未结束的异步操作(Bash 子进程还在打印日志)。`execute` 一旦返回立即关 `acceptingUpdates` 标志位,之后所有 `onUpdate` 调用==静默丢弃==,防止污染已结束的工具调用。

### 第 5 步:afterToolCall —— 后置钩子(可修改结果)

| 场景 | 做什么 | 怎么做 |
|---|---|---|
| 脱敏 | 替换敏感信息 | 返回 `{ content:[{type:"text", text:"[已脱敏]"}] }` |
| 审计 | 记录调用详情 | 读 result 写日志,返回 `undefined`(不改结果) |
| 修错 | 错误结果修正为正常 | 返回 `{ isError:false, content:[...] }` |
| 早停 | 当前批次后停止 | 返回 `{ terminate:true }` |

合并语义是字段级覆盖——提供了就替换,没提供就保留原值。

### 管道终点:ToolResultMessage

```ts
{
    role: "toolResult",
    toolCallId: "call_abc123",      // 关联到原始 ToolCall
    toolName: "read",
    content: [{ type: "text", text: "1│ import { Agent }..." }],
    details: { language: "typescript" },  // 给 UI 的元数据
    isError: false,
    timestamp: 1700000000000,
}
```

追加到对话历史,下一轮作为上下文发给模型。

---

## 三、并行 vs 串行:一票否决 + 三阶段

模型一次回复可能含多个 ToolCall。直觉是 `Promise.all` 并行,但两个 edit 操作同文件并行会互相覆盖。

### Pi 调度策略:一票否决

```js
const hasSequentialToolCall = toolCalls.some(
    (tc) => tools?.find(t => t.name === tc.name)?.executionMode === "sequential",
);
if (config.toolExecution === "sequential" || hasSequentialToolCall) {
    return executeToolCallsSequential(...);   // 整批串行
}
return executeToolCallsParallel(...);
```

> [!question] 为什么一票否决而不是只串行冲突工具?
> "哪些工具会冲突"很难精确判断。edit 和 edit 操作不同文件就安全?万一有依赖关系呢?Pi 选择==保守策略:宁可多等,不可出错==。

### 并行执行三阶段设计

```
阶段1 准备(顺序):  A: prepare→validate→beforeToolCall
                  B: prepare→validate→beforeToolCall   ← 必须顺序(可能有副作用)
阶段2 执行(并行):  A、B、C 的 tool.execute 同时(Promise.all)  ← 只有 execute 并行
阶段3 事件(有序):  end 按完成顺序发;result 按调用顺序发       ← 模型依赖调用顺序
```

> [!info] v0.80.2 内置工具默认并行
> 7 个内置工具(read/write/edit/bash/grep/find/ls)都没显式声明 `executionMode`,默认 `"parallel"`。Edit 工具文件安全靠内部 `withFileMutationQueue`(`file-mutation-queue.ts:32-61`,Edit 在 `edit.ts:312` 调用)——同文件编辑串行化,是工具自己做的第二道防线,不依赖外层 `executionMode`。

---

## 四、永不抛出:工具出错也是一条消息

### 6 种错误,1 种产物

| 哪一步出错 | 怎么处理 | 最终产物 |
|---|---|---|
| 工具未找到 | 直接返回错误结果 | `ToolResultMessage { isError:true, content:"Tool xxx not found" }` |
| prepareArguments 抛异常 | try-catch 捕获 | `ToolResultMessage { isError:true }` |
| Schema 验证失败 | try-catch 捕获 | `ToolResultMessage { isError:true }` |
| beforeToolCall 阻止 | 返回阻止结果 | `ToolResultMessage { isError:true }` |
| tool.execute 抛异常 | executePreparedToolCall catch | `ToolResultMessage { isError:true }` |
| afterToolCall 抛异常 | finalizeExecutedToolCall catch | `ToolResultMessage { isError:true }` |

==所有错误最终形态都是 ToolResultMessage,没有一种以"抛异常"形式逃出管道==。

### 关键代码:tool.execute 的双重防护

```ts
// agent-loop.ts:628-669
async function executePreparedToolCall(prepared, signal, emit) {
    const updateEvents: Promise<void>[] = [];
    let acceptingUpdates = true;
    try {
        const result = await prepared.tool.execute(
            prepared.toolCall.id, prepared.args, signal,
            (partialResult) => {
                if (!acceptingUpdates) return;     // settle 后的孤儿回调忽略
                updateEvents.push(/* 发 tool_execution_update */);
            },
        );
        acceptingUpdates = false;
        await Promise.all(updateEvents);
        return { result, isError: false };
    } catch (error) {
        acceptingUpdates = false;
        await Promise.all(updateEvents);  // 先等进度事件发完,再编码错误
        return {
            result: createErrorToolResult(error instanceof Error ? error.message : String(error)),
            isError: true,
        };
    } finally {
        acceptingUpdates = false;  // 兜底关闭闸门
    }
}
```

三个关键决策:① 异常被 catch 不穿透;② 异常被"翻译"成正常结果(长得像正常结果,只是 content 装错误文本);③ 进度事件先发完再编码错误(否则 UI 看到"工具先报错再吐最后一行进度"的诡异画面)。

### 异常 → 消息:接收者不同

```
异常(穿透前):                消息(编码后):
Error: ENOENT                  { role:"toolResult", isError:true,
  → 一路穿透管道                  content:[{text:"ENOENT: no such file..."}] }
  → 打断 Agent Loop              → 追加到对话历史
  → UI 卡死                      → 下一轮发给模型 → 模型自己决定怎么办
```

> [!important] 异常和消息的区别不在"内容是什么"
> 两者描述同一件事,区别在==接收者是谁==。异常接收者是调用栈(打断循环);消息接收者是模型(消化错误继续)。Pi 选择把异常翻译成消息,让"工具出错"成为模型可见、可处理的正常信息流。

### 为什么"伪装成消息"是最佳实践?

| 错误场景 | 模型看到后的合理反应 |
|---|---|
| `read("/path/a.ts")` 报"文件不存在" | 先 `ls` 看目录有什么,找正确文件名再读 |
| `edit` 报"oldText 找不到匹配" | 先 `read` 看实际内容,调整 oldText 重试 |
| `bash("npm run build")` 报"模块未找到" | `npm install` 后再 build |
| `bash("rm -rf /")` 被阻止 | 换安全写法或向用户解释 |

> [!success] 让模型自己决定下一步,比框架替它决定更好
> 每种场景正确动作都不同,只有模型有足够上下文判断。框架直接抛异常等于放弃模型自我纠错能力——用户只能崩溃后手动重启。==错误不会终止流程,而是成为下一步决策的输入==——这是 Agent 比传统脚本更"智能"的关键。

### 错误描述越具体,模型纠错能力越强

```
模糊错误(不可取):              具体 error.message(推荐):
{ content:[{text:"Read failed"}]  { content:[{text:"Offset 200 is beyond
  isError:true }                     end of file (100 lines total)"}]
                                   isError:true }
```

模型看"Read failed"只能盲目重试;看"Offset 200 is beyond end of file (100 lines total)"能立刻明白"文件只有 100 行,offset 给错了",下次给 `offset:50`。==具体错误描述等于给模型一份"怎么改才对"的提示==。

### Pi 的真实做法:两层错误处理,分层负责

```
第一层(工具内部,主动):识别已知错误类型,包装成具体可读描述
└── Read/Edit/Bash 都是这样——Bash 把"已输出的内容"附在错误里
└── 目的:给模型提供"为什么失败、怎么改才对"的具体线索

第二层(框架兜底,被动):executePreparedToolCall 的 catch
└── 只在工具没识别出来时生效
└── 不创造新错误描述,只把 error.message 原样透传给模型
└── 目的:保证任何异常都不会穿透到 Agent Loop
```

**Bash 工具的教科书级写法**(`bash.ts:390-407`):
```ts
} catch (err) {
    const snapshot = await finishOutput();  // 先固定已输出内容
    const { text } = formatOutput(snapshot, "");
    if (err.message === "aborted") throw new Error(appendStatus(text, "Command aborted"));
    if (err.message.startsWith("timeout:")) throw new Error(appendStatus(text, `Command timed out after ${...} seconds`));
    if (exitCode !== 0 && exitCode !== null) throw new Error(appendStatus(outputText, `Command exited with code ${exitCode}`));
    throw err;    // 识别不了的异常,原样抛出交给框架兜底
}
```

主动识别已知类型用 `appendStatus` 把"已输出内容+具体原因"打包成新 Error;识别不了的才 `throw err` 原样抛出。

### 自定义工具最佳实践

```ts
execute: async (id, params, signal, onUpdate) => {
    try {
        // 业务逻辑
        return { content: [...], details: {...} };
    } catch (err) {
        if (err instanceof MyKnownErrorA) throw new Error(`具体描述A:${err.message}。建议修复...`);
        if (err instanceof MyKnownErrorB) throw new Error(`具体描述B:${err.message}。可能原因...`);
        throw err;  // 识别不了的,让框架兜底 catch 透传 err.message
    }
}
```

两条原则:
1. **能识别的错误一定要包装**:附"是什么错、为什么、怎么办"线索。"文件 /a.ts 不存在,目录下有 [b.ts, c.ts]"比"文件不存在"强 10 倍
2. ==识别不了的不要硬编码描述==:直接 `throw err`,让框架透传 `err.message`。不要写 `throw new Error("操作失败")`——那把所有未知错误涂成同一种颜色,模型无法区分

---

## 五、【进阶】Operations 抽象:工具执行不等于系统调用

### 问题:工具代码写死了系统调用

`fs.readFileSync(path)` 写死——只认本地文件系统。想 Mock 测试、SSH 远程、Docker 容器执行,就得改工具代码。

### 解法:工具不直接调系统 API,调接口

```ts
export interface ReadOperations {
    readFile: (absolutePath: string) => Promise<Buffer>;
    access: (absolutePath: string) => Promise<void>;
    detectImageMimeType?: (absolutePath: string) => Promise<string | null>;
}

execute: async (toolCallId, params, signal, onUpdate, ctx) => {
    const ops = options?.operations ?? defaultReadOperations;
    await ops.access(absolutePath);
    const buffer = await ops.readFile(absolutePath);  // 通过接口读,不直接调 fs
}
```

```ts
// 本地(默认)                    // 测试(Mock)                      // 远程(SSH)
createReadToolDefinition(cwd)     createReadToolDefinition(cwd, {     createReadToolDefinition(cwd, {
                                       operations: {                      operations: {
                                           readFile: () => Buffer.from("mock"),  readFile: (p) => sshExec(`cat ${p}`),
                                           access: () => {},                     access: (p) => sshExec(`test -r ${p}`),
                                       }                                       }
                                   })                                     })
```

> [!info] 接口按工具需求裁剪,非大一统
> Read 只要 `readFile`/`access`;Grep 只要 `isDirectory`/`readFile`;Bash 只要 `exec`。每个工具只声明自己需要的方法,==不多不少==。

| 工具 | 接口 | 方法 |
|---|---|---|
| Read | `ReadOperations` | `readFile`, `access` |
| Write | `WriteOperations` | `writeFile`, `mkdir` |
| Edit | `EditOperations` | `readFile`, `writeFile`, `access` |
| Bash | `BashOperations` | `exec` |
| Grep | `GrepOperations` | `isDirectory`, `readFile` |
| Find | `FindOperations` | `exists`, `glob` |
| Ls | `LsOperations` | `exists`, `stat`, `readdir` |

---

## 六、方法论提炼:四个设计模式

1. **分层接口递进法**:基础层管"能描述"(Tool),运行时层加"能执行"(AgentTool),产品层加"能展示和扩展"(ToolDefinition)。包装器桥接层间差异
2. **管道+钩子模式**:核心流程是管道(prepare→validate→execute),前后各一个钩子(before/after)可拦截或修改。==管道内每步出错都不抛异常,统一编码为正常消息==
3. **错误即消息原则**:每步出错都编码成 `isError:true` 的 ToolResultMessage。模型自己根据错误信息决定下一步。即便未知异常也用 `String(error)` 兜底成消息,==绝不让原始异常穿透打断 Agent Loop==
4. **Operations 抽象法**:工具不直接调系统 API,通过最小化 Operations 接口间接调用。测试可 Mock,远程可 SSH,不改工具代码

---

## 关键源码索引

| 位置 | 内容 |
|---|---|
| `ai/types.ts:433-437` | Tool(第一层) |
| `agent/types.ts:371-394` | AgentTool(第二层) |
| `coding-agent/extensions/types.ts:435-482` | ToolDefinition(第三层) |
| `tool-definition-wrapper.ts:5-18` | wrapToolDefinition(包装器) |
| `agent-loop.ts:562-626` | prepareToolCall(五步管道前 3 步) |
| `agent-loop.ts:628-669` | executePreparedToolCall(第 4 步+框架兜底 catch) |
| `agent-loop.ts:671-714` | finalizeExecutedToolCall(第 5 步) |
| `agent-loop.ts:716-721` | createErrorToolResult(错误消息搬运函数) |
| `bash.ts:390-407` | Bash 工具主动识别错误的典范 |
| `read.ts:275` | Read 工具附加文件总行数 |
| `edit.ts:330` | Edit 工具附加文件路径 |
| `read.ts:43-50` | ReadOperations(Operations 抽象) |

下一章:[[M06-消息系统-Agent的记忆组织传递]] —— `ToolResultMessage` 长什么样?Agent 内部消息和发给模型的消息一样吗?

> [!note]- 版本说明
> 基于 Pi **v0.80.2**,代码分析以 earendil-works/pi 仓库实际源码为准。原文:[dg-ai-notes.pages.dev/modules/ch05-tools](https://dg-ai-notes.pages.dev/modules/ch05-tools/)
