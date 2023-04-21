---
title: LLVM CMake 构建系统
categories:
  - LLVM
  - CMake
---

* toc
{:toc}

# LLVM CMake 构建系统

## CMake 基础知识

这里只列举看懂LLVM CMake文件必要的知识点，详细的文档请参考[官方手册](https://cmake.org/cmake/help/book/mastering-cmake/index.html)[^1]。

### 整体把握

1. 基于CMake的构建系统被组织为一系列逻辑上的`targets`，这和Make中的`target`很类似。所谓的`target`可以理解为希望构建出的东西，可以是一个执行文件，或者一个库，也可以是一组封装的命令。
   * `add_executable`可以创建一个二进制的`target`，会生成对应的执行文件。
   * `add_library`可以创建一个库的`target`，会产生对应的库。
   * `add_custom_target`可创建一个没有输出的`target`，与`makefile`中的`.PHONY`对应，常用于执行一些命令。
   
2. 每个有输出的`target`默认都会加入CMake的`all target`，也就是说生成makefile后，使用make命令将会构建这些目标。可以通过加入`EXCLUDE_FROM_ALL`选项来关闭该功能。

3. 编写`CMakeLists.txt`使用的是cmake语言，该语言由**注释**、**变量**与**命令**三部分组成。注释以'#'开始，变量通过`set`定义，命令则分宏、函数与基本内置命令。变量需要关注的是作用域，cmake只有`add_subdirectory`与函数调用这唯二的情形会创建新的作用域，每个变量的作用域从当前定义点开始，在其子作用域以及后续的空间发挥作用。子作用域可以重新定义同名变量覆盖父作用域，但是出了作用域又会恢复，除非加入`PARENT_SCOPE`选项，该选项的作用等同于在该函数调用的下一行定义该变量。以下是一个例子：

   ```cmake
   function(modify_test)
     set(test boo PARENT_SCOPE)
     message(${test}) # 输出"foo"
   endfunction()
   
   set(test foo) # 此刻开始test的值是"foo"
   message(${test}) # 输出"foo"
   modify_test()
   # 从此处开始test的值开始变为"boo"
   message(${test}) # 输出"boo"
   ```

   基本命令是CMake内部实现的，看手册就好；函数是用户自定义的命令，开辟了一个新的作用域；宏与函数类似，唯一的区别在于宏不开辟新的作用域。

4. `Modules`是CMake提供的用于支持代码重用的功能。模块也是一段普通的CMake命令，CMake可以通过查找`CMAKE_MODULE_PATH`指令的目录及其子目录查询对应的模块文件。模块分为查找模块（Find Modules）以及实用模块（Utility Modules）。查找模块支持`find_package`命令来确定某些软件库或者头文件的位置，提供一些结果变量或者描述函数等；实用模块是一些自定义函数的封装，用于提供一些特定功能。

3. 可以利用`try_compile`与`try_run`命令执行系统环境的检测。基本思路是写一小段测试程序然后用编译器编译或者运行，然后CMake可以根据编译/运行结果置一些变量，以此来判断系统是否支持某些功能。可以参考[这篇文档](https://cmake.org/cmake/help/book/mastering-cmake/chapter/System%20Inspection.html#system-properties)。

### 术语&命令

* `project`：设置项目名称、版本以及需要用到的语言等。该命令也会置几个`project`名字、目录相关的变量。
* `include`：加载并运行某个文件或者模块中的CMake代码。和高级语言中的`include`类似，只是将对应文件的代码包含到调用处。该命令首先搜索CMake的内置模块目录，失败则查询变量`CMAKE_MODULE_PATH`指定的目录。
* `set`：设置一个变量为给定值。CMake的变量类型分为普通变量，`Cache`变量以及环境变量。其中`Cache`变量具有全局作用域，可以在项目的任意文件中访问到。
* `unset`：取消一个变量。
* `find_program`：从某些目录中查找给定名字的执行程序，这些目录包括1)一些特定的CMake变量指定的目录。如`CMAKE_PROGRAM_PATH`、`CMAKE_PREFIX_PATH`对应路径下的bin或者sbin子目录等。2)某些环境变量指定的目录。3)当前命令的`PATHS`选项指定的目录。4)系统的标准环境变量，如`PATH`。

- `option`：提供一个用于可选的bool值选项。使用语法为：

  ```cmake
  option(<variable> "help_text" [value])
  ```

  如果`<variable>`已经被设置，则该命令什么都不做；否则定义一个bool变量，其值默认为OFF，可由`value`覆盖。

* `list`：列表操作命令，分为列表读取、查找、修改与排序四大子命令。
* `string`：字符串操作命令，具体看手册。
* `file`：文件操作命令。提供文件内容的读写、文件系统处理、路径转换、文件传输等功能。

- `add_compile_definitions`：将预处理定义加入编译器的命令行，其语法如下：

  ```cmake
  add_compile_definitions(<definition> ...)
  ```

  其中`<definition>`是`VAR`或者`VAR=XXX`的形式，在编译命令行中分别对应`-DVAR`以及`-DVAR=XXX`的选项。

* `find_package`：在一些目录中查找特定的模块并包含到当前文件位置执行。该命令有多种模式，在Module Mode下，会查找CMake安装时提供的内置模块路径以及`CMAKE_MODULE_PATH`指定的路径。

* `execute_process`：执行若干个子进程，可以接收多个命令选项，每个选项都会创建一个进程运行。如果指定了`RESULT_VARIABLE`和`OUTPUT_VARIABLE，则会分别记录进程的退出条件以及标准输出。

* `configure_file`：复制一个模板文件的内容，并根据当前CMake的变量值来修改模板的内容，结果放到目标文件。典型地，模板中的`@VAR@`以及`${VAR}`（没有加`@ONLY`）会被展开成变量的值；`#cmakedefine VAR`会根据`VAR`是否在CMake中定义了而被展开成`#define VAR`或者被注释；`#cmakedefine01 VAR`类似地会被展开成`#define VAR 1`或者`#define VAR 0`。

* `set_property`：在给定作用域内设置一个命名属性。这个命令在LLVM CMake中非常常见，典型地用法是设置一些`target`的属性，最后构建`target`时会读取属性来设置一些构建命令。值得一提的是在LLVM的CMake中引入的`component`这一概念就是一个普通的`target`附带了`LLVM_COMPONENT_NAME`、`LLVM_LINK_COMPONETS`与`COMPONENT_HAS_JIT`三个属性。

  > An LLVM component is a cmake target with the following cmake properties
  > eventually set:
  >
  >   - LLVM_COMPONENT_NAME: the name of the component, which can be the name of
  >     the associated library or the one specified through COMPONENT_NAME
  >   - LLVM_LINK_COMPONENTS: a list of component this component depends on
  >   - COMPONENT_HAS_JIT: (only for group component) whether this target group
  >     supports JIT compilation
  >     Additionnaly, the ADD_TO_COMPONENT <component> option make it possible to add this
  >     component to the LLVM_LINK_COMPONENTS of <component>.

* `get_property`：从给定作用域中获取某个属性的值，并将值保存到特定变量中。

*  - [ ] `install`：下次一定

## CMakeLists.txt分析

以`llvm-project/llvm/CMakeLists.txt`为例子进行分析：

1. 首先定义项目名、项目的版本号以及项目的语言：

   ```cmake
   project(LLVM
     VERSION ${LLVM_VERSION_MAJOR}.${LLVM_VERSION_MINOR}.${LLVM_VERSION_PATCH}
     LANGUAGES C CXX ASM)
   ```
   在我阅读的这个版本中，`LLVM_VERSION_MAJOR`、`LLVM_VERSION_MINOR`以及`LLVM_VERSION_PATCH`分别被设置为17,0,0。

2. 包含`GNUInstallDirs`模块以获取一些在GNU系统下的安装目录：

   ```cmake
   include(GNUInstallDirs)
   ```

   `GNUInstallDirs.cmake`模块是CMake内置的模块，随CMake的安装而自动安装，可以通过`locate GNUInstallDirs`命令获取本机上该模块的安装路径。

   ```bash
   $ locate GNUInstallDirs
   ...
   /usr/share/cmake/Modules/GNUInstallDirs.cmake
   ```

   在`GNUInstallDirs.cmake`的文件头有该模块的说明：

   >Define GNU standard installation directories
   >
   >Provides install directory variables as defined by the
   >`GNU Coding Standards`_.
   >
   >Result Variables
   >^^^^^^^^^^^^^^^^
   >
   >Inclusion of this module defines the following variables:
   >
   >``CMAKE_INSTALL_<dir>``
   >
   >``CMAKE_INSTALL_FULL_<dir>``
   >
   >where ``<dir>`` is one of:
   >
   >``BINDIR``
   >  user executables (``bin``)
   >``SBINDIR``
   >  system admin executables (``sbin``)
   >...

   文件的描述非常清晰，该模块提供了GNU的标准安装路径，并定义了一些结果变量供使用。

3. 检测CMake命令是否指令了`CMAKE_BUILD_TYPE`。LLVM要求构建时必须传入`-DCMAKE_BUILD_TYPE=<type>`，否则构建系统拒绝执行：

    ```cmake
      if (NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
        message(FATAL_ERROR "
      No build type selected. You need to pass -DCMAKE_BUILD_TYPE=<type> in order to configure LLVM.
        ...")
      endif()
    ```

4. 检测`LLVM_ENABLE_PROJECTS`与`LLVM_ENABLE_RUNTIMES`是否是合法的架构与运行时库。

5. 添加LLVM自定义的模块搜索路径(设置`CMAKE_MODULE_PATH`），之后即可包含这些目录下的模块：

   ```cmake
   list(INSERT CMAKE_MODULE_PATH 0
     "${CMAKE_CURRENT_SOURCE_DIR}/cmake"
     "${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules"
     "${LLVM_COMMON_CMAKE_UTILS}/Modules"
     )
   ```

6. 包含`CPack`打包模块。有关`CPack`的部分还没看，有兴趣的可自行参考[相关文档](https://cmake.org/cmake/help/book/mastering-cmake/chapter/Packaging%20With%20CPack.html)。

7. 定义`CMAKE_INSTALL_PACKAGEDIR`变量。这个变量的定义在`GNUInstallPackageDir`模块中，包含该模块即可获取该变量：

   ```cmake
   include(GNUInstallPackageDir)
   ```

   `CMAKE_INSTALL_PACKAGEDIR`是一个路径变量，变量默认值是lib/cmake。

8. 定义了一个`LLVM_ALL_TARGETS`变量表示支持的所有后端架构，如果要新增一个后端架构，可以在其中新增架构名：

    ```cmake
    set(LLVM_ALL_TARGETS
      AArch64
      AMDGPU
      ...
      X86
      XCore
      )
    ```

9. 定义了一个`LLVM_TARGETS_TO_BUILD`变量表示需要构建的后端架构，默认是所有架构，也可以在cmake命令行中手动加入`-DLLVM_TARGETS_TO_BUILD=<needed targets>`来避免所有架构的编译：

    ```cmake
    set(LLVM_TARGETS_TO_BUILD "all"
    CACHE STRING "Semicolon-separated list of targets to build, or \"all\".")
    
    if( LLVM_TARGETS_TO_BUILD STREQUAL "all" )
    set( LLVM_TARGETS_TO_BUILD ${LLVM_ALL_TARGETS} )
    endif()
    ```

10. 查找`FindPython3`模块获取组件`Interpreter`，并将其包含到当前文件：

    ```cmake
    find_package(Python3 ${LLVM_MINIMUM_PYTHON_VERSION} REQUIRED
        COMPONENTS Interpreter)
    ```

    - [ ] CMake中的`COMPONET`表示什么？

11. 包含模块`config-ix`实施系统的配置检查，执行LLVM的脚本`config.guess`获取主机的triple（位于`LLVM_HOST_TRIPLE`变量中）：

    ```cmake
    include(config-ix)
    ```

12. 设置变量`LLVM_TARGET_TRIPLE`。除非`LLVM_TARGETS_TO_BUILD=native`，即native就是target，此时`LLVM_TARGET_TRIPLE`与`LLVM_NATIVE_TRIPLE`一样。否则该变量为空。

    ```cmake
    include(SetTargetTriple)
    set_llvm_target_triple()
    ```

13. 根据要构建的架构(`LLVM_TARGETS_TO_BUILD`)配置一些架构相关的变量：

    ```cmake
    set(LLVM_ENUM_TARGETS "")
    set(LLVM_ENUM_ASM_PRINTERS "")
    set(LLVM_ENUM_ASM_PARSERS "")
    set(LLVM_ENUM_DISASSEMBLERS "")
    set(LLVM_ENUM_TARGETMCAS "")
    set(LLVM_ENUM_EXEGESIS "")
    foreach(t ${LLVM_TARGETS_TO_BUILD})
      set( td ${LLVM_MAIN_SRC_DIR}/lib/Target/${t} )
    
      list(FIND LLVM_ALL_TARGETS ${t} idx) # 检测LLVM_TARGETS_TO_BUILD中的变量是否合法
      list(FIND LLVM_EXPERIMENTAL_TARGETS_TO_BUILD ${t} idy)
      if( idx LESS 0 AND idy LESS 0 )
        message(FATAL_ERROR "The target `${t}' is experimental and must be passed "
          "via LLVM_EXPERIMENTAL_TARGETS_TO_BUILD.")
      else()
        set(LLVM_ENUM_TARGETS "${LLVM_ENUM_TARGETS}LLVM_TARGET(${t})\n")
      endif()
      ...
      if( EXISTS ${td}/AsmParser/CMakeLists.txt )
        set(LLVM_ENUM_ASM_PARSERS
          "${LLVM_ENUM_ASM_PARSERS}LLVM_ASM_PARSER(${t})\n")
      endif()
      ...
    endforeach()
    ```

    如果`LLVM_TARGETS_TO_BUILD="X86;Mips"`，那么最后`LLVM_ENUM_PARSERS`就是：

    ```cpp
    LLVM_ASM_PARSER(X86)
    LLVM_ASM_PARSER(Mips)
    ```

    这些变量会被用于生成头文件：

    ```cpp
    configure_file(
      ${LLVM_MAIN_INCLUDE_DIR}/llvm/Config/AsmParsers.def.in
      ${LLVM_INCLUDE_DIR}/llvm/Config/AsmParsers.def
      )
    ```
    
    其中AsmParsers.def.in的内容如下：
    
    ```cpp
    #ifndef LLVM_ASM_PARSER
    #  error Please define the macro LLVM_ASM_PARSER(TargetName)
    #endif
    
    @LLVM_ENUM_ASM_PARSERS@
    
    #undef LLVM_ASM_PARSER
    ```
    
    展开生成的AsmParsers.def如下：
    
    ```cpp
    #ifndef LLVM_ASM_PARSER
    #  error Please define the macro LLVM_ASM_PARSER(TargetName)
    #endif
    
    LLVM_ASM_PARSER(X86)
    LLVM_ASM_PARSER(Mips)
    
    #undef LLVM_ASM_PARSER
    ```
    
14. 将llvm/include目录与build/include目录加入项目的`include`目录。

    ```cmake
    include_directories( ${LLVM_INCLUDE_DIR} ${LLVM_MAIN_INCLUDE_DIR})
    ```

15. 引入LLVM自定义的Module，用于处理子目录的自定义命令与tblgen相关的命令：

    ```cmake
    include(AddLLVM)
    include(TableGen)
    
    include(LLVMDistributionSupport)
    ```

16. 开始构建子目录，需要保证被依赖的库先构建/定义，否则会出现循环依赖：

    ```cmake
    add_subdirectory(lib/Demangle)
    add_subdirectory(lib/Support) # Support依赖Demangle库，所以Demangle最先构建
    add_subdirectory(lib/TableGen) # TableGen依赖Support库，所以紧随其后
    
    add_subdirectory(utils/TableGen)
    
    add_subdirectory(include/llvm)
    add_subdirectory(lib)
    ...
    ```

17. 最后执行安装命令，（`install`命令还没认真看，后面再补上相关介绍，逃~~~

    ```cmake
    install(DIRECTORY include/llvm include/llvm-c
      DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
      COMPONENT llvm-headers
      FILES_MATCHING
      PATTERN "*.def"
      PATTERN "*.h"
      PATTERN "*.td"
      PATTERN "*.inc"
      PATTERN "LICENSE.TXT"
      )
                                                                               
    install(DIRECTORY ${LLVM_INCLUDE_DIR}/llvm ${LLVM_INCLUDE_DIR}/llvm-c
      DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
      COMPONENT llvm-headers
      FILES_MATCHING
      PATTERN "*.def"
      PATTERN "*.h"
      PATTERN "*.gen"
      PATTERN "*.inc"
      # Exclude include/llvm/CMakeFiles/intrinsics_gen.dir, matched by "*.def"
      PATTERN "CMakeFiles" EXCLUDE
      PATTERN "config.h" EXCLUDE
      )
    ```

    

## 注解

[^1]: 要完全掌握CMake的使用细节需要花不少时间，而且很容易就忘，我的习惯是先大致了解该语言的整体架构，然后遇到不会的内容再去查文档。
