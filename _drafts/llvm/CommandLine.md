[toc]
# llvm CommandLine库的使用（其一）

当我们自己编写一个程序时，第一步是实现命令行参数的解析，将用户的输入转化成方便程序处理的值。有没有一种究级的命令行参数解析库来直接使用？这里推荐一个非常值得一试的C++库--llvm CommandLine库。

使用该库的理由有几点：

1. 该库使用非常简单。新定义一个选项时只需定义一个选项对象。并且选项对象可以存储解析的选项值，可以像操作普通数据类型一样操作选项对象。
2. 该库实现地非常优雅。它充分利用了C++元编程的优势，可以作为理解C++元编程的一个入门实例。
3. 该库扩展方便。当你想自定义数据类型来保存选项解析值时，只需自己实现一个`option parser`，并实现其中的`parse`函数解析选项值字符串即可。

## 快速使用导引

要使用CommandLine库，首先需要包含该库的头文件，然后运行时调用解析函数完成参数的解析。参数有哪些以及是何种类型等需要用户使用一种声明式语法来定义一个参数变量。

1. 包含如下头文件

   ```cpp
   #include "llvm/Support/CommandLine.h"
   ```

2. 在main函数的开始调用解析函数

   ```cpp
   int main(int argc, char **argv) {
     cl::ParseCommandLineOptions(argc, argv);
     ...
   }
   ```

3. 使用声明式语法来声明需要解析哪些选项，选项的种类是什么，比如我们想用一个`Unix`标准的输出参数`-o <filename>`可以这么声明：

   ```cpp
   cl::opt<string> OutputFilename("o", cl::desc("specify output filename"), cl::value_desc("filename"));
   ```

   在程序运行时加入`-help`得到对应的输出如下：

   ```cpp
   USAGE: program [options]
   
   OPTIONS:
     -h                - Alias for -help
     -help             - display available options (-help-hidden for more)
     -o <filename>     - Specify output filename
   ```

## 使用说明

* 选项中的`modifier`(如`cl::desc`)是顺序无关的，如下两种声明是等价的。

  ```cpp
  // the first way
  cl::opt<string> OutputFilename("o", cl::desc("specify output filename"), cl::value_desc("filename"))
  // the second way
  cl::opt<string> OutputFilename(cl::desc("specify output filename"), "o", cl::value_desc("filename"));
  ```

* 可以给选项提供**别名**，只需用`cl::alias`声明一个别名即可，`cl::alias`中需要有一个`cl::aliasopt modifier`声明别名的对象。

  ```cpp
  cl::opt<bool> Force ("f", cl::desc("Overwrite output files"));
  cl::opt<bool> Quiet ("quiet", cl::desc("Don't print informational messages")); // option for -quit
  cl::alias     QuietA("q", cl::desc("Alias for -quiet"), cl::aliasopt(Quiet));  // -q is an alias of -quit
  ```

* `enum`与`cl::values`配合可以限定选项值的可选值。假定我们需要设置一个选项用于表示特定优化级别，优化级别有`-g -O1 -O2 -O3`四种，可以这么声明选项：

  ```cpp
  enum OptLevel {
      Debug, O1, O2, O3
  };
  cl::opt<OptLevel> Optimization(cl::desc("specify optimization level"),
                                 cl::values(clEnumValN(Debug, "-g", "No optimization, enable debuging"),
                                            clEnumVal(O1, "Enable trival optimization"),
                                            clEnumVal(O2, "Enable default optimization"),
                                            clEnumVal(O3, "Enable expensive optimization")));
  ```

  如果同时指定同一选项的多个参数，那么以最后一个参数为准（可以认为逐个进行参数解析，最后一个覆盖了之前的值）。如下的优化级别最终是`-O1`。

  ```bash
  bash$ ./test -g -O2 -O1
  ```

  以上选项的声明在使用选项`-help`输出时的结果如下：

  ```text
  ...
  specify optimization level:
   -g     -No optimization, enable debuging
   -O1    -Enable trival optimization
   -O2    -Enable default optimization
   -O3    -Enable expensive optimization
  ...
  ```

  当然也可以给这些替代选项命名，在构造函数中添加名字属性即可。

* 可以使用`cl::list`对象来解析一系列选项，典型的例子是链接器`ld`可以跟任意个`.o`文件作为参数：

  ```cpp
  cl::list<std::string> InputFiles(cl::Positional, cl::OneOrMore)
  ```

* 同样可以使用`cl::bits`对象来解析选项列表，不同的是`cl::bits`适合做查询（本质上是一种bitmap构成的哈希），而`cl::list`适合做遍历，因此该类适合收集标记，如下是一个用`cl::bits`收集dump标记的使用样例：

  ```cpp
  enum DebugLev {
      nodebuginfo, quick, detailed
  };
  cl::opt<DebugLev> DebugLevel(
      cl::desc("Set the debuging level"),
      cl::values(clEnumValN(nodebuginfo, "none", "disable debug information"),
          clEnumVal(quick, "enable quick debug information"),
          clEnumVal(detailed, "enable detailed debug information"))
  );
  ```

* 可以使用`cl::OptionCategory`声明一个选项类型，然后在选项定义中添加`cl::cat`属性可以将同类型的选项聚合在一起方便查看。

## 选项属性与选项修饰

命令行库使用声明式的方式定义了选项值的数据类型，并且可在构造函数中加入任意的参数，这些参数可以分为两类：

1. 其一是用于说明选项内在特征的属性参数。如选项格式(像`cl::Positional`)、选项描述(`cl::desc`)、选项值描述(`cl::value_desc`)等。
2. 其二是用于控制选项解析、输出的修饰参数。如控制`-help`是否输出选项信息(`cl::Hidden`)、控制选项值是否必须(`cl::ValueRequired`)等。

~~没要非常清晰的准则划分参数是属性类还是修饰类。我个人的判断方式是：属性是命令行解析器解析时的**操作数**；修饰是**控制解析器操作方式**的控制参数。能不能区分没关系，查手册就好了。~~

选项属性（除名字属性外）使用时需要使用构造函数，而选项修饰仅仅只是一个标记。

### 选项属性（Option Attributes)

1. 名字属性声明了选项的名字是啥。手册上说明除了**位置选项**外其余选项都需要有名字。只需要在选项的构造函数中加入一个双引号声明的名字即可：

   ```cpp
   cl::opt<bool> Quiet("quiet");
   ```
    - [x] `enum`类型的选项名貌似是通过`cl::values`+`clEnumValue`实现的？

      **解答:**`enum`类型的选项是字面量类型，没有名字，`cl::values`中定义的名字是为了解析时匹配命令行参数使用的。

2. `cl::desc`属性描述了选项的信息，当程序使用`-help`时会输出该描述信息。

3. `cl::value_desc`属性则描述了选项值的信息。

4. `cl::init`属性声明了标量选项的初始值，如果没有声明本属性，则选项值由选项类型的默认构造函数确定。

5. `cl::location`属性声明了外部存储变量。

6. `cl::aliasopt`属性声明`cl::alias`选项是谁的别名。

7. `cl::values`属性声明由通用解析器使用的string-value映射，该属性接收一系列（option，value，description）构成的三元组。

8. `cl::multi_val`声明该选项有多个选项值，且选系那个值由该属性的参数决定：

   ```cpp
   cl::list<int> Triple("mytriple", cl::multi_val(3));
   ```

9. `cl::cat`声明该选项是属于哪个category。

10. `cl::callback`属性接收一个回调函数，每次命令行匹配一个该选项就调用该回调一次。

10. `cl::sub`属性声明该选项属于那个子命令。比如`git commit -h`，`git pull -h`中子命令是`commit`与`pull`。`h`是子命令的选项。

### 选项修饰（Option Modifiers)

选项修饰是`cl::opt`以及`cl::list`中用于控制选项如何解析以及`-help`如何输出的参数。可以分为如下6类：

1. 隐藏`-help`中某些选项的输出。

   * `cl::NotHidden`表明选项会出现在`-helpe`与`-help-hidden`的列表中。
   * `cl::Hidden`表明只会出现在`-helpr-hidden`中。
   * `cl::ReallyHidden`表明选项不应该出现在任何help输出中。

2. 控制选项允许以及要求出现的次数。

   本组修饰指明一个选项允许（或者要求）在命令行中出现的次数：

   * `cl::Optional`是`cl::opt`以及`cl::alias`的默认modifier，表明该选项是可选的，要么不出现，要么出现一次。这种情形最为常见。

   * `cl::ZeroOrMore`是`cl::list`的默认modifier，同样表明选项列表中可以有0个或者多个选项。

   * `cl::Required` modifier表明该选项必须出现一次。一般是位置选项，如程序必须有一个输入文件。

   * `cl::OneOrMore`表示该选项必须至少出现一次。这个一般声明在`cl::list`中。

   * `cl::ConsumeAfter`表示选项列表中最后一个位置选项后的所有参数列表。一个常见的例子是:

     ```bash
     bash$ bash my.sh -a -b -c
     ```

     如果选项按照如下方式声明，则该选项存储`-a,-b,-c`三个字符串：

     ```cpp
     cl::list<std::string> ScriptArgs(cl::ConsumeAfter)
     ```

3. 控制选项值是否必须指定。

   本组选项修饰指定一个选项是否要有选项值。在该库中，一个值要么由等号指定（如`-index-depth=17`)，要么由尾随字符串指定（如`-o a.out`）

   * `cl::ValueOptional`是布尔类型的默认modifier，表示该选项的值是可选的。要使能一个布尔变量，只需要有该选项或者`-foo=true`或者`-foo true且有cl::ValueRequired`。
   * `cl::ValueRequired`表示选项值是必须的，这也是除了[`匿名enum(unamed alternatives using generic parser)`](https://llvm.org/docs/CommandLine.html#id12)外大部分选项的默认值。该模式告知解析器如果一个选项后面没跟`=`,则其值在后一个参数中。
   * `cl::ValueDisallowd`表示该选项不能有值，这个是匿名`enum`类型选项的默认值。

4. 控制其他格式化选项。

   格式化选项组用于指定命令行选项具有一些与其他普通命令行参数不同的特殊功能，你只能使用其中的一个选项。

   * `cl::NormalFormatting`是所有选项的默认参数，表示普通的选项。
   * `cl::Positional`表示这是一个没有关联选项的位置参数。
   * `cl::ConsumeAfter`表示用于捕捉"解释器风格"的参数。
   * `cl::Prefix`表示这个选项可以作为值的前缀，如`-L/usr/lib`，`-DNAME=value`。

5. 控制选项分组。

   * `cl::Grouping`用于将多个选项聚合在一起（位置参数`cl::Positional`除外），实现类似一些Unix风格命令将多个选项拼接在一起的功能。如`ls -labF`，只使用一个`-`将4个选项聚在一起。

6. 混杂选项修饰。

   混在选项修饰是唯一可以混杂在一起的标志，

## 库实现细节

命令行库的实现是比较有意思的，要讲解它需要较多篇幅，因此会在另一篇博客中进行讲解。不过在讲解它实现之前，可以先看看下面一些问题。带着疑问看实现会更有效些。

### 问题导入

在使用该库的过程中，首先感受到的是库的易用性，其次就是伴随着一些实现相关的疑问，这些强大的功能是如何实现的？我的疑问主要记录在以下几点：

1. 参数是声明式提取，如下面这一条声明表示需要提取一个`-o`参数，但是实际解析确实运行时的，通过`ParseCommandLineOptions`来提取所有参数。那么在参数声明阶段做了啥操作呢？我能想到的解释是声明只是在某种数据结构内做了一些标记，表示存在这种参数，该参数有那些属性。然后在运行时解析阶段读取标记来进行匹配。

   ```cpp
   // 声明参数
   cl::opt<string> OutputFilename("o", cl::desc("Specify output filename"), cl::value_desc("filename"));
   
   // 解析参数
   int main(int argc, char **argv) {
     cl::ParseCommandLineOptions(argc, argv);
     ...
   }
   ```

2. 在使用声明式语法声明一些选项后怎么将其关闭？这一点在使用`CommandLine`库作为第三方库使用时颇为麻烦。考虑我们想自定义一些选项类似`-o <outputfile>`，当程序使用`-help`输出时我们只想输出这些自定义的选项信息，但是由于llvm的其他库在内部大量使用了其他编译相关的选项，导致输出极其膨胀：

   ```
   bash$ ./test -help
   USAGE: test [options] <input file>
   
   OPTIONS:
   
   Color Options:
       --color              -Use colors in output
   General Options:
       --arch64-neon-syntax=<value> -xxx
       ...
   ```

3. 选项值可选的选项是怎么确定选项值的？直观的实现方式是根据上下文来确定，如果选项后一个字符串不是一个选项名，那么就把它当该选项的选项值？

   

