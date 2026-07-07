---
title: ESP-IDF项目风格
date: 2026-02-18
description: 因为想要规范化自己的开发习惯，于是我在阅读乐鑫文档后，简单总结了一个标准IDF项目该有的样子及对各个部分的简要解释。
tags:  
    - 嵌入式
    - ESP32
categories: 
    - 研究
---

> 参考文档
> https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-guides/build-system.html

# 项目架构与风格

## 基本概念
**项目** 
特指一个目录，其中包含了构建 **应用程序** （即可执行文件）所需的全部文件和配置，还包含了其他支持型文件，
例如分区表、数据分区或文件系统分区，以及引导加载程序。

**项目配置** 
保存在项目根目录下名为 `sdkconfig` 的文件中，
可以通过 `idf.py menuconfig` 进行修改，且一个项目只能包含一个项目配置。

**应用程序** 
是由 `ESP-IDF` 构建得到的可执行文件。
- 一个项目通常会构建两个应用程序：
    - 项目应用程序（可执行的主文件，即用户自定义的固件）
    - 引导加载程序（启动并初始化项目应用程序）。

**组件**
组件是 `COMPONENT_DIRS` 列表中包含 `CMakeLists.txt` 文件的任何目录。
- 组件将会被编译成静态库（.a 文件）并链接到应用程序。
- 部分组件由 `ESP-IDF` 官方提供，其他组件则来源于其它开源项目。

**目标** 
特指运行构建后应用程序的硬件设备。
运行 `idf.py --list-targets` 可以查看当前 `ESP-IDF` 版本中支持目标的完整列表。

## 项目基本结构

**main/**
它是一种特殊的**组件**，main组件内包含项目的入口文件
在项目的入口文件中定义了`app_main()`作为程序的入口函数

**build/**
存放构建输出的目录    
- CMake 会配置项目，并在此目录下生成临时的构建文件。
- 随后，在主构建进程的运行期间，该目录还会保存临时目标文件、库文件以及最终输出的二进制文件。
- 此目录通常不会添加到项目的源码管理系统中，也不会随项目源码一同发布。
  

**sdkconfig**
执行 `idf.py menuconfig` 时会创建或更新此文件，文件中保存了项目中所有组件（包括 ESP-IDF 本身）的配置信息。
- `sdkconfig.old` 文件是前者的备份
- 本文件存储了IDF Menu Config的配置项

**component/**
可选目录，包含了项目的部分自定义组件，可以在顶层CMakeLists.txt中设置 `EXTRA_COMPONENT_DIRS` 变量来手动查找其他位置的组件
- 该目录包含构成项目的标准组件，各个组件之间应低耦合，最终在main的入口文件中调用并实现任务创建等基本行为
- 有助于构造可复用的代码或导入第三方组件

**managed_components/**
由 IDF 组件管理器创建，用于存储由 IDF 组件管理器管理的组件
- 请勿手动修改 "`managed_components`" 目录下的内容。如果需要进行更改，可以将组件复制到 `components` 目录下。
- "`managed_components`" 目录通常不在 Git 中进行版本控制，也不会与项目源代码一起分发。

**CMakeLists.txt**
组织IDF项目的核心文件

### CMake文件结构

#### 顶层CMakeList
每个项目都有一个顶层 CMakeLists.txt 文件，包含整个项目的构建设置。默认情况下，项目 CMakeLists 文件会非常小。

- **最小项目示例**:
    ```CMake
    cmake_minimum_required(VERSION 3.16)   # 选择该项目使用的最低CMake版本
    include($ENV{IDF_PATH}/tools/cmake/project.cmake) # IDF的标准模板，会导入 CMake 的其余功能来完成配置项目、检索组件等任务
    project(myProject) # 会创建项目本身，并指定项目名称。该名称会作为最终输出的二进制文件的名字
    ```

- **可选项目变量**
  参考文档:  https://github.com/espressif/esp-idf/blob/v5.5.1/tools/cmake/project.cmake
    ```
    COMPONENT_DIRS：组件的搜索目录，默认为 IDF_PATH/components、 PROJECT_DIR/components、和 EXTRA_COMPONENT_DIRS。如果你不想在这些位置搜索组件，请覆盖此变量。
    
    EXTRA_COMPONENT_DIRS：用于搜索组件的其它可选目录列表。路径可以是相对于项目目录的相对路径，也可以是绝对路径。
    
    COMPONENTS：用于指定要构建到项目中的组件名称列表，默认为 COMPONENT_DIRS 目录下检索到的所有组件。使用此变量可以“精简”项目，从而缩短构建时间。请注意，如果一个组件通过 COMPONENT_REQUIRES 指定了它依赖的另一个组件，则会自动将其添加到 COMPONENTS 中，所以 COMPONENTS 列表可能会非常短。另外，还可以通过设置 MINIMAL_BUILD 构建属性 来指定 COMPONENTS 中的 main 组件。
    
    BOOTLOADER_IGNORE_EXTRA_COMPONENT：可选组件列表，位于 bootloader_components/ 目录中，引导加载程序编译时会忽略该列表中的组件。使用这一变量可以将一个组件有条件地包含在项目中。
    
    BOOTLOADER_EXTRA_COMPONENT_DIRS：可选的附加路径列表，引导加载程序编译时将从这些路径中搜索要编译的组件。注意，这是一个构建属性。
    ```



#### 组件CMakeList
这个文件用来在IDF中注册组件自身的相关源代码,以便于后续与项目其他部分进行链接.

- esp-idf的外部组件注册行为发生在该外部组件的根目录cmakelist文件中
- 当 CMake 运行项目配置时，它会记录本次构建包含的组件列表，它可用于调试某些组件的添加/排除
    - 如果组件存在重名,IDF搜索组件的优先级:
        - 项目目录下的组件
        - `EXTRA_COMPONENT_DIRS` 中的组件
        - 项目目录下 `managed_components/` 目录中的组件
        - `IDF_PATH/components` 目录下的组件
    - 将ESP-IDF组件复制到项目目录下,就可以在不修改IDF源代码的前提下修改IDF的内部组件
    - 需要fullclean后重新构建项目才能看到新组件的路径

- 创建一个新组件后,可以使用IDF的内置函数将其快速添加到构建系统中.
    ```CMake
    idf_component_register(
        SRCS "foo.c" "bar.c"    # 该组件中需要加入构建系统的源代码文件
        INCLUDE_DIRS "include"  # 添加头文件到全局include搜索路径中
        REQUIRES mbedtls)       # 添加组件前置依赖关系, 声明该组件需要使用什么其他组件
    ```
    - 这个函数将会生成一个与根目录文件夹名称一致的库,并在最终链接至应用程序中
    - 使用相对路径
    - 支持一些额外的参数,参考这里
      https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-guides/build-system.html#cmake-component-register
    - 组件变量参考
    ```
    COMPONENT_DIR：组件目录，即包含 CMakeLists.txt 文件的绝对路径，它与 CMAKE_CURRENT_SOURCE_DIR 变量一样，路径中不能包含空格。
    
    COMPONENT_NAME：组件名，与组件目录名相同。
    
    COMPONENT_ALIAS：库别名，由构建系统在内部为组件创建。
    
    COMPONENT_LIB：库名，由构建系统在内部为组件创建。
    
    COMPONENT_VERSION：组件版本，由 idf_component.yml 指定并由 IDF 组件管理器设置。
    ```

    - 可以把KConfig添加到与CMakelist.txt同级的组件根目录中,就可以为该组件添加一些独有配置了.

**组件依赖**
- main组件默认依赖所有组件,无需传递`REQUIRES` 或 `PRIV_REQUIRES`
- 即便不声明 `REQUIRES` 或 `PRIV_REQUIRES`, 各组件也会默认依赖一些基础组件
  通用组件包括：cxx、newlib、freertos、esp_hw_support、heap、log、soc、hal、esp_rom、esp_common、esp_system。
- 如果将 `COMPONENTS` 变量设置为项目直接使用的最小组件列表，那么构建系统会扩展到包含所有组件。完整的组件列表为：
    - `COMPONENTS` 中明确提及的组件。
    - 这些组件的依赖项（以及递归运算后的组件）。每个组件都依赖的 通用组件。
    - 会显著缩短构建时间

### 添加自定义Bootloader
// TODO 

## 不使用IDF标准CMake函数添加外部组件
// TODO


## 编写多目标
> [!WARNING]
>
> 猜你在找:esp-idf 测试
>
> https://docs.espressif.com/projects/esp-idf/zh_CN/release-v5.3/esp32/api-guides/build-system.html#cmake-buildsystem-api
> https://github.com/espressif/esp-idf/tree/7024ea10/examples/build_system/cmake/idf_as_lib
> 需要注意,似乎IDF从6.0版本才支持自定义多主目标
> 我使用的SDK版本是5.3,所以需要使用CMake来手动配置多目标。

```CMake
cmake_minimum_required(VERSION 3.16)
project(my_custom_app C)

# 导入提供 ESP-IDF CMake 构建系统 API 的 CMake 文件
include($ENV{IDF_PATH}/tools/cmake/idf.cmake)

# 在构建中导入 ESP-IDF 组件，可以视作等同 add_subdirectory()
# 但为 ESP-IDF 构建增加额外的构建过程
# 具体构建过程
idf_build_process(esp32)

# 创建项目可执行文件
# 使用其别名 idf::newlib 将其链接到 newlib 组件
add_executable(${CMAKE_PROJECT_NAME}.elf main.c)
target_link_libraries(${CMAKE_PROJECT_NAME}.elf idf::newlib)

# 让构建系统知道项目到可执行文件是什么，从而添加更多的目标以及依赖关系等
idf_build_executable(${CMAKE_PROJECT_NAME}.elf)
```
