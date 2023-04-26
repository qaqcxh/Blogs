---
title: LLVM 寄存器信息描述
categories: LLVM
typora-root-url: ../../..
---

[toc]

# LLVM 寄存器信息描述

## MCRegister

`MCRegister`表示一个low level的寄存器，通常作为`MCInst`中的一个操作数。以我目前的理解，`MCRegister`对应一个物理寄存器。

实现上，表示寄存器只需要一个编号即可，所以`MCRegister`的数据成员就一个`unsigned`。编号与物理寄存器的对应关系是由`TableGen`决定的。

```cpp
using MCPhysReg = uint16_t; 

class MCRegister {
    unsigned Reg; // 寄存器编号，取值有如下的特殊含义
    // 0 不是一个寄存器
    // [1, 2^30) 表示物理寄存器
    // [2^30, 2^31）表示stack slot
    // [2^31, 2^32）表示虚拟寄存器
}
```

- [ ] `MCRegister`应该表示一个物理寄存器了，为什么注释中说它也能用于表示虚拟寄存器？

## MCRegisterClass

`MCRegisterClass`表示一个寄存器类，是具有某种共同特征或者作用于某个共同场合的寄存器集合，比如具有64位位宽的寄存器可以归为一类：

```cpp
// GR64 Register Class...
const MCPhysReg GR64[] = {
  X86::RAX, X86::RCX, X86::RDX, X86::RSI, X86::RDI, X86::R8, X86::R9, X86::R10, X86::R11, X86::RBX, X86::R14, X86::R15, X86::R12, X86::R13, X86::RBP, X86::RSP, X86::RIP, 
};
```

`MCRegisterClass`的数据成员如下：

```cpp
class MCRegisterClass {
public:
  using iterator = const MCPhysReg*;
  using const_iterator = const MCPhysReg*;

  const iterator RegsBegin;    // 指向类的成员数组，如GR64
  const uint8_t *const RegSet; // 包含的寄存器的位图，如GR64Bits
  const uint32_t NameIdx;      // 寄存器类名字在RegClassStrings数组的索引，如GR64对应57
  const uint16_t RegsSize;     // 该寄存器类包含的寄存器个数，如GR64中是17个
  const uint16_t RegSetSize;   // 包含寄存器位图的大小，如sizeof(GR64Bits)
  const uint16_t ID;           // 该寄存器类的ID，如X86::GR64RegClassID
  const uint16_t RegSizeInBits;// 该类中寄存器的比特数，如64
  const int8_t CopyCost;       // 在该寄存器类中寄存器复制的开销，如1
  const bool Allocatable;      // 其中的寄存器是否可分配，true or false
  ...
}
```

这些成员的含义已在注释中说明，我们以之前列举的`GR64 MCRegisterClass`为例进行解释，该寄存器类的定义在`X86MCRegisterClasses[X86::GR64RegClassID]`中，内容如下：
```cpp
{ GR64, GR64Bits, 57, 17, sizeof(GR64Bits), X86::GR64RegClassID, 64, 1, true },
```

1. `RegsBegin`指向该寄存器类的成员数组，此例中就是前面列举的`const MCPhysReg GR64[]`数组。`RegsSize`描述这个数组的大小，此例是17。

2. `RegSet`是一个位图，位图中第N位是1表示编号是N的物理寄存器存在；`RegSetSize`是该位图的大小，因为我们并不需要建立所有物理寄存器的位图，只需要能表示该`RegClass`中最大编号的物理寄存器即可。

3. `NameIdx`是该寄存器类的名字，只不过存的是名字在字符数组中的下标，可以在`X86RegClassStrings`中验证：

   ```cpp
   extern const char X86RegClassStrings[] = {
       ...
       /* 57 */ "GR64\0" //第57字节开始表示GR64的名字
   }
   ```

4. `ID`表示寄存器类的标识；`RegSizeInBits`表示其中的寄存器大小，显然`GR64`都是64位的寄存器；`CopyCost`表示`GR64`中寄存器互相复制的开销，这里显示只有1，表示开销很小；`Allocatable`表示该寄存器类是否可以用于寄存器分配，显然`GR64`都是可分配物理寄存器。（对应的控制寄存器是不可分配的）。

## MCRegisterDesc

`MCRegisterDesc`是**描述寄存器信息的核心类**。基本上一个寄存器的子寄存器(`SubRegs`)、超寄存器(`SuperRegs`)都由它描述。如果某个寄存器A的位域被另一个寄存器B完全覆盖，则可以称A是B的子寄存器或者说B是A的超寄存器。如X86中`AH,AL`是`AX`的子寄存器，`EAX,RAX`是`AX`的超寄存器。

`MCRegisterDesc`类在实现上是一个[POD](https://stackoverflow.com/questions/146452/what-are-pod-types-in-c)。数据成员如下：

```cpp
struct MCRegisterDesc {
  uint32_t Name;      // Printable name for the reg (for debugging)
  uint32_t SubRegs;   // Sub-register set, described above
  uint32_t SuperRegs; // Super-register set, described above

  // Offset into MCRI::SubRegIndices of a list of sub-register indices for each
  // sub-register in SubRegs.
  uint32_t SubRegIndices;

  // RegUnits - Points to the list of register units. The low 4 bits holds the
  // Scale, the high bits hold an offset into DiffLists. See MCRegUnitIterator.
  uint32_t RegUnits;

  /// Index into list with lane mask sequences. The sequence contains a lanemask
  /// for every register unit.
  uint16_t RegUnitLaneMasks;
};
```

我们以`RAX`这一寄存器为例来分析这些数据成员的含义。首先看一下有关`RAX`的`MCRegisterDesc`，其在`const MCRegisterDesc X86RegDesc[X86::RAX]`中的定义如下：

```cpp
{ 1229, 235, 2, 6, 1636, 8 }
```

*  可以看到`RAX`的`Name`是1229。获取一个寄存器实际的字符串名字是通过`MCRegisterInfo::getName`函数得到的：

    ```cpp
    const char *getName(MCRegister RegNo) const {
      return RegStrings + get(RegNo).Name;
    }
    ```

    所以`MCRegisterDesc::Name`保存的是`RegStrings`中的下标。在X86架构下，对应的是`X86RegStrings`，其在1229字节处的字符串恰好就是"RAX":

    ```cpp
    extern const char X86RegStrings[] = {
        ...
        /* 1229 */ "RAX\0"
    };
    ```

* `RAX`的`SubRegs`的值是235，查看下`MCSubRegIterator`这个迭代器是如何获取子寄存器值的。该迭代器的定义如下：

    ```cpp
    class MCSubRegIterator : public MCRegisterInfo::DiffListIterator {
    public:
      MCSubRegIterator(MCRegister Reg, const MCRegisterInfo *MCRI, bool IncludeSelf = false) {
        init(Reg, MCRI->DiffLists + MCRI->get(Reg).SubRegs); // 初始化差分链表
        if (!IncludeSelf) // 开始时指向自身，如果不需要包含自身则提前自增
          ++*this;
      }
    };
    ```

    在迭代器的构造函数中调用了`init`，并且使用`MCRegisterDesc::SubRegs`与一个`DiffLists`相加的结果做为`init`的一个参数。该`init`用于初始化`MCRegisterInfo::DiffListIterator`的`uint16_t Val`与`const MCPhysReg *List`两个成员：

    ```cpp
    void init(MCPhysReg InitVal, const MCPhysReg *DiffList) {
      Val = InitVal;
      List = DiffList;
    }
    ```

    迭代器的解引用操作符的定义的实现是直接返回`Val`:

    ```cpp
    MCRegister operator*() const { return Val; }
    ```

    所以获取子寄存器的值是通过读取`MCSubRegIterator::Val`来实现的。构造时，如果`IncludeSelf`为真，则迭代器的第一个初始值就是该寄存器本身。那么后面是如何获取下一个子寄存器并且是何时终止的呢？解答的关键在于迭代器的自增运算符，该算符的实现如下：

    ```CPP
    void operator++() {
      // The end of the list is encoded as a 0 differential.
      if (!advance())
        List = nullptr;
    }
    
    MCRegister advance() {
      assert(isValid() && "Cannot move off the end of the list.");
      MCPhysReg D = *List++;
      Val += D;
      return D;
    }
    
    ```

    自增运算符将功能委托给了`advance`函数，该函数将`*List`的值累加到`Val`中即获取到了下一个子寄存器，同时`List`自增指向链表的下一个元素，当`List`为空时迭代器即终止。

    现在我们可以知道子寄存器是如何获取的了：首先获取具体架构的`DiffLists`差分链表，该链表保存的是一组元素的差值。在X86下该链表是`X86RegDiffLists`。然后将`MCRegDesc::SubRegs`索引`DiffLists`得到对应的子寄存器差分链表。本例中对应的第235项如下：

    ```cpp
    extern const MCPhysReg X86RegDiffLists[] = {
        ...
        /* 235 */ 65507, 65517, 65535, 65535, 39, 0,
        ...
    }
    ```

    最后将原寄存器的编号累加即可获取最后的寄存器值[^1]：

    ```
    51=>22(51+65507)->3(22+65517)->2(3+65535)->1(2+65535)->40(1+39) 下一个0是结束标志，这一串对应的寄存器编号如下：
    RAX=>EAX->AX->AL->AH->HAX
    ```

* `RAX`的`SuperRegs`是2，其解析的方式和`SubRegs`是相同的，都是索引`X86RegDiffLists`，对应的表项是0，也就是没有超寄存器。

* 下一个数据成员是`SubRegIndices`,`RAX`中该成员的值是6。先看下`SubRegIndices`是如何解析的，然后再理解其含义。

  解析该成员的方法在`MCSubRegIndexIterator`迭代器中，先看下这个迭代器的实现方式：

  ```cpp
  class MCSubRegIndexIterator {
    MCSubRegIterator SRIter;  // 该SubRegIndex对应的SubReg
    const uint16_t *SRIndex;  // 当前SubRegIndex的地址，这里可以看到它是一个uint16
  
  public:
    /// Constructs an iterator that traverses subregisters and their
    /// associated subregister indices.
    MCSubRegIndexIterator(MCRegister Reg, const MCRegisterInfo *MCRI)
      : SRIter(Reg, MCRI) {
      SRIndex = MCRI->SubRegIndices + MCRI->get(Reg).SubRegIndices;
    }
  
    /// Returns current sub-register.
    MCRegister getSubReg() const {
      return *SRIter;
    }
  
    /// Returns sub-register index of the current sub-register.
    unsigned getSubRegIndex() const {
      return *SRIndex;
    }
  
    /// Returns true if this iterator is not yet at the end.
    bool isValid() const { return SRIter.isValid(); }
  
    /// Moves to the next position.
    void operator++() {
      ++SRIter;
      ++SRIndex;
    }
  };
  ```

  看完之后可以知道：

  1. `MCSubRegIndex`是关联一个`SubReg`的，其顺序和`SubReg`的顺序是一致的。并且这个迭代器的终止条件是根据`MCSubRegIterator`的终止条件实现。
  2. `MCSubRegIndex`是以`uint16_t`数组的形式存在，每次自增都会指向下一个元素。
  3. `MCRegisterDesc::SubRegIndices`保存的是一个偏移，通过加上`MCRegisterInfo::SubRegIndices`即可获得该寄存器的`SubRegIndex`链表。在X86架构下，`MCRegisterInfo::SubRegIndices`是`const uint16_t X86SubRegIdxLists[]`。

  现在可以解析`RAX`的`SubRegIndices`了，其值是6，找到`X86SubRegIdxLists`的第6项，提取长度为5(`IncludeSelf`为false时`RAX`的子寄存器有5个)的列表：

  ```cpp
  extern const uint16_t X86SubRegIdxLists[] = {
    ...
    /* 6 */ 6, 4, 1, 2, 5, 0,
    ...
  };
  ```
  
  由此可以得到`RAX`所有子寄存器与其`Index`的对应关系：
  
  ```
  (EAX->6),(AX->4),(AL->1),(AH->2),(HAX->5)
  ```

  我们可以写段小程序验证以上的推导：
  
  ```cpp
  #include "llvm/MC/MCRegisterInfo.h"
  #include "llvm/ADT/Triple.h"
  #include "llvm/Support/TargetSelect.h"
  #include "llvm/MC/TargetRegistry.h"
  #include "llvm/Target/TargetOptions.h"
  #include "llvm/Target/TargetMachine.h"
  #include <optional>
  #include <iostream>
  
  using namespace llvm;
  
  int main() {
      // 1. 获取X86的Target
      std::string TargetTriple("x86_64-unknown-linux-gnu");
      InitializeAllTargetInfos();
      InitializeAllTargets();
      InitializeAllTargetMCs();
      InitializeAllAsmParsers();
      InitializeAllAsmPrinters();
      std::string Error;
      auto Target = TargetRegistry::lookupTarget(TargetTriple, Error);
      if (!Target) {
          errs() << Error;
          return 1;
      }
  
      // 2. 创建X86的TargetMachine
      auto CPU = "generic";
      auto Features = "";
      TargetOptions opt;
      auto RM = Optional<Reloc::Model>();
      auto TM = Target->createTargetMachine(TargetTriple, CPU, Features, opt, RM);
  
      // 3. 获取MCRegisterInfo接口类，验证MCSubRegIndexIterator
      const MCRegisterInfo *MCRI = TM->getMCRegisterInfo();
      for(MCSubRegIndexIterator SRII(51, MCRI); SRII.isValid(); ++SRII) // !!! 这里51必须适配你llvm库中X86::RAX的值，可以在X86GenRegisterInfo.inc中找到，不同版本值可能不同
          std::cout << MCRI->getName(SRII.getSubReg()) << " idx:" << SRII.getSubRegIndex() << std::endl;
  }
  ```
  
  输出的结果如下：
  
  ![](/Img/TestMCSubRegIndexIterator.png)
  
  现在我们知道了每个子寄存器对应的`Index`，那么这个`Index`有什么含义呢？首先最容易想到的是作为子寄存器的标识，不同子寄存器的`Index`肯定不一样，其次通过查找`SubRegIndex`的使用情况，发现两个与`Index`相关的函数：
  
  ```cpp
  unsigned MCRegisterInfo::getSubRegIdxSize(unsigned Idx) const { // 获取子寄存器的大小
    assert(Idx && Idx < getNumSubRegIndices() &&
           "This is not a subregister index");
    return SubRegIdxRanges[Idx].Size;
  }
  
  unsigned MCRegisterInfo::getSubRegIdxOffset(unsigned Idx) const { // 获取子寄存器的偏移
    assert(Idx && Idx < getNumSubRegIndices() &&
           "This is not a subregister index");
    return SubRegIdxRanges[Idx].Offset;
  }
  ```
  
  这两个函数中`SubRegIndex`被用于索引`SubRegIdxRanges`数组，在X86架构下，这个数组是`X86SubRegIdxRanges`，其定义如下：
  
  ```cpp
  extern const MCRegisterInfo::SubRegCoveredBits X86SubRegIdxRanges[] = {
    { 65535, 65535 },
    { 0, 8 },	// sub_8bit
    { 8, 8 },	// sub_8bit_hi
    { 8, 8 },	// sub_8bit_hi_phony
    { 0, 16 },	// sub_16bit
    { 16, 16 },	// sub_16bit_hi
    { 0, 32 },	// sub_32bit
    { 0, 65535 },	// sub_mask_0
    { 65535, 65535 },	// sub_mask_1
    { 0, 128 },	// sub_xmm
    { 0, 256 },	// sub_ymm
  };
  ```
  
  这个数组中成员的类型是`SubRegCoveredBits`，是一个(offset, size)二元组，表示子寄存器的起始比特与长度。我们用`EAX`来验证下，它的`SubRegIndex`是6，对应{0, 32}，符合预期！所以`SubRegIndex`另外一种含义是其在`<target>SubRegIdxRanges`数组的下标。

* `RAX`的`RegUnit`是1636。为了理解该值所代表的意义，不妨看下`MCRegUnitIterator`是如何获取一个寄存器的`RegUnit`的：

    ```cpp
    class MCRegUnitIterator : public MCRegisterInfo::DiffListIterator {
    public:
      ...
      MCRegUnitIterator(MCRegister Reg, const MCRegisterInfo *MCRI) {
        assert(Reg && "Null register has no regunits");
        assert(MCRegister::isPhysicalRegister(Reg.id()));
        // Decode the RegUnits MCRegisterDesc field.
        unsigned RU = MCRI->get(Reg).RegUnits;
        unsigned Scale = RU & 15;
        unsigned Offset = RU >> 4;
    
        // Initialize the iterator to Reg * Scale, and the List pointer to
        // DiffLists + Offset.
        init(Reg * Scale, MCRI->DiffLists + Offset);
        advance();
      }
    
      MCRegUnitIterator &operator++() {
        MCRegisterInfo::DiffListIterator::operator++();
        return *this;
      }
    };
    ```

    这个迭代器和之前遇到的一样，都是一个初始值`Val`以及`List`构成的`DiffListIterator`。不过`MCRegisterDesc::RegUnit`不是一个简单的下标，而是由低四位的`Scale`以及高位的`Offset`组成，`Reg*Scale`得到`Val`，然后`Offset`索引`DiffLists`。按照之前的套路，`RAX`的1636(0x664)译码之后得到4的`Scale`与102的`Offset`，对应的表项如下：

    ```cpp
    extern const MCPhysReg X86RegDiffLists[] = {
      ...
      /* 102 */ 65332, 1, 14, 0,
      ...
    }
    ```

    对应的值是：

    ```
    0, 1, 15
    ```
    
    此时还并不知道这些值代表什么含义，那么可以搜索一下哪些地方会用到`MCRegUnitIterator`，看它们是如何用这个迭代器的结果的。可以观察到有如下一个函数片段：
    
    ```cpp
    for (RI = MCRegUnitIterator(Reg, MCRI); RI.isValid(); ++RI) {
      for (RRI = MCRegUnitRootIterator(*RI, MCRI); RRI.isValid(); ++RRI) {
          ...
      }
    }
    ```
    
    `MCRegUnitIterator`的解引用被`MCRegUnitRootIterator`直接使用了！在其中`RegUnit`被用作`RegUnitRoots`的索引：
    
    ```cpp
    MCRegUnitRootIterator(unsigned RegUnit, const MCRegisterInfo *MCRI) {
      assert(RegUnit < MCRI->getNumRegUnits() && "Invalid register unit");
      Reg0 = MCRI->RegUnitRoots[RegUnit][0];
      Reg1 = MCRI->RegUnitRoots[RegUnit][1];
    }
    ```
    
    在X86架构下，这个`RegUnitRoots`是`X86RegUnitRoots`，我们看它的第0、1、15项：
    
    ```cpp
    extern const MCPhysReg X86RegUnitRoots[][2] = {
      { X86::AH }, // 0
      { X86::AL }, // 1
      ...
      { X86::HAX }, // 15
    }
    ```
    
     分别对应`AH`,`AL`,`HAX`。
    
    可以在之前的验证代码中加入以下一段来看看推导结果：
    
    ```cpp
        for (MCRegUnitIterator RUI(51, MCRI); RUI.isValid(); ++RUI)
            for (MCRegUnitRootIterator RRUI(*RUI, MCRI); RRUI.isValid(); ++RRUI)
                std::cout << MCRI->getName(*RRUI) << std::endl;
    ```
    
    输出如下：
    
    ![](/Img/TestMCRegUnit.png)
    
    符合预期！所以`MCRegisterDesc::RegUnit`保存的是一个组合值，译码后用于索引`DiffLists`得到`RegUnit`，而`RegUnit`需要继续索引`RegUnitRoots`才能得到最后的`root reg unit`。这些所谓的`root reg unit`可以认为是**最小的子寄存器**，它们是组成其他寄存器的最小单元。

* 最后看一下`RAX`的`RegUnitLaneMask`这个数据成员，它的值是8。在说明它之前，先读下`LaneMask`的一段注释：

  > This is typically used to track liveness at sub-register granularity.
  > Lane masks for sub-register indices are similar to register units for
  > physical registers. The individual bits in a lane mask can't be assigned
  > any specific meaning. They can be used to check if two sub-register
  > indices overlap.                                                                         
  > Iff the target has a register such that:                                                                         
  >   getSubReg(Reg, A) overlaps getSubReg(Reg, B)                                                            
  > then:                                                               
  >   (getSubRegIndexLaneMask(A) & getSubRegIndexLaneMask(B)) != 0

  大概说的是`LaneMask`可保存元素之间的覆盖性，如果两个子寄存器有重叠，那么它们的`LaneMask`相与必然不为0，两者互为充要条件。有了这个理解之后再来解析这个8的含义，在`MCRegUnitMaskIterator`中有对`MCRegisterDesc::RegUnitLaneMask`的使用：
  
  ```cpp
  MCRegUnitMaskIterator(MCRegister Reg, const MCRegisterInfo *MCRI)
    : RUIter(Reg, MCRI) {
      uint16_t Idx = MCRI->get(Reg).RegUnitLaneMasks;
      MaskListIter = &MCRI->RegUnitMaskSequences[Idx]; // 用于索引RegUnitMaskSequences
  }
  ```
  
  这个`RegUnitLaneMasks`被用作`RegUnitMaskSequences`的下标，而该数组在X86下是`X86LaneMaskLists`，它的第8项是：
  
  ```cpp
  extern const LaneBitmask X86LaneMaskLists[] = {
    ...
    /* 8 */ LaneBitmask(0x0000000000000002), LaneBitmask(0x0000000000000001), LaneBitmask(0x0000000000000008), LaneBitmask::getAll(),
    ...
  }
  ```
  
  于是可以得到如下映射：
  
  | RegUnit | LaneBitmask |
  | :-----: | :---------: |
  |   AH    |     0x2     |
  |   AL    |     0x1     |
  |   HAX   |     0x8     |
  
  这几个`LaneBitmask`可以在`SubRegIndexLaneMaskTable`中找到对应的定义：
  ```cpp
  static const LaneBitmask SubRegIndexLaneMaskTable[] = {
    LaneBitmask::getAll(),
    LaneBitmask(0x0000000000000001), // sub_8bit
    LaneBitmask(0x0000000000000002), // sub_8bit_hi
    LaneBitmask(0x0000000000000004), // sub_8bit_hi_phony
    LaneBitmask(0x0000000000000007), // sub_16bit
    LaneBitmask(0x0000000000000008), // sub_16bit_hi
    LaneBitmask(0x000000000000000F), // sub_32bit
    LaneBitmask(0x0000000000000010), // sub_mask_0
    LaneBitmask(0x0000000000000020), // sub_mask_1
    LaneBitmask(0x0000000000000040), // sub_xmm
    LaneBitmask(0x0000000000000040), // sub_ymm
   };
  ```
  
  我们可以验证这个`LaneMaskTable`是否符合之前的定义，比如sub_16bit是7，它和sub_8bit(1)、sub_8bit_hi(2)都有重叠，而7与1和2相与都不为0，符合条件！

## MCRegisterInfo



- [x] `MCReigsterDesc`中`SubRegs`与`SuperRegs`是一个`uint32_t`，不是常见的容器类型，那么一个整数是如何表示寄存器集合呢？难道是一个表的索引？

  **解答：**`SubRegs、SuperRegs`是相对`<Target>RegDiffLists`的偏移，`DiffLists`一串差分表。给定一个初始值，可以通过累加差分表得到一列最终的结果表。

- [x] `SubRegIndicies`表示什么？

  **解答：**`SubRegIndicies`表示一组子寄存器，它是一个索引，是相对`<target>SubRegIdxLists`的偏移。`<target>SubRegIdxLists`中的元素表示一个具体的子寄存器位置，它也是一个索引，通过索引`SubRegIdxRanges`，可以确定子寄存器的起始比特以及长度。

- [x] `RegUnits`又是什么概念？

  **解答：**`RegUnits`是最小的子寄存器单元，任何其他子寄存器都可以认为是若干个`RegUnits`组成。`RegUnits`也是索引`DiffLists`，计算后的一组小表可以通过`<target>RegUnitRoots`解析得到。

- [ ] `RegUnitLaneMasks`是什么东西？

## TargetRegisterInfo



## X86TargetRegisterInfo



## 注解

[^1]: 不同LLVM版本某个架构下寄存器的编号有可能不同，比如本文中使用的RAX，LLVM-17中其值是51，但是在LLVM-15中是49，实际使用时应该用枚举常量。
