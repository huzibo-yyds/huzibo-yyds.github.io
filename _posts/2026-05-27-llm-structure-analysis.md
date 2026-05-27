---
title: "LLM 结构解析"
date: 2026-05-27
permalink: /posts/2026/05/llm-structure-analysis/
tags:
  - LLM
  - Transformer
  - Attention
  - Decoder-only
  - AI
excerpt: "从 next-token prediction 出发，由浅入深理解 Decoder-only LLM 的整体结构、Transformer Block、Self-Attention、FFN、KV Cache 与常见组件。"
toc_label: "目录"
toc_items:
  - title: "LLM 最外层在做什么"
    id: "what-llm-does"
  - title: "从文本到向量"
    id: "text-to-vector"
  - title: "文本先变成 token ids"
    id: "token-ids"
    level: 3
  - title: "Embedding：token id 变成向量"
    id: "embedding"
    level: 3
  - title: "多层 Transformer Block"
    id: "transformer-blocks"
  - title: "一个 Transformer Block 的内部"
    id: "inside-transformer-block"
    level: 3
  - title: "attention 模块"
    id: "attention-module"
    level: 3
  - title: "FFN 模块"
    id: "ffn-module"
    level: 3
  - title: "涉及算子"
    id: "operators"
    level: 3
  - title: "Self-Attention：token 之间互相看"
    id: "self-attention"
  - title: "Multi-Head：拆成多个注意力头"
    id: "multi-head"
    level: 3
  - title: "Attention 分数、Mask 和 Softmax"
    id: "attention-score-mask-softmax"
    level: 3
  - title: "用注意力权重汇总 Value"
    id: "weighted-value"
    level: 3
  - title: "FFN / MLP：加工每个 token 自己"
    id: "ffn-mlp"
  - title: "Norm 和 Residual 的作用"
    id: "norm-residual"
  - title: "Final Norm、lm_head 和 logits"
    id: "final-norm-lm-head-logits"
  - title: "采样下一个 token"
    id: "sample-next-token"
  - title: "推理时的 KV Cache"
    id: "kv-cache"
  - title: "整体伪代码"
    id: "pseudo-code"
  - title: "进一步"
    id: "further-reading"
  - title: "参考资源"
    id: "references"
    level: 3
  - title: "论文和现代组件"
    id: "papers-modern-components"
    level: 3
---

🤖 不纠结每个算子的实现细节，而是从“一个 LLM 如何从文本生成下一个 token”开始，由浅入深看完整结构。LLaMA、Mistral、Qwen、Gemma、GPT 类模型都可以先按这个框架理解。

> 以 LLaMA 类 Decoder-only LLM 为例

## 1. LLM 最外层在做什么 {#what-llm-does}
LLM 本质上是一个 next-token predictor：
```text
给定前面的 token，预测下一个 token。
```

例如：
```text
输入：今天天气很
输出候选：
  好: 8.2
  冷: 6.1
  热: 5.7
  ...
```
这些分数叫 `logits`。模型会通过 greedy、top-k、top-p 等策略选出下一个 token，比如选出“好”，再把它拼回输入，继续预测后面的 token。

## 2. 从文本到向量 {#text-to-vector}

### 2.1 文本先变成 token ids {#token-ids}
模型不能直接处理字符串，要先经过 **tokenizer**：
```text
"Hello world" -> [15496, 995]
```
这些整数是 token 在词表里的编号。注意：编号大小本身没有语义，`15496` 并不表示它比 `995` “更大”或“更重要”，它只是词表中的位置。

### 2.2 Embedding：token id 变成向量 {#embedding}
📌 将token从离散符号 -> 连续向量

模型有一张**可训练的 embedding table**：
```text
embedding_table.shape = [vocab_size, hidden_size]
```
例如：
```text
vocab_size = 100000
hidden_size = 4096
```

`Embedding` 做的事就是按 token id 查表：
```python
hidden = embedding_table[token_ids]
```

如果：
```text
token_ids.shape = [seq_len]
```
那么：
```text
hidden.shape = [seq_len, hidden_size]
```

此时，每个 token 都从一个离散编号变成了一个连续向量。对应的算子语义是：
```text
Embedding / Gather
```

## 3. 多层 Transformer Block {#transformer-blocks}
Embedding 之后，hidden 会进入很多层 Transformer block：
```python
hidden = embedding(token_ids)

for layer in layers:
    hidden = transformer_block(hidden)
```

一个典型规模可能是：
`hidden_size = num_heads * head_dim`
```text
num_layers = 32      //32 层 Transformer Block
hidden_size = 4096   //embedding后hidden向量维度
num_heads = 32       //注意力头，一个 token 不只用一种方式看上下文，而是分成 32 组视角去看
head_dim = 128       // attention head 的向量维度（hidden_size = num_heads * head_dim
                     // 💬含义：原本每个 token 是 4096 维，做 attention 时会拆成 32 个头，每个头 128 维
```

每一层都在加工所有 token 的表示。
- 浅层可能更偏局部、词形、短语模式；
- 深层会逐渐形成更抽象的语义和任务相关表示。

### 3.1 一个 Transformer Block 的内部 {#inside-transformer-block}
现代 LLaMA 类模型通常使用 Pre-Norm 结构：
```text
输入 hidden
  -> RMSNorm
  -> Self-Attention
  -> Residual Add
  -> RMSNorm
  -> FFN / MLP
  -> Residual Add
输出 hidden
```

伪代码：
```python
hidden = hidden + self_attention(rms_norm(hidden))  # attention
hidden = hidden + ffn(rms_norm(hidden))             # FFN
```

这里有两个核心子模块：
- `Self-Attention`：让 token 之间交换信息。
- `FFN / MLP`：对每个 token 自己的 hidden 向量做非线性加工。

还有两个稳定训练的设计：
- `RMSNorm / LayerNorm`：稳定数值尺度。
- `Residual Add`：保留原始信息，并帮助梯度传播。

### 3.2 attention模块 {#attention-module}
> 负责 token 和 token 之间的信息交互
```
hidden
  -> RMSNorm
  -> Self-Attention
  -> Residual Add
```
对应代码：
```python
x = rms_norm(hidden)
attn_out = self_attention(x)
hidden = hidden + attn_out
```

📌 逐步解析
- **RMSNorm**
    - 作用是先把 hidden 的数值尺度稳定一下
        - 原因：因为一层层传下去，hidden 的数值可能变得过大、过小，或者分布漂移。RMSNorm 会让输入进入 attention 前更稳定。
        - 本质是，先将hidden调整到一个稳定的数值范围，再交给attention处理
- **Self-Attention**
    - 关键计算；输入输出shape一致 `[batch, seq_len, hidden_size]`
- **Residual Add**
    - 残差链接，不是直接用 attention 的输出替换 hidden
    - 视觉上，原来的信息保留，attention学到的信息作为增量补充上去

### 3.3 FNN模块 {#ffn-module}
> 对每个 token 内部表示做加工、变换、提炼
```
hidden
  -> RMSNorm
  -> FFN / MLP
  -> Residual Add
```
对应代码
```python
x = rms_norm(hidden)
ffn_out = ffn(x)
hidden = hidden + ffn_out
```
📌 逐步解析
- RMSNorm
    - Attention 之后 hidden 已经发生变化，所以进入 FFN 前再 norm 一次
- **FFN / MLP**
    - FFN 是 Feed-Forward Network，前馈网络，也常叫 MLP
    - 对每个 token 自己的向量做非线性加工
- Residual Add

经典 FFN：
```
h = gelu(x @ W1)
ffn_out = h @ W2
```
LLaMA 类模型常用 SwiGLU：
```
gate = x @ W_gate
up = x @ W_up
h = silu(gate) * up
ffn_out = h @ W_down
```
FFN 的输入输出 shape 也保持一致：
- 中间可能扩展到更大维度
```
输入:  [batch, seq_len, hidden_size]
输出:  [batch, seq_len, hidden_size]
```

💡 FNN承载了LLM大部分参数 （2/3）
### 3.4 涉及算子 {#operators}
```
RMSNorm:
Mean / Multiply / Rsqrt / Multiply

Self-Attention:
Linear / MatMul / Reshape / Transpose / Softmax / BatchMatMul

Residual Add:
Add

FFN:
Linear / SiLU or GELU / Multiply / Linear

Shape 变换:
Reshape / Transpose / Concatenate
```

---

## 4. Self-Attention：token 之间互相看 {#self-attention}
Self-Attention 解决的问题是：
```text
当前 token 应该关注前文中的哪些 token？
```

比如：
```text
小明把书放进书包，因为他要上学
```

模型需要知道“他”可能指“小明”。Attention 就是用来建模 token 之间关系的。

Attention 的第一步是从 hidden 生成 Q/K/V：
```python
Q = hidden @ Wq
K = hidden @ Wk
V = hidden @ Wv
```

可以这样直观理解：
```text
Q: Query，当前 token 想找什么信息
K: Key，其他 token 提供什么索引
V: Value，其他 token 真正携带的内容
```

对应算子：
```text
Linear / MatMul / GEMM
```

### 4.1 Multi-Head：拆成多个注意力头 {#multi-head}

假设：
```text
hidden_size = 4096
num_heads = 32
head_dim = 128
```

因为：
```text
4096 = 32 * 128
```

所以可以把 Q/K/V 从：
```text
[seq_len, hidden_size]
```

变成：
```text
[num_heads, seq_len, head_dim]
```
每个 head 都可以学习一种不同的关注模式。有的 head 可能关注局部词序，有的关注实体指代，有的关注更长距离依赖。

对应算子：
```text
Reshape / Transpose
```

### 4.2 Attention 分数、Mask 和 Softmax {#attention-score-mask-softmax}
每个 token 的 Q 会和所有 token 的 K 做点积：
```python
scores = Q @ K.T / sqrt(head_dim)
```
`scores[i, j]` 可以理解为：
```text
第 i 个 token 对第 j 个 token 的关注程度。
```

Decoder-only LLM 不能偷看未来 token，所以要加 causal mask：
```text
允许看到：
token 4 -> token 1,2,3,4

不允许看到：
token 4 -> token 5,6,...
```

mask 矩阵大概是：
```text
1 0 0 0
1 1 0 0
1 1 1 0
1 1 1 1
```

被屏蔽的位置会被设成一个很大的负数：
```python
scores = np.where(mask, scores, -1e30)
```
然后用 softmax 把分数变成权重：
```python
attn_prob = softmax(scores)
```

对应算子：
```text
BatchMatMul / Scale / Where / Softmax
```

### 4.3 用注意力权重汇总 Value {#weighted-value}
拿到 attention probability 后，用它对 V 做加权求和：（attn_prob是当前token 的Q 与其他算子的K点积的结果
```python
context = attn_prob @ V
```

含义是：
```text
当前 token 根据注意力权重，从其他 token 的 Value 中汇总信息。
```

多个 head 的结果会再拼回 hidden 维，然后经过一个输出线性层：
```python
context = concat_heads(context)
attn_out = context @ Wo
```

对应算子：
```text
BatchMatMul / Transpose / Reshape / Linear
```

---

## 5. FFN / MLP：加工每个 token 自己 {#ffn-mlp}
Attention 负责 token 之间的信息交互；FFN 负责对每个 token 自己的 hidden 向量做非线性变换。
经典 FFN：
```python
h = gelu(hidden @ W1)
out = h @ W2
```

LLaMA 类模型更常见的是 SwiGLU：
```python
gate = hidden @ W_gate
up = hidden @ W_up
h = silu(gate) * up
out = h @ W_down
```

结构可以画成：

```text
hidden
  -> gate_proj -> SiLU ----\
                            * -> down_proj -> output
  -> up_proj --------------/
```

这里 `gate` 分支决定哪些信息通过，`up` 分支提供要被加工的内容。

对应算子：

```text
Linear / SiLU / Multiply / Linear
```

## 6. Norm 和 Residual 的作用 {#norm-residual}
Transformer block 中反复出现：
```python
hidden = hidden + module(norm(hidden))
```
`Norm` 的作用是稳定 hidden 的数值尺度，避免中间激活分布一层层漂移。LLaMA 类模型常用 `RMSNorm`：
```python
x_norm = x / sqrt(mean(x * x) + eps) * weight
```

`Residual Add` 的作用是把原始 hidden 直接加回去：
```python
hidden = hidden + module_output
```

它有两个好处：
- 保留原始信息，模型可以学习“增量修改”。
- 让梯度更容易传回前面的层。

对应算子：
```text
RMSNorm / LayerNorm / Add
```

## 7. Final Norm、lm_head 和 logits {#final-norm-lm-head-logits}
⭐️ 把hidden 向量变成“每个词的分数”

所有 Transformer block 结束后，通常还有一个 final norm：
```python
hidden = final_norm(hidden)
```

然后用 `lm_head` 把 hidden 映射到词表大小：
- 线性层 lm_head，目的将hidden转到词表大小（language modeling head｜语言模型输出头
```python
logits = hidden @ lm_head.T
```

如果：
```text
hidden.shape = [seq_len, hidden_size]
lm_head.shape = [vocab_size, hidden_size]
```
那么：
- `logits` 是模型给每个候选 token 的 **原始分数**
```text
logits.shape = [seq_len, vocab_size]
```

生成时通常只看最后一个 token 的 logits：
```python
next_logits = logits[-1]
```

对应算子：
```text
RMSNorm / LayerNorm / MatMul / Linear
```

## 8. 采样下一个 token {#sample-next-token}
🚀 如何生成新的token

`logits` 是每个 token 的未归一化分数。可以先做 softmax：
```python
prob = softmax(next_logits)
```

然后选择下一个 token：
```text
greedy: 选择分数最高的 token
top-k: 只从最高 k 个 token 中采样
top-p: 从累计概率达到 p 的候选集合中采样
```

对应算子：
```text
Softmax / Argmax / TopK / Sort / Cumsum / Multinomial
```

## 9. 推理时的 KV Cache {#kv-cache}
如果每生成一个 token 都重新计算所有历史 token 的 K/V，会非常慢。⭐️ KV Cache 的思想是：
```text
第 1 步：算 K1, V1，存起来
第 2 步：只算新 token 的 K2, V2，拼到缓存后面
第 3 步：只算新 token 的 K3, V3，再拼到缓存后面
```

这样 decode 阶段每一步只需要处理新 token 的 Q，同时复用历史 K/V。

对应算子：
```text
Concat / Slice / Gather / Attention
```

KV Cache 是 LLM 推理加速的关键之一。

- KV Cache 不是整个模型只有一份。每一层 Transformer block 的 K/V 都不同，因为每一层的 hidden 不同
    - 因此会占显存
- KV Cache 为什么能加速
    - 不用Cache：每生成一个 token，都重新处理整个历史序列
    - 使用Cache：每生成一个 token，只处理新 token
    - 在decode阶段，能够显著减少计算量
- 代价，显存占用
    - 粗略的大小估计
          `num_layers * 2(K/V) * batch * num_heads * seq_len * head_dim * dtype_size`
## 10. 整体伪代码 {#pseudo-code}
把上面的结构串起来，一个 LLaMA 类模型可以简化成：
```python
hidden = embedding(token_ids)

for layer in layers:
    x = rms_norm(hidden)
    attn_out = self_attention(x)
    hidden = hidden + attn_out

    x = rms_norm(hidden)
    ffn_out = swiglu_ffn(x)
    hidden = hidden + ffn_out

hidden = final_rms_norm(hidden)
logits = hidden @ lm_head.T
next_token = sample(logits[-1])
```

一句话总结：
```text
LLM = Embedding + 多层 Transformer Block + final norm + lm_head
Transformer Block = Norm + Attention + Residual + Norm + FFN + Residual
Attention 负责 token 之间通信，FFN 负责单个 token 内部加工。
```


## 进一步 {#further-reading}

### 参考资源 {#references}
| 顺序  | 资源                                                                                                                                          | 适合看什么                                                                                                                                                                                                                                                                                            |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | [![](https://poloclub.github.io/favicon.ico)Transformer Explainer](https://poloclub.github.io/transformer-explainer/)                       | 最适合入门。它用 GPT-2 小模型可视化 Embedding、Q/K/V、Attention、MLP、logits、temperature、top-k/top-p，和我们 MD 里的结构非常对应。页面明确把 Transformer 拆成 Embedding、Transformer Block、Output Probabilities 三块。([![](https://poloclub.github.io/favicon.ico)poloclub.github.io](https://poloclub.github.io/transformer-explainer/)) |
| 2   | [The Illustrated Transformer](https://jalammar.github.io/illustrated-transformer/)                                                          | 图解经典 Transformer，适合理解 Q/K/V、self-attention、encoder/decoder。对“为什么 attention 能让 token 互相看”讲得很直观。([jalammar.github.io](https://jalammar.github.io/illustrated-transformer/?source=post_page-----83066192d9ab--------------------------------&utm_source=openai))                                    |
| 3   | [The Illustrated GPT-2](https://jalammar.github.io/illustrated-gpt2/)                                                                       | 更贴近 Decoder-only LLM。讲 GPT-2 如何做 next-token prediction、masked self-attention、KV 缓存思想、embedding 和输出 logits。([jalammar.github.io](https://jalammar.github.io/illustrated-gpt2/?utm_source=openai))                                                                                                 |
| 4   | [LLM Visualization by Brendan Bycroft](https://bbycroft.net/llm)                                                                            | 3D 可视化 GPT 风格 LLM 的数据流。适合建立“token 从 embedding 一路流过 block、attention、MLP、logits”的整体空间感。([bbycroft.net](https://bbycroft.net/llm))                                                                                                                                                                  |
| 5   | [![](https://huggingface.co/favicon.ico)Hugging Face LLM Course: Transformer Architectures](https://huggingface.co/course/chapter1/6?fw=pt) | 适合理清 encoder-only、decoder-only、encoder-decoder 的区别，以及为什么 LLM 多数是 decoder-only。([![](https://huggingface.co/favicon.ico)huggingface.co](https://huggingface.co/course/chapter1/6?fw=pt&utm_source=openai))                                                                                        |
| 6   | [The Annotated Transformer](https://nlp.seas.harvard.edu/annotated-transformer)                                                             | 代码级理解经典 Transformer。它把论文结构用 PyTorch 风格代码逐步实现，适合把“结构图”落到代码。([nlp.seas.harvard.edu](https://nlp.seas.harvard.edu/annotated-transformer?utm_source=openai))                                                                                                                                         |
| 7   | [![](https://sebastianraschka.com/favicon.ico)Build a Large Language Model From Scratch](https://sebastianraschka.com/llms-from-scratch)    | 系统性从 tokenizer、embedding、attention、GPT model、pretraining、finetuning 一路做出来。非常适合你现在这种“由浅入深”的学习路线。([![](https://sebastianraschka.com/favicon.ico)sebastianraschka.com](https://sebastianraschka.com/llms-from-scratch?utm_source=openai))                                                           |
| 8   | [Karpathy nanoGPT](https://github.com/karpathy/nanogpt) / [build-nanogpt](https://github.com/karpathy/build-nanogpt)                        | 代码很短，适合理解 GPT 最小实现。看完前面的结构后，再看这个会很爽：Embedding、Block、Attention、MLP、lm_head 都能对应上。([github.com](https://github.com/karpathy/nanogpt?utm_source=openai)) ([github.com](https://github.com/karpathy/build-nanogpt?utm_source=openai))                                                                |
| 9   | [动手学深度学习：注意力机制与 Transformer](https://zh-v2.d2l.ai/d2l-zh.pdf)                                                                               | 中文体系化教材。适合补基础：注意力机制、多头注意力、Transformer、训练和优化。([zh-v2.d2l.ai](https://zh-v2.d2l.ai/d2l-zh.pdf?utm_source=openai))                                                                                                                                                                                  |


### 论文和现代组件 {#papers-modern-components}

| 资源                                                                                                                         | 建议怎么看                                                                                                                                                                                                                 |
| -------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [![](https://arxiv.org/favicon.ico)Attention Is All You Need](https://arxiv.org/abs/1706.03762)                            | Transformer 原始论文。重点看 scaled dot-product attention、multi-head attention、FFN、residual、layer norm。先不必死磕公式，配合 Illustrated Transformer 看。([![](https://arxiv.org/favicon.ico)arxiv.org](https://arxiv.org/abs/1706.03762)) |
| [![](https://arxiv.org/favicon.ico)LLaMA: Open and Efficient Foundation Language Models](https://arxiv.org/abs/2302.13971) | 现代 LLM 结构参考。重点看 LLaMA 相比原始 Transformer 的组件变化：RMSNorm、SwiGLU、RoPE。([![](https://arxiv.org/favicon.ico)arxiv.org](https://arxiv.org/abs/2302.13971))                                                                    |
| [![](https://arxiv.org/favicon.ico)RoFormer: Rotary Position Embedding](https://arxiv.org/abs/2104.09864)                  | RoPE 来源。适合理解为什么现代 LLM 不一定用绝对位置编码，而是对 Q/K 做旋转位置编码。([![](https://arxiv.org/favicon.ico)arxiv.org](https://arxiv.org/abs/2104.09864?utm_source=openai))                                                                  |
| [![](https://arxiv.org/favicon.ico)Root Mean Square Layer Normalization](https://arxiv.org/abs/1910.07467)                 | RMSNorm 原论文。重点理解：RMSNorm 去掉 LayerNorm 的减均值，只保留尺度归一化，现代 LLaMA 类模型很常用。([![](https://arxiv.org/favicon.ico)arxiv.org](https://arxiv.org/abs/1910.07467?utm_source=openai))                                               |
| [![](https://arxiv.org/favicon.ico)GLU Variants Improve Transformer](https://arxiv.org/abs/2002.05202)                     | SwiGLU / GeGLU 的来源之一。适合理解为什么现代 LLM 的 FFN 常用门控结构，而不是简单 ReLU/GELU。([![](https://arxiv.org/favicon.ico)arxiv.org](https://arxiv.org/abs/2002.05202?utm_source=openai))                                                   |
| [![](https://arxiv.org/favicon.ico)FlashAttention](https://arxiv.org/abs/2205.14135)                                       | 更偏实现优化层。语义仍是 attention，但通过 IO-aware 分块避免显式存完整 attention 矩阵。你理解完标准 attention 后再看。([![](https://arxiv.org/favicon.ico)arxiv.org](https://arxiv.org/abs/2205.14135?utm_source=openai))                                   |
