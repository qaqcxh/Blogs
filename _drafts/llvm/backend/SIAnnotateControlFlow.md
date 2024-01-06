# SIAnnotateControlFlow

## 1. 引言

* IR在经过`StructurizeCFG`之后变成了结构化的CFG。其中的分支与循环都插入了合适的FLOW块以方便后续的执行掩码[^1]生成。

* 本Pass的作用是给结构化的控制流插入Intrinsic，以标识IF/THEN/ELSE/LOOPS等不同的基本块，方便后续给基本块生成exec_mask寄存器的控制指令。

  > Annotate the control flow with intrinsics so the backend can recognize if/then/else and loops.

## 2. 阅读目标

* **理解如何确定基本块的类型**。

* **了解如何处理不同类型的基本块**。

## 3. 确定基本块类型

### 3.1 类型

* 结构化CFG的基本块分别有IF/THEN BLOCK/ELSE/ELSE BLOCK/END_IF/IF_BREAK 6种基本块。见下图：

![hello](/home/qwqcxh/Blogs/Img/SIAnnotate.svg)

* 这些基本块中IF，THEN BLOCK与ELSE BLOCK是CFG的普通基本块，ELSE，END_IF与IF_BREAK属于**FLOW**基本块，是`StructurizeCFG`额外插入的控制基本块。需要注意一个基本块可能有多种类型。
* FLOW基本块**都位于栈中**，表示处于Open还未处理的控制流。出入栈的方式参见问题与解答。

### 3.2 判断

* THEN BLOCK与ELSE BLOCK可以根据分支类型判定，如果是**无条件跳转且不在栈中**，则属于这两种类型。
* END_IF位于栈中，有两种情形：
  1. 只有一个后继(无条件跳转)，则可马上判定。
  2. 有两个后继，说明END_IF同时也是一个IF，此时需要排除所有其他基本块后才能确定。
* 

## 4. 不同类型基本块的处理



## 3. 代码分析

* 本Pass是一个function pass，会dfs遍历函数内的基本块，对于每个基本块BB：
  1. 如果BB的块尾是无条件跳转的。
     1. 如果BB位于栈顶，则可判定是END_IF，`closeControlFlow`。
     2. 遍历下一个基本块。
  2. 如果BB的第二个后继已经被访问过： // 不可能是IF, ELSE
     1. 并且BB位于栈顶，则可判定是END_IF，`closeControlFlow`。
     2. 如果BB被它的第二个后继支配，则可判定是IF_BREAK，`handleLoop`。
     3. 遍历下一个基本块。
  3. 如果BB位于栈顶：
     1. 如果BB的分支条件来源与phi，并且判断是一个ELSE基本块，则`insertElse`，并遍历下一个。
     2. 否则判定为END_IF，`closeControlFlow`。
  4. 最后判定当前基本块是一个新的IF，`openIf`。



## 4. 问题与解答

- [ ] Pass中何时入栈与出栈，栈中的(BB, Value)对表示什么含义？

  入栈的case：

  1. 处理IF时，会将successor(1)入栈。该后继可能是END_IF或者ELSE。
  2. 处理ELSE时，会将successor(1)入栈。该后继是END_IF。

  出栈的case：

  1. 处理ELSE时，会出栈，得到IF的exec_mask。

## 注解

[^1]: AMDGPU使用exec_mask寄存器来屏蔽一个wavefront中不需要执行的线程。

