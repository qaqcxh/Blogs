---
title: LLVM 后端架构之TargetMachine（其二）
category: LLVM
typora-root-url: ../../..
---

[toc]

# LLVM 后端架构之TargetMachine

摘要

## TargetMachine的设计与实现

* `TargetMachine`用户无需手动修改或继承，需要直接操作的是`LLVMTargetMachine`.
* 
* 

## 如何实现一个新的TargetMachine

1. 创建一个空的`TargetMachine`并将其注册到`Target`中。

2. 实现`TargetMachine`。

   * 首先需要将`MCAsmInfo`的创建函数注册到`Target`中，这由*LoongArchMiniTargetDesc.cpp*中的`LLVMInitializeLoongArchMiniTargetMC`函数完成注册。

   * 其次需要调用`MCAsmInfo`的创建函数，并将创建的结果保存到`TargetMachine`中，这一步是在`TargetMachine`的构造函数中由`initAsmInfo`完成的。但是`createMCAsmInfo`需要`MCRegisterInfo`作为参数，所以需要先实现`MCRegisterInfo`。

   * 实现`MCRegisterInfo`的td文件并注册。否则`TargetMachine`的`initAsmInfo`失败，无法构造`TargetMachine`。

   * 实现`MCInstrInfo`的td文件并注册。

   * 实现`MCSubTargetInfo`的td文件并注册。

3. 实现`TargetMachine->addPassesToEmitFile`中需要使用的`MC`接口：

   1. 在`LoongArchMiniTargetMachine`中实现`LoongArchMiniPassConfig`管理`CodeGen`需要用到的pass。
      1. 在`PassConfig`中实现`addInstSelector`接口，该接口将加入指令选择所需要的pass，一般是`SelectionDAGISel`的子类。
      2. 实现指令选择的pass--`LoongArchMiniDAGToDAGISel`。目前并不需要全部实现，可以只实现pass最基本的接口，`runOnMachineFunction`，以及pass的`ID`。另外还需要实现一个`Select`纯虚函数，这个函数直接调用默认生成的`SelectCode`即可。// 用list重新描述这段话
      3. 不出意外的话会出意外，此时编译会报`LoongArchMini::RET`未声明。这是一个`SDNode`的opcode，所以需要实现下，按照约定在相应文件(`LoongArchMiniISelLowering.h`)中定义即可。

   2. 添加`AsmPrinter`的支持，使得`MachineInstr`能下降到目标文件(.o)或者汇编文件(.s)。为此你需要先实现用于构造它的MC工具类：
      1. 实现`MCInstPrinter`并注册到`Target`中。
      2. 实现`MCCodeEmitter`。
      3. 实现`MCAsmBackend`。
      4. 实现`AsmStreamer`。
      5. 实现`AsmPrinter`。

4. 在`TargetMachine`中添加`TargetLoweringObjectFile`类，以及实现`getObjFileLowering`接口。

5. 现在基本具备了从IR到`MachineInstr`的框架了。但是使用命令：

   ```bash
   llc -march=loongarchmini ~/practice/llvm/test.ll -filetype=null -stop-after=loongarchmini-isel
   ```

   会显示：

   ```bash
   Running pass 'LoongArchMini DAG->DAG Pattern Instruction Selection' on function '@main' 出错
   ```

   原因在于`TargetMachine->getSubTargetImpl`没有实现，返回一个`nullptr`。所以你需要：

   * 实现`<YourTarget>SubTarget`类。
     - [ ] 实现`<Target>ABI` 
     
     - [ ] 实现`<Target>FrameLowering`
       - [x] 继承`TargetFrameLowering`基类，实现`<YourTarget>FrameLowering`。暂时只实现其中的纯虚函数使编译通过。
     
       - [ ] 
     
     - [ ] 实现`<Target>InstrInfo`
     - [ ] 实现`<Target>RegisterInfo`
     - [ ] 实现`<Target>TargetLowering`
       - [ ] 实现`bool CanLowerReturn`接口
     - [ ] 实现`SelectionDAGTargetInfo`
   * 实现`getSubTargetImpl`
   
     - [x] 1. 继承LLVMTargetMachine实现`<YourTarget>TargetMachine`
     - [x] 2. 在`<YourTarget>TargetMachine`中重载`getSubTargetImpl`函数



## 如何添加一条指令

1. 在td中加入指令描述。
2. 实现LoongArchMiniMCDisassembler类：
3. 注册MCInstPrinter:
   1. 在MCTargetDesc/LoongArchMiniMCTargetDesc.cpp中加入LoongArchMiniInstPrinter的create函数，并将其注册到TargetMachin。

## 疑问记录
- [x] 要在llvm支持一个后端架构，首先需要实现`Target`接口并注册，之后需要实现`TargetMachine`接口并将其注册到`Target`中，那么注册是如何进行的？

  **解答：**在*Target/LoongArchMini/LoongArchMiniTargetMahcine.h*文件的`LLVMInitializeLoongArchMiniTarget`函数内调用注册模板即可完成注册。

- [x] `TargetMachine`与`Target`类是什么关系？从代码来看，`Target`包含`TargetMachine`的创建方法，是否意味着`Target`是更顶层的类，描述目标后端架构的顶层信息，而`TargetMachine`是其核心成员或者内部接口？

  **解答：**是的，这个问题可以由[这篇博客]({% link _posts/llvm/2023-5-7-Target.md %})解答。

- [x] `Target`与`TargetMachine`中为什么都有`MCAsmInfo,MCRegisterInfo,MCInstrInfo`等MC信息？

  **解答：**`Target`中提供了这些特征类的构造函数，但是并没有提供保存这些构造结果的数据结构。事实上，这些类是保存在`TargetMachine`中的。

- [ ] `PassConfig`的作用是什么？应该是控制`CodeGen`过程中架构需要用到的pass，那么具体是哪些pass呢？另外`TargetPassConfig`作为默认的`PassConfig`是否是一个接口，每个架构都需要自定义实现自己的pass?

- [ ] 只定义一个`SelectionDAGISel`pass后为什么`llc`实际显示还添加了大量pass？这些pass是何时添加的，作用是什么？

- [ ] `Immutable Pass`一般不运行pass，而是提供一些数据信息供其他pass使用，那么其他pass如何使用`Immutable Pass`的信息呢？

- [ ] `TargetLowering`中的`LowerReturn`的实现：

  - [ ] 根据调用约定，在处理return时应该将返回值放入合适的位置


## 注解

