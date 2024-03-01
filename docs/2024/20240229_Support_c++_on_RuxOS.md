# 在RuxOS上支持c++
时间：2024/2/29

作者：徐金阳

摘要：介绍了在RuxOS上解决链接时c++标准库符号找不到的方法，从而可以在RuxOS上编译运行c++应用程序。

## 问题描述

在RuxOS上编译应用程序时，以x86_64架构下的[wamr](https://github.com/syswonder/rux-wamr)为例，编译通过，生成`wamr.o`目标文件，但在链接生成`wamr_x86_64_qemu-q35.elf`时显示`undefined reference`，其中未定义的符号都是`std`、`new`、`delete`等c++相关的函数。

## 解决方法

找到使用的x86_64-linux-musl-gcc编译器的根目录，例如`/opt/x86_64-linux-musl-cross/`，找到其中的`x86_64-linux-musl/lib/libstdc++.a`，将其加入到链接应用程序的命令中。即直接将该静态库链接到应用程序目标文件`wamr.o`，再与RuxOS的符号和musl libc的符号链接生成`.elf`文件。

若链接时还有`Unwind`相关的符号显示`undefined reference`，这是c++中异常处理相关的符号。可将`lib/gcc/x86_64-linux-musl/11.2.1/`（版本号`11.2.1`可能不同）目录下的`libgcc_eh.a`链接进来。

若链接时还出现符号`__clrsbsi2`未定义，此时还需要链接`libgcc.a`中的`_clrsbsi2.o`目标文件，可以用如下命令将该目标文件从`libgcc.a`中提取出来：

```bash
x86_64-linux-musl-ar x /opt/x86_64-linux-musl-cross/x86_64-linux-musl/lib/libgcc.a _clrsbsi2.o
```

再将`_clrsbsi2.o`链接到应用程序目标文件`wamr.o`。

若链接时还有`__dso_handle`未定义，这是由 GCC 编译器生成的一种用于处理 C++ 全局对象析构的内部符号。可在main函数之前手动加上如下定义：

```c++
extern "C" {
__attribute__((weak)) void *__dso_handle;
}
```

## 局限性

只能静态链接`libstdc++.a`。

## 支持的c++版本

取决于编译器`x86_64-linux-musl-gcc`提供的`libstdc++.a`静态库的c++版本。

可通过`x86_64-linux-musl-gcc -v`查看`musl gcc`编译器版本。各c++版本和编译器版本关系如下：

1. C++20：gcc8。
2. C++17：gcc7 完全支持，gcc6 和 gcc5 部分支持。
3. C++14：gcc5 完全支持，gcc4 部分支持。
4. C++11：gcc4.8.1 及以上可以完全支持，gcc4.3 部分支持，gcc4.3 以下版本不支持。

或可通过以下命令逐个检查各版本是否被支持：

```bash
x86_64-linux-musl-g++ -std=c++11 -E - < /dev/null
x86_64-linux-musl-g++ -std=c++14 -E - < /dev/null
x86_64-linux-musl-g++ -std=c++17 -E - < /dev/null
# 若GCC版本在9及以下则用-std=c++2a
x86_64-linux-musl-g++ -std=c++20 -E - < /dev/null
x86_64-linux-musl-g++ -std=c++23 -E - < /dev/null
```

若不支持则显示`error: unrecognized command-line option '-std=c++版本号'`。测试时11.2.1版本的`x86_64-linux-musl-gcc`支持c++23。

## 标准模板库（STL）支持情况

支持。

`libstdc++.a`中有各模板库符号的定义，`x86_64-linux-musl/include/c++/11.2.1`目录下也有各模板库的头文件所以支持`std::list`等STL的内容。

## 示例

见`apps/c/cpp`目录下的c++应用程序，运行如下命令即可编译运行：

```bash
make A=apps/c/cpp ARCH=aarch64 LOG=info SMP=4 run MUSL=y
```
