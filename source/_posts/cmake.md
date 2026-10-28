---
title: cmake
tags: cmake
date: 2025-08-26 10:35:14
---

## 前言

文章选用cmake版本为3.28.6
我喜欢这个上有老下有小的版本，令人踏实

推荐一个cmake书籍: [CMakeCookbook 中文版](https://www.bookstack.cn/read/CMake-Cookbook/content-preface-preface-chinese.md)

## cmake 的使用场景和优势

- 使用场景：cmake是个构建工具，广泛用在c/c++语言。

- 使用cmake的好处无非围绕三个字：跨平台。我们在编译c/c++代码的时候，在Linux系统可能使用GCC，在windows系统可能使用微软提供的MSVC构建工具，他们的构建指令是不一样的，GCC需要编写Makefile，MSVC需要编写.sln文件，那么如果我们开发一个软件需要在两个平台下使用，就要分别写一套构建文件，这很麻烦，但如果使用cmake，他会根据你构建工具来生成对应的构建文件。

## cmake中的一些概念

- CMakeLists.txt 就像是GCC中的Makefile，MSVC的.sln文件一样，cmake也是使用文件来构建项目，我们所有的构建指令，调试指令都是写在CMakeLists.txt文件当中。

## cmake基础使用

### 一段最简单的cmake程序

~~~cmake
cmake_minimum_required(VERSION 3.10) # 设置版本限制

project(hello-world LANGUAGES CXX) # 设置项目名称和所用语言

add_executable(hello-world hello-world.cpp) # 构建可执行文件
~~~

### cmake中的生成静态(动态)链接库并链接

静态链接库

~~~cmake
add_library(message
  STATIC
    Message.hpp
    Message.cpp
  )

add_executable(hello-world hello-world.cpp)

target_link_libraries(hello-world message)
~~~

动态链接库
~~~cmake
add_library(message
  SHARED
    Message.hpp
    Message.cpp
  )

add_executable(hello-world hello-world.cpp)

target_link_libraries(hello-world message)
~~~

### cmake中的条件语句

~~~cmake
set(USE_LIBRARY OFF)
message(STATUS "Compile sources into a library? ${USE_LIBRARY}")

if(USE_LIBRARY)
    # add_library will create a static library
    # since BUILD_SHARED_LIBS is OFF
    add_library(message ${_sources})
    add_executable(hello-world hello-world.cpp)
    target_link_libraries(hello-world message)
else()
    add_executable(hello-world hello-world.cpp ${_sources})
endif()
~~~

### cmake中指定编译器

~~~shell
$ cmake -D CMAKE_CXX_COMPILER=clang++ ..
~~~

### cmake中指定构建类型

构建类型一般包含Debug版本和Release版本
- Debug版本一般用于程序员在开发工程中便于调试的版本，可能包含很多调试需要的log信息，编译速度较慢
- Relase版本是真正的发行版本，是指在项目准备好进行发布的版本。

~~~cmake
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()
message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")
~~~

切换构架版本

~~~shell
$ cmake -D CMAKE_BUILD_TYPE=Debug ..
~~~

### cmake中设定语言标准

这个一般出现在c++中，因为c++的版本迭代，不同的版本支持的语言特性不同，因此需要指定语言标准。

~~~cmake
set_target_properties(animals
  PROPERTIES
    CXX_STANDARD 14
    CXX_EXTENSIONS OFF
    CXX_STANDARD_REQUIRED ON
    POSITION_INDEPENDENT_CODE 1
)
~~~

- CXX_STANDARD 会设置我们想要的标准。
- CXX_EXTENSIONS 告诉CMake，只启用ISO C++标准的编译器标志，而不使用特定编译器的扩展。
- CXX_STANDARD_REQUIRED 指定所选标准的版本。如果这个版本不可用，CMake将停止配置并出现错误。当这个属性被设置为OFF时，CMake将寻找下一个标准的最新版本，直到一个合适的标志。这意味着，首先查找C++14，然后是C++11，然后是C++98。（译者注：目前会从C++20或C++17开始查找）

### cmake中处理平台相关的源代码

理想情况下，应该避免依赖于平台的源代码，但是有时我们没有选择，特别是当要求配置和编译不是自己编写的代码时。本示例中，将演示如何使用CMake根据操作系统编译源代码。

~~~cpp
#include <cstdlib>
#include <iostream>
#include <string>
std::string say_hello() {
#ifdef IS_WINDOWS
  return std::string("Hello from Windows!");
#elif IS_LINUX
  return std::string("Hello from Linux!");
#elif IS_MACOS
  return std::string("Hello from macOS!");
#else
  return std::string("Hello from an unknown system!");
#endif
}
int main() {
  std::cout << say_hello() << std::endl;
  return EXIT_SUCCESS;
}
~~~

可以在cmake中定义标志

~~~cmake
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
  target_compile_definitions(hello-world PUBLIC "IS_LINUX")
endif()
if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
  target_compile_definitions(hello-world PUBLIC "IS_MACOS")
endif()
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
  target_compile_definitions(hello-world PUBLIC "IS_WINDOWS")
endif()
~~~

## find_package

`find_package`是用于发现和设置包的CMake模块的命令。这些模块包含CMake命令，用于标识系统标准位置中的包。CMake模块文件称为`Find<name>.cmake`，当调用`find_package(<name>)`时，模块中的命令将会运行。

