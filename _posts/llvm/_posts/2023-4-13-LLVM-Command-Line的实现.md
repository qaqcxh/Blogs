---
categories:
 - LLVM
---
## class opt的实现

### 基类opt_storage模板的介绍
* `opt_storage`是一个存储选项值的模板类，其模板参数如下：

  ```cpp
  template <class DataType, bool ExternalStorage, bool isClass>
  ```

  1. `DataType`是选项值的数据类型。
  2. `ExternalStorage`是选项值的存储方式（由本模板类存储or外部变量存储）。**false**时表示值存储在由`cl::location`指定的变量中。
  3. `isClass`是判断`DataType`是否是**class**的谓词。

     - [x] `isClass`影响什么？`DataType`是class或者是基本数据类型有和影响？

       **解答：**`isClass`控制了`opt_storage`的类定义，如果是类则采用继承的方式保存选项值，否则采用内部`DataType`基本变量保存选项值。

* `opt_storage`模板有一个主模板与两个特化：
  
    ```cpp
    template <class DataType, bool ExternalStorage, bool isClass>
    class opt_storage { ... } /// 主模板的定义，匹配ExternalStorage为true的情形
    
    template <class DataType> /// 特化1，匹配DataType是class的情形
    class opt_storage<DataType, false, true> : public DataType { ... }
    
    template <class DataType> /// 特化2，匹配DataType是基本数据类型的情形
    class opt_storage<DataType, false, false> : public DataType { ... }
    ```
    主模板类是采用外部变量存储选项值的类定义，两个特化是匹配采用内部存储选项值的类定义，其中特化1是数据类型为class的定义，特化2是普通数据类型的定义。
    
    * 采用内部存储，且选项数据类型是class的特化1的实现细节如下：
    
      1. 通过继承`DataType`来存储选项值：
    
         ```cpp
         template <class DataType>
         class opt_storage<DataType, false, true> : public DataType { ... }
         ```
    
      2. 封装`getValue()`与`setValue()`来操作选项值：
    
         ```cpp
         DataType &getValue() { return *this; } // Derived->Base的类型转换
         const DataType &getValue() const { return *this; } // 匹配const选项
         ```
    
         `getValue()`的实现比较直接，直接将`opt_storage`类型转换成基类就搞定了。并且实现了const与非const两个版本。
    
         ```cpp
         template <class T> void setValue(const T &V, bool initial = false) {
             DataType::operator=() { return *this; } // 调用DataType的赋值算子完成赋值操作
             if (initial)
                 Default = V; // 选项有默认值
         }
         ```
    
         `setValue()`的实现也很简单，直接调用基类的赋值运算符即可。因为`=`的右值可以为多个数据类型，所以这个函数是一个模板成员函数。
    
      3. 有些选项存在默认值，`opt_storage`内部有一个默认值存储变量以及默认值获取函数`getDefault`：
    
         ```cpp
         OptionValue<DataType> Default; // 默认值变量
         const OptionVal<DataType> &getDefault() const { return Default; } // 默认值获取函数
         ```
    
         可以看到`opt_storage`内部使用[`OptionValue<DataType>`](###OptionValue模板类的介绍)来存储默认值，但是比较奇怪的一点是在`DataType`是class的情形下，`Default`内部并没有一个实际的`DataType`存储默认值，其内部成员只是一些无用的接口函数：
    
         ```cpp
         // 从基类OptionValueBase<DataType, true>继承的成员
         bool hasValue() const { return false; } // 表示Default无默认值
         const DataType &getValue() const { llvm_unreachable("no default value"); } // 这里提示更加明显了
         ```
    
         从代码实例来看，对于选项值是类的`opt_storage`，其内部的`Default`只是一个占位的成员，并不存储实际内容。所以有一个疑问：
    
         - [ ] 为什么不让选项值是class的`opt_storage`含有默认值，理论上只要有构造函数就能设置默认选项值了？
    
    * 采用内部存储，且选项数据类型是基本数据的特化2的实现细节如下：
    
      1. 使用一个`DataType`成员来存储选项值：
    
         ```cpp
         template <class DataType>
         class opt_storage<DataType, false, false> {
         	DataType Value;
             ...
         }
         ```
    
      2. 与特化1一样的`getValue()`，`setValue()`实现。
    
      3. 与特化1一样的`OptionValue<DataType> default`，`getDefault()`定义与实现。
    
      4. 新增一个指针代理运算符函数：
    
         ```cpp
         DataType operator->() const { return Value; } // 如果DataType是指针类型，则直接返回该指针类型，提供*DataType内部成员的访问
         ```
    
    * 采用外部存储的主模板的实现细节如下：
    
      1. 使用一个指针指向实际保存选项值的变量：
      
         ```cpp
         template <class DataType, bool ExternalStorage, bool isClass>
         class opt_storage {
             DataType *Location = nullptr; // 指向实际保存的变量
             OptionValue<DataType> Default; // 默认值
             ...
         }
         ```
      
      2. 使用`setLocation(Option &O, DataType &L)`来设置保存变量：
      
         ```cpp
         bool setLocation(Option &O, DataType &L) {
             if (Location)
                 return O.error("cl::location(x) specified more than once!");
             Location = &L;
             Default = L; // 初始化默认值，如果DataType是类则
             return false; // 返回值true表示有问题，false表示正常
         }
         ```
      
      3. 使用与其他特化一样的机制通过`getValue()`，`setValue()`获取与设置值。
      
      4. 使用与特化2一样的方式获取默认值`getDefault()`。
      
      5. 设置了一个`DataType`类型转换算子：
      
         ```cpp
         operator DataType() const { return this->getValue(); }
         ```

* 小结一下`opt_storage`这个类可以在**内部或者外部**存储选项值，主要提供`getValue(),setValue()`与`getDefault()`三个API函数。

#### OptionValue模板类的介绍

OptionValue的类继承关系如下图所示：

<img src="/Img/OptionValue.svg"  />

##### GenericOptionValue 抽象基类

* 该类是一个抽象类，只定义了一个`compare`比较的接口：

  ```cpp
  virtual bool compare(const GenericOptionValue) = 0;
  ```

* 其余包含默认构造、默认拷贝构造、默认赋值、默认析构4个函数：

##### OptionValueCopy类

* 该类内部存放一个简单的选项值。包含一个`DataType`的值以及一个`Valid`标志：

  ```cpp
  DataType Value;
  bool Valid = false;
  ```

* 该类提供的接口包括：

  1. 默认构造、赋值与析构函数。

  2. 判断是否有选项值`hasValue`，设置选项值`setValue`以及获取选项值`getValue`:

     ```cpp
     bool hasValue() const { return Valid; }
     void setValue(const DataType &V) {
         Valid = true;
         Value = V;
     }
     const DataType& getValue() const {
         assert(Valid);
         return Value;
     }
     ```

  3. 实现`GenericOptionValue`的接口比较函数`compare`：

     ```cpp
     bool compare(const DataType &V) const { return Valid && (Value != V); }
     ```

##### OptionValueBase类

* `OptionValueBase`存在一个主模板与一个特化：

  ```cpp
  template <class DataType, bool isClass>
  struct OptionValueBase : public GenericOptionValue { ... } // 主模板，isClass为true时匹配该主模板，safely does nothing
  
  template <class DataType>
  struct OptionValueBase<DataType, false> : OptionValueCopy<DataType> { ... } // DataType非类时匹配该偏特化模板
  ```

* 主模板匹配**类**的选项类型，内部实现了`compare`，`hasValue`，`getValue`，`setValue`等接口，但是基本不做任何事情。
* 特化模板匹配**非类**的选项类型，内部直接继承`OptionValueCopy`的接口函数，这些接口直接操作`OptionValueCopy`内部的选项值。

##### OptionValue类

* `OptionValue`是选项值的顶层类，直接通过继承`OptionValueBase`进行实现：

  ```cpp
  template <class DataType>
  struct OptionValue final : OptionValueBase<DataType, std::is_class<DataType>::value> { ... }
  ```

* 显然`OptionValue`继承了`OptionValueBase`的接口`compare`，`hasValue`，`getValue`，`setValue`。这些接口根据`DataType`是否是类有不同的实现，如果是类基本就是啥都不干。这么设计的理由是？

* `OptionValue`除了继承的接口外，自己实现了一个不同类型的赋值函数模板：

  ```cpp
  // 一些选项从不同数据类型中获取值
  template <class DT> OptionValue<DataType> &operator=(const DT &V) {
      this->setValue(V);
      return *this;
  }
  ```


### 基类Option的介绍

该类存储与选项相关的所有属性与修饰内容，提供选型属性与选项修饰的设置与访问函数：

```cpp
private:
  uint16_t NumOccurrences; // The number of times specified
  // Occurrences, HiddenFlag, and Formatting are all enum types but to avoid
  // problems with signed enums in bitfields.
  uint16_t Occurrences : 3; // enum NumOccurrencesFlag
  // not using the enum type for 'Value' because zero is an implementation
  // detail representing the non-value
  uint16_t Value : 2;
  uint16_t HiddenFlag : 2; // enum OptionHidden
  uint16_t Formatting : 2; // enum FormattingFlags
  uint16_t Misc : 5;
  uint16_t FullyInitialized : 1; // Has addArgument been called?
  uint16_t Position;             // Position of last occurrence of the option
  uint16_t AdditionalVals;       // Greater than 0 for multi-valued option.

public:
  StringRef ArgStr;   // The argument string itself (ex: "help", "o")
  StringRef HelpStr;  // The descriptive text message for -help
  StringRef ValueStr; // String describing what the value of this option is
  SmallVector<OptionCategory *, 1>
      Categories;                    // The Categories this option belongs to
  SmallPtrSet<SubCommand *, 1> Subs; // The subcommands this option belongs to.
```

#### 功能类OptionCategory

该类用于描述一个选项类别。一个选项类别的标识是一个**名字**与**可选的描述**，另外由于`-help`能按照选项的`OptionCategory`来进行打印，因此解析器需要能遍历所有的`OptionCategory`，这就需要提供一个全局的数据结构供注册了。其核心成员如下：

```cpp
class OptionCategory {
StringRef const Name; // 类别名字
StringRef const Description; // 可选的描述

void registerCategory(); // 注册函数
...
}
```

注册的数据结构为`GlobalParser->RegisteredOptionCategories`。

#### 功能类SubCommand

`cl::SubCommand`用于声明一个逻辑独立的子命令种类。如果命令行参数可以从逻辑上分为若干个子功能，那么就可以声明多个`cl::SubCommand`，并将选项通过`cl::sub`属性进行归类。归类后就可以利用`cl::SubCommand`提供的功能接口判断是否有该子命令的选项、有哪些选项等。如下是一个例子：

```cpp
#include "llvm/Support/CommandLine.h"

using namespace llvm;

int main(int argc, char **argv) {
  cl::subcommand Add("add", "加法运算");
  cl::subcommand Sub("sub", "减法运算");

  cl::opt<int> Addend("addend", cl::desc("加数"), cl::sub(Add));
  cl::opt<int> Minuend("minuend", cl::desc("被减数"), cl::sub(Sub));
  cl::opt<int> Subtrahend("subtrahend", cl::desc("减数"), cl::sub(Sub));

  cl::ParseCommandLineOptions(argc, argv);

  if (Add) {
    // 执行加法运算
    int sum = Addend + cl::getSubcommand(Add)->getAdditionalArgsCount();
    // ...
  } else if (Sub) {
    // 执行减法运算
    int difference = Minuend - Subtrahend;
    // ...
  }

  return 0;
}
```

实现上，`cl::SubCommand`的成员可以划分为3类：

1. 子命令的标识，主要有**名字**与可选的**描述**两个成员：

   ```cpp
   StringRef Name;
   StringRef Decription;
   StringRef getName() const { return Name; }
   StringRef getDescription() const { return Description; }
   ```

2. 注册相关成员函数。解析器应该能知道所有的`cl::SubCommand`，因而在构造时需要将其注册到`GlobalParser`中。

   ```cpp
   void registerSubCommand();
   void unregisterSubCommand();
   ```

3. 子命令包含的选项成员：

   ```cpp
   SmallVector<Option *, 4> PositionalOpts;
   SmallVector<Option *, 4> SinkOpts;
   StringMap<Option *> OptionsMap;
   ```

## 一般修饰符（选项属性+选项修饰）的实现及应用

在`cl::opt`，`cl::list`等类的构造函数中出现的参数统称为修饰符(modifier)，在源代码层面都认为是modifier，但是在一些官方手册中会将其细分为`Option Attribute`与`Option Modifer`，这一点需要注意。`Option Attribute`修饰符是一些类，需要构造这些类来携带特定信息；而`Option Modifier`则是一类标记，所以从实现角度这样细分是合理的（虽然使用上并不需要关心。

### 选项属性

除了名字属性可以是一个普通字符串外，其余选项属性都是一个class。这些class都实现了一个`apply`接口：

```cpp
void apply(Option &O) const { O.xxx; } //将本属性的值apply到选项O上
```

#### `cl::desc`属性

描述选项信息的字符串，内部就是一个简单字符串：

```cpp
struct desc {
    StringRef Desc;
    desc(StringRef Str) : Desc(Str) {}
    void apply(Option &O) const { O.setDescription(Desc); }
}
```

#### `cl::value_desc`属性

描述选项值的字符串，实现同`cl::desc`：

```cpp
struct value_desc {
    StringRef Desc;
    value_desc(StringRef Str) : Desc(Str) {}
    void apply(Option &O) const { O.setValueStr(Desc); }
}
```

#### `cl::init`与`cl::list_init`属性

按照之前的描述，选项属性对应一个class，因此`cl::init`的结果必然是一个携带`apply`接口的类。一个直观的想法是将`cl::init`实现为一个模板类，但是类模板不具有模板参数推导能力，每次使用都得显示声明默认值的类型，像下面这样：

```cpp
cl::init<int>(10);
cl::init<double>(10.0);
cl::init<std::string>("-");
```

这样实现略显麻烦，由于模板函数可以进行参数推导，所以一种经典的做法是用模板函数封装模板类。实现方式如下：

1. 定义描述初始值的模板类，该类实现`apply`接口：

   ```cpp
   template <class Ty> struct initializer {
       const Ty &Init;
       initializer(const Ty &Val) : Init(Val) {}
       template <class Opt> void apply(Opt &O) const { O.setInitialValue(Init); }
   }
   ```

2. 使用模板函数封装该类，函数接收初始值并自动推导类型，构造并返回上面的模板类：

   ```cpp
   template <class Ty> initializer<Ty> init(const Ty &Val) {
       return initializer<Ty>(Val);
   }
   ```

`cl::list_init`与`cl::init`的实现类似，不同的是内部初始值是一个数组：
```cpp
template <class Ty> struct list_initializer {
    ArrayRef<Ty> Inits;
    list_initializer(ArrayRef<Ty> Vals) : Inits(Vals) {}
    template <class Opt> void apply(Opt &O) const { O.setInitialValues(Inits); }
}

template <class Ty> list_initializer<Ty> list_init(ArrayRef<Ty> Vals) {
    return list_initializer<Ty>(Vals);
}
```
#### `cl::location`属性

该属性允许用户定义外部变量存储选项值的解析结果，而不必保存在选项内部。该属性可根据外部变量自动推导类型，返回对应的属性类。因此也是通过模板类+模板函数实现：

1. 模板类为`LocationClass`，内部一个外部变量的引用，实现了`apply`接口：

   ```cpp
   template <class Ty> struct LocationClass {
       Ty &Loc;
       LocationClass(Ty &L) : Loc(L) {}
       template <class Opt> void apply(Opt &O) const { O.setLocation(O, Loc);}
   }
   ```

2. 模板函数如下：

   ```cpp
   template <class Ty> LocationClass<Ty> location(Ty &L) {
       return LocationClass<Ty>(L);
   }
   ```

#### `cl::cat`属性

该属性指定选项属于哪一个类别，因此该类的数据成员就是选项类别：

```cpp
struct cat {
    OptionCategory &Category;
    cat(OptionCategory &c) : Category(c) {}
    template<class Opt> void apply(Opt &O) const { O.}
}
```

#### `cl::sub`属性

该属性指定选项所属的子命令：

```cpp
struct sub {
    SubCommand &Sub;
    sub(SubCommand &S) : Sub(S) {}
    template <class Opt> void apply(Opt &O) const { O.addSubCommand(Sub); }
}
```

#### `cl::cb`属性

该属性指定一个选项相关的回调函数，每当解析其遇到一个该属性时就会调用一次注册的回调。为了自动推导输入的回调函数类型，该属性的实现也是利用模板类+模板函数的机制：

1. 实现一个保存回调函数的模板类：

   ```cpp
   template <typename R, typename Ty> struct cb {
       std::function<R(Ty)> CB;
       cb(std::function<R(Ty)> CB) : CB(CB) {}
       template <typename Opt> void apply(Opt &O) const { O.setCallBack(CB); }
   }
   ```

2. 实现模板函数做输入函数的类型推导，并生成对应的属性类：

   ```cpp
   template <typename F>
   cb<typename detail::callback_traits<F>::result_type,
      typename detail::callback_traits<F>::arg_type> // 模板参数依赖的名字
   callback(F CB) {
          using result_type = typename detail::callback_traits<F>::result_type;
          using arg_type = typename detail::callback_traits<F>::arg_type;
          return cb<result_type, arg_type>(CB);
      }
   ```

3. 上面用到的`callback_traits`实现如下：

   * 首先需要定义`callback_traits`的主模板：

     ```cpp
     template <typename F>
     struct callback_traits : public callback_traits<decltype(&F::operator()) {}
     ```

     主模板是暴露给用户使用的，提供一个函数类型就能提取返回类型与参数类型。要获取具体类型，常见的做法是用偏特化来匹配。

   * 偏特化实现如下：

     ```cpp
     template <typename R, typename C, typename... Args>
     struct callback_traits<R (C::*)(Args...) const> {
         using result_type = R;
         using arg_type = std::tuple_element_t<0, std::tuple<Args...>>;
     }
     ```

#### `cl::values`属性

有一类特殊的选项，选项本身表示一个特定的枚举值。这类选项称为字面量选项（手册有时也叫作alternative)。如llvm opt工具中可以手动指定要运行的优化：

```bash
bash$ opt -mem2reg -dce -instcombine hello.ll
```

上面的`mem2reg,dec,instcombine`就是字面量选项。要支持这类选项，需要在声明选项时加入`cl::values`属性表明字面量的取值范围。其实现及使用如下：

1. 需要一个类来实现属性类的`apply`接口并储存字面量值的信息，这个类由`ValueClass`实现：

   ```cpp
   class ValuesClass {
       SmallVector<OptionEnumValue, 4> Values;
   public:
       ValuesClass(std::initializer_list<OptionEnumValue> Options)
           : Values(Options) {}
       template <class Opt> void apply(Opt &O) const { // apply 接口
           for (const auto &Value : Values)
               O.getParser().addLiteralOption(Value.Name, Value.Value, Value.Description); // 保存在Opt内部的parser中，这些OptionEnumValue会被用于解析。
       }
   }
   
   struct OptionEnumValue {
       StringRef Name;
       int Value;
       StringRef Description;
   }
   ```

2. `ValuesClass`可接收任意数量的`OptionEnumValue`，但是需要用初始化列表构造，可以通过使用函数参数包来实现变长参数：

   ```cpp
   template <typename... OptsTy> ValuesClass values(OptsTy... Options) {
       return ValueClass({Options...});
   }
   ```

### 选项修饰标记

选项修饰标记只是一个枚举值，对选项标记的解析需要不同的处理（选项属性有对应的类，并提供了`apply`接口进行处理）。为了让修饰标记与属性类用相同的接口处理，可以用模板特化来分别实现：

1. 所有modifier都通过一个`applicator::opt`函数来解析。

2. 选项属性可以作为主模板来实现，调用属性类的`apply`接口完成选项解析：

   ```cpp
   template <class Mod> struct applicator {
       template <class Opt> static opt(const Mod &M, Opt &O) { M.apply(O); }
   }
   ```

3. 每个选项修饰标志都通过特化该主模板来分别实现：

   ```cpp
   // 处理选项名modifier
   template <unsigned n> struct applicator<char[n]> {
       template <class Opt> static opt(StringRef Str, Opt &O) {
           O.setArgStr(Str);
       }
   };
   // 处理NumOccurrences标记
   template <> struct applicator<NumOccurrencesFlag> {
       template <class OPt> static opt(NumOccurencesFlag N, Opt &O) {
           O.setNumOccurrencesFlag(N);
       }
   };
   ...
   ```

现在，所有modifier都可以通过实例化`applicator`并调用其中的`opt`方法来解析。要解析`cl::opt`等类中的可变构造函数参数，可以使用函数参数包进行实现：

```cpp
template <class Opt, class Mod, class... Mods>
void apply(Opt *O, const Mod &M, const Mods&... Ms) {
    applicator<Mod>::opt(M, O);
    apply(O, Ms...);
}

template <class Opt, class Mod> // 递归基础
void apply(Opt *O, const Mod &M) {
    applicator<MOd>::opt(M, O);
}
```

## class list的实现

### 基类list_storage的实现

`list_storage`是存储`cl::list`中同类型选项的容器。其目的与`opt_storage`一样保存选项值。且同样支持内部存储与外部存储的不同特化实现。

#### 外部存储主模板

* 默认的存储方式是外部存储，包含选项值数据类型`DataType`与容器数据类型`StorageClass`两个模板参数：

   ```cpp
   template <class DataType, class StorageClass> class list_storage { ... }
   ```

* `list_storage`内部数据成员一个是外部存储变量的指针、默认选项值数组以及一个默认赋值标记：

  ```cpp
  // 外部存储指针
  StorageClass *Location = nullptr;
  // 默认值数组
  std::vector<OptionValue<DataType>> Default = std::vector<OptionValue<DataType>>();
  // 是否是默认值标记
  bool DefaultAssigned = false;
  ```

* 成员函数主要包括设置外部变量、添加解析出的选项值、获取默认值等：

  ```cpp
  bool setLocation(Option &O, StorageClass &L) {
      ...
      Location = &L;
      ...
  }
  
  template <class T> void addValue(const T &V, bool initial = false) {
      Location->push_back(V);
      if (initial)
          Default.push_back(V);
  }
  ```

#### 内部存储特化模板

* 内部存储将`StorageClass`特化为了bool值，只有一个`DataType`模板参数：

  ```cpp
  template <class Sat
  ```

  

## class bits的实现

## option parser的实现

命令行解析器(`CommandLineParser`)会将命令行参数划分为空格分割的单词流(`Tokens`)，然后将单词匹配注册的选项名，如果匹配成功，则将**选项**及**选项值字符串**交给`option parser`解析。所以`option parser`的大致作用是解析选项值字符串，并将值保存到`parse`函数的输出参数中。

```cpp
// O - 解析器所在的选项
// ArgName - 选项名字符串
// Arg - 选项值字符串
// Val - 解析结果输出参数
bool parse(Option &O, StringRef ArgName, StringRef Arg, DataType &Val);
```



`option parser`也是一个模板类，先介绍主模板的实现，然后介绍几个普通数据类型的特化实现。

### `Option Parser`主模板

* 主模板继承了`generic_parser_base`中的一些接口，这些接口主要是一些打印相关的函数。

  ```cpp
  template <class DataType> class parser : public generic_parser_base { ... }
  ```

* `parser`主模板内记录了`DataType`类型的选项值数组，当用`cl::values`修饰某个选项时，该modifier内声明的候选值最终将被记录在该数据成员中：

  ```cpp
  SmallVector<OptionInfo, 8> Values;
  ```

  `OptionInfo`是描述字面量选项（`enum类型的选项`)的结构，包括名字，描述字符串以及字面量值：

  ```cpp
  class OptionInfo : public GenericOptionInfo { // 继承name, helpStr
  public:
      OptionInfo(StringRef name, DataType v, StringRef helpStr) : GenericOptionInfo(name, helpStr), V(v) {}
      OptionValue<DataType> V; //字面量值
  }
  ```

* `parser`主模板内实现的成员函数主要包括：

  1. `-help, -print-options`等相关的一些打印函数。

  2. 最核心的`parse`选项解析函数。

     该函数将选项值字符串与`Values`中注册的可选值进行比较，名字匹配返回字面量值即可。

     ```cpp
     bool parse(Option &O, StringRef ArgName, StringRef Arg, DataType &V) {
         StringRef ArgVal; // 命令行中该选项的选项值字符串
         if (Owner.hasArgStr()) ArgVal = Arg;
         else ArgVal = ArgName; // 字面量选项以枚举名当作名字
         for (size_t i = 0, e = Values.size(); i != e; ++i)
             if (Values[i].Name == ArgVal) { // 名字匹配
                 V = Values[i].V.getValue();
                 return false;
             }
         return O.error("...");
     }
     ```

  3. 辅助添加`cl::values`的映射表操作函数。

     ```cpp
     template <class DT>
     void addLiteralOption(StringRef Name, const DT &V, StringRef HelpStr) {
         OptionInfo X(Name, static_cast<DataType>(V), HelpStr);
         Values.push_back(X); // 在映射表中加入一项，扩冲选项值值域
         AddLiteralOption(Owner, Name); // 插入CommandLineParser的OptionsMap
     }
     
     void removeLiteralOption(StringRef Name) {
         unsigned N = findOption(Name);
         Values.erase(Values.begin() + N);
     }
     ```

### `Option Parser`特化模板

* 特化模板的基类是`basic_parser`，`basic_parser`的基类则是实现各种打印接口的函数。

  ```cpp
  template <class DataType> class basic_parser : public basic_parser_impl {
  public:
      using parser_data_type = DataType;
      using OptVal = OptionValue<DataType>;
      
      basic_parser(Option &O) : basic_parser_impl(O) {}
  }
  ```

* 特化模板类都是对普通数据类型进行特化，因此并不需要内部数据成员，解析函数的实现也比较直接，以`std::string`数据类型以及`int`数据类型为例：

  ```cpp
  template <> class parser : public basic_parser<std::string> {
  public:
      bool parse(Option &O, StringRef, StringRef Arg, std::string &Value) {
          Value = Arg.str();
          return false;
      }
      ...
  }
  
  template <> class parser : public basic_parser<int> {
  public:
      bool parse(Option &O, StringRef, StringRef Arg, std::string &Value) {
          if (Arg.getAsInteger(0, Value))
              return O.error("...");
          return false;
      }
  }
  ```

## 阅读疑问记录

- [ ] `cl::opt`会在一个全局的数据结构中记录以便解析，那么当`cl::opt`出作用域需要析构时析构函数是否会将它们从全局数据结构中删除？逻辑上最好删除，待求证。
- [ ] 如果自定义一个class用作`cl::opt`的数据类型，用默认的option parser需要定义
- [ ] 为什么`OptionValue<DataType>`在`DataType`是class的时候内部并不保存该类型的默认值？是有什么实现难点？要搞清楚的话得看默认值在代码哪些地方被使用，如果该值是自定义类会发生什么。
