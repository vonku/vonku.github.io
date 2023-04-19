---
title: ChChorOS-1-GDB
date: 2023-04-19 22:34:45
categories: OS
tags:
---


### 前言
> 本章是 AArch64 qemu gdb 调试的基础。

<!--more-->

#### 启动调试

```
sudo make qemu-gdb
sudo make gdb
```

#### gdb调试命令

**设置断点**
```
b stack_test
```
**查看断点**
```
info breakpoint
```
**清除断点**

```
clear location
```
location 为某一行代码的行号或者某个具体的函数名。
```
delete [breakpoints][num]
```
删除某一个特定的断点或全部断点。
**禁用/使能断点**

```
disable [breakpoints][num]
enable [breakpoints][num]
```
**查看变量**

```
info variables
```
**查看寄存器**

```
info register sp
info register fp  // $x29
```
**查看内存**

```
x/32xg $x29
```
命令格式：  x /nfu <addr>

- n 是要显示内存单元的个数
- f 表示显示格式
  - x 十六进制
  - d 十进制
  - u 十进制无符号整型
  - o 八进制
  - t 二进制
  - i 指令地址格式
  - c 字符串格式
  - f 浮点格式

- u 一个地址单元的长度
  - b/h/w/g，分别表示单字节、双字节、四字节、八字节

#### 二进制分析

**查看ELF头部信息**

```
readelf -h build/kernel.img
```
**查看ELF包含的程序段**

```
readelf -S build/kernel.img
```
**查看ELF的LMA和VMA**

```
objdump -h build/kernel.img
```
**汇编单步调试**

```
si
```
**实时查看寄存器变化**

```
打开gdb之后运行 layout regs
```