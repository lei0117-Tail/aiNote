---
title: M01 开篇 —— 为什么 Pi-Agent 值得你花时间
date: 2026-08-16
source: https://dg-ai-notes.pages.dev/modules/ch01-overview/
version: Pi v0.80.2
module: M01
tags:
  - pi-agent
  - agent
  - coding-agent
  - 架构
  - 学习笔记
  - ai
aliases:
  - Pi-Agent 开篇
  - Pi 是什么
---

# M01 开篇 —— 为什么 Pi-Agent 值得你花时间

> [!abstract] 一句话定义
> **Pi 是一款极简、可扩展的终端编码 Agent 外壳(coding agent harness)**,由 libGDX 作者 Mario Zechner 创建,全部用 TypeScript 编写,MIT 协议开源。

Pi 有 ==三个身份==,指向同一个项目:
1. **编码工具** —— 日常好用的终端 Agent
2. **学习教材** —— 可读完的 Agent 设计教科书
3. **开发 SDK** —— 构建自己 Agent 的三层堆栈

---

## 关键数字(v0.80.2)

| 指标           | 数值                             | 含义                                                       |
| ------------ | ------------------------------ | -------------------------------------------------------- |
| GitHub Stars | 64,000+                        | 十个月增长,社区验证需求                                             |
| 内置工具         | 4 核心 + 3 辅助                    | 核心:`read`/`write`/`edit`/`bash`;辅助:`grep`/`find`/`ls`    |
| 系统提示词        | 静态模板 ~90 词(运行时 200-400 词)      | 对比 Claude Code 的数万字                                      |
| TUI 代码量      | ~12,000 行                      | 核心 `tui.ts` 单文件约 1,700 行                                 |
| 支持供应商        | 35 个 KnownProvider(约 27 个独立品牌) | Anthropic/OpenAI/Google/Groq/Ollama 等                    |
| 核心包数量        | 4 个                            | `pi-ai` / `pi-agent-core` / `pi-tui` / `pi-coding-agent` |
| 运行模式         | 4 种                            | 交互 / print-JSON / RPC / SDK                              |

> [!note] 数字勘误
> 官网早期营销材料常说"4 个内置工具"、"15+ 供应商"、"约 600 行 TUI"——前两者指核心 4 工具(不含辅助)和早期知名厂商,"600 行 TUI"是早期版本。v0.80.2 实际已增长到约 12,000 行。

---

## 四层架构:四个核心包各司其职

```
┌──────────────────────────────────────────┐
│          pi-coding-agent                 │  ← 完整 CLI 产品 + SDK
│  系统提示词 · 内置工具 · 会话管理 · 扩展  │
├──────────────────────────────────────────┤
│  pi-tui              │  pi-agent-core    │  ← 终端 UI + Agent 引擎
│  差分渲染 · 组件系统  │  AgentLoop · 工具 │
│                      │  系统 · 事件流    │
├──────────────────────┴───────────────────┤
│              pi-ai                       │  ← 多供应商 LLM 抽象
│  统一 API · 上下文交接 · 流式 · Token 追踪│
└──────────────────────────────────────────┘
```

> [!important] 堆栈关系
> - **三层堆栈**:`pi-ai → pi-agent-core → pi-coding-agent`(每层可独立使用)
> - **正交 UI 库**:`pi-tui` 与 Agent 体系==完全解耦==,只被 coding-agent 单向依赖
> - 你可以只用 `pi-ai` 调模型,或用 `pi-agent-core` 在自己应用里跑 Agent Loop,完全不碰 CLI
> - 外围还有实验性 `pi-orchestrator`(v0.80.x 新增,多 Agent 编排,不在学习主线)

---

## 视角一:作为编码 Agent —— 积木,而非整车

> [!quote] 核心定位
> Pi 不是又一个 Cursor 或 Claude Code——它是==一盒让你按自己心意拼出自己 CodingAgent 的积木==。

| 类比 | 特征 |
|---|---|
| Cursor / Claude Code | 造好的**整车**,坐进去就能开 |
| Pi | 发动机、底盘、转向柱——**零件 + 一份"已验证能跑"的保证** |

这个定位是 Pi 一切设计决策的源头:
- 系统提示词只有 ~1,000 token → "该说什么"由你决定
- 只内置 4 个工具 → 更多内置工具 = 更多不可改变的约束
- 没有 MCP / 计划模式 / 子 Agent / 待办 → 这些是"整车功能",Pi 留给你用扩展去搭

> [!tip] 适用人群
> - ✅ 生活在终端里、熟悉 tmux 和容器、对关不掉的功能烦躁 → **Pi**
> - ✅ 要零配置开箱即用 → Cursor 或 Claude Code
> - 这不是优劣问题,是工作方式契合度问题

### 五根定制杠杆

Pi 缺的功能,全都可以自己造。==极简核心 + 强大杠杆==,让你拿到的是"一个能长成任何形状的 Agent"。

| 杠杆                 | 作用                     | 特点                                      |
| ------------------ | ---------------------- | --------------------------------------- |
| **扩展(Extensions)** | 最强一根,加新工具/命令/钩子/TUI 组件 | TypeScript 文件,自动加载 + ==热重载==            |
| **技能(Skills)**     | 加新知识("遇到 X 该怎么做")      | 渐进式披露,按需进上下文,平时 0 token                 |
| **提示词模板**          | 可复用工作流                 | markdown 模板,支持参数,斜杠命令加载                 |
| **主题(Themes)**     | TUI 皮肤                 | 会话中切换,实时生效                              |
| **Pi 包(Packages)** | 把上面四样打包分发              | `pi install npm:@foo/pi-tools` 或 `git:` |

> [!info] 扩展 vs 技能
> - **扩展** = 给 Agent 加新**能力**(工具/命令/模式)
> - **技能** = 给 Agent 加新**知识**(任务做法)
> - 两者可叠加:扩展可注册技能,技能可调用扩展工具

### 默认配置的日常红利

Pi 在 **TerminalBench**(约 82 项任务评估)排名第二(用 Claude Opus 4.5,仅次于 Terminus)——尽管没 MCP、子 Agent、计划模式、后台 bash、内置待办。极简没牺牲能力。

1. **上下文干净** —— 系统 prompt + 工具定义 < 1,000 token(对比 Claude Code 数万)。可用 `SYSTEM.md` 替换整个系统提示词
2. **透明到骨头** —— 每条消息、每次工具调用 IO、跨会话成本追踪、HTML/JSON 导出全可见
3. **模型自由** —— 35 个 KnownProvider,会话中途 `/model` 或 `Ctrl+L` 切换;`pi-ai` 底层处理跨供应商上下文交接(思考轨迹转换、签名 blob 回放)
4. **树状会话(DAG)** —— `/tree` 跳到任意历史消息分叉,所有分支活在同一文件
5. **YOLO 模式** —— 默认不经审批执行。理由:审批弹窗导致"弹窗疲劳",沦为"安全表演(security theater)";建议用容器化做安全边界,需审批则 ~50 行扩展实现

### 上手 & 接国内模型

```bash
curl -fsSL https://pi.dev/install.sh | sh
# 或
npm install -g --ignore-scripts @earendil-works/pi-coding-agent
```

接国内模型(智谱/DeepSeek/Kimi/Qwen)用 `~/.pi/agent/models.json`,启动时由 `ModelRegistry.create()` 自动读取:

```json
{
  "providers": {
    "deepseek": {
      "baseUrl": "https://api.deepseek.com",
      "api": "openai-completions",
      "apiKey": "<your-key>",
      "models": [
        { "id": "deepseek-v4-flash", "name": "DeepSeek V4 Flash" },
        { "id": "deepseek-v4-pro", "name": "DeepSeek V4 Pro",
          "contextWindow": 1000000, "maxTokens": 384000 }
      ]
    }
  }
}
```

> [!warning] 安全提醒
> `apiKey` 明文存放。务必把 `.pi/` 加进 `.gitignore`,否则一个 `git add .` 就会泄露。

关键字段:`providers`(顶层字典,键名自起)、`api`(协议:`openai-completions`/`anthropic-messages`/`openai-responses`)、`baseUrl`、`apiKey`、`models`(`id` 是真实模型名,`name` 是 TUI 显示名)、`contextWindow`/`maxTokens`(可选,影响压缩策略)。

切换方式:会话中 `/model` 临时切;`settings.json` 设 `defaultProvider`+`defaultModel` 设默认;`pi models` 查列表。

---

## 视角二:作为学习教材 —— Agent 设计教科书

> [!important] 为什么是 Pi?因为它足够小
> Pi 核心循环只有几百行,但设计质量不"简陋"——TerminalBench 排名第二。
> ==你可以在有限时间内真正"读完"一个高质量 Agent 的全部核心代码。==这种事对 Claude Code、LangChain 都不可能。

### 减法哲学:真正的教育在取舍里

看"什么都做了"的框架,只学到"他们做了什么";看==刻意什么都不做==的框架,才学到"做 Agent 到底需要什么"。Pi 官网 "What we didn't build" 是一份倒过来的宣言:

| Pi 不做   | 为什么不做                                           | 替代方案                         |
| ------- | ----------------------------------------------- | ---------------------------- |
| MCP 支持  | MCP 服务器(如 Playwright MCP)会灌入 13,700+ token 工具描述 | 带 README 的 CLI 工具,Agent 按需读取 |
| 子 Agent | 增加复杂度,降低可观察性                                    | tmux 多实例,或专用扩展               |
| 权限弹窗    | 导致"弹窗疲劳",沦为安全表演                                 | 容器化隔离,或扩展搭审批                 |
| 计划模式    | 计划写到 markdown 更持久可复用                            | 写 `plan.md` 文件               |
| 后台 bash | tmux 已解决                                        | 用 tmux                       |
| 内置待办    | `TODO.md` 更灵活                                   | markdown 文件或自建扩展             |

### 本教程章节地图

| 章 | 主题 | 难度 |
|---|---|---|
| 1 | 开篇总览 | 入门 |
| [[M02-三层架构-Pi-Agent的骨骼]] | 项目结构与分层架构 | 入门 |
| [[M03-Agent-Loop-让模型转动的引擎]] | Agent Loop | ★ 核心 |
| [[M04-模型调用-一行代码驾驭多模型]] | 模型调用 | ★ 核心 |
| [[M05-工具系统-Agent的手脚如何管住]] | 工具系统 | ★ 核心 |
| [[M06-消息系统-Agent的记忆组织传递]] | 消息系统 | ★ 核心 |
| [[M07-事件驱动-Agent的神经系统]] | 事件驱动架构 | 进阶 |
| [[M08-上下文工程-有限窗口装无限对话]] | 上下文工程 | 进阶 |
| [[M09-上下文压缩-对话太长怎么办]] | 上下文压缩 | 进阶 |
| [[M10-会话管理-存储恢复与分叉]] | 会话管理 | 进阶 |

每章回答三层:**是什么**(概念)、**怎么做**(源码)、**为什么这样做**(取舍)。前 6 章建议顺序通读,第 7 章起可按需跳读。

---

## 视角三:作为 SDK —— 构建你自己的 Agent

### SDK 三层堆栈 + 正交 UI 库

```
pi-ai  →  pi-agent-core  →  pi-coding-agent     (三层堆栈,每层独立可用)
                ↑
            pi-tui  (正交 UI 库,与 Agent 无关)
```

| 层                 | 职责           | 依赖         | 复用场景                                |
| ----------------- | ------------ | ---------- | ----------------------------------- |
| `pi-ai`           | 只管调模型        | 无 Agent 概念 | 聊天机器人、文档分析、代码审查等任何调 LLM 的项目         |
| `pi-agent-core`   | 只管跑循环        | 依赖 `pi-ai` | 数据分析/客服/自动化测试等任意 Agent              |
| `pi-coding-agent` | 完整 CLI + SDK | 组装下面两层     | 嵌入自己应用(headless 模式)                 |
| `pi-tui`          | 终端 UI 库      | ==零依赖==兄弟包 | 任何终端交互界面(CLI/dashboard/TUI 游戏/REPL) |

> [!example] SDK 调用示意
> ```ts
> // Layer 1: 只调模型
> import { getModel, stream } from '@earendil-works/pi-ai/compat';
> const model = getModel('anthropic', 'claude-sonnet-4-5');
> const context = { systemPrompt: 'You are helpful.',
>   messages: [{ role: 'user', content: 'Hello!' }] };
> for await (const event of stream(model, context)) {
>   if (event.type === 'text_delta') process.stdout.write(event.delta);
> }
>
> // Layer 3: 嵌入应用
> import { createAgentSession } from '@earendil-works/pi-coding-agent';
> const session = await createAgentSession({
>   cwd: '/path/to/project',
>   model: getModel('anthropic', 'claude-sonnet-4-5'),
> });
> session.subscribe((e) => { if (e.type === 'turn_end') console.log('一轮完成'); });
> await session.prompt('Read the codebase and explain the architecture.');
> ```

`pi-tui` 是 Mario(libGDX 作者)的老本行作品,约 12,000 行:差分渲染(每帧只重绘变化单元格)、保留模式 UI(类 React 声明式)、内置组件(自动补全输入框/markdown 渲染/语法高亮/模糊搜索)。它根本不在乎你在调 LLM 还是做别的事,是 Pi 最容易脱离 Pi 单独复用的部分。

### 四种运行模式

| 模式 | 用途 | 示例 |
|---|---|---|
| 交互模式 | 日常编程 TUI | `pi` |
| print/JSON 模式 | 脚本和 CI/CD | `pi -p "explain this code"` |
| RPC 模式 | stdin/stdout 交换 JSON | 集成进非 Node.js 程序 |
| SDK 模式 | 嵌入自己应用 | `createAgentSession()` |

多模式设计让 Pi 从"开发者手边工具"无缝演进为"生产系统 Agent 能力提供者",项目成长后不需要换框架。OpenClaw 等项目已在生产用 Pi SDK。

---

## Pi 的对立面:两种相反哲学

| 路线     | 代表                                              | 起点     | 用户姿态               |
| ------ | ----------------------------------------------- | ------ | ------------------ |
| **全包** | Claude Code、Everything Claude Code(214K+ Stars) | 从"满"开始 | 慢慢删,被推送着适应工具       |
| **极简** | Pi                                              | 从"空"开始 | ==让你来填==,工具适应你的工作流 |

> [!quote]
> 像 Claude Code 和 Codex CLI 这类工具,优化的是"在精心打磨的环境里尽快拿到第一次成功"…… Pi 把优先级挪到了"对工具的所有权"上。它不给你计划模式,它给你==构造一个完全按你心意行事的计划模式所需的构建块==。——社区观察者 Pasquale

一位社区观察者把 Pi 称作"世上最可驾驭(steerable)的外壳"——可驾驭不是因为它响应快,而是因为你对它每一个动作都有==否决权和改造权==。

---

## 总结

Pi 是一个"三位一体"的项目:

- **作为工具**:极简、透明、可驾驭的终端编码 Agent。上下文干净、模型自由、树状会话、YOLO 默认
- **作为教材**:高质量、可读完的 Agent 设计参考。10 章覆盖从 Agent Loop 到会话管理的核心决策点
- **作为 SDK**:层次分明、可独立复用的开发套件。三层堆栈 + 正交 `pi-tui`,四种运行模式覆盖本地到生产

> [!success] 最重要的一点
> Pi 证明了==做减法是一种有竞争力的产品立场==。在一个朝"全包"狂奔的赛道里,**"我不需要的,就不会被构建"**这句话本身,就是一项真正的功能。

---

> [!note]- 版本说明
> 本文档系列基于 Pi **v0.80.2** 编写。代码分析以 earendil-works/pi 仓库实际源码为准(教程链接指向 main 分支,可能与 v0.80.2 有微小差异)。
> 原文地址:[dg-ai-notes.pages.dev/modules/ch01-overview](https://dg-ai-notes.pages.dev/modules/ch01-overview/)
