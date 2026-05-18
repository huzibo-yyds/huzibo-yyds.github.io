---
title: "TVM Relay 入门：用 ResNet18 跑通一次完整推理"
date: 2026-05-18
permalink: /posts/2026/05/tvm-relay-resnet18-beginner-guide/
tags:
  - TVM
  - Relay
  - ResNet18
  - ONNX
  - Compiler
toc_label: "目录"
toc_items:
  - title: "为什么从 ResNet18 开始"
    id: "why-resnet18"
  - title: "整体路线"
    id: "overall-flow"
  - title: "运行示例"
    id: "run-example"
  - title: "图片预处理：把图片变成张量"
    id: "preprocess-image"
  - title: "模型导入：ONNX 到 Relay"
    id: "onnx-to-relay"
  - title: "模型编译：relay.build 做了什么"
    id: "relay-build"
  - title: "运行时：Graph Executor"
    id: "graph-executor"
  - title: "Relay 和 Relax 的区别"
    id: "relay-vs-relax"
  - title: "常用 target 和 PassContext"
    id: "target-passcontext"
  - title: "初学者常见问题"
    id: "beginner-faq"
  - title: "最小模板"
    id: "minimal-template"
  - title: "下一步"
    id: "next-step"
---

这篇文章面向刚开始接触 TVM 的同学。目标不是讲完 TVM 的所有概念，而是用一个 ResNet18 图片分类示例，跑通一条最小但完整的 Relay 推理链路：

```text
ONNX 模型
  -> Relay IRModule
  -> relay.build
  -> Graph Executor
  -> 分类结果
```

对应示例脚本位于 TVM 0.19.0 仓库中：

```text
toys/resnet18/resnet18_relay_classify.py
```

如果你后面要继续做自定义算子、Relay pass、模型编译优化或硬件后端接入，这个例子可以作为起点。它足够小，方便观察每一步；同时又是真实模型，不是只有 `add`、`multiply` 的玩具计算。

## 为什么从 ResNet18 开始 {#why-resnet18}

TVM 的概念很多：frontend、IRModule、Relay、TIR、target、schedule、runtime、executor。初学时如果直接上一个复杂模型，很容易陷入“代码能跑，但不知道每一层在做什么”的状态。

ResNet18 适合作为第一个例子，原因有三个：

1. 模型结构经典，卷积、BatchNorm、ReLU、Pooling、Dense 等算子都比较常见。
2. 输入输出直观，输入是一张图片，输出是 ImageNet 类别概率。
3. Relay 对 ONNX ResNet18 的导入和编译路径比较成熟，适合先建立信心。

这篇文章里，我们关注的是 TVM 编译和运行模型的主路径，而不是 ResNet18 网络结构本身。

## 整体路线 {#overall-flow}

脚本从 `main()` 开始，核心流程可以压缩成下面几步：

```python
model_path = fetch_file(...)
labels_path = fetch_file(...)
image_path = fetch_file(...)

labels = load_labels(labels_path)
input_data = preprocess_image(image_path)

onnx_model = onnx.load(model_path)
input_name = first_model_input_name(onnx_model)
shape_dict = {input_name: input_data.shape}

mod, params = relay.frontend.from_onnx(
    onnx_model,
    shape=shape_dict,
    freeze_params=True,
)

with tvm.transform.PassContext(opt_level=3):
    lib = relay.build(mod, target="llvm", params=params)

module = graph_executor.GraphModule(lib["default"](tvm.cpu(0)))
module.set_input(input_name, input_data)
module.run()
output = module.get_output(0).numpy()
```

再画成概念图：

```text
JPG 图片
  -> preprocess_image()
  -> NCHW float32 tensor

ONNX ResNet18
  -> relay.frontend.from_onnx()
  -> Relay IRModule + params
  -> relay.build()
  -> GraphExecutorFactoryModule
  -> graph_executor.GraphModule
  -> set_input() + run() + get_output()
  -> top-k 分类结果
```

初学时建议先把这条链路背下来。后面无论换成 ViT、BERT，还是换成自定义算子，很多工作本质上都是在这条链路的某个位置插入新逻辑。

## 运行示例 {#run-example}

在 TVM 0.19.0 仓库根目录运行：

```bash
cd /Users/huzi/Documents/Code/tvm/apache-tvm-0.19.0
.venv-0.19/bin/python toys/resnet18/resnet18_relay_classify.py
```

输出类似：

```text
Loading ONNX ResNet18
Converting ONNX to Relay graph
Compiling with relay.build for llvm
Running inference with graph executor
image: /Users/huzi/Documents/Code/tvm/apache-tvm-0.19.0/toys/cache/kitten.jpg
input: x (1, 3, 224, 224)
top classifications:
 1.  282 tiger cat                      0.7885
 2.  281 tabby                          0.1399
 3.  285 Egyptian cat                   0.0451
 4.  287 lynx                           0.0039
 5.  292 tiger                          0.0007
```

换自己的图片：

```bash
.venv-0.19/bin/python toys/resnet18/resnet18_relay_classify.py --image-path /path/to/image.jpg
```

只看前三个分类：

```bash
.venv-0.19/bin/python toys/resnet18/resnet18_relay_classify.py --topk 3
```

如果第一次运行需要下载 ONNX 模型、标签或测试图片，脚本会先放到本地 cache。后续运行会直接复用本地文件。

## 图片预处理：把图片变成张量 {#preprocess-image}

神经网络不能直接接收 JPG 文件。ResNet18 需要的输入是：

```text
(1, 3, 224, 224)
```

这个 shape 的含义是：

```text
1    batch size，一次输入 1 张图片
3    RGB 三个通道
224  高度
224  宽度
```

`preprocess_image()` 做了这些事情：

1. 打开图片并转成 RGB。
2. 短边缩放到 256。
3. 中心裁剪成 224 x 224。
4. 像素值从 `0 ~ 255` 缩放到 `0 ~ 1`。
5. 使用 ImageNet 的 mean/std 做归一化。
6. 从 HWC 格式转成 CHW 格式。
7. 加上 batch 维度，变成 NCHW。

关键代码是：

```python
data = np.asarray(image).astype("float32") / 255.0
data = (data - IMAGE_NET_MEAN) / IMAGE_NET_STD
data = np.transpose(data, (2, 0, 1))
return np.expand_dims(data, axis=0).astype("float32")
```

可以把这一步理解为：

```text
人能看的图片 -> 模型能吃的张量
```

很多初学者第一次跑模型时，问题不在 TVM 编译，而在预处理。比如通道顺序写成 NHWC、忘记归一化、mean/std 用错，模型都可能正常运行，但分类结果会明显不对。

## 模型导入：ONNX 到 Relay {#onnx-to-relay}

Relay 是 TVM 里传统且成熟的一代深度学习图 IR。对 ONNX 模型来说，最常见的导入方式是：

```python
mod, params = relay.frontend.from_onnx(
    onnx_model,
    shape=shape_dict,
    freeze_params=True,
)
```

这里有三个核心对象：

```text
onnx_model
  外部框架模型，本例中是 ResNet18 ONNX 文件。

mod
  Relay IRModule，可以理解为 TVM 内部的模型计算图。

params
  模型权重，比如卷积权重、BatchNorm 参数、全连接层权重等。
```

`shape_dict` 告诉 TVM 输入张量的名字和形状：

```python
shape_dict = {"x": (1, 3, 224, 224)}
```

脚本里没有硬编码输入名，而是从 ONNX graph 中找到第一个非 initializer 输入：

```python
input_name = first_model_input_name(onnx_model)
shape_dict = {input_name: input_data.shape}
```

这样写比直接假设输入名叫 `"x"` 更稳一点。不同导出工具生成的 ONNX 输入名可能不一样，比如 `"input"`、`"images"`、`"data"`。

如果想观察 Relay IR，可以在导入后打印：

```python
print(mod)
print(mod["main"])
```

ResNet18 的 IR 会比较长，但你能看到类似这些算子：

```text
nn.conv2d
nn.batch_norm
nn.relu
nn.max_pool2d
nn.global_avg_pool2d
dense
```

这一步之后，模型已经从 ONNX 世界进入了 TVM 的 Relay 世界。

## 模型编译：relay.build 做了什么 {#relay-build}

编译入口是：

```python
target = "llvm"

with tvm.transform.PassContext(opt_level=3):
    lib = relay.build(mod, target=target, params=params)
```

这行代码很关键。`relay.build()` 不是简单地“保存模型”，而是把 Relay IRModule 编译成可以被 runtime 执行的产物。

大致链路是：

```text
Relay IRModule
  -> Relay optimization passes
  -> operator implementation selection
  -> TE compute / schedule
  -> TIR
  -> LLVM codegen
  -> runtime module
```

其中：

```text
target = "llvm"
```

表示编译到本机 CPU。对于本地入门示例，这是最方便的 target。

```python
tvm.transform.PassContext(opt_level=3)
```

表示使用一组常见优化。初学阶段一般先用 `opt_level=3`。如果你在调试 IR 变化，也可以临时降到 `0` 或 `1`，减少优化带来的干扰。

脚本里还手动调用了一次：

```python
opt_mod, _ = relay.optimize(mod, target=target, params=params)
```

这不是运行模型必须的步骤，只是为了把优化后的 Relay IR 打印出来，方便观察 `relay.build()` 前模型图发生了什么变化。

`relay.build()` 返回的通常是一个 factory module。后面通过：

```python
lib["default"](dev)
```

实例化出真正绑定到某个设备的 runtime module。

## 运行时：Graph Executor {#graph-executor}

Relay 编译后，本例使用 Graph Executor 执行模型：

```python
dev = tvm.cpu(0)
module = graph_executor.GraphModule(lib["default"](dev))
module.set_input(input_name, input_data)
module.run()
output = module.get_output(0).numpy()
```

逐句看：

```python
dev = tvm.cpu(0)
```

选择第 0 个 CPU 设备。

```python
module = graph_executor.GraphModule(lib["default"](dev))
```

从编译结果中创建 Graph Executor 运行时模块。

```python
module.set_input(input_name, input_data)
```

设置模型输入。

```python
module.run()
```

执行一次推理。

```python
output = module.get_output(0).numpy()
```

取第 0 个输出，并转成 NumPy 数组。

如果模型有多个输出，可以用：

```python
num_outputs = module.get_num_outputs()
out0 = module.get_output(0).numpy()
out1 = module.get_output(1).numpy()
```

Graph Executor 的好处是直观：编译得到 `lib`，绑定设备得到 runtime module，设置输入，执行，读取输出。对 Relay 入门来说，它比一次性引入更多 runtime 概念更容易理解。

## Relay 和 Relax 的区别 {#relay-vs-relax}

TVM 现在同时能看到 Relay 和 Relax。刚开始学的时候，最容易困惑的是：为什么有的教程用 `relay.build()`，有的教程用 `tvm.compile()` 和 `relax.VirtualMachine()`？

一句话区分：

```text
Relay 是 TVM 传统且成熟的一代深度学习图 IR。
Relax 是 TVM 新一代高层 IR，更强调动态 shape、现代模型和 TensorIR 统一。
```

常见流程对比如下：

| 项目 | Relay | Relax |
|---|---|---|
| 常见导入 ONNX | `relay.frontend.from_onnx` | `tvm.relax.frontend.onnx.from_onnx` |
| 编译入口 | `relay.build` | `tvm.compile` |
| 优化配置 | `PassContext(opt_level=3)` | `relax.get_pipeline(...)` |
| 常见运行器 | `graph_executor.GraphModule` | `relax.VirtualMachine` |
| 初学体验 | 老教程多，路径稳定 | API 更新更多，概念更现代 |

Relay 示例：

```python
mod, params = relay.frontend.from_onnx(onnx_model, shape=shape_dict)

with tvm.transform.PassContext(opt_level=3):
    lib = relay.build(mod, target="llvm", params=params)

module = graph_executor.GraphModule(lib["default"](dev))
module.set_input(input_name, input_data)
module.run()
output = module.get_output(0).numpy()
```

Relax 示例大致是：

```python
mod = tvm_onnx.from_onnx(
    onnx_model,
    shape_dict={input_name: input_data.shape},
    dtype_dict={input_name: "float32"},
)

mod = relax.get_pipeline("zero")(mod)
executable = tvm.compile(mod, target="llvm")

vm = relax.VirtualMachine(executable, dev)
output = vm["main"](tvm.runtime.tensor(input_data, dev))
```

所以它们不是简单替换 API 名字。IR、编译入口、运行器和参数组织方式都不同。

如果你当前主要使用 TVM 0.19.0、已有 Relay 教程、或者要写 Relay pass / 自定义 Relay op，那么先把 Relay 跑通会更直接。Relax 可以作为后续了解的新路线。

## 常用 target 和 PassContext {#target-passcontext}

`target` 决定编译出来的代码在哪里运行。

本机 CPU：

```python
target = "llvm"
dev = tvm.cpu(0)
```

CUDA GPU：

```python
target = "cuda"
dev = tvm.cuda(0)
```

OpenCL：

```python
target = "opencl"
dev = tvm.opencl(0)
```

ARM CPU 交叉编译时常见：

```python
target = "llvm -mtriple=aarch64-linux-gnu"
```

入门阶段，优先用 `llvm`。它依赖少，方便把注意力放在 TVM 编译链路本身。

`PassContext` 的 `opt_level` 可以粗略理解为优化强度：

```text
0  基本不优化，适合观察原始 IR
1  基础优化
2  更多图优化
3  常用默认选择，示例和教程里很常见
```

一般先写：

```python
with tvm.transform.PassContext(opt_level=3):
    lib = relay.build(mod, target=target, params=params)
```

如果遇到奇怪的类型推导、shape 或 pass 相关问题，可以临时降低优化等级辅助定位。

## 初学者常见问题 {#beginner-faq}

**1. `params` 是什么，为什么要传给 `relay.build`？**

`params` 是模型权重。比如 ResNet18 中的卷积权重、BatchNorm 参数、全连接层权重等。

```python
lib = relay.build(mod, target=target, params=params)
```

传入 `params` 后，TVM 可以把权重绑定进编译结果。如果忘记传，可能会导致编译失败、运行时要求额外设置很多权重，或者输出结果不对。

**2. `freeze_params=True` 是什么含义？**

它表示尽量把参数作为 Relay 常量嵌入计算图。对推理场景来说，这通常更方便，也有利于后续优化。

**3. shape 写错会怎样？**

常见后果包括：

1. ONNX 转 Relay 时报 shape 不匹配。
2. 编译时报类型推导错误。
3. 运行时报输入维度错误。
4. 程序能跑，但分类结果很差。

所以导入模型时，输入名和输入 shape 要非常仔细。

**4. 看到 operators have not been tuned 是错误吗？**

不是错误。类似提示一般表示 TVM 没有找到 AutoTVM/TopHub 的调优日志，所以会使用默认 schedule。结果仍然应该是正确的，只是性能不一定最好。

示例脚本里设置了：

```python
os.environ.setdefault("TOPHUB_LOCATION", "NONE")
```

这是为了避免 `relay.build` 自动尝试下载 TopHub 日志。入门时先保证功能跑通，性能调优可以放到后面。

**5. 为什么脚本里要兼容 `onnx.mapping`？**

TVM 0.19 的 Relay ONNX frontend 会使用 `onnx.mapping`，但新版 ONNX 包已经调整了相关模块。脚本里补了一个兼容映射：

```python
if "onnx.mapping" not in sys.modules and hasattr(onnx, "_mapping"):
    mapping_module = types.ModuleType("onnx.mapping")
    mapping_module.TENSOR_TYPE_TO_NP_TYPE = {
        key: value.np_dtype for key, value in onnx._mapping.TENSOR_TYPE_MAP.items()
    }
    sys.modules["onnx.mapping"] = mapping_module
```

这只是为了让旧版 TVM frontend 能和新版 ONNX 包配合使用。如果你使用较旧的 ONNX 版本，可能不需要这段代码。

## 最小模板 {#minimal-template}

后面写 Relay 程序时，可以先从下面这个模板开始：

```python
import tvm
from tvm import relay
from tvm.contrib import graph_executor

target = "llvm"
dev = tvm.cpu(0)

# 1. 从外部框架导入模型
mod, params = relay.frontend.from_onnx(
    onnx_model,
    shape={"x": (1, 3, 224, 224)},
    freeze_params=True,
)

# 2. 编译
with tvm.transform.PassContext(opt_level=3):
    lib = relay.build(mod, target=target, params=params)

# 3. 创建运行时
module = graph_executor.GraphModule(lib["default"](dev))

# 4. 设置输入并运行
module.set_input("x", input_data)
module.run()

# 5. 获取输出
output = module.get_output(0).numpy()
```

再简化成三个核心动作：

```text
from_xxx()      负责导入模型
relay.build()   负责编译模型
GraphModule     负责运行模型
```

初学阶段优先记住这些名字：

```text
relay.frontend.from_onnx
relay.build
tvm.transform.PassContext
graph_executor.GraphModule
module.set_input
module.run
module.get_output
target = "llvm"
dev = tvm.cpu(0)
```

这些 API 构成了 Relay 推理路径的骨架。

## 下一步 {#next-step}

跑通 ResNet18 之后，可以继续做几件事：

1. 打印 `mod` 和 `opt_mod`，对比优化前后的 Relay IR。
2. 换一张图片，确认预处理和 top-k 输出是否合理。
3. 换一个 ONNX 模型，练习处理不同输入名和输入 shape。
4. 导出编译结果：

```python
lib.export_library("resnet18_relay.so")
```

然后重新加载：

```python
loaded_lib = tvm.runtime.load_module("resnet18_relay.so")
module = graph_executor.GraphModule(loaded_lib["default"](dev))
```

5. 在 Relay IR 上写一个简单 pass，比如统计 `nn.conv2d` 数量，或者替换某些基础算子。

等你理解了：

```text
ONNX -> Relay -> relay.build -> Graph Executor
```

这条链路之后，再去看自定义 Relay op、TE compute、schedule、TIR lowering、codegen，就会清楚很多。TVM 的学习曲线不算平，但只要先抓住这条主线，后面的概念就不再是一堆散点。
