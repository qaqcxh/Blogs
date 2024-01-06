---
title: AMDGPU 编译流程
categories: LLVM,AMDGPU
typora-root-url: ../../..
---

[toc]

# AMDGPU编译流程

## 引言

随着大模型的兴起，GPU架构凭借其并行性优势在模型训练以及推理中占据了重要地位。通过和CPU配合形成异构系统，可以充分利用异构系统的优势，提高系统的处理能力。为了适配异构系统的编程模型，自然就需要开发GPU设备上的编译技术。本文将以AMDGPU为例，浅述clang是如何将hip编译为异构系统上的执行文件。

## hipcc背后的秘密

在开发完一个hip(或者cuda)程序后，我们可以直接像编译普通c++程序那样编译hip程序，并且会得到一个看起来和普通ELF并无二致的执行文件，然后在GPU机器上执行该程序就可以进行计算。

以如下demo.cpp为例：

```cpp
#include <hip/hip_runtime.h>

__global__ void addOne(int *v) {
  *v += 1;
}

int main() {
  int h_v = 0;
  int *d_v;
  hipMalloc(&d_v, sizeof(int));
  hipMemcpy(d_v, &h_v, sizeof(int), hipMemcpyHostToDevice);
  addOne<<<1,1>>>(d_v);
  hipMemcpy(&h_v, d_v, sizeof(int), hipMemcpyDeviceToHost);
  printf("v is %d\n", h_v);
}
```

使用hipcc demo.cpp -o demo即可编译得到执行文件demo，运行后即可得到结果1：

```bash
$ hipcc demo.cpp -o demo
$ sudo ./demo
$ v is 1
```

实际上hipcc作为一个driver，隐藏了大量的编译细节，要探究实际的编译过程，我们可以添加-###选项：

```bash
hipcc -### demo.cpp -o demo
```

-###将把实际的编译命令输出到屏幕，上面的例子将得到以下输出：

![](/../Img/AMDGPU/AMDCompile_1.png)

图1：提取libbc-clang_rt.builtins供编译device代码时使用

![img](/../Img/AMDGPU/AMDCompile_2.png)

  图2：通过设置triple为amdgcn来编译device侧代码，得到demo-gfx908.o

![img](/../Img/AMDGPU/AMDCompile_3.png)

​                                         图3：操作同图1，提取得到的libbc-clang_rt.builtins供device侧obj文件链接使用

![img](/../Img/AMDGPU/AMDCompile_4.png)

​                                           图4：链接device测的obj文件demo-gfx908.o得到device侧可执行文件

![img](/../Img/AMDGPU/AMDCompile_5.png)

​                                          图5：使用clang-offload-bundler将device可执行文件封装成一个bundle(.hipfb)

![img](/../Img/AMDGPU/AMDCompile_6.png)

​        图6：通过设置triple为x86_64编译host侧代码，同时使用-fcuda-include-gpubinary接收device侧可执行文件(.hipfb)

![img](/../Img/AMDGPU/AMDCompile_7.png)

​                                          图7：最后将携带device binary的host obj文件链接为可执行文件

## 编译流程小结

1. hipcc首先unbundle提取libclang_rt.builtins中的bc字节码库
2. 然后通过设置triple，使用clang编译demo.cpp中device测的代码，编译过程中会连同device-lib以及1中的libbc-clang_rt.builtins一起编译。**得到device侧obj文件**

![img](/../Img/AMDGPU/AMDCompile_8.png)

3. hipcc再次unbundle提取libclang_rt.builtins中的bc字节码库（操作同1）

4. 使用lld将2得到的device obj以及3得到的libbc-clang_rt.builtin**链接成device侧可执行文件**。

![img](/../Img/AMDGPU/AMDCompile_9.png)

5. 使用clang-offload-bundler将devcie侧可执行文件**封装成一个bundle(.hipfb)**

6. 设置triple编译demo.cpp中host测的代码，编译过程中会接收5中的到的bundle，得到一个携带**device bundle的host obj文件。**

7. 使用lld将6中的host obj链接为最终的执行文件（携带device bundle)，device侧执行文件作为一个section放在host elf的只读段中。

![img](/../Img/AMDGPU/AMDCompile_10.png)

## 为什么需要将libclang_rt.builtins unbundle两次？

在第3节中我们可以看到步骤1和3都unbundle了libclang_rt.builins。使用md5sum可以发现unbundle得到的结果完全一致：

![img](/../Img/AMDGPU/AMDCompile_11.png)

那么hipcc没有复用第一次的结果可能是没有缓存之前的文件，第一次unbundle的结果随后会被移除。该猜测没有进行验证，如有谬误，欢迎指正。

## bundle与clang-offload-bundler

### 5.1 什么是clang-offload-bundler

对于异构的单源编程语言，工具链会实施多次编译得到host与device的code object，为了得到与传统非异构平台类似的单输出文件，可以使用clang-offload-bundler将host与若干device code objects捆束在一起。基本做法是将device(官方文档称为offload device)的code objects以数据的形式嵌入到host的code object。之后运行时runtime可以根据code objects的类型做相应的提取。

### 5.2 基本用法

clang-offload-bundler的使用非常简单，参考[官方文档](https://clang.llvm.org/docs/ClangOffloadBundler.html#usage)。

### 5.3 bundle文件格式

bundler可以将文本形式的输入捆束在一起，也可以将二进制形式的输入捆束在一起。但是不能将文本与二进制混合进行bundle。

1. 文本形式文件bundle之后的布局就是将每个输入用特定的注释包裹起来，然后拼接在一起。其BNF描述如下：

![img](/../Img/AMDGPU/AMDCompile_12.png)

假定我们有如下两个text文件

```cpp
#include <iostream>

int main() {
  std::cout << "I am text1" << std::endl;
}
#include <iostream>

int main() {
  std::cout << "I am text2" << std::endl;
}
```

使用如下命令将text1.ii与text2.ii bundle起来：

```bash
~/llvm-project/llvm/build/bin/clang-offload-bundler -type=ii -targets=host-x86_64-unknown-linux-gnu,hipv4-x86_64-unknown-linux-gnu -input=text1.ii -input=text2.ii -output=res.ii
```

得到的结果文件如下：

```cpp
// __CLANG_OFFLOAD_BUNDLE____START__ host-x86_64-unknown-linux-gnu-
#include <iostream>

int main() {
  std::cout << "I am text1" << std::endl;
}

// __CLANG_OFFLOAD_BUNDLE____END__ host-x86_64-unknown-linux-gnu-

// __CLANG_OFFLOAD_BUNDLE____START__ hipv4-x86_64-unknown-linux-gnu-
#include <iostream>

int main() {
  std::cout << "I am text2" << std::endl;
}

// __CLANG_OFFLOAD_BUNDLE____END__ hipv4-x86_64-unknown-linux-gnu-
```

1. 二进制形式的bundle文件布局如下：

![img](/../Img/AMDGPU/AMDCompile_13.png)

bundle文件首先是标识文件类型的magic string以及所包含的bundle项数，之后就是枚举每一项的4元组(代码对象偏移，代码对象大小，ID字符串长度，ID字符串）。在这段元数据信息之后就是一段对齐的填0字节，之后就是具体的二进制代码对象了。

以图5得到的fatbin为例，使用xxd得到其16进制如下：

![img](/../Img/AMDGPU/AMDCompile_14.png)

- 首先24字节是__CLANG_OFFLOAD_BUNDLE__的magic string。
- 之后8字节是0x2表示有2个bundle。
- 第一个bundle的前8字节是code object偏移，这里显示是0x1000也就是在第一个4k处。
- 第一个bundle的8-15字节是code object大小，为0
- 第一个bundle的16-23字节是bundle ID字符串长，这里是0x19，正好是后面25字节host-x86_64-unknown-linux的长度。
- 第二个bundle的前8字节是的偏移也是0x1000(因为第一个code object大小是0）
- 第二个bundle的8-15字节的大小是0x2538
- 第二个bundle的16-23字节表示的bundle ID串长是0x1f，表示hipv4-amdgcn-amd-amdhsa--gfx908的长度。
- 在第0x1000处刚好是device code object的起始，是一个AMD的ELF文件：

![img](/../Img/AMDGPU/AMDCompile_15.png)

## 参考文献

1. https://clang.llvm.org/docs/ClangOffloadBundler.html#creating-a-heterogeneous-device-archive
