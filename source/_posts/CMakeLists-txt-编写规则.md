---
title: CMakeLists.txt 编写规则
date: 2024-11-03 12:28:39
categories:
- linux
tags: 
- cmake
- CMakeLists.txt
typora-root-url: ./..
---

 参考：

[CMake 保姆级教程（上） | 爱编程的大丙](https://subingwen.cn/cmake/CMake-primer/#2-6-3-总结)

[cmake使用详细教程（日常使用这一篇就足够了）_cmake教程-CSDN博客](https://blog.csdn.net/iuu77/article/details/129229361?ops_request_misc={"request_id":"3BDC4C1F-AC50-4EB2-99A3-1FD7204C3A17","scm":"20140713.130102334.."}&request_id=3BDC4C1F-AC50-4EB2-99A3-1FD7204C3A17&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~top_positive~default-2-129229361-null-null.142^v100^pc_search_result_base8&utm_term=cmake&spm=1018.2226.3001.4187)

> **需要注意的是CMakeLists.txt文件中的指令不区分大小写**

# 1. 注释

## 1.1 注释行

CMake通过**‘#’**进行单行注释，比如

```yaml
# Cmake版本至少为3.1
cmake_minimum_required(VERSION 3.1)
```

## 1.2 注释块

CMake通过**‘if(FALSE)’**进行块注释，比如

```yaml
if(FALSE)
    # CMake版本至少为3.1
    # CMake版本至少为3.1
    # CMake版本至少为3.1
endif()
cmake_minimum_required(VERSION 3.1)
```

# 2. CMakeLists.txt的编写

## 2.1 同意目录下的源文件

如果只有一个源文件hello.cpp，内容如下：

```cpp
#include <iostream>
using namespace std;
 
int main()
{
    cout << "Hello World!" << endl;
    return 0;
}
```

那么CMakeLists.txt可以这样写：

```yaml
cmake_minimum_required(VERSION 3.1)
project (LearnCMake LANGUAGES CXX)
add_executable(hello hello.cpp)
```

- cmake_minimum_required：指定使用的 cmake 的**最低**版本；
- project：定义工程名称，并可指定工程的版本、工程描述、web主页地址、支持的语言（默认情况支持所有语言），如果不需要这些都是可以忽略的，只需要指定出工程名字即可； 
  - 这里项目名称为 LearnCMake ，项目使用的语言是 C++，CXX表示C++

```yaml
# PROJECT 指令的语法是：
project(<PROJECT-NAME> [<language-name>...])
project(<PROJECT-NAME>
       [VERSION <major>[.<minor>[.<patch>[.<tweak>]]]]
       [DESCRIPTION <project-description-string>]
       [HOMEPAGE_URL <url-string>]
       [LANGUAGES <language-name>...])

# 示例
project(GrpcServer  
        VERSION 1.0.0  
        DESCRIPTION "A gRPC server implementation in C++"  
        HOMEPAGE_URL "https://example.com/grpcserver"  
        LANGUAGES CXX)
```

- add_executable：定义工程会生成一个可执行程序，可执行程序名为hello，源文件名称为hello.cpp

```yaml
# 单个源文件
add_executable(可执行程序名 源文件名称)
# 多个源文件名（空格隔开）
add_executable(hello add.c div.c main.c mult.c sub.c)
# 多个源文件名（封号隔开）
add_executable(hello  add.c;div.c;main.c;mult.c;sub.c)
```

我们一般需要单独创建一个build文件夹，用于存放

- **可执行文件**：已经编译过的可执行文件，如Windows上的.exe文件，Linux和Mac上的二进制文件。
- **中间文件**：编译过程中生成的中间文件，如目标文件（.o或.obj）、汇编文件（.s）等。
- **构建脚本**：用于帮助构建项目的脚本文件，如Makefile、CMakeLists.txt的生成版本或Ninja文件等。
- **库文件**：如果项目包含库代码的编译，那么还会生成静态库（.a）或动态库（.so、.dll）。

我们这里创建一个build文件夹，进入之后执行cmake并编译

```yaml
mkdir build
cd ./build/
cmake ..
```

build文件夹内会生成以下文件：



![img](/images/$%7Bfiilename%7D/format,png-1730608148424-328.png)

执行make命令，使用makefile进行编译

```yaml
make
```

![img](/images/$%7Bfiilename%7D/format,png-1730608148424-329.png)

生成可执行程序hello，并运行

```yaml
./hello
```

![img](/images/$%7Bfiilename%7D/format,png-1730608148424-330.png)

## 2.2 SET指令

**SET**指令是 CMake 中用于定义和设置变量的命令，它允许你创建一个新的变量或者修改一个已存在的变量的值。

### ***2.2.1 使用SET将多个源文件存储至指定变量***

如果我们有五个源文件需要添加：

```yaml
add_executable(hello add.c;div.c;main.c;mult.c;sub.c)
```

如果有多个源文件，我们可以定义一个变量，将文件名对应的字符串存储起来，在cmake里定义变量需要使用**set**

```yaml
# SET 指令的语法是：
# [] 中的参数为可选项, 如不需要可以不写
SET(VAR [VALUE] [CACHE TYPE DOCSTRING [FORCE]])
```

我们使用SET指令将这五个文件名和自定义的变量联系起来，通过添加这个变量名即可同时调用这五个文件

```yaml
set(SRC_LIST add.c;div.c;main.c;mult.c;sub.c)
add_executable(hello  ${SRC_LIST})
```

如果源文件特别多，一个个添加会特别慢，此时可以使用cmake中的函数存储这些源文件。

```
aux_source_directory(dir var)
```

他的作用是把**dir**目录中的所有源文件都储存在**var**变量中，然后需要用到源文件的地方用变量var来取代。

上述代码可以使用该函数改写为：(我们一般把.cpp文件放在src文件夹下，.h放在include文件夹下)

```
aux_source_directory(. SRC_LIST)
add_executable(hello  ${SRC_LIST})
```

### ***2.2.2 指定C++标准***

C++标准对应有一宏叫做**DCMAKE_CXX_STANDAR**，我们可以通过SET指令指定我们使用的C++标准：

```yaml
#增加-std=c++11
set(CMAKE_CXX_STANDARD 11)
#增加-std=c++14
set(CMAKE_CXX_STANDARD 14)
#增加-std=c++17
set(CMAKE_CXX_STANDARD 17)
```

该宏一般与宏**CMAKE_CXX_STANDARD_REQUIRED**进行配合使用，该宏为ON时， 如果编译器不支持你指定的 C++ 标准（比如C++17），CMake 配置过程将会失败：

```
set(CMAKE_CXX_STANDARD_REQUIRED ON)
```

### ***2.2.3 指定输出的路径***

在CMake中指定可执行程序输出的路径，也对应一个宏，叫做**EXECUTABLE_OUTPUT_PATH**，可通过SET指令进行设置：

```
set(HOME /home/robin/Linux/Sort)
set(EXECUTABLE_OUTPUT_PATH ${HOME}/bin)
```

- 如果子目录bin不存在，会自动生成

## 2.3 file和aux_source_directory

这两个指令都可以用于搜索指定目录下的文件，后者用于查找指定目录下的**所有源文件**，我们在2.1中使用过，这里不再做介绍。

### ***2.3.1 file***

```
file(GLOB/GLOB_RECURSE 变量名 要搜索的文件路径和文件类型)
```

在 CMake 中，**file(GLOB ...)** 和 **file(GLOB_RECURSE ...)** 命令用于从指定的目录中搜索匹配特定模式的文件，并将这些文件的列表存储在一个变量中。

- GLOB: 将指定目录下搜索到的满足条件的所有文件名生成一个列表，并将其存储到变量中。
- GLOB_RECURSE：**递归**搜索指定目录，将搜索到的满足条件的文件名生成一个列表，并将其存储到变量中。

如果我们有一个目录如下：

```yaml
project/  
├── CMakeLists.txt  
├── src/  
│   ├── main.cpp  
│   └── utils/  
│       └── helper.cpp
```

如果我们像包含src/目录下的所有.cpp文件，只需：

```yaml
# 使用 GLOB 只搜索 src 目录下的文件（不递归），并存储至变量SRC_FILES 
file(GLOB SRC_FILES "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp")  
  
# 或者使用 GLOB_RECURSE 递归搜索 src 目录及其子目录中的文件  
file(GLOB_RECURSE SRC_FILES_RECURSIVE "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp")  
  
# 然后你可以使用这些变量来添加可执行文件或库  
add_executable(MyExecutable ${SRC_FILES_RECURSIVE})
```

- 宏 CMAKE_CURRENT_SOURCE_DIR 是 CMake 提供的一个内置变量，用于表示当前正在处理的CMakeLists.txt文件所在的源代码目录的路径 	
  - 如果不同文件夹下有不同的CMakeLists.txt文件，那么CMAKE_CURRENT_SOURCE_DIR 随着CMakeLists.txt文件所处的路径变化而变化。

## 2.4 包含头文件

### ***2.4.1 头文件均在一个目录下***

在CMake中通过命令 **include_directories** 设置要包含的目录。

```yaml
# 在指定 dir目录下寻找头文件
include_directories ( dir )
```

如果我们有一个目录如下：

```
$ tree
.
├── build
├── CMakeLists.txt
├── include
│   └── head.h
└── src
    ├── add.cpp
    ├── div.cpp
    ├── main.cpp
    ├── mult.cpp
    └── sub.cpp
```

我们可以使用如下指令包含头文件head.h

```
include_directories(${PROJECT_SOURCE_DIR}/include)
```

- 宏 **PROJECT_SOURCE_DIR** 是 CMake 在处理project()命令时自动设置的变量，它指向包含最近一次调用project()命令定义的项目的源目录

该目录下的CMakeLists.txt文件这样写：

```yaml
cmake_minimum_required(VERSION 3.1)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
project (LearnCMake LANGUAGES CXX)

include_directories(${PROJECT_SOURCE_DIR}/include)
file(GLOB SRC_LIST ${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp)
add_executable(hello ${SRC_LIST})
```

### ***2.4.2 头文件源文件分离，并包含多个文件夹***

假设我们有如下目录结构：

```
LEARN_CMAKE
├── CMakeLists.txt
│   ├── inc_dir1
│   │   └── myfunc1.h
│   ├── inc_dir2
│   │   └── myfunc2.h
│   ├── main_dir
│   │   └── hello.cpp
│   ├── src_dir1
│   │   └── myfunc1.cpp
│   └── src_dir2
│       └── myfunc2.cpp
```

该目录下的CMakeLists.txt文件这样写：

```yaml
cmake_minimum_required(VERSION 3.1)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 项目名称
project (LearnCMake LANGUAGES CXX)

# 头文件
include_directories(${PROJECT_SOURCE_DIR}/inc_dir1 ${PROJECT_SOURCE_DIR}/inc_dir2)

# 获取源文件
file(GLOB SRC_LIST1 ${CMAKE_CURRENT_SOURCE_DIR}/src_dir1/*.cpp)
aux_source_directory(${CMAKE_CURRENT_SOURCE_DIR}/src_dir2 SRC_LIST2)
aux_source_directory(${CMAKE_CURRENT_SOURCE_DIR}/main_dir MAIN_DIR)

# 生成可执行文件
add_executable(hello ${SRC_LIST1} ${SRC_LIST2} ${MAIN_DIR})
```

## 2.5 生成动态库和静态库

- 动态库在编译时并不会将库的代码直接复制到最终的可执行文件中，而是在程序运行时动态地加载库。这种机制有助于节省磁盘空间和内存，因为多个程序可以共享同一个动态库的副本。
- 当我们提到动态库的“后者”时，如果是指与动态库相对的概念，那么通常指的是静态库。静态库在编译时会被完整地复制到最终的可执行文件中，因此每个使用静态库的程序都会有一个库的副本。

有些时候我们编写的源代码并不需要将他们编译生成可执行程序，而是生成一些静态库或动态库提供给第三方使用，下面来讲解在cmake中生成这两类库文件的方法。

### *2.5.1 静态库*

- 在Linux中静态库以~作为前缀, 以`.a`作为后缀, 中间是库的名字自己指定即可, 即: `libxxx.a`

- 在Windows中静态库一般以`lib`作为前缀, 以`.lib`作为后缀, 中间是库的名字需要自己指定, 即: `libxxx.lib`

静态库的生成需使用如下指令：

```
add_library(库名称 STATIC 源文件1 [源文件2] ...) 
```

- **库名称：**创建的静态库的名字。在构建成功后，这个名字会出现在输出目录中。
- **STATIC**：这个关键字指示你想要创建一个**静态库**。
- **源文件1 [源文件2] ...**：这些是构成静态库的源文件列表。你可以列出多个源文件，CMake 会将它们编译并打包进静态库中。

我们一般只需要指定出库的名字和STATIC关键字就可以，另外两部分在生成该文件的时候会**自动填充**。

假设我们有如下目录结构：

```yaml
$ tree
.
├── build
├── CMakeLists.txt
├── include          # include目录下存放头文件
│   └── head.h
├── lib              # lib目录下存放生成的库
└── src              # src目录下存放源文件
    ├── add.cpp
    ├── div.cpp
    ├── main.cpp
    ├── mult.cpp
    └── sub.cpp
```

如果我们要将其生成为静态库，该目录下的CMakeLists.txt文件这样写：

```yaml
cmake_minimum_required(VERSION 3.1)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
project (LearnCMake LANGUAGES CXX)

include_directories(${PROJECT_SOURCE_DIR}/include)

file(GLOB SRC_LIST ${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp)
add_library(hello STATIC ${SRC_LIST})
```

这样最终就会生成对应的静态库文件 **hello.a**或者**hello.lib**

### ***2.5.2 动态库***

- 在Linux中动态库以`lib`作为前缀, 以`.so`作为后缀, 中间是库的名字自己指定即可, 即: `libxxx.so`

- 在Windows中动态库一般以`lib`作为前缀, 以`.dll`作为后缀, 中间是库的名字需要自己指定, 即: `libxxx.dll`

动态库的指令和静态库类似，只不过关键字**STATIC**需改为**SHARED**

```
add_library(库名称 SHARED 源文件1 [源文件2] ...) 
```

编写对应的CMakeLists.txt文件，最后生成对应的动态库文件**hello.so**或者**hello.dll**

### ***2.5.3 指定库输出路径***

我们使用宏 **LIBRARY_OUTPUT_PATH** 来指定静态/动态库的输出路径。而对于动态库，因为动态库有可执行权限，所以不仅可以像可执行程序一样使用宏 **EXECUTABLE_OUTPUT_PATH** 指定输出路径，而且可以使用宏 **LIBRARY_OUTPUT_PATH**；但静态库默认不具有可执行权限，所以静态库只可以用宏 **LIBRARY_OUTPUT_PATH** 来指定输出路径。

假设我们有如下目录结构：

```
$ tree
.
├── build
├── CMakeLists.txt
├── include          # include目录下存放头文件
│   └── head.h
├── lib              # lib目录下存放生成的库
└── src              # src目录下存放源文件
    ├── add.cpp
    ├── div.cpp
    ├── main.cpp
    ├── mult.cpp
    └── sub.cpp
```

如果我们要将其生成为库，并指定输出路径，该目录下的CMakeLists.txt文件这样写：

```
cmake_minimum_required(VERSION 3.1)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
project (LearnCMake LANGUAGES CXX)
# 包含头文件
include_directories(${PROJECT_SOURCE_DIR}/include)
# 查找源文件
file(GLOB SRC_LIST ${PROJECT_SOURCE_DIR}/src/*.cpp)
# 设置动态库/静态库生成路径
set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)
# 生成动态库
add_library(hello SHARED ${SRC_LIST})
# 生成静态库
add_library(hello STATIC ${SRC_LIST})
```

## 2.6 链接库文件

如果lib目录下已经存在库文件，我们应该如何用lib下的静态/动态库。

假设我们有如下目录结构：

```
$ tree
.
├── build
├── CMakeLists.txt
├── bin              $ bin目录下存放项目生成的可执行文件
├── include          # include目录下存放头文件
│   └── head.h
├── lib              # lib目录下存放生成的库
    ├── libhello.a
    └── libhello.so
└── src              # src目录下存放源文件
    ├── add.cpp
    ├── div.cpp
    ├── main.cpp
    ├── mult.cpp
    └── sub.cpp
```

### ***2.6.1 链接静态库***

在cmake中，链接静态库的命令如下：

```
link_libraries(<static lib> [<static lib>...])
```

- 静态库名称可以是全名 libxxx.a，也可以是掐头（lib）去尾（.a）之后的名字 xxx

如果该静态库不是系统提供的（自己制作或者使用第三方提供的静态库）可能出现静态库找不到的情况，此时可以将静态库的路径也指定出来：

```
link_directories(<lib path>)
```

该目录下的CMakeLists.txt文件这样写：

```yaml
cmake_minimum_required(VERSION 3.1)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
project (LearnCMake LANGUAGES CXX)
# 包含头文件
include_directories(${PROJECT_SOURCE_DIR}/include)
# 查找源文件
file(GLOB SRC_LIST ${PROJECT_SOURCE_DIR}/src/*.cpp)
# 包含静态库路径
link_directories(${PROJECT_SOURCE_DIR}/lib)
# 查找库文件
# 第一个参数：变量，用于存储查找到的库文件   第二个参数：要查找的库文件  第三个参考：指定目录下查找
find_library(HELLO_LIB libhello.a ${PROJECT_SOURCE_DIR}/lib)
# 链接静态库
link_libraries(${HELLO_LIB})
# 生成可执行程序
add_executable(hello ${SRC_LIST})
```

find_library函数用于查找指定目录下的库文件，并将其存储至指定变量中。

### ***2.6.2 链接动态库***

在cmake中链接动态库的命令如下:

```
target_link_libraries(
    <target> 
    <PRIVATE|PUBLIC|INTERFACE> <item>... 
    [<PRIVATE|PUBLIC|INTERFACE> <item>...]...)
```

- target：指定要加载的库的文件的名字 	

  - 该文件可能是一个源文件
  - 该文件可能是一个动态库/静态库文件
  - 该文件可能是一个可执行文件
  
- PRIVATE/PUBLIC/INTERFACE：动态库的访问权限，默认为PUBLIC 

  - 如果各个动态库之间没有依赖关系，无需做任何设置，三者没有没有区别，一般无需指定，使用默认的 PUBLIC 即可。
  - 动态库的链接具有传递性，如果动态库 A 链接了动态库B、C，动态库D链接了动态库A，此时动态库D相当于也链接了动态库B、C，并可以使用动态库B、C中定义的方法。 	
    - **PUBLIC**：在public后面的库会被Link到前面的target中，并且里面的符号也会被导出，提供给第三方使用。
    - **PRIVATE**：在private后面的库仅被link到前面的target中，并且终结掉，第三方不能感知你调了啥库
    - **INTERFACE**：在interface后面引入的库不会被链接到前面的target中，只会导出符号。

> **动态库的链接和静态库是完全不同的**
>
> 1. 静态库会在生成可执行程序的链接阶段被打包到可执行程序中，所以可执行程序启动，静态库就被加载到内存中了。
> 2. 动态库在生成可执行程序的链接阶段不会被打包到可执行程序中，当可执行程序被启动并且调用了动态库中的函数的时候，动态库才会被加载到内存。因此，在cmake中指定要链接的动态库的时候，应该将**命令写到生成了可执行文件之后**。

该目录下的CMakeLists.txt文件这样写：

```yaml
cmake_minimum_required(VERSION 3.1)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
project (LearnCMake LANGUAGES CXX)
# 包含头文件
include_directories(${PROJECT_SOURCE_DIR}/include)
# 查找源文件
file(GLOB SRC_LIST ${PROJECT_SOURCE_DIR}/src/*.cpp)
# 包含动态库路径
link_directories(${PROJECT_SOURCE_DIR}/lib)
# 查找库文件
# 第一个参数：变量，用于存储查找到的库文件   第二个参数：要查找的库文件  第三个参考：指定目录下查找
find_library(HELLO_LIB libhello.so ${PROJECT_SOURCE_DIR}/lib)
# 生成可执行程序
add_executable(hello ${SRC_LIST})
# 链接的动态库
target_link_libraries(hello pthread ${HELLO_LIB})
```

其中，pthread 和 libhello都是可执行程序hello需要链接的动态库名称。

***2.6.3 target_link_libraries 和 link_libraries 的区别***

- target_link_libraries 是更现代和推荐的用法，适合精细控制和管理项目的库依赖。
- link_libraries 适合简单的项目或早期的 CMake 版本，但在复杂的项目中会带来管理上的困难

现在一般推荐使用*target_link_libraries*

*a. target_link_libraries*

仅对指定的目标（如可执行文件或库）在编译时需要链接哪些库生效。

```
target_link_libraries(<target> <LIBRARIES>)
```

比如

```
target_link_libraries(MyExecutable PRIVATE mylib)
```

**优点：**

- 可以指定链接库的作用范围（PRIVATE、PUBLIC、INTERFACE），从而控制库的可见性和传递性。
- 可以单独为每个目标配置链接库，不会影响其他目标。
- 支持接口库（interface libraries）和导出库（exported libraries）。

*b. link_libraries*

对整个目录内的所有目标生效.

**缺点：**

- 由于是全局的，无法控制库的作用范围（PRIVATE、PUBLIC 等），所有目标都会使用这些库。
- 缺乏模块化，不适合复杂的项目结构或多目标项目。

## 2.7 message指令

message命令用于在配置和生成过程中向用户输出信息。基本语法如下：

```
message([SEND_ERROR | FATAL_ERROR | WARNING | AUTH_WARNING | STATUS | DEPRECATION_WARNING] "message-text")
```

- SEND_ERROR：将消息作为错误输出，并继续执行（但通常会停止在产生错误的命令）。
- FATAL_ERROR：将消息作为错误输出，并立即停止处理 CMakeLists.txt 文件。
- WARNING：将消息作为警告输出。
- AUTH_WARNING：输出一个授权警告（通常用于策略警告）。
- STATUS：输出普通状态信息（这是**默认行为**，如果不指定类型）。
- DEPRECATION_WARNING：输出一个弃用警告。

举例：

```
if(UNIX)  
    message(STATUS "Building for UNIX-like system.")  
elseif(WIN32)  
    message(STATUS "Building for Windows.")  
else()  
    message(WARNING "Unknown platform. Proceeding with caution.")  
endif()
```

## 2.8 移除操作

若我们通过file或者aux_source_directory对某一个目录进行搜索，但搜索结果中并不是全部的源文件都是我们需要的，这时候我们需要通过将我们不需要的源文件移除。可以使用list命令：

```
list(REMOVE_ITEM <list> <value1> [<value2> ...])
```

- `<list>` 是要操作的列表的名称。
- `<value1>`, `<value2>`, ... 是希望从列表中移除的元素

举例：

```yaml
# 假设我有一个列表包含以下元素
set(my_list "a" "b" "c" "d" "e")
# 删除b和d
list(REMOVE_ITEM my_list "b" "d")
```

假设我们有如下目录结构：

```
$ tree
.
├── build
├── CMakeLists.txt
├── include          # include目录下存放头文件
│   └── head.h
├── lib              # lib目录下存放生成的库
└── src              # src目录下存放源文件
    ├── add.cpp
    ├── div.cpp
    ├── main.cpp
    ├── mult.cpp
    └── sub.cpp
```

若源文件不包含sub.cpp，该目录的CMakeLists.txt文件这样写：

```yaml
cmake_minimum_required(VERSION 3.1)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
project (LearnCMake LANGUAGES CXX)
# 包含头文件
include_directories(${PROJECT_SOURCE_DIR}/include)
# 查找源文件
file(GLOB SRC_LIST ${PROJECT_SOURCE_DIR}/src/*.cpp)
# 移除前日志
message(STATUS "message: ${SRC_LIST }")
# 移除 main.cpp
list(REMOVE_ITEM SRC_LIST ${PROJECT_SOURCE_DIR}/sub.cpp)
# 移除后日志
message(STATUS "message: ${SRC_LIST }")
# 生成可执行文件
add_executable(hello ${SRC_LIST})
```

## 2.9 find_library和find_package

**find_library**用于查找指定目录下的指定文件，并将其存储至变量中。

```yaml
# 第一个参数：变量，用于存储查找到的库文件   第二个参数：要查找的库文件  第三个参考：指定目录下查找
find_library(HELLO_LIB libhello.so ${PROJECT_SOURCE_DIR}/lib)
```

**find_package**根据系统环境找到指定的软件包或库，并将其相关信息（如库的路径、头文件的路径等）提供给项目使用。

```
find_package(<PackageName> [version] [REQUIRED] [COMPONENTS components...])
```

- `<PackageName>`: 需要查找的软件包的名称，例如 Boost、OpenCV 等。
- [version]: 可选，指定软件包的版本号。如果需要特定版本，可以指定版本号，例如 1.70.0。
- [REQUIRED]: 可选，表示该包是必需的。如果 CMake 找不到该包，它会停止并报错。
- [COMPONENTS components...]: 可选，指定要查找的组件或模块。例如，某些库可能有多个子模块（如 Boost 中的 system、filesystem 模块），你可以通过此选项指定只查找需要的部分。

举例：

```
find_package(Boost 1.70 REQUIRED COMPONENTS system filesystem)
```

查找 Boost 库的版本 1.70，并确保 system 和 filesystem 两个组件被找到。如果找不到这些组件，CMake 将报错并停止构建。

# 3. 常用的宏

| 宏                          | 功能                                                         |
| --------------------------- | ------------------------------------------------------------ |
| PROJECT_NAME                | 用函数project(demo)指定的项目名称                            |
| PROJECT_SOURCE_DIR          | 一般为工程的根目录，指向包含最近一次调用project()命令定义的项目的源目录 |
| PROJECT_BINARY_DIR          | 执行cmake命令的目录，一般为build                             |
| CMAKE_CURRENT_SOURCE_DIR    | 当前处理的CMakeLists.txt文件所在目录                         |
| CMAKE_CURRENT_BINARY_DIR    | 当前处理的二进制输出目录，也就是 CMake 将生成构建文件（例如 Makefile 或其他生成器文件）的目录 |
| CMAKE_CURRENT_LIST_DIR      | CMakeLists.txt的完整路径                                     |
| CMAKE_CURRENT_LIST_LINE     | 当前所在行                                                   |
| EXECUTABLE_OUTPUT_PATH      | 重新定义目标二进制可执行文件的存放位置                       |
| LIBRARY_OUTPUT_PATH         | 重新定义目标链接库文件的存放位置                             |
| CMAKE_CXX_STANDARD          | 指定C++标准                                                  |
| CMAKE_CXX_STANDARD_REQUIRED | 如果编译不符合指定C++标准，编译                              |

# 4. 示例

这是我们之前在linux上编译grpc通信代码用到的CMakeLists.txt文件，现在可以分析每条指令的作用是什么：

```yaml
# 指定cmake版本最低为3.1
cmake_minimum_required(VERSION 3.1)
# 项目名称为GrpcServer，编程语言C++
project(GrpcServer LANGUAGES CXX)
# 将 CMAKE_CURRENT_SOURCE_DIR 和 CMAKE_CURRENT_BINARY_DIR 自动添加到包含路径中，这些目录下的头文件被自动包含
set(CMAKE_INCLUDE_CURRENT_DIR ON)
# C++版本17
set(CMAKE_CXX_STANDARD 17)
# 如果不满足C++17，强制编译失败
set(CMAKE_CXX_STANDARD_REQUIRED ON)
# 查找Threads 库，如果找不到该包，编译停止并报错
find_package(Threads REQUIRED)
# 使用模块查找的规则，优先使用 CMake 自带的 FindProtobuf.cmake 文件，而不是使用配置模式（即通过 protobuf-config.cmake 文件进行查找）
set(protobuf_MODULE_COMPATIBLE TRUE)
# 查找protobuf库，并要求找到的是基于CONFIG模式的配置文件(protobuf-config.cmake)，且找不到会报错
find_package(Protobuf CONFIG REQUIRED)
# 输出protobuf版本号
message(STATUS "Using protobuf ${Protobuf_VERSION}")
# 将protobuf的核心库libprotobuf赋值给变量_PROTOBUF_LIBPROTOBUF 
set(_PROTOBUF_LIBPROTOBUF protobuf::libprotobuf)
# 将grpc的反射库赋值给_REFLECTION 
set(_REFLECTION gRPC::grpc++_reflection)
# Find gRPC installation
# Looks for gRPCConfig.cmake file installed by gRPC's cmake installation.
find_package(gRPC CONFIG REQUIRED)
message(STATUS "Using gRPC ${gRPC_VERSION}")

set(_GRPC_GRPCPP gRPC::grpc++)
# 添加可执行文件和源文件
file(GLOB SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/*.cpp)
file(GLOB PBSOURCES ${CMAKE_CURRENT_SOURCE_DIR}/*.cc)
# 生成可执行程序GrpcServer 
add_executable(GrpcServer ${SOURCES}
        ${PBSOURCES})
# 链接库动态库文件
target_link_libraries(GrpcServer
    ${_REFLECTION}
    ${_GRPC_GRPCPP}
    ${_PROTOBUF_LIBPROTOBUF})
```

