---
title: Mach-O文件分析
date: 2020-05-22 14:14:46
tags:
categories:
	- iOS
---

### Mach-O文件

`Mach-O` 其实是 `Mach Object` 文件格式的缩写，是 `mac` 以及 `iOS` 上可执行文件的格式， 类似于 `windows` 上的 `PE` 格式 ( Portable Executable ) , `linux` 上的 `elf` 格式 ( Executable and Linking Format )  .

它是一种用于可执行文件、目标代码、动态库的文件格式。作为 `a.out` 格式的替代，`Mach-O` 提供了更强的扩展性。

但是除了可执行文件外 , 其实还有一些文件也是使用的 `Mach-O` 的文件格式 .

属于 `Mach-O` 格式的常见文件

> - 目标文件 .o
> - 库文件
>   - .a
>   - .dylib
>   - Framework
> - 可执行文件
> - dyld ( 动态链接器 )
> - .dsym ( 符号表 )

`Mach-O` 并非一定是可执行文件 , 它是一种文件格式 , 分为 `Mach-O Object` 目标文件 、 `Mach-O ececutable` 可执行文件、 `Mach-O dynamically` 动态库文件、 `Mach-O dynamic linker` 动态链接器文件、 `Mach-O dSYM companion` 符号表文件 , 等等 .

### Mach-O文件结构

具体可以分为几个部分

文件头 mach64 Header
加载命令 Load Commands
文本段 __TEXT
数据段 __Data
动态库加载信息 Dynamic Loader Info
入口函数 Function Starts
符号表 Symbol Table
动态库符号表 Dynamic Symbol Table
字符串表 String Table
![](1.jpg)

![](2.jpg)

### Load Commands - 加载命令

Mach-O文件包含非常详细的加载指令，这些指令非常清晰地指示加载器如何设置并且加载二进制数据。Load Commands紧紧跟着二进制文件头。

![](3.png)

### 参考地址

MachO 文件结构详解：https://juejin.im/post/5c67e7efe51d45164c75993b#heading-5

Mach-O文件格式和程序从加载到执行过程：https://blog.csdn.net/bjtufang/article/details/50628310

趣探 Mach-O：加载过程：https://juejin.im/post/5a0c5c3451882554bd509a46#heading-4

