The quantized folder holds the implementation of the low-level quantized kernel.
The kernels are registered in `torch::_ops` namespace, and operate on the quantized `at::Tensor` data type.
You can learn more about the quantized tensors in the [quantized tensor API wiki](https://github.com/pytorch/pytorch/wiki/Introducing-Quantized-Tensor) page.

This document serves as an entry point for quantized kernel implementation.

## Implementing native quantized ops

The new quantized ops are almost always located under the `ATen/native/quantized/cpu` folder.
For the sake of an example, let us implement an element-wise quantized logical AND operation under `ATen/native/quantized/cpu/qand.cpp`.

### Step 0. Implement the quantized function

Before writing the quantized kernel and registering it, let us implement a quantized function.
That would assist in any further discussion.
The snippet below shows the implementation of a quantized AND operator, with the support of all implemented quantized types.

```c++
Tensor quantized_and(Tensor qa, Tensor qb) {
  // Some type checks for qa and qb should be here...
  Tensor qc;
  double scale = qa.q_scale();
  long zero_point qa.q_zero_point();

  auto iter = TensorIterator::binary_op(qc, qa, qb);

  AT_DISPATCH_QINT_TYPES(qa.scalar_type(), "quantized_and", [&]() {
    Tensor qc = at::_empty_affine_quantized(qa.sizes(),
                                            at::device(kCPU).dtype(SCALAR_TYPE),
                                            scale, zero_point);
    cpu_kernel(iter, [&](scalar_t a_value, scalar_t b_value) -> scalar_t {
      return scalar_t(a_value.val_ & b_value.val_);
    });
  });
  return qc;
}
```

The code above is fairly straight-forward:
It takes two quantized tensors `qa` and `qb`, and uses `binary_kernel` to produce a quantized tensor `qc`.
We also use the [`TensorIterator`](https://caffe2.ai/doxygen-c/html/structat_1_1_tensor_iterator.html) in this example.
The only part that that requires explicit explanation is the `AT_DISPATCH_QINT_TYPES`.
This macro makes sure that the underlying code works with all quantized types.
It provides several useful "aliases":

- `SCALAR_TYPE` -- `ScalarType` of the quantized tensor (e.g. `kQInt8`)
- `scalar_t` -- quantized data type (dtype, e.g. `qint8`)
- `underlying_t` -- underlying POD data type (dtype, e.g. `int8_t`)

The macro takes three arguments:

1. Quantized data type. This will define what the "aliases" are.
In the example above, the resulting tensor will be the same as the `qa.scalar_type()`.
2. Function name. This argument is currently used for error reporting.
3. Implementation lambda. The main implementation should sit in the body of this lambda.
it should also use the aliases for the quantized data types instead of the explicit data types.

### Step 1. Create the kernel

All kernels must be classes inheriting from `torch::OperatorKernel`.
The implementation itself should be under the `operator()` method.
In the `qand.cpp` file, we create the following

```c++
class QuantizedAnd final : public torch::OperatorKernel {
 public:
  Tensor operator(Tensor qa, Tensor qb) {
    return quantized_and(qa, qb);
  }
};
```

### Step 2a. Register the kernel

The registration is done using the `torch::RegisterOperators().op(...)`.

```c++
static auto registry = torch::RegisterOperators().op(
    "quantized::and(Tensor qa, Tensor qb) -> Tensor",
    torch::RegisterOperators::options()
      .kernel<QuantizedAnd>(QuantizedCPUTensorId()));
```

The registry takes two arguments:

1. **Function schema string**: This schema describes the usage of the op.
In the example above the schema is `"quantized::and(Tensor qa, Tensor qb) -> Tensor"`.
This translates to `torch._ops.ops.quantized.and` function in Python of the appropriate signature.
**Note:** The arguments signature in the schema is optional, and can also be written as `"quantized::and"` (without args).
2. **Registration options** should be of type `torch::RegisterOperators::options()`.
To attach a kernel to it, use `.kernel<KERNEL_CLASS>(DISPATCH_KEY)`.
In quantized ops you almost always want to use the `QuantizedCPUTensorId()` dispatcher.

### Step 2b. [Optional] Registering the operation with the `native_functions.yaml`

In some cases, if the signature of the quantized function and its non-quantized counterpart are the same, it is worth adding it to the `ATen/native/native_functions.yaml`.
A detailed explanation on this file can be found [here](https://github.com/pytorch/pytorch/blob/master/aten/src/ATen/native/README.md).

**If adding a new entry to the `native_functions.yaml`:**

```yaml
- func: quantized_and(Tensor qa, Tensor qb) -> Tensor
  dispatch:
    QuantizedCPU: quantized_and
```

**If adding to an existing entry in the `native_functions.yaml`:**

If you find an entry in the yaml file, and would like to add a quantized kernel to it, you can just add a new dispatch entry for it.
For example, let's assume there existed a `logical_and` function in the YAML file.
In that case, modification would look as:

```yaml
- func: logical_and(Tensor a, Tensor b) -> Tensor
  dispatch:
    CPU: _logical_and_cpu    # Assume this existed
    CUDA: _logical_and_cuda  # Assume this existed
    QuantizedCPU: quantized_and  # We add this line
```

### Putting it all together

The final file `ATen/native/quantized/cpu/qand.cpp` would look as follows

```c++
#include <ATen/ATen.h>
#include <ATen/NativeFunctions.h>  // Need that for the `native_functions.yaml`
#include <ATen/core/Type.h>
#include <ATen/core/op_registration/op_registration.h>
#include <ATen/native/TensorIterator.h>
#include <ATen/native/cpu/Loops.h>

namespace at { namespace native {
Tensor quantized_and(Tensor qa, Tensor qb) {
  // The awesome op implementation...
  return qc;
}

namespace {
class QuantizedAnd final : public torch::OperatorKernel {
 public:
  Tensor operator(Tensor qa, Tensor qb) {
    return quantized_and(qa, qb);
  }
};

static auto registry = torch::RegisterOperators().op(
    "quantized::and(Tensor qa, Tensor qb) -> Tensor",
    torch::RegisterOperators::options()
      .kernel<QuantizedAnd>(QuantizedCPUTensorId()));

}  // namespace
}}  // namespace at::native
```

Notice that we try to keep all the kernels in the anonymous namespace.
The reason for that is that we access the kernels only through the `torch` namespace and this prevents symbol clashes in the linker.

### Step 3. Administrative stuff

Before the op can be used, it needs to be compiled.
If the op is placed under `native/quantized/cpu`, this already done for you.
However, if the location is changed, two files must be notified:

- *`caffe2/aten/TARGETS`* -- You can follow the same example, and add your path in somewhere in that file. Notice in this file we places the path to the quantized source files:
```bash
ATEN_NATIVE_CPP = glob([
  # ...
  "src/ATen/native/quantized/**/*.cpp",
])
```

- *`caffe2/aten/src/ATen/CMakeLists.txt`* -- Again, following the example, you must add your paths.
The current quantization paths are added as

```bash
FILE(GLOB native_quantized_cpp
          "native/quantized/*.cpp"
          "native/quantized/cpu/*.cpp")
```

## Using quantized ops

### Python

Usage in Python is pretty easy.
To implement the python quantized function using our kernel, you can do the following

```python
from torch._ops import ops

def quantized_and(qa, qb):
  # Notice the schema changed from `quantized::and` to `quantized.and`
  return ops.quantized.and(qa, qb)
```

**Note:** If writing new pytorch functions that use quantized kernels, it is strongly encouraged to place them in the `torch/nn/quantized/functional.py`.

### C++

You should not need to use the registered kernels in C++.
Although **officially not supported**, you can use the following

```c++
namespace at { namespace native {
namespace dispatch_tools {
/* Creates a stack of inputs consumable by the dispatcher.*/
template<class... Inputs>
inline std::vector<torch::IValue> makeStack(Inputs&&... inputs) {
  return {std::forward<Inputs>(inputs)...};
}

/* Given an operator handle, calls it using some arguments.*/
template<class... Args>
inline std::vector<torch::IValue> callOp(const torch::OperatorHandle& op,
                                       Args... args) {
  auto stack = makeStack(std::forward<Args>(args)...);
  auto kernel = torch::Dispatcher::singleton().lookup(op, &stack);
  kernel.call(&stack);
  return stack;
}

/* Finds the op and calls the callOp on it.*/
template<class... Args>
inline std::vector<torch::IValue> callOp(const char* func_name,
                                       const char* overload_name,
                                       Args... args) {
  const torch::optional<torch::OperatorHandle> op_handle
    = torch::Dispatcher::singleton().findSchema(func_name, overload_name);
  assert(op_handle.has_value());
  return callOp(op_handle.value(), args...);
}
}  // dispatch_tools

// This is your new function
Tensor quantized_and(Tensor qa, Tensor qb) {
  return dispatch_tools::callOp("quantized::and", "", qa, qb);
}
}}  // namespace at::native
```

The `dispatch_tools` is just a local namespace created for a sake of example.
