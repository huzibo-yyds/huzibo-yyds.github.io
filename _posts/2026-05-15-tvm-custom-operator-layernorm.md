---
title: "从模型导入到自定义算子：TVM 自定义 LayerNorm 全流程"
date: 2026-05-15
permalink: /posts/2026/05/tvm-custom-operator-layernorm/
tags:
  - TVM
  - Relay
  - Custom Operator
  - LayerNorm
  - Compiler
toc_label: "目录"
toc_items:
  - title: "背景：TVM 编译模型时发生了什么"
    id: "tvm-compile-flow"
  - title: "从 ResNet18 到 ViT-Tiny"
    id: "resnet18-to-vit-tiny"
  - title: "目标：把 LayerNorm 替换成自定义 Relay op"
    id: "replace-layernorm"
  - title: "第一步：注册 C++ Relay 算子"
    id: "register-cpp-relay-op"
  - title: "第二步：接上 Python API 和 schedule"
    id: "python-api-and-schedule"
  - title: "第三步：写 Relay pass 替换 LayerNorm"
    id: "relay-pass"
  - title: "第四步：重新编译 TVM"
    id: "rebuild-tvm"
  - title: "第五步：运行 baseline 和 custom 对比"
    id: "runtime-compare"
  - title: "几个踩坑点"
    id: "pitfalls"
  - title: "当前限制"
    id: "limitations"
  - title: "总结"
    id: "summary"
---

最近在 TVM 0.19.0 上做了一个小实验：给 Relay 增加一个真正参与编译链路的自定义算子 `custom.layer_norm`，再写一个 pass，把 ViT-Tiny ONNX 模型里的 LayerNorm 子图替换成这个自定义算子。最后分别跑 baseline 和 custom 两条路径，确认输出一致。

这篇文章把这个过程整理成一条完整路线：先从 TVM 如何加载模型说起，再进入 Relay 自定义算子的注册、pass 替换、编译、运行和结果验证。

## 背景：TVM 编译模型时发生了什么 {#tvm-compile-flow}

以 Relay 为例，TVM 执行一个外部模型通常是这几步：

```text
外部模型
  -> Relay frontend
  -> Relay IRModule
  -> Relay passes
  -> relay.build
  -> TE compute / schedule
  -> TIR
  -> LLVM / CUDA / Metal 等 codegen
  -> runtime 执行
```

一个最小化的 ONNX + Relay + Graph Executor 流程大概是：

```python
import onnx
import tvm
from tvm import relay
from tvm.contrib import graph_executor

target = "llvm"
dev = tvm.cpu(0)

onnx_model = onnx.load("model.onnx")

mod, params = relay.frontend.from_onnx(
    onnx_model,
    shape={"x": (1, 3, 224, 224)},
    freeze_params=True,
)

with tvm.transform.PassContext(opt_level=3):
    lib = relay.build(mod, target=target, params=params)

module = graph_executor.GraphModule(lib["default"](dev))
module.set_input("x", input_data)
module.run()
output = module.get_output(0).numpy()
```

这里最关键的一点是：`relay.build()` 不是简单“保存模型”，而是把 Relay IR 编译成可运行产物。它会继续走算子实现选择、TE compute、schedule、TIR lowering、目标代码生成和 runtime 封装。

因此，如果只是写一个 pass 改了 Relay 图，但新节点没有 compute / schedule / lowering 路径，`relay.build()` 仍然会失败。

## 从 ResNet18 到 ViT-Tiny {#resnet18-to-vit-tiny}

前期我用 ResNet18 跑通过一个基本流程：ONNX 模型导入 TVM，经过编译后在 runtime 上执行，并对图片输出 top-k 分类结果。这个实验的价值不在 ResNet18 本身，而是先确认几件事：

- 模型前端能正常把 ONNX 转成 TVM IR。
- `target="llvm"` 的本机 CPU 编译链路可用。
- runtime 能设置输入、执行并取回输出。
- 预处理、softmax、top-k 等模型周边流程是通的。

在 TVM 0.19.0 上，Relay 仍然是一个很适合观察自定义算子链路的入口。于是下一步就从“能加载并运行模型”，推进到“能改写模型图，并让新算子真的编译执行”。

## 目标：把 LayerNorm 替换成自定义 Relay op {#replace-layernorm}

这次实验的目标是新增一个 Relay op：

```text
custom.layer_norm(data, gamma, beta, axis=-1, epsilon=1e-5, center=True, scale=True)
```

然后写一个 pass，把 ViT-Tiny ONNX 转 Relay 后的 LayerNorm 语义替换成它。

最终有两条路径：

```text
baseline:
  ONNX -> Relay -> relay.build -> graph executor

custom:
  ONNX -> Relay -> ReplaceLayerNormWithCustom
       -> Relay with custom.layer_norm
       -> relay.build -> graph executor
```

两条路径使用同一张测试图片、同一个 target 和同一种 runtime，最后比较 logits 和 top-k，确认改写没有破坏语义。

## 第一步：注册 C++ Relay 算子 {#register-cpp-relay-op}

真正新增 primitive op，不能只在 Python 里写一个函数名。TVM 需要知道这个算子：

- 名字是什么。
- 属性有哪些。
- 输入输出类型关系是什么。
- 如何生成 TE compute。
- 使用什么 schedule。

本实验新增了：

```text
src/relay/op/custom/layer_norm.cc
```

它注册 `custom.layer_norm`，复用 TVM 里的 `LayerNormAttrs`，并通过 type relation 约束输出 shape / dtype 与 `data` 一致，`gamma`、`beta` 是归一化维度对应的一维张量。

底层 compute 没有重新手写 LayerNorm 数学公式，而是复用已有 TOPI：

```text
topi::nn::layer_norm
```

这样做的重点不是创造一个高性能 kernel，而是打通“自定义 Relay op 能进入 TVM 默认编译链路”的完整路径。

这里还有一个容易忽略的点：只注册 Relay 节点还不够。如果没有 compute / schedule，TVM 在 lowering 时还是不知道怎么把它变成可执行循环。也就是说：

```text
Relay op 名字
  != 可以被 relay.build 编译执行的算子
```

## 第二步：接上 Python API 和 schedule {#python-api-and-schedule}

C++ 注册后，还需要把它暴露到 Python 侧，方便 pass 构造 Relay Call。本实验新增了：

```text
python/tvm/relay/op/custom.py
python/tvm/relay/op/_custom.py
python/tvm/relay/op/_custom_make.py
```

三者职责可以这样理解：

```text
custom.py
  面向用户和 pass，提供 relay.op.custom.layer_norm(...)

_custom_make.py
  初始化 FFI，把 C++ MakeCustomLayerNorm 接到 Python

_custom.py
  注册 backend schedule，让 relay.build 可以继续 lower
```

调用关系大概是：

```text
relay.op.custom.layer_norm(...)
  -> _custom_make.layer_norm(...)
  -> C++ MakeCustomLayerNorm(...)
  -> Relay Call(op="custom.layer_norm")
```

同时在 `python/tvm/relay/op/__init__.py` 中导入 `custom` 和 `_custom`，保证 `import tvm.relay.op` 时 API 和 schedule 注册都会生效。

## 第三步：写 Relay pass 替换 LayerNorm {#relay-pass}

新增 pass 位于：

```text
toys/vit_tiny/custom_layer_norm_pass.py
```

核心结构是：

```python
@relay.transform.function_pass(opt_level=1)
class ReplaceLayerNormWithCustom:
    ...
```

它内部用 `ExprMutator` 遍历 Relay 表达式，支持两类替换。

第一类是直接命中的 Relay op：

```text
nn.layer_norm(data, gamma, beta)
  -> custom.layer_norm(data, gamma, beta)
```

第二类是 ONNX frontend 实际生成的分解子图。ViT-Tiny 里的 LayerNorm 不一定会直接变成 `nn.layer_norm`，更常见的是一串基础算子组合：

```text
mean(x)
subtract(x, mean)
power(subtract, 2)
mean(power)
add(var, epsilon)
sqrt(...)
divide(subtract, sqrt)
multiply(..., gamma)
add(..., beta)
```

pass 识别这个结构后，把它重组为：

```text
custom.layer_norm(x, gamma, beta, axis=-1, epsilon=...)
```

为什么这里选 `ExprMutator`，而不是 `DFPatternCallback`？核心原因是 LayerNorm 的等价写法比较多。比如 `add` 和 `multiply` 可能左右交换，归一化形式也可能写成 `divide` 或乘以倒数。如果用 dataflow pattern，需要写很多变体。

`ExprMutator` 更像手写递归遍历器，可以把识别逻辑拆成多个小函数：判断 mean、判断 variance、判断 gamma/beta、判断 epsilon、判断是不是最后一维归一化。代码更长，但调试时更容易知道是哪条语义约束没有满足。

## 第四步：重新编译 TVM {#rebuild-tvm}

因为这次新增的是 C++ 层 op 注册，所以必须重新编译 TVM。否则 Python 侧看不到新的 FFI constructor，`relay.build` 也找不到对应 compute / schedule。

本机使用的环境是：

```text
/Users/huzi/miniconda3/envs/tvm-0.19
```

编译命令：

```bash
cd /Users/huzi/Documents/Code/tvm/apache-tvm-0.19.0/build
/Users/huzi/miniconda3/envs/tvm-0.19/bin/cmake .. \
  -G Ninja \
  -DCMAKE_MAKE_PROGRAM=/Users/huzi/miniconda3/envs/tvm-0.19/bin/ninja
/Users/huzi/miniconda3/envs/tvm-0.19/bin/ninja -j8
```

如果只是修改 Python pass，一般不需要重新编译 C++，重启 Python 进程即可。但如果新增或修改了 `src/relay/op/custom/layer_norm.cc`，就需要重新链接 `libtvm.dylib`。

## 第五步：运行 baseline 和 custom 对比 {#runtime-compare}

完整对比脚本做了这些事：

1. 加载 ViT-Tiny ONNX、preprocessor config、ImageNet labels 和测试图片。
2. 用 `relay.frontend.from_onnx` 转成 Relay。
3. baseline 路径直接 `relay.build`。
4. custom 路径先运行 `ReplaceLayerNormWithCustom`，再 `relay.build`。
5. 两边都用 `graph_executor.GraphModule` 执行。
6. 对比 top-k、`max_abs_diff`、`max_rel_diff` 和 `np.allclose`。

运行：

```bash
cd /Users/huzi/Documents/Code/tvm/apache-tvm-0.19.0
/Users/huzi/miniconda3/envs/tvm-0.19/bin/python toys/vit_tiny/vit_tiny_layernorm_compare.py
```

实验结果中，pass 替换了 25 个 LayerNorm：

```text
custom.layer_norm replacements: 25
custom.layer_norm calls in custom IR: 25
```

baseline 和 custom 的 top-5 一致：

```text
1.  282 tiger cat
2.  281 tabby
3.  285 Egyptian cat
4.  287 lynx
5.  761 remote control
```

数值误差：

```text
max_abs_diff: 1.0967255e-05
max_rel_diff: 0.0014497685
allclose(rtol=0.0001, atol=0.0001): True
```

这说明 `custom.layer_norm` 不是只停留在 Relay 图里的“假节点”，而是真的走完了：

```text
Relay pass
  -> relay.build
  -> FTVMCompute
  -> TE compute
  -> schedule
  -> TIR
  -> LLVM codegen
  -> graph executor runtime
```

## 几个踩坑点 {#pitfalls}

**1. 自定义 pass 不等于自定义算子**

pass 只负责改 IR。如果替换出来的新 op 没有完整注册，build 阶段还是会失败。

**2. Python wrapper 不等于 FFI constructor**

`custom.py` 只是易用 API。真正构造 Relay Call 的底层入口来自 C++ 注册的 global function，需要 `_custom_make.py` 初始化。

**3. 有 compute 还需要 schedule**

TE compute 描述“怎么算”，schedule 描述“怎么执行”。Relay build 两者都需要。

**4. ONNX 导入后的图未必保留高级语义**

模型里叫 LayerNorm，导入 Relay 后可能只是 `mean/subtract/power/sqrt/divide/multiply/add`。写 pass 时要面向真实 IR，而不是面向想象中的模型结构。

**5. 先做 baseline，再做 custom**

baseline 是判断自定义路径是否正确的锚点。没有 baseline，后面即使 custom 能跑，也很难判断它是不是语义一致。

## 当前限制 {#limitations}

这次实现主要目标是打通全流程，不是优化 LayerNorm 性能。当前限制包括：

- 主要验证 `target="llvm"` CPU。
- 只覆盖 ViT-Tiny 常见的 float32、最后一维归一化、带 gamma/beta 的 LayerNorm。
- 底层复用 TOPI `layer_norm` compute，没有手写新的高性能 schedule。
- pattern matcher 针对当前 ViT-Tiny ONNX frontend 输出编写，换模型或换导出方式可能需要扩展。

## 总结 {#summary}

TVM 自定义算子真正完整的链路，不只是“注册一个名字”或者“写一个 pass”。它至少包含：

```text
模型导入
  -> Relay IR
  -> 自定义 op 注册
  -> Python API / FFI
  -> compute / schedule
  -> Relay pass 替换
  -> relay.build
  -> TIR / codegen
  -> runtime 验证
```

这次 `custom.layer_norm` 实验的收获是：把一个真实模型中的 LayerNorm 子图替换成自定义 Relay op，并且能编译、运行、和 baseline 对齐。对后续继续做算子融合、后端接入或硬件相关 lowering 来说，这是一个比较扎实的起点。
