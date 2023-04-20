---
title: LLVM CMake 构建系统
categories:
  - LLVM
  - CMake
---

 [toc]

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

3. `Modules`是CMake提供的用于支持代码重用的功能。模块也是一段普通的CMake命令，CMake可以通过查找`CMAKE_MODULE_PATH`指令的目录及其子目录查询对应的模块文件。模块分为查找模块（Find Modules）以及实用模块（Utility Modules）。查找模块支持`find_package`命令来确定某些软件库或者头文件的位置，提供一些结果变量或者描述函数等；实用模块是一些自定义函数的封装，用于提供一些特定功能。

### 术语&命令

* `project`：

* `include`：
* `set`：
* `list`：

- `option`：提供一个用于可选的bool值选项。使用语法为：

  ```cmake
  option(<variable> "help_text" [value])
  ```

  如果`<variable>`已经被设置，则该命令什么都不做；否则定义一个bool变量，其值默认为OFF，可由`value`覆盖。

- `add_compile_definitions`：将预处理定义加入编译器的命令行，其语法如下：

  ```cmake
  add_compile_definitions(<definition> ...)
  ```

  其中`<definition>`是`VAR`或者`VAR=XXX`的形式，在编译命令行中分别对应`-DVAR`以及`-DVAR=XXX`的选项。

* `file`：
* `execute_process`：
* `find_package`：
* `configure_file`：

## CMakeLists.txt分析

以`llvm-project/llvm/CMakeLists.txt`为例子进行分析：

- 首先定义项目名、项目的版本号以及项目的语言。

   ```cmake
   project(LLVM
     VERSION ${LLVM_VERSION_MAJOR}.${LLVM_VERSION_MINOR}.${LLVM_VERSION_PATCH}
     LANGUAGES C CXX ASM)
   ```

- 定义了一个`LLVM_ALL_TARGETS`变量表示支持的所有后端架构：

   ```cmake
   set(LLVM_ALL_TARGETS
     AArch64
     AMDGPU
     ...
     X86
     XCore
     )
   ```

- 定义了一个`LLVM_TARGETS_TO_BUILD`变量表示需要构建的后端架构，默认是所有架构：

   ```cmake
   set(LLVM_TARGETS_TO_BUILD "all"
       CACHE STRING "Semicolon-separated list of targets to build, or \"all\".")
       
   if( LLVM_TARGETS_TO_BUILD STREQUAL "all" )
     set( LLVM_TARGETS_TO_BUILD ${LLVM_ALL_TARGETS} )
   endif()
   ```

   



## 注解

[^1]: 要完全掌握CMake的使用细节需要花不少时间，而且很容易就忘，我的习惯是先大致了解该语言的整体架构，然后遇到不会的内容再去查文档。
