---
title: SelectionDAGISel Pass的实现
category: LLVM
version: draft
---

[TOC]

# LLVM 中的指令选择过程（SelectionDAGISel）



- [ ] 指令选择所处编译流程中的阶段
- [ ] 指令选择的目标
- [ ] 基本实现方式
- [ ] LLVM的实现

## 前置知识

## 1. 什么是TargetLowering?

* `GetReturnInfo`
  1. `ComputeValueVTs`将LLVM的返回值类型分解成一系列`EVT`类型，如分解聚集类型(`aggregate type`)。
  2. 计算`EVT`合法化(`promote`,`expand`)之后的基本类型，结果记录到`OutputArg`数组中。

- [x] 为什么需要对`RET`进行`lowering`？

  LLVM IR中`RET`可以返回任何数据类型，如`struct`,`array`等聚集类型、非法位宽整型、向量类型等。这些类型最终都需要下降成目标机器支持的基本类型，以供CPU运算。换句话说就是变成寄存器运算支持的基本类型，如`i32，i64`。

- [ ] `LowerReturn`过程中如果返回值类型被合法化为少数几个基本类型，那么返回值是否会存放在寄存器中？

  按照System V ABI规定，X86-64的返回值只有一个，并位于`RAX`寄存器中。但是在LLVM中，返回值的一部分也可以位于`RAX,RDX,RCX`中，并在函数调用完成后直接使用`RDX,RCX`获取值，这种做法目前来看

- [ ] 接上一个问题，如果合法化后的数据类型太多，寄存器放不下，会触发LLVM中称之为`sret-demotion`的操作，这个操作的作用是什么？直观的猜想是将其放入栈中，是否如此？如果是的话怎么放？

## 2. SelectionDAGISel有哪些数据成员？

```cpp
TargetMachine &TM; // SelectionDAGISel的构建过程是Triple->Target->TargetMachine->PassConfig->SelectionDAGISel，所以在构造该pass时TM是已知的，可以直接传入。
const TargetLibraryInfo *LibInfo; // TODO!
std::unique_ptr<FunctionLoweringInfo> FuncInfo; // 将LLVM IR Function翻译为Machine Function IR的辅助函数，比较杂乱
SwiftErrorValueTracking *SwiftError; // TODO!
MachineFunction *MF; // MF由MachineFunctionPass根据Function IR自动创建
MachineRegisterInfo *RegInfo;
SelectionDAG *CurDAG; // SelectionDAG
std::unique_ptr<SelectionDAGBuilder> SDB; // 用于构建SDNode的工具类，核心功能是利用各种visit函数将IR转化为对应的SDNode
AAResults *AA = nullptr;
AssumptionCache *AC = nullptr;
GCFunctionInfo *GFI = nullptr;
CodeGenOpt::Level OptLevel;
const TargetInstrInfo *TII;
const TargetLowering *TLI;
bool FastISelFailed;
SmallPtrSet<const Instruction *, 4> ElidedArgCopyInstrs;

/// Current optimization remark emitter.
/// Used to report things like combines and FastISel failures.
std::unique_ptr<OptimizationRemarkEmitter> ORE;
```



## 实现过程

### 1. 构建SelectionDAG



### 2. 



#### The work flow of SelectionDAGIsel Pass

1. If we already selected that function, we do not need to run SDIsel.
2. If some phi nodes of this function contains side effect incoming values
(such as trap constant expression) and the edge is critical, split that edge.
3. Initialize SelectionDAG and `FunctionLoweringInfo`(FuncInfo::set).
4. Initialize SelectionDAGBuilder.
5. SelectAllBasicBlocks
    1. Intialize Function lowering info of CurrDAG.
    2. LowerArguments
        1. Lower argument types and set up the incoming argument description vector.(InputArg vector)
            1. Traverse every argument and lower them into register value type. 
                1. Lower Type into Value Types by calling ComputeValueVTs, in this step, aggregate Type are lowered into element type vectors.
                2. Compute ArgFlags to describe this **argument**
                3. Lower Value Type into Register Value Types, for every RegisterVT, create a InputArg in InputArg vector.
        3. Call the target to lower argument into real low level value like reg and stack frame.
            1. Assign locations to all of the incoming arguments. For every register type in step 5.2.2, try to assign a location(reg or stack) to them and record these info in CCState.
            2. Create corresponding SDValue for every assign locations. For register location, just create a virtual register node and a CopyFromReg node, while for memory location, we create a FrameIndex node and a Load node.
            3. Push every argument SDValue into InVals vector.
        4. Set up the argument values, which means binding every argument with its final SDValue.
            1. For every argument, we calculate its value types first, and for every value composed this argument, we assemble the specified legal parts into it. After every value is assembled, we create a MergeValue node to represent that argument.
            2. Bind this argument with its SDValue.
    3. Iterate over all basic blocks in the function.
        1. SelectBasicBlock.
            1. **Lower instructions which is in this basic block and not elided into SDValue.**
                1. Set up outgoing PHI node register values before emitting the terminator. Remember the virtual registers that need to be added to the Machine PHI nodes as input.
                2. Call corresponding visit function to lower this instruction.
            2. **CodeGenAndEmitDAG. Emit the lowered DAG as machine code.**
                1. Run the DAG combiner in pre-legalize mode.
                2. Legalize types1, hack on DAG until it only uses operations and types that the target supports.
                3. Run the DAG combiner in post-type-legalize mode.
                4. Legalize vectors.
                5. Legalize types2. hack on DAG until it only uses operations and types that the target supports.
                6. Run the DAG combiner in post-type-legalize mode.
                7. DAG legalization.
                8. Run the DAG combiner in post-legalize mode.
                9. Instruction select all of the operations to machine code, adding the code to the MachineBasicBlock.
                10. Schedule machine code.
                11. Emit machine code to BB. This can change 'BB' to the last block being inserted into.
        3. FinishBasicBlock.
6. `RegInfo->EmitLiveInCopies` emit the copies into the top of the block before emitting the code for the block if the first basic block in the function has live in ins that need to be copied into vregs.