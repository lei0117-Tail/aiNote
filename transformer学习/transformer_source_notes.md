# Transformer 翻译原理 & GPT 生成原理

---

## 一、Transformer 翻译原理（Encoder-Decoder）

### 整体结构

翻译模型由两部分组成：

```
源句（中文）→ [Encoder] → enc_output
                                ↓ K, V
目标句（英文）→ [Decoder] ←────────── Cross-Attention
                   ↓
              输出投影 → 预测下一个词
```

### Encoder：理解源语言

输入"我喜欢你"，Encoder 做的事：

```
"我喜欢你" → Token ID → Embedding + 位置编码
                               ↓
                    × N 层 Self-Attention + FFN
                               ↓
                enc_output（每个词的上下文向量）
```

Self-Attention 是**双向**的，"喜欢"可以同时看到"我"和"你"，每个词的输出向量都融合了整句话的语义。

### Decoder：生成目标语言

Decoder 每次生成一个词，循环直到结束符 `<eos>`：

```
第1轮：输入 [<bos>]               → 预测 "I"
第2轮：输入 [<bos>, I]            → 预测 "like"
第3轮：输入 [<bos>, I, like]      → 预测 "you"
第4轮：输入 [<bos>, I, like, you] → 预测 <eos> → 停止
```

每个 Decoder 层内部有三个子层：

| 子层 | Q | K / V | 作用 |
|------|---|-------|------|
| Masked Self-Attn | 已生成的词 | 已生成的词（causal mask 挡住未来） | 理解"自己已经说了什么" |
| Cross-Attention | 当前目标词 | Encoder 输出（enc_output） | 从源句里提取对应信息 |
| FFN | — | — | 非线性特征变换 |

**Cross-Attention 就是中英对齐发生的地方**：
- 生成 "I" 时，"I" 的 Q 会和 enc_output 里"我"的 K 点积最高 → 注意力集中在"我"
- 生成 "like" 时，"like" 的 Q 对齐"喜欢"
- 这个对齐关系不是人工写的词典，而是**训练时通过反向传播自动学出来的**

### 第一个词是怎么生成的？

第一轮 Decoder 只有 `<bos>`（开始符），没有任何目标词。

```
Q = <bos> 的向量
Cross-Attention: Q 看整个 enc_output（"我喜欢你"）
输出投影打分 → "I" 得分最高 → 预测 "I"
```

`<bos>` 通过训练，学会了"看到以'我'开头的中文句时，预测 'I'"的模式。

### 训练 vs 推理

| | 训练（Teacher Forcing） | 推理（自回归） |
|--|--|--|
| Decoder 输入 | 喂真实的目标序列（不管上一步预测对不对） | 把上一步预测的词喂给下一步 |
| 速度 | 快（所有位置并行计算） | 慢（必须一步一步串行） |

---

## 二、GPT 生成原理（Decoder Only）

### 和翻译模型的本质区别

GPT **没有 Encoder，也没有 Cross-Attention**，只有一种注意力：

```
翻译模型：  问题 → [Encoder] → K/V  +  [Decoder Cross-Attn] Q=目标词
GPT：       问题和答案拼在一起，统一用 Self-Attention 处理，Q=K=V=同一序列
```

### 处理"市盈率是什么"的完整过程

**第1轮：问题整体输入，预测第一个答案词**

```
输入序列：[市, 盈, 率, 是, 什, 么]
               ↓
Self-Attention（Q=K=V=上面6个词，causal mask 下三角）：
  "市"的Q 只看 "市"的K/V
  "盈"的Q 只看 "市,盈"的K/V
  ……
  "么"的Q 能看全部6个词的K/V  ← 融合了整个问题的语义
               ↓
取"么"位置的输出向量
               ↓
输出投影（打分矩阵）× 词表 → 每个词一个分数
               ↓
argmax → "市"（答案第1个字）

同时：[市,盈,率,是,什,么] 的 K、V 全部缓存（KV Cache）
```

**第2轮：用新生成的词作为 Q，查询历史 K/V**

```
新词 "市" → Embedding → 向量
                  ↓ × W_Q
                  Q₆（只有1个Q）

K_cache = [K市, K盈, K率, K是, K什, K么] + K₆（新算的）
V_cache = [V市, V盈, V率, V是, V什, V么] + V₆（新算的）

Attention: Q₆ · K_cache ᵀ → 注意力权重 → 加权取 V_cache
                  ↓
输出投影 → 预测 "盈"
```

**第3、4、5……轮：重复上面的过程**

```
每轮：
  Q  = 上一轮新生成的词的向量（只有1个）
  K/V cache = 问题 + 所有已生成词的缓存（不断追加）
  → 预测下一个词
  → 直到生成 <eos> 停止
```

### 打分表（输出投影）是什么？

每轮预测的最后一步都是这个矩阵乘法：

```
当前输出向量 [d_model 维]
      × 输出投影矩阵 W [d_model × vocab_size]
      = 打分向量 [vocab_size 维]  ← 词表里每个词一个分数
                ↓
           argmax → 得分最高的词
```

W 是训练时通过反向传播学出来的——预测错了 Loss 大，梯度调整 W，让正确词的得分越来越高。W 和 Token Embedding 共享同一个矩阵（参数减半，效果更好）。

### KV Cache 的价值

没有 KV Cache：每轮都重新算所有词的 K/V，大量重复计算。

有 KV Cache：问题部分的 K/V 只算一次，之后每轮只算新词的 K/V。

```
K/V cache 随时间增长：
  ████████████████  问题（第1轮算好，固定不变）
  ░░░░░░░          已生成的答案（每轮追加1个）

Q 每轮只有1个：上一轮刚生成的词的向量
```

---

## 三、两种架构对比

| | 翻译模型（Enc-Dec） | GPT（Decoder Only） |
|--|--|--|
| 问题/源句处理 | 单独经过 Encoder | 直接作为输入序列的一部分 |
| Cross-Attention | ✅ 有，Q=目标词，K/V=Encoder输出 | ❌ 没有 |
| Self-Attention | Q=K=V=目标序列（causal） | Q=K=V=问题+答案拼接序列 |
| 问题和答案的关系 | 物理隔离，两个独立模块 | 拼在一起，统一处理 |
| 代表模型 | 原始 Transformer、T5、MarianMT | GPT 系列、LLaMA、DeepSeek、Qwen |
| 适合任务 | 翻译、摘要（输入输出语言/格式差异大） | 对话、续写、代码生成（开放式生成） |

