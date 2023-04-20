---
title: TargetMachine的接口设计
category: LLVM
---

[toc]

# TargetMachine 的接口设计



## 疑问记录
- [x] 要在llvm支持一个后端架构，首先需要实现`TargetMachine`接口。然后将这个后端进行注册，那么注册是如何进行的？

  可以看看`llvm/MC/TargetRegistry`文件。

- [ ] `TargetMachine`与`Target`类是什么关系？从代码来看，`Target`包含`TargetMachine`的创建方法，是否意味着`Target`是更顶层的类，描述目标后端架构的顶层信息，而`TargetMachine`是其核心成员或者内部接口？
