---
title: AMDGPU Host启动分析
categories: 
 - AMDGPU
typora-root-url: ../../../
---

* toc
{:toc}

## 1. 引言

HIP程序在编译完后会得到一个host ELF文件。而device代码在编译完后会被bundle，作`.hip_fatbin section`嵌入在该ELF中。那么在运行host ELF之后，HIP是如何将host ELF携带的device代码传递给runtime的呢？它又是如何进行kernel launch的呢？本文将进行解答。

## 2. hip中main函数调用之前发生了什么？

考虑如下hip程序：

```cpp
#include <hip/hip_runtime.h>
#include <stdio.h>

__global__ void demoKernel(int *a) {
 *a += 1;
}

int main() {
 int x = 0;
 int *d_x;
 hipMalloc((void**)&d_x, sizeof(int));
 hipMemcpy(d_x, &x, sizeof(int), hipMemcpyHostToDevice);
 demoKernel<<<1,2>>>(d_x);
 hipMemcpy(&x, d_x, sizeof(int), hipMemcpyDeviceToHost);
}
```

使用以下命令得到host端的IR：

```bash
hipcc -S -emit-llvm --offload-host-only demo.cpp -o demo.ll
```

在IR中可以首先找到全局构造器**llvm.global_ctors:**

![img]({{ site.baseurl }}/Img/AMDGPU/HostLaunch_1.png)

构造器的相关说明可以查看[llvm语言手册](https://llvm.org/docs/LangRef.html#the-llvm-global-ctors-global-variable)。简单来说就是一个全局数组，每项保存一个(优先级，函数指针，关联数据或函数)的三元组，编译完后程序将按照优先级依次调用这些构造函数，全部执行完后才会调用main。c++一般有两种函数会被包含在该全局构造器中：

1. 用于给全局变量初始化的函数。
2. 带有**__attribute__((constructor))**属性的函数。

在本例中，只有**__hip_module_ctor**这一个构造函数。我们用llvm-extract从demo.ll中提取该函数，观察其依赖的数据变量以及代码逻辑：

```bash
/opt/rocm-5.4.6/llvm/bin/llvm-extract -func=__hip_module_ctor demo.ll -S -o hip_module_ctor.ll
```

   ![img]({{ site.baseurl }}/Img/AMDGPU/HostLaunch_2.png)

1. 首先明确下__hip_module_ctor依赖的几个全局变量以及runtime函数：

   - **@_Z10demoKernelPi**是一个void (i32*)的函数指针，在demo.ll中有其完整定义，它指向一个**__device_stub__demoKernel**的桩函数。该桩函数有以下两个作用：

      - 作为kernel函数的ID。**每个kernel都有唯一一个桩函数**，桩函数的命名规则为"__device_stub__"+kernel名。host在launch kernel时需要告知runtime执行哪个kernel，那么只需传入该指针作为kernel的唯一标识，runtime将查询该ID映射的内核函数，进而执行。
      - 作为kernel函数的启动接口。理论上在完成__hipPushCallConfiguration之后可以直接调用桩函数，由桩函数最后调用hipLaunchKernel进入runtime。但是实际上clang并没有用到做种方式，而是显示地在main中调用hipLaunchKernel。所以第二种作用其实用处不大。

   - **@0**定义了一个kernel名字符串数组。本例为：@0 = private unnamed_addr constant [17 x i8] c"_Z10demoKernelPi\00", align 1

   - **@__hip_fatbin_wrapper**定义了一个fatbin的描述变量。其IR定义在demo.ll中为：

      ![img]({{ site.baseurl }}/Img/AMDGPU/HostLaunch_3.png)

      这个描述变量的类型可以在runtime中找到：

      ![img]({{ site.baseurl }}/Img/AMDGPU/HostLaunch_4.png)

      我们重点关注其binary成员，该指针指向了__hip_fatbin这个符号，而这个符号就是.hip_fatbin section的起始地址。通过该指针可以访问嵌入在host中的device bundle文件。

   - **@__hip_gpubin_handle** 是一个FatBinaryInfo**类型的变量，在调用__hipRegisterFatBinary后，返回一个描述runtime内部保存fatbin信息的变量地址。对于host侧而言，我们只关心该指针是否是null，是null表示还没注册过fatbin，否则反之。__

2. 了解前面提到的变量含义之后，__hip_module_ctor的代码逻辑就非常简单了：

   1. 首先判断__hip_gpubin_handle指针是否为空，如果为空，则说明fatbin还没注册，调用__hipRegisterFatBinary。
   2. __hipRegisterFatBinary接收一个__CudaFatBinaryWrapper类型的变量，通过其中的binary指针找到.hip_fatbin bundle的位置，解析其中的所有kernel。最后返回一个FabBinaryInfo**的指针。
   3. 用步骤b返回的指针赋值给__hip_gpubin_handle，之后其他module的__hip_module_ctor就不需要再注册fatbin了。
   4. 最后，使用__hipRegisterFunction将host端的桩函数与runtime内部的kernel函数进行关联。

3. 关于这段注册逻辑，[hip编程手册](https://raw.githubusercontent.com/RadeonOpenCompute/ROCm/rocm-4.5.2/AMD_HIP_Programming_Guide.pdf)有一段精炼紧凑的描述，可以作为以上过程的一个小结：

   ![img]({{ site.baseurl }}/Img/AMDGPU/HostLaunch_5.png)

## 3. 用c++描述__hip_module_ctor的代码逻辑

在理解了demo.ll中各变量以及__hip_module_ctor的逻辑之后，我们可以将其用c++进行描述：

```c
// runtime api原型声明
hipError_t hipLaunchKernel(const void *functioin_address, dim3 numBlocks, dim3 dimBlock, void **args, size_t sharedMemBytes, hipStream_t stream);
hipError_t __hipPopCallConfiguration(dim3 *gridDim, dim3 *blockDim, size_t *sharedMem, hipStream_t *stream);
struct FatbinInfo;

struct __CudaFatBinaryWrapper {
    unsigned int magic;
    unsigned int version;
    void *binary;
    void *dummy1;
}

// device_stub
void __device_stub__demoKernel(int *args) {
    void **kernel_args = (void **)args;
    dim3 grid_dim, block_dim;
    size_t sharedMem;
    hipStream_t stream;
    __hipPopCallConfiguration(&grid_dim, &block_dim, &sharedMem, &stream);
    hipLaunchKernel(__device_stub__demoKernel, grid_dim, block_dim, sharedMem, stream);
    return;
}
    
void (*demoKernel)(int *) = __device_stub_demoKernel; // stub函数指针
char kernel_name[] = "_Z10demoKernelpi\00"; // kernel函数名
extern __attribute__((section(".hip_fatbin")) const char __hip_fatbin; // 引用位于.hip_fatbin section的__hip_fatbin符号
// 定义一个位于.hipFatBinSegement中的wrapper变量
static __attribute__((section(".hipFatBinSegment")) const struct __CudaFatBinaryWrapper __hip_fatbin_wrapper = {1212764230, 1, &__hip_fatbin, nullptr};
// 定义一个fatbin注册标记
__attribute__((selectany,visibility("hidden"))) FatbinInfo **__hip_gpubin_handle = nullptr;
// 构造函数
static __attribute__((constructor)) void __hip_module_ctor() {
    if (!__hip_gpubin_handle) {
        __hip_gpubin_handle = __hipRegisterFatBinary(&__hip_fatbin_wrapper);
    }
    __hipRegisterFunction(__hip_gpubin_handle, demoKernel, kernel_name, kernel_name, -1, nullptr, nullptr, nullptr, nullptr, nullptr); 
    atexit(__hip_module_dtor); // 在exit时调用__hip_module_dtor
}
```

对以上翻译代码做点说明：

- __hip_fatbin是链接时的一个内部符号，与之相似的有edata, etext标识数据段、代码段的结尾。参考[这个](https://linux.die.net/man/3/etext)。
- __hip_gpubin_handle被标记为comdat any，也就是允许多个编译单元本地有多个定义，链接时随便选取一个。
- 每个模块的__hip_module_ctor最后应该都位于llvm.global_ctors内，会串行执行，因此不用担心__hip_module_ctor会有并发问题。

## 4. main中的kernel launch

![img]({{ site.baseurl }}/Img/AMDGPU/HostLaunch_6.png)

一切是那么的朴素而自然，简单的一个__hipPushCallConfiguration，然后调用device_stub函数，桩函数的逻辑可查看第3节中的C++等价代码。

## 参考文献

1. [llvm 全局构造器](https://llvm.org/docs/LangRef.html#the-llvm-global-ctors-global-variable)
2. [device stub命名](https://reviews.llvm.org/D68578)
3. [HIP编译说明](https://github.com/ROCm/HIP/blob/develop/docs/developer_guide/build.md)
4. [HIP 编程手册](https://raw.githubusercontent.com/RadeonOpenCompute/ROCm/rocm-4.5.2/AMD_HIP_Programming_Guide.pdf)
5. [rocm runtime源代码仓库](https://github.com/ROCm/clr)
