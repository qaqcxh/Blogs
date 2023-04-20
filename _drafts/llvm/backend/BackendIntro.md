---
title: llvm 新增一个后端架构需要了解哪些类
categories: LLVM
---

[toc]

# llvm 新增一个后端架构需要了解哪些类

对于一个新手来说，在llvm后端新增加一个架构是一个不小的挑战。要实现这个目标，首先需要对后端涉及到的一些类有个大致了解。当然第一遍阅读肯定是粗糙的，你不可能一遍了解所有类的精确用途。但是我觉得可以尝试做到一下几点：

* **知道支持一个新架构需要enable哪些功能**。比如要对架构进行注册、需要提供类来描述CPU的特征、需要添加机器指令描述、需要设置调用约定、指令选择和匹配等。
* **了解要`enable`一个功能需要修改哪些类**。至少有个方向知道要看哪些类。
* 了解在实现一个类的接口**背后发生了什么**，**影响**了编译`pipeline`的哪个阶段，对整体编译的作用是啥。

另外llvm实现的类非常多，阅读的时候先抓核心基类，了解这个类的大致功能，等实际需要了解更多细节的时候再详细阅读。

## TargetMachine

> Primary interface to the complete machine description for the target machine.  All target-specific information should be accessible through this interface.

按照注释上的说明，这个类是描述目标机器的主接口，提供了所有架构相关的信息。那么我们可以想象一下哪些信息是架构相关的呢？

1. **指令信息**。指令是软硬件的接口，作为系统软件，最关注的应该就是硬件指令了。有指令描述才能做指令选择。
2. **寄存器信息**。汇编指令可以直接操作物理寄存器，对于编译器来说肯定需要知道机器架构有哪些物理寄存器，这样才可以做寄存器分配。
3. **硬件参数**。编译器需要知道一些详细的硬件参数才能做针对性地指令选择和调度。硬件参数包括微架构的一些特征，如发射宽度、功能部件数、指令延迟。
4. **存储模型**。如机器的大小端、硬件支持的基本数据类型。

以上是我简单想到的一些东西，那么实际`TargetMachine`描述了哪些信息呢？

1. 该架构的`DataLayout`。

2. 该架构的一些字符串标识：

   ```cpp
   Triple TargetTriple; // Triple string
   std::string TargetCPU; // CPU name
   std::string TargetFS; // target feature string
   ```

3. 汇编、寄存器、机器指令、子架构信息：

   ```cpp
   std::unique_ptr<const MCAsmInfo> AsmInfo;
   std::unique_ptr<const MCRegisterInfo> MRI;
   std::unique_ptr<const MCInstrInfo> MII;
   std::unique_ptr<const MCSubtargetInfo> STI;
   ```

4. 一些硬件功能选项以及其他后端代码生成的算法接口等：

   ```cpp
   const TargetOptions DefaultOptions;
   mutable TargetOptions Options;
   ...
   virtual bool
   addPassesToEmitFile(PassManagerBase &, raw_pwrite_stream &,
                       raw_pwrite_stream *, CodeGenFileType,
                       bool /*DisableVerify*/ = true,
                       MachineModuleInfoWrapperPass *MMIWP = nullptr) {
     return true;
   }
   
   virtual bool addPassesToEmitMC(PassManagerBase &, MCContext *&,
                                  raw_pwrite_stream &,
                                  bool /*DisableVerify*/ = true) {
     return true;
   }
   ...
   ```





## 构建系统

### 架构注册

1. 
