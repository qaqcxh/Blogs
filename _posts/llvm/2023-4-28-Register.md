---
title: LLVM 寄存器信息描述
categories: LLVM
typora-root-url: ../../..
---

* toc
{:toc}

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
  
  ![]({{ site.baseurl }}/Img/TestMCSubRegIndexIterator.png)
  
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
    
    ![]({{ site.baseurl }}/Img/TestMCRegUnit.png)
    
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

`MCRegisterInfo`是整个`MCRegister`相关的**接口类**。它是对以上几个类的封装，所有对物理寄存器的访问都应该通过`MCRegisterInfo`的接口来使用。

1. `MCRegisterInfo`应该能得到任意物理寄存器的信息。所以内部有一个`MCRegisterDesc`的数组：

   ```cpp
   const MCRegisterDesc *Desc; // Pointer to the descriptor array
   unsigned NumRegs;           // Number of entries in the array
   ```

   该数组在X86下是通过`InitX86MCRegisterInfo`函数初始化的，两成员分别被赋值`X86RegDesc`与292。获取某个物理寄存器信息时，直接使用`operator []`或`get`函数即可：

   ```cpp
   const MCRegisterDesc &operator[](MCRegister RegNo) const {
     assert(RegNo < NumRegs &&
            "Attempting to access record for invalid register number!");
     return Desc[RegNo];
   }
   
   /// Provide a get method, equivalent to [], but more useful with a
   /// pointer to this object.
   const MCRegisterDesc &get(MCRegister RegNo) const {
     return operator[](RegNo);
   }
   ```

   此外，`MCRegisterInfo`提供了每个物理寄存器的名字数组，可以用`getName`方法获取名字：

   ```cpp
   const char *RegStrings;     // Pointer to the string table.
   
   /// Return the human-readable symbolic target-specific name for the
   /// specified physical register.
   const char *getName(MCRegister RegNo) const {
     return RegStrings + get(RegNo).Name;
   }
   ```

   在所有物理寄存器中，有两个特殊的寄存器被单独列举出来。一个是返回地址寄存器，另一个是程序计数寄存器：

   ```cpp
   MCRegister RAReg; // Return address register
   MCRegister PCReg; // Program counter register
   ```

   可直接调用对应的get方法进行获取。

2. `MCRegisterInfo`提供了对寄存器类的访问，在实现上所有寄存器类被保存在一个数组中。`MCRegisterInfo`提供了一个数组地址与大小来引用该数组：

   ```cpp
   const MCRegisterClass *Classes; // Pointer to the regclass array
   unsigned NumClasses;            // Number of entries in the array
   const char *RegClassStrings;    // Pointer to the class strings.
   ```

   这两成员在X86下被初始化为`X86MCRegisterClasses`与126。表明X86下共有126个寄存器类。

   同样地，寄存器类也有对应的名字数组与`get`方法：

   ```cpp
   const char *RegClassStrings;  // Pointer to the class strings.
   
   const char *getRegClassName(const MCRegisterClass *Class) const {
     return RegClassStrings + Class->NameIdx;
   }
   ```

3. 提供获取物理寄存器所包含的`RegUnit`的方法。回忆一下`RegUnit`是最小不可划分的子寄存器。它的值被编码在`DiffLists`差分表中，每个`RegUnit`需要再索引`RegUnitRoots`才能得到寄存器编号。

   ```cpp
   const MCPhysReg *DiffLists;  // Pointer to the difflists array
   const MCPhysReg (*RegUnitRoots)[2]; // Pointer to regunit root table.
   unsigned NumRegUnits;        // Number of regunits. also length of RegUnitRoots.
   ```

   以上数据成员分别被初始化为`X86RegDiffLists`，`X86RegUnitRoots`,173。

   要遍历一个物理寄存器所包含的`RegUnit`，LLVM提供了`MCRegUnitIterator`与`MCRegUntiRootIterator`跌代器供使用，典型的方式如下：

   ```cpp
   for (MCRegUnitIterator RUI(RegNo, MCRI); RUI.isValid(); ++RUI) {
       for (MCRegUnitRootIterator RURI(*RUI, MCRI); RURI.isValid(); ++RURI) {
           visit(*RURI); // *RURI即是regunit的物理寄存器
       }
   }
   ```
   此外，`MCRegisterInfo`中有一个`RegUnitMaskSequnce`指向`RegUnit`的`LaneBitmask`。获取一个物理寄存器的`LaneBitmask`也需要通过`MCRegUnitMaskIterator`迭代器遍历。

4. `MCRegisterInfo`也封装了对一个物理寄存器的`SubRegIndices`的访问。比如我们想得到某个子寄存器的起始比特与大小，需要先获得该`SubReg`的`Index`。与此相关的两个数据成员是`SubRegIndices`数组与`SubRegIdxRanges`数组：

   ```cpp
   const uint16_t *SubRegIndices; // Pointer to the subreg lookup array.
   unsigned NumSubRegIndices; // Number of subreg indices.
   const SubRegCoveredBits *SubRegIdxRanges; // Pointer to the subreg covered bit ranges array.
   ```

   `SubRegIndices`是一个索引表，布局没有特殊安排，由`TableGen`自动生成，被初始化为`X86SubRegIdxLists`。`SubRegIdxRanges`是每一个子寄存器的起止范围，被赋值为`X86SubRegIdxRanges`。

   访问它们同样需要使用迭代器，这里用的是`MCSubRegIndexIterator`。典型用法如下：

   ```cpp
   for (MCSubRegIndexIterator SRII(RegNo, MCRI); SRII.isValid(); ++SRII) {
       visit(SRII.getSubRegIndex, SRII.getSubReg...); // visit是你自己的函数
   }
   ```

5. - [ ] 最后是和`Dwarf`等进行寄存器映射的结构，暂时不准备看它的实现。

   ```cpp
   unsigned L2DwarfRegsSize;
   unsigned EHL2DwarfRegsSize;
   unsigned Dwarf2LRegsSize;
   unsigned EHDwarf2LRegsSize;
   const DwarfLLVMRegPair *L2DwarfRegs;  // LLVM to Dwarf regs mapping
   const DwarfLLVMRegPair *EHL2DwarfRegs;// LLVM to Dwarf regs mapping EH
   const DwarfLLVMRegPair *Dwarf2LRegs;  // Dwarf to LLVM regs mapping
   const DwarfLLVMRegPair *EHDwarf2LRegs;// Dwarf to LLVM regs mapping EH
   DenseMap<MCRegister, int> L2SEHRegs;  // LLVM to SEH regs mapping
   DenseMap<MCRegister, int> L2CVRegs;   // LLVM to CV regs mapping
   ```

## TargetRegisterClass

前面涉及的类都位于`MC`层，该层比较`low-level`，关注硬件相关的属性。而`TargetRegisterXXX`相关的寄存器类则与`CodeGen`相关，与ABI、寄存器分配算法等联系精密。所以理解这些类需要对LLVM后端代码生成比较熟悉了，下面将讲解一些实现相关的细节，有关**CodeGen接口设计**部分留待后续。

`TargetRegisterClass`是一个所有成员都是`public`的类，`X86`架构下每个寄存器类都有唯一对应的`TargetRegisterClass`。该类的数据成员如下：

```cpp
class TargetRegisterClass {
public:
  using iterator = const MCPhysReg *;
  using const_iterator = const MCPhysReg *;
  using sc_iterator = const TargetRegisterClass* const *;

  // Instance variables filled by tablegen, do not use!
  const MCRegisterClass *MC;
  const uint32_t *SubClassMask;
  const uint16_t *SuperRegIndices;
  const LaneBitmask LaneMask;
  /// Classes with a higher priority value are assigned first by register
  /// allocators using a greedy heuristic. The value is in the range [0,63].
  const uint8_t AllocationPriority;
  /// Whether the class supports two (or more) disjunct subregister indices.
  const bool HasDisjunctSubRegs;
  /// Whether a combination of subregisters can cover every register in the
  /// class. See also the CoveredBySubRegs description in Target.td.
  const bool CoveredBySubRegs;
  const sc_iterator SuperClasses;
  ArrayRef<MCPhysReg> (*OrderFunc)(const MachineFunction&);
  ...
}
```

这个类没有构造函数，所有成员都是`TableGen`显示初始化完成。以`GR8RegClass`为例对该类进行分析，其定义如下：

```cpp
extern const TargetRegisterClass GR8RegClass = {
  &X86MCRegisterClasses[GR8RegClassID],
  GR8SubClassMask,
  SuperRegIdxSeqs + 2,
  LaneBitmask(0x0000000000000001),
  0,
  false, /* HasDisjunctSubRegs */
  false, /* CoveredBySubRegs */
  NullRegClasses,
  GR8GetRawAllocationOrder
};
```

* 首先每个`TargetRegisterClass`对象需要知道该`RegClass`包含的寄存器成员。这可以直接利用`MCRegisterClass`提供的接口实现，因此其内部通过引入一个`MC`成员来指向对应的`MCRegisterClass`。此例是`X86MCRegisterClasses[GR8RegClassID]`。

* `TargetRegisterClass`的`SubClassMask`提供子类的访问功能。其实寄存器类就是一个集合，一个子类就是该集合的子集。要理解这个成员的实现，可以查看该成员的访问函数，在`getSubClassMask`函数中，有如下注释：

    > Returns a bit vector of subclasses, including this one. The vector is indexed by class IDs.
    >                                                                 
    > To use it, consider the returned array as a chunk of memory that contains an array of bits of size NumRegClasses. Each 32-bit chunk contains a bitset of the ID of the subclasses in big-endian style.
    >                                                                    
    > I.e., the representation of the memory from left to right at the bit level looks like:
    > [31 30 ... 1 0] [ 63 62 ... 33 32] ... [ XXX NumRegClasses NumRegClasses - 1 ... ]
    > Where the number represents the class ID and XXX bits that should be ignored.
    >                                                                    
    > See the implementation of hasSubClassEq for an example of how it
    > can be used.

  其中解释了`SubClassMask`的含义，它是一些`uint32_t`组成的位图，这些位图按照32位一组分割成块，每个块由低到高顺序编码。可以在`hasSubClassEq`函数中佐证这一点：
  
  ```cpp
  /// Returns true if RC is a sub-class of or equal to this class.
  bool hasSubClassEq(const TargetRegisterClass *RC) const {
    unsigned ID = RC->getID();
    return (SubClassMask[ID / 32] >> (ID % 32)) & 1;
  }
  ```
  
  `GR8RegClass`中该掩码是`GR8SubClassMask`，定义如下：
  
  ```cpp
  static const uint32_t GR8SubClassMask[] = {
    0x0000001d, 0x00000000, 0x00000000, 0x00000000, 
    0x400800c0, 0x3fc7fe5e, 0xdeed2e30, 0x00000fef, // sub_8bit
    0x00080000, 0x08c19a00, 0x1a000000, 0x00000463, // sub_8bit_hi
  };
  ```
  
  可以看到这个掩码数组所表示的比特位是超过X86寄存器类的个数的(119个)，4个`uint32_t`即可表示，即只有第一行表示子寄存器类，后两行的含义后续讲解。
  
  我们写一段小程序来遍历`GR8RegClass`的`SubClass`：
  
  ```cpp
  #include "llvm/TargetParser/Triple.h"
  #include "X86RegisterInfo.h"
  #include <iostream>
  
  #define GET_REGINFO_ENUM
  #include "X86GenRegisterInfo.inc"
  
  using namespace llvm;
  
  int main() {
      Triple X86Triple("x86_64-unknown-linux-gnu");
      X86RegisterInfo X86RI(X86Triple);
      const TargetRegisterClass *GR8RC = X86RI.getRegClass(X86::GR8RegClassID);
      for (BitMaskClassIterator BMCI(GR8RC->getSubClassMask(), X86RI);
           BMCI.isValid(); ++BMCI) {
          std::cout << X86RI.getRegClassName(X86RI.getRegClass(BMCI.getID()))
                    << std::endl;
      }
  }
  ```
  
  在*llvm-project/llvm/lib/Target/X86/CMakeLists.txt*中添加两行构建代码：
  
  ```cmake
  add_executable(TestTargetRegisterInfo <path_to_above_test_file>)
  target_link_libraries(TestTargetRegisterInfo LLVMX86CodeGen)
  ```
  
  然后编译运行得到如下输出：
  
  ![]({{ site.baseurl }}/Img/TestTargetSubClassMask.png)

  四个`SubRegClass`所包含的寄存器确实是`GR8RegClass`的子集，与掩码中第一个0x1d对应。

* 与`SubClass`对应，`TargetRegisterClass`提供访问`SuperClass`的成员及接口：

  ```cpp
  const sc_iterator SuperClasses;
  ```

  该成员实际是一个以0结尾的数组，遍历即可。

* `TargetRegisterClass`有一个与`SuperClass`名字接近但是意义完全不同的概念：`super-register class`，超寄存器类。与此相关的数据成员是`SuperRegIndices`，在`GR8RegClass`中该成员的值是`SuperRegIdxSeqs + 2`：

  ```cpp
  static const uint16_t SuperRegIdxSeqs[] = {
    ...
    /* 2 */ 1, 2, 0,
    ...
  };
  ```

  在代码中关于`SuperRegIndices`的注释如下：

  > Returns a 0-terminated list of sub-register indices that project some super-register class into this register class. The list has an entry for each Idx such that:                                                                          There exists SuperRC where:
  >     For all Reg in SuperRC:
  >       this->contains(Reg:Idx)

  翻译过来是说该成员指向一串0结尾的超寄存器类(super-register class)索引列表。每个索引对应的超寄存器类中，包含的寄存器在该索引下的子寄存器属于当前`TargetRegisterClass`。

  此例`GR8RegClass`对应的索引列表为1，2。那么如何得到每个索引对应的超寄存器类呢？

  此时需要查看`SuperRegClassIterator`超寄存器类迭代器的实现细节。其构造函数如下：

  ```cpp
  // Create a SuperRegClassIterator that visits all the super-register classes of RC. When IncludeSelf is set, also include the (0, sub-classes) entry.
  SuperRegClassIterator(const TargetRegisterClass *RC,
                        const TargetRegisterInfo *TRI,
                        bool IncludeSelf = false)
    : RCMaskWords((TRI->getNumRegClasses() + 31) / 32),
      Idx(RC->getSuperRegIndices()), Mask(RC->getSubClassMask()) {
    if (!IncludeSelf)
      ++*this;
  }
  ```

  1. `RCMaskWords`被初始化为4。联想到之前`SubClassMask`是一个以32bit为块的掩码数组，可以推出该变量是所有寄存器类位图所占用的块数。
  2. `Idx`被初始化为`SuperRegIndices`，也就是我们想分析的成员。
  3. `Mask`被初始化为`SubClassMask`。难道超寄存器类也被编码在`SubClassMask`中？

  接下来看下这个迭代器的自增算符是如何实现的：

  ```cpp
  /// Advance iterator to the next entry.
  void operator++() {
    assert(isValid() && "Cannot move iterator past end.");
    Mask += RCMaskWords; // Mask每次加4个块，也就是每次越过GR8SubClassMask的一样
    SubReg = *Idx++; // SubReg就是索引列表中当前被访问的索引
    if (!SubReg)
      Idx = nullptr;
  }
  ```

  好家伙，超寄存器类(super-register class)原来被编码到`SubClassMask`中了，`GR8SubClassMask`除了第一行是子类的位图掩码，其余两行表示的是超寄存器类的位图。

  用如下代码来读取这些编码是否是超寄存器类的位图：

  ```cpp
  #include "llvm/TargetParser/Triple.h"
  #include "X86RegisterInfo.h"
  #include <iostream>
  
  #define GET_REGINFO_ENUM
  #include "X86GenRegisterInfo.inc"
  
  using namespace llvm;
  
  int main() {
      Triple X86Triple("x86_64-unknown-linux-gnu");
      X86RegisterInfo X86RI(X86Triple);
  
      const TargetRegisterClass *GR8RC = X86RI.getRegClass(X86::GR8RegClassID);
      for (SuperRegClassIterator SRI(GR8RC, &X86RI); SRI.isValid(); ++SRI) {
          std::cout << "Index: " << SRI.getSubReg() << "\n";
          const uint32_t *SuperRegMask = SRI.getMask();
          for (BitMaskClassIterator BMCI(SuperRegMask, X86RI); BMCI.isValid();
               ++BMCI) {
              std::cout << X86RI.getRegClassName(X86RI.getRegClass(BMCI.getID()))
                        << std::endl;
          }
      }
  }
  ```

  输出结果如下：

  ```bash
  Index: 1 // 对应sub_8bit
  GR16
  GR16_NOREX
  GR16_ABCD
  LOW32_ADDR_ACCESS_RBP_with_sub_8bit
  ...
  Index: 2 // 对应sub_8bit_hi
  GR16_ABCD
  GR32_ABCD
  ...
  ```

  与预期一致！

* `TargetRegisterClass`含有一个该寄存器类的`LaneMask`，其值是所有包含的寄存器的所有子寄存器的`LaneMask`的并(就是一个或运算)。

  ```cpp
  const LaneBitmask LaneMask;
  
  /// Returns the combination of all lane masks of register in this class.
  /// The lane masks of the registers are the combination of all lane masks
  /// of their subregisters. Returns 1 if there are no subregisters.
  LaneBitmask getLaneMask() const {
    return LaneMask;
  }
  ```

* `CoveredBySubRegs`与`HasDisjunctSubRegs`是`TargetRegisterClass`的两个标记，分别表示该寄存器类是否能由所有子寄存器覆盖(如`AX`可以由`AH`,`AL`覆盖而`RAX`无法由`EAX`覆盖)以及是否有至少两个不覆盖的寄存器。

  ```cpp
  /// Whether the class supports two (or more) disjunct subregister indices.
  const bool HasDisjunctSubRegs;
  /// Whether a combination of subregisters can cover every register in the
  /// class. See also the CoveredBySubRegs description in Target.td.
  const bool CoveredBySubRegs;
  ```

  这两个标记的确切含义还没有通过具体使用的代码以及相关上下文进行求证，仅代表我当前按照注释理解的观点。

* `TargetRegisterClass`还有一个与寄存器分配优先级相关的成员，值越大优先级越高：

  ```cpp
  /// Classes with a higher priority value are assigned first by register
  /// allocators using a greedy heuristic. The value is in the range [0,63].
  const uint8_t AllocationPriority
  ```

## TargetRegisterInfo

- [ ] TODO

## X86TargetRegisterInfo

- [ ] TODO

## 注解

[^1]: 不同LLVM版本某个架构下寄存器的编号有可能不同，比如本文中使用的RAX，LLVM-17中其值是51，但是在LLVM-15中是49，实际使用时应该用枚举常量。
