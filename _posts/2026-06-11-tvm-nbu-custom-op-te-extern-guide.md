---
title: "TVM 自定义算子接入：从 Relay 注册到 te.extern Runtime"
date: 2026-06-11
permalink: /posts/2026/06/tvm-nbu-custom-op-te-extern-guide/
tags:
  - TVM
  - Relay
  - te.extern
  - Custom Op
  - Compiler
excerpt: "从 C++ 侧 Relay 算子注册出发，整理 Python API、FTVMCompute、FTVMSchedule、te.extern、PackedFunc runtime 和常见报错的完整接入套路。"
toc_label: "目录"
toc_items:
  - title: "总体链路"
    id: "overall-chain"
  - title: "从 C++ 算子里读哪些信息"
    id: "read-cpp-op"
  - title: "Relay op 名字"
    id: "relay-op-name"
    level: 3
  - title: "输入个数和输入顺序"
    id: "input-count-order"
    level: 3
  - title: "attrs 字段"
    id: "attrs-fields"
    level: 3
  - title: "type relation 输出"
    id: "type-relation-output"
    level: 3
  - title: "TNonComputational 和 TOpPattern"
    id: "noncomputational-pattern"
    level: 3
  - title: "Python API wrapper 怎么写"
    id: "python-api-wrapper"
  - title: "_custom.py 基础模板"
    id: "custom-py-template"
  - title: "te.extern 每个参数是什么意思"
    id: "te-extern-parameters"
  - title: "tuple 输入算子怎么处理"
    id: "tuple-inputs"
  - title: "runtime PackedFunc stub 模板"
    id: "runtime-packedfunc-stub"
  - title: "生成 compute/schedule 的机械步骤"
    id: "mechanical-steps"
  - title: "找 op name"
    id: "step-op-name"
    level: 3
  - title: "找 tensor inputs"
    id: "step-tensor-inputs"
    level: 3
  - title: "找需要传 runtime 的 attrs"
    id: "step-runtime-attrs"
    level: 3
  - title: "写 te.extern"
    id: "step-write-te-extern"
    level: 3
  - title: "注册 schedule"
    id: "step-register-schedule"
    level: 3
  - title: "示例：nbu.transpose"
    id: "example-transpose"
  - title: "示例：nbu.fused_layer_norm"
    id: "example-layer-norm"
  - title: "常见报错和定位"
    id: "common-errors"
  - title: "FTVMCompute already registered"
    id: "ftvmcompute-registered"
    level: 3
  - title: "Generic function already registered for xxx_strategy"
    id: "generic-function-registered"
    level: 3
  - title: "Non-primitive-call nodes should have been transformed away"
    id: "non-primitive-call"
    level: 3
  - title: "runtime PackedFunc not found"
    id: "packedfunc-not-found"
    level: 3
  - title: "set_input shape mismatch"
    id: "set-input-shape-mismatch"
    level: 3
  - title: "最小检查清单"
    id: "checklist"
  - title: "推荐文件组织"
    id: "file-organization"
---

本文记录一种固定套路：从 C++ 侧 Relay 算子注册代码中提取信息，生成 Python 侧 `_nn.py` 或 `_custom.py` 里的 `FTVMCompute + FTVMSchedule`，并通过 `te.extern -> tir.call_packed -> Python PackedFunc` 把算子计算转到 Python runtime。

这套方法适合先跑通 Relay build 和 graph executor 流程，或者用 Python/NumPy 做正确性验证。它不是高性能实现方案，因为运行期会跨 C++/Python 边界，并且通常会产生 host copy。

## 1. 总体链路 {#overall-chain}

完整执行链路是：

```text
C++ nn.cc
  -> RELAY_REGISTER_OP("nbu.xxx")
  -> type relation 推导输出 shape/dtype
  -> TVM_REGISTER_GLOBAL("nbu._make.xxx")

Python API
  -> def xxx(...): return _make.xxx(...)

Python compute/schedule
  -> reg.register_compute("nbu.xxx")
  -> te.extern(...)
  -> tvm.tir.call_packed("nbu.runtime.xxx", ...)
  -> reg.register_schedule("nbu.xxx", schedule_nbu_extern)

relay.build
  -> FuseOps 把 op 包成 Primitive function
  -> LowerTE 查 FTVMCompute/FTVMSchedule
  -> 生成 TIR PrimFunc
  -> Relay call 变成 call_lowered(...)

graph_executor.run
  -> 执行 TIR PrimFunc
  -> call_packed 查找 "nbu.runtime.xxx"
  -> 调 Python PackedFunc
  -> runtime 函数写回 out.copyfrom(...)
```

要分清两个阶段：

- `relay.build` 阶段只需要 `FTVMCompute + FTVMSchedule` 能生成 TIR。
- `module.run()` 阶段才需要注册 runtime PackedFunc。

因此，如果 runtime 还没接，理论上 `relay.build` 仍然可以成功；真正跑 executor 时才会报找不到 `nbu.runtime.xxx`。

## 2. 从 C++ 算子里读哪些信息 {#read-cpp-op}

看一个 C++ 算子时，重点读五块。

### 2.1 Relay op 名字 {#relay-op-name}

看：

```cpp
static const Op& op = Op::Get("nbu.fused_add_relu");
RELAY_REGISTER_OP("nbu.fused_add_relu")
```

Python 侧注册必须使用完全相同的名字：

```python
@reg.register_compute("nbu.fused_add_relu")
...
reg.register_schedule("nbu.fused_add_relu", schedule_nbu_extern)
```

### 2.2 输入个数和输入顺序 {#input-count-order}

看：

```cpp
.set_num_inputs(2)
.add_argument("data0", "Tensor", ...)
.add_argument("data1", "Tensor", ...)
```

以及 Make 函数里的 `Call`：

```cpp
auto call = Call(op, {data0, data1}, Attrs(attrs), {});
```

这决定 `_custom.py` 里的：

```python
data0, data1 = inputs
```

也决定 `call_packed` 参数顺序：

```python
tvm.tir.call_packed(
    "nbu.runtime.fused_add_relu",
    ins[0],
    ins[1],
    outs[0],
)
```

输入顺序必须和 C++ `Call(op, {...})` 一致。

### 2.3 attrs 字段 {#attrs-fields}

看 Make 函数：

```cpp
attrs->out_shape = out_shape;
attrs->out_dtype = out_dtype;
attrs->epsilon = epsilon;
attrs->axis = axis;
```

这些字段在 Python compute 中通过 `attrs.xxx` 读取：

```python
epsilon = float(attrs.epsilon)
axis = int(attrs.axis)
```

通常 `out_shape/out_dtype` 不需要手动传给 runtime，因为它们已经体现在 `out_type.shape/out_type.dtype` 和 `outs[0]` 里。

只有当 runtime 逻辑确实需要某个 attr，比如 `axis`、`epsilon`、`take_axis`，才把它作为普通 scalar 参数传给 `call_packed`。

### 2.4 type relation 输出 {#type-relation-output}

看 type relation 里最后的 Assign：

```cpp
reporter->Assign(types[types.size() - 1], TensorType(out_shape, out_dtype));
```

这决定 Python compute 中：

```python
te.extern(
    out_type.shape,
    ...,
    dtype=out_type.dtype,
)
```

`out_type.shape` 和 `out_type.dtype` 是 Relay 类型推导之后的结果。

### 2.5 TNonComputational 和 TOpPattern {#noncomputational-pattern}

一般自定义 extern 算子保留：

```cpp
.set_attr<TOpPattern>("TOpPattern", kOpaque)
```

如果你希望这个 op 通过 `FTVMCompute -> te.extern` 生成真实 kernel，通常不要设置：

```cpp
.set_attr<TNonComputational>("TNonComputational", true)
```

原因是 `FuseOps` 遇到 `TNonComputational=true` 的普通 op，可能不会把它包成 Primitive function，后面 graph executor codegen 会看到残留的普通 call：

```relay
nbu.fused_concats(...)
```

然后报：

```text
Non-primitive-call nodes should have been transformed away.
The graph executor code generator expects all calls to be call_lowered
```

如果这个 op 只是纯 shape/view/alias 元操作，才考虑 `TNonComputational=true`。如果它会通过 runtime 真正写输出 buffer，就不要标。

## 3. Python API wrapper 怎么写 {#python-api-wrapper}

如果 C++ 注册了：

```cpp
TVM_REGISTER_GLOBAL("nbu._make.fused_add_relu").set_body_typed(MakeNbuAddRelu);
```

Python API 通常写：

```python
def fused_add_relu(
    data0,
    data1,
    broadcast_add,
    addlhs_shape,
    addrhs_shape,
    nonlr_mode,
    out_shape,
    out_dtype,
):
    return _make.fused_add_relu(
        data0,
        data1,
        broadcast_add,
        addlhs_shape,
        addrhs_shape,
        nonlr_mode,
        out_shape,
        out_dtype,
    )
```

注意：Python API 参数不等于 Relay op 的 tensor inputs。

例如上面有 8 个 Python API 参数，但真正进入 Relay Call 的 tensor input 只有：

```cpp
Call(op, {data0, data1}, Attrs(attrs), {})
```

所以 `_custom.py` 里的 `inputs` 只有两个：

```python
data0, data1 = inputs
```

其它参数是 attrs。

## 4. _custom.py 基础模板 {#custom-py-template}

文件顶部需要：

```python
import tvm
from tvm import te
from tvm.target import generic_func

from . import op as reg
```

建议所有 extern-backed nbu 算子共用一个 schedule：

```python
@generic_func
def schedule_nbu_extern(attrs, outs, target):
    """Generic schedule for nbu extern-backed ops."""

    with target:
        return te.create_schedule([out.op for out in outs])
```

单输入算子模板：

```python
# ---------------nbu.xxx---------------

@reg.register_compute("nbu.xxx")
def compute_nbu_xxx(attrs, inputs, out_type):
    """Lower nbu.xxx to te.extern + runtime PackedFunc."""

    (data,) = inputs

    return [
        te.extern(
            out_type.shape,
            [data],
            lambda ins, outs: tvm.tir.call_packed(
                "nbu.runtime.xxx",
                ins[0],
                outs[0],
            ),
            name="nbu_xxx",
            dtype=out_type.dtype,
        )
    ]


reg.register_schedule("nbu.xxx", schedule_nbu_extern)

# ---------------nbu.xxx---------------
```

多输入算子模板：

```python
# ---------------nbu.xxx---------------

@reg.register_compute("nbu.xxx")
def compute_nbu_xxx(attrs, inputs, out_type):
    """Lower nbu.xxx to te.extern + runtime PackedFunc."""

    data0, data1, data2 = inputs

    return [
        te.extern(
            out_type.shape,
            [data0, data1, data2],
            lambda ins, outs: tvm.tir.call_packed(
                "nbu.runtime.xxx",
                ins[0],
                ins[1],
                ins[2],
                outs[0],
            ),
            name="nbu_xxx",
            dtype=out_type.dtype,
        )
    ]


reg.register_schedule("nbu.xxx", schedule_nbu_extern)

# ---------------nbu.xxx---------------
```

带 scalar attr 的模板：

```python
# ---------------nbu.xxx---------------

@reg.register_compute("nbu.xxx")
def compute_nbu_xxx(attrs, inputs, out_type):
    """Lower nbu.xxx to te.extern + runtime PackedFunc."""

    data, gamma, beta = inputs
    epsilon = float(attrs.epsilon)
    axis = int(attrs.axis)

    return [
        te.extern(
            out_type.shape,
            [data, gamma, beta],
            lambda ins, outs: tvm.tir.call_packed(
                "nbu.runtime.xxx",
                ins[0],
                ins[1],
                ins[2],
                outs[0],
                epsilon,
                axis,
            ),
            name="nbu_xxx",
            dtype=out_type.dtype,
        )
    ]


reg.register_schedule("nbu.xxx", schedule_nbu_extern)

# ---------------nbu.xxx---------------
```

## 5. te.extern 每个参数是什么意思 {#te-extern-parameters}

以：

```python
te.extern(
    out_type.shape,
    [data0, data1],
    lambda ins, outs: tvm.tir.call_packed(
        "nbu.runtime.fused_add_relu",
        ins[0],
        ins[1],
        outs[0],
    ),
    name="nbu_fused_add_relu",
    dtype=out_type.dtype,
)
```

逐项解释：

```python
out_type.shape
```

输出 tensor 的 shape。来自 C++ type relation 的 `TensorType(...shape..., ...dtype...)`。

```python
[data0, data1]
```

extern op 的输入 tensor 列表。顺序必须和 C++ `Call(op, {data0, data1}, ...)` 一致。

```python
lambda ins, outs: ...
```

生成 TIR extern body 的函数。这里不是直接执行 Python 计算，而是在 lowering 阶段生成 TIR 语句。

```python
ins[0], ins[1]
```

对应输入 buffer。

```python
outs[0]
```

对应 TVM runtime 分配好的输出 buffer。

```python
tvm.tir.call_packed("nbu.runtime.fused_add_relu", ...)
```

生成运行期 PackedFunc 调用。executor 执行到这里时，会在 TVM 全局 registry 里查找同名函数。

```python
name="nbu_fused_add_relu"
```

TE extern op 内部名字，用于 TIR/PrimFunc 命名。它不是 runtime PackedFunc 名。

```python
dtype=out_type.dtype
```

输出 dtype。

## 6. tuple 输入算子怎么处理 {#tuple-inputs}

有些 Relay op 的输入是一个 tuple，比如 concat：

```relay
%x = (%a, %b, %c)
nbu.fused_concats(%x)
```

C++ type relation 里可能看到：

```cpp
const auto* data = types[0].as<TupleTypeNode>();
```

这类算子在 Python compute 里不要简单写死：

```python
(data,) = inputs
```

更稳的写法是把输入展开成动态列表：

```python
# ---------------nbu.fused_concats---------------

@reg.register_compute("nbu.fused_concats")
def compute_nbu_fused_concats(attrs, inputs, out_type):
    """Lower nbu.fused_concats to te.extern + runtime PackedFunc."""

    input_tensors = list(inputs)
    axis = int(attrs.axis)

    return [
        te.extern(
            out_type.shape,
            input_tensors,
            lambda ins, outs: tvm.tir.call_packed(
                "nbu.runtime.fused_concats",
                *ins,
                outs[0],
                axis,
            ),
            name="nbu_fused_concats",
            dtype=out_type.dtype,
        )
    ]


reg.register_schedule("nbu.fused_concats", schedule_nbu_extern)

# ---------------nbu.fused_concats---------------
```

对应 runtime stub 可以写成可变参数：

```python
def nbu_fused_concats_runtime(*args) -> None:
    *inputs, out, axis = args
    out.copyfrom(np.zeros(out.shape, dtype=out.dtype))
```

## 7. runtime PackedFunc stub 模板 {#runtime-packedfunc-stub}

runtime 文件只需要保证接口正确、能写回输出，就可以先跑通 executor。

```python
from __future__ import annotations

from collections import Counter

import numpy as np
import tvm


CALL_COUNTS = Counter()


def _record(name: str) -> None:
    CALL_COUNTS[name] += 1


def _zero_output(out) -> None:
    out.copyfrom(np.zeros(out.shape, dtype=out.dtype))


def reset_call_counts() -> None:
    CALL_COUNTS.clear()


def get_call_count(name: str) -> int:
    return CALL_COUNTS[name]


def nbu_xxx_runtime(data0, data1, out) -> None:
    _record("nbu.runtime.xxx")
    _zero_output(out)


def register_nbu_runtime_packed_funcs() -> None:
    tvm.register_func("nbu.runtime.xxx", nbu_xxx_runtime, override=True)
```

核心规则：

- runtime 函数参数顺序必须和 `_custom.py` 的 `call_packed` 顺序一致。
- 输出不要 `return result`，而是 `out.copyfrom(result)`。
- `tvm.register_func` 的字符串必须和 `call_packed` 的字符串完全一致。
- `override=True` 方便反复调试。

## 8. 生成 compute/schedule 的机械步骤 {#mechanical-steps}

拿到一个 C++ 算子后，按下面步骤生成。

### Step 1: 找 op name {#step-op-name}

```cpp
Op::Get("nbu.transpose")
RELAY_REGISTER_OP("nbu.transpose")
```

得到：

```python
@reg.register_compute("nbu.transpose")
reg.register_schedule("nbu.transpose", schedule_nbu_extern)
```

### Step 2: 找 tensor inputs {#step-tensor-inputs}

看：

```cpp
.set_num_inputs(1)
Call(op, {data}, Attrs(attrs), {})
```

得到：

```python
(data,) = inputs
```

### Step 3: 找需要传 runtime 的 attrs {#step-runtime-attrs}

如果 runtime 只需要输出 buffer 的 shape/dtype，不需要额外传。

如果有数学参数：

```cpp
attrs->epsilon = epsilon;
attrs->axis = axis;
```

得到：

```python
epsilon = float(attrs.epsilon)
axis = int(attrs.axis)
```

并加到 `call_packed` 参数末尾。

### Step 4: 写 te.extern {#step-write-te-extern}

```python
return [
    te.extern(
        out_type.shape,
        [...inputs...],
        lambda ins, outs: tvm.tir.call_packed(
            "nbu.runtime.xxx",
            ...ins...,
            outs[0],
            ...attrs...
        ),
        name="nbu_xxx",
        dtype=out_type.dtype,
    )
]
```

### Step 5: 注册 schedule {#step-register-schedule}

```python
reg.register_schedule("nbu.xxx", schedule_nbu_extern)
```

## 9. 示例：nbu.transpose {#example-transpose}

C++ 信息：

```cpp
Expr MakeNbuTranspose(Expr data, Array<IndexExpr> tp_ishape,
                      Array<IndexExpr> tp_axes, Array<IndexExpr> oshape,
                      DataType out_dtype) {
  ...
  static const Op& op = Op::Get("nbu.transpose");
  auto call = Call(op, {data}, Attrs(attrs), {});
  return call;
}

RELAY_REGISTER_OP("nbu.transpose")
  .set_num_inputs(1)
  .add_argument("data", "Tensor", "The input tensor.")
  .add_type_rel("NbuTranspose", NbuTransposeRel)
  .set_attr<TOpPattern>("TOpPattern", kOpaque);
```

生成：

```python
# ---------------nbu.transpose---------------

@reg.register_compute("nbu.transpose")
def compute_nbu_transpose(attrs, inputs, out_type):
    """Lower nbu.transpose to te.extern + runtime PackedFunc."""

    (data,) = inputs

    return [
        te.extern(
            out_type.shape,
            [data],
            lambda ins, outs: tvm.tir.call_packed(
                "nbu.runtime.transpose",
                ins[0],
                outs[0],
            ),
            name="nbu_transpose",
            dtype=out_type.dtype,
        )
    ]


reg.register_schedule("nbu.transpose", schedule_nbu_extern)

# ---------------nbu.transpose---------------
```

对应 runtime：

```python
def nbu_transpose_runtime(data, out) -> None:
    out.copyfrom(np.zeros(out.shape, dtype=out.dtype))


tvm.register_func("nbu.runtime.transpose", nbu_transpose_runtime, override=True)
```

## 10. 示例：nbu.fused_layer_norm {#example-layer-norm}

C++ 信息：

```cpp
Call(op, {data, gamma, beta}, Attrs(attrs), {})
attrs->epsilon = epsilon;
```

生成：

```python
# ---------------nbu.fused_layer_norm---------------

@reg.register_compute("nbu.fused_layer_norm")
def compute_nbu_fused_layer_norm(attrs, inputs, out_type):
    """Lower nbu.fused_layer_norm to te.extern + runtime PackedFunc."""

    data, gamma, beta = inputs
    epsilon = float(attrs.epsilon)

    return [
        te.extern(
            out_type.shape,
            [data, gamma, beta],
            lambda ins, outs: tvm.tir.call_packed(
                "nbu.runtime.fused_layer_norm",
                ins[0],
                ins[1],
                ins[2],
                outs[0],
                epsilon,
            ),
            name="nbu_fused_layer_norm",
            dtype=out_type.dtype,
        )
    ]


reg.register_schedule("nbu.fused_layer_norm", schedule_nbu_extern)

# ---------------nbu.fused_layer_norm---------------
```

## 11. 常见报错和定位 {#common-errors}

### 11.1 FTVMCompute already registered {#ftvmcompute-registered}

报错：

```text
Attribute FTVMCompute of custom.numpy_layer_norm is already registered with same plevel=10
```

原因：

- 同一个 op 的 `reg.register_compute` 执行了两次。
- 或同一份 Python 包被不同 import 路径导入两次。
- 或 TVM 源码和业务包里同时注册了同名 op compute。

解决：

- 保证每个 op 的 compute 只注册一次。
- 不要同时在 `tvm/relay/op/_custom.py` 和 `hzb/op/_custom.py` 注册同一个 op。
- 避免 `python.hzb...` 和 `hzb...` 两套 import 路径混用。

### 11.2 Generic function already registered for xxx_strategy {#generic-function-registered}

报错：

```text
Generic function already registered for nbu.transpose_strategy
```

原因：

- `reg.register_schedule("nbu.transpose", schedule_nbu_extern)` 重复执行。

解决：

- 检查是否手写注册一次，又在列表循环里注册一次。
- 检查 op list 是否重复。
- 检查 Python module 是否被重复 import。

### 11.3 Non-primitive-call nodes should have been transformed away {#non-primitive-call}

报错：

```text
The graph executor code generator expects all calls to be call_lowered, but found:
nbu.fused_concats(...)
```

原因：

- 这个 op 没有被 `FuseOps + LowerTE` 变成 `call_lowered`。
- 常见原因是 C++ 注册了 `TNonComputational=true`。
- 也可能是没有注册 `FTVMCompute/FTVMSchedule`。

解决：

- 确认 `_custom.py` 已 import，compute/schedule 已注册。
- 对需要真实执行的 extern op，删除：

```cpp
.set_attr<TNonComputational>("TNonComputational", true)
```

- 确认 build 后 IR 里出现：

```relay
call_lowered(@tvmgen_default_fused_nbu_xxx, ...)
```

### 11.4 runtime PackedFunc not found {#packedfunc-not-found}

可能在 `module.run()` 时出现：

```text
Cannot find function nbu.runtime.xxx
```

原因：

- `_custom.py` 的 `call_packed("nbu.runtime.xxx", ...)` 已生成。
- 但当前 Python 进程没有执行：

```python
tvm.register_func("nbu.runtime.xxx", ...)
```

解决：

- 在 `module.run()` 前调用：

```python
register_nbu_runtime_packed_funcs()
```

### 11.5 set_input shape mismatch {#set-input-shape-mismatch}

报错：

```text
array shape do not match the shape of NDArray
(1, 3, 224, 224) vs (1, 224, 224, 3)
```

含义：

```text
传入 numpy array shape vs executor 期望 input shape
```

解决：

- 打印：

```python
print("feeding input:", input_data.shape)
print("executor input:", module.get_input(input_name).shape)
```

- 两者必须完全一致。
- 如果自定义 pass 把入口从 NCHW 改成 NHWC，则 `set_input` 前需要：

```python
input_data = input_data.transpose(0, 2, 3, 1).astype("float32")
```

## 12. 最小检查清单 {#checklist}

为一个新 nbu 算子接 `te.extern` 前，检查：

- C++ `RELAY_REGISTER_OP("nbu.xxx")` 名字确定。
- C++ `Call(op, {...})` 输入顺序确定。
- C++ type relation 能推导输出 `TensorType(shape, dtype)`。
- Python API wrapper 调 `_make.xxx(...)`。
- `_custom.py` 有 `@reg.register_compute("nbu.xxx")`。
- `_custom.py` 有 `reg.register_schedule("nbu.xxx", schedule_nbu_extern)`。
- `te.extern` 的 input list 和 `call_packed` 的 `ins[...]` 顺序一致。
- `call_packed("nbu.runtime.xxx", ...)` 字符串和 runtime 注册字符串一致。
- runtime 函数通过 `out.copyfrom(...)` 写输出。
- 不重复注册 compute/schedule。
- 对真实执行的 extern op，不设置 `TNonComputational=true`。

## 13. 推荐文件组织 {#file-organization}

建议把三类代码分开：

```text
python/hzb/op/nn.py
  Python Relay API wrapper:
  def fused_add_relu(...): return _make.fused_add_relu(...)

python/hzb/op/_nn.py 或 _custom.py
  FTVMCompute + FTVMSchedule:
  @reg.register_compute("nbu.xxx")
  te.extern(...)

toys/.../nbu_runtime.py
  runtime PackedFunc:
  tvm.register_func("nbu.runtime.xxx", ...)
```

`nn.py` 是“用户如何构造 Relay Call”。

`_nn.py/_custom.py` 是“relay.build 如何 lower 这个 Call”。

`xx_runtime.py` 是“executor.run 如何执行 call_packed”。
