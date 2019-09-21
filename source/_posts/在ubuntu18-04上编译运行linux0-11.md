---
title: 在ubuntu18.04上编译运行linux0.11
date: 2019-09-21 09:07:43
tags: linux0.11 kernal ubuntu
---

>本文的目的是在Ubuntu18.04上编译运行Linux0.11。

系统是虚拟机下的ubuntu18.04LTS，主要软件环境是Bochs+gcc+vim+Linux 0.11 源代码。

后续希望能编写自己的应用程序，修改Linux 0.11的源代码，用gcc编译后，在Bochs的虚拟环境中运行、调试目标代码，在实践中加深对Linux操作系统的理解。

话不多说，现在开始吧。

<!--more-->

## 1. 准备环境
本文主要参考了哈工大的[《操作系统之基础》][link1]的课程，该课程提供了基本的实验环境压缩包**hit-oslab**，下载链接: http://pan.baidu.com/s/1rP0Bc1_DUVCL-7_g_YJHeQ 提取码: u3nb。

``` shell
# 下载hit-oslab-linux-20110823.tar.gz到个人目录/home/vonku，进入该目录 
$ cd /home/vonku/

# 解压
$ tar -zxvf hit-oslab-linux-20110823.tar.gz 

# 查看是否解压成功
$ ls -al
# 解压后得到的oslab就是实验环境的主文件夹
```

现在我们查看一下文件结构：

![abc](在ubuntu18-04上编译运行linux0-11/1.png)

- Image文件

oslab工作在一个宿主操作系统之上，我们使用的是Ubuntu18.04，我们在宿主操作系统上完成对Linu0.11的开发、修改和编译之后，在linux-0.11目录下会生成一个名为Image文件，这个文件就是编译之后的目标文件。

该文件已经包含引导和所有内核的二进制代码。如果拿来一张软盘，从它的0上去扇区开始，逐字节写入Image文件的内容，就可以从这张软盘启动一台真正的计算机，并进入Linux0.11内核。

>**oslab**采用bochs模拟器加载这个Image文件，模拟执行Linux0.11。

- bochs

bochs 目录下是与 bochs 相关的执行文件、数据文件和配置文件。

- run 脚本
run 是运行 bochs 的脚本命令。

运行后 bochs 会自动在它的虚拟软驱 A 和虚拟硬盘上各挂载一个镜像文件，软驱上挂载是 linux-0.11/Image，硬盘上挂载的是 hdc-0.11.img。

因为 bochs 配置文件中的设置是从软驱 A 启动，所以 Linux 0.11 会被自动加载。

而 Linux 0.11 会驱动硬盘，并 mount 硬盘上的文件系统，也就是将 hdc-0.11.img 内镜像的文件系统挂载到 0.11 系统内的根目录 —— /。在 0.11 下访问文件系统，访问的就是 hdc-0.11.img 文件内虚拟的文件系统。

- hdc-0.11.img 文件
hdc-0.11.img 文件的格式是 Minix 文件系统的镜像。

Linux 所有版本都支持这种格式的文件系统，所以可以直接在宿主 Linux 上通过 mount 命令访问此文件内的文件，达到宿主系统和 bochs 内运行的 Linux 0.11 之间交换文件的效果。hdc-0.11.img 内包含有：

```
- Bash shell；
- 一些基本的 Linux 命令、工具，比如 cp、rm、mv、tar；
- vi 编辑器；
- gcc 1.4 编译器，可用来编译标准 C 程序；
- as86 和 ld86；
- Linux 0.11 的源代码，可在 0.11 下编译，然后覆盖现有的二进制内核。
```
## 2. 编译内核
#### step1. 安装8086 16位编译链接器
 首先进入到linux-0.11目录，然后执行make命令：
```shell
$ cd ./linux-0.11/
$ make all
```
本以为直接make，内核就编译好了。但是事情总是没那么简单...

make之后，系统提示:
```
make as86 : 命令未找到
```
根据提示知道，是as86没找到，需要安装as86，as86是8086(16位机器)的编译器。同样地，我们还缺少16位 x86 的链接器 ld86。

```shell
# apt-cache 可以查询和显示已安装和可安装软件包的可用信息
$ apt-cache search as86 ld86
bin86 - 16-bit x86 assembler and loader

#因此，安装 bin86 即可
$ sudo apt-get install bin86
```
#### step2. 安装gcc环境
这时候我们再make一下，然后提示
```
gcc-3.4 未找到
```
原因很简单，我们没安装gcc，或者gcc太新了。
```shell
# 再用上边的方法搜索一下
$ apt-cache search gcc-3.4
libalien-wxwidgets-perl - Perl module for locating wxwidgets binaries

# 因此安装一下 libalien-wxwidgets-perl
$ sudo apt-get install ibalien-wxwidgets-perl
```

本以为gcc-3.4可以了。但是呢，还是不行...

去网上找了一圈，在[这篇][link2]和[这篇][link3]博客上找到了解决方法。
```shell
# 整理一下步骤
$ sudo apt-get install ncurses-dev
$ sudo apt-get install bison
$ sudo apt-get install flex
$ sudo apt-get install build-essential
```
然后去官网上下载deb安装包，下载地址为 http://old-releases.ubuntu.com/ubuntu/pool/universe/g/gcc-3.4/ :
```shell
gcc-3.4-base_3.4.6-6ubuntu3_amd64.deb
gcc-3.4_3.4.6-6ubuntu3_amd64.deb
cpp-3.4_3.4.6-6ubuntu3_amd64.deb
g++-3.4_3.4.6-6ubuntu3_amd64.deb
libstdc++6-dev_3.4.6-6ubuntu3_amd64.deb
```
把这些安装包都放到同一个目录后，执行下面的命令进行安装：
```shell
$ dpkg -i *.deb
```
增加gcc-3.4的可选项，然后切换版本到gcc-3.4。
```shell
$ ls /usr/bin/gcc* -ll

# 增加 gcc 候选项
$ sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-3.4 30

# 选择gcc版本
$ sudo update-alternatives --config gcc

# 这时候再查看gcc版本号，就是gcc-3.4了
$ gcc -v
gcc version 3.4.6 (ubuntu 3.4.6-6ubuntu3)
```

现在gcc版本号对了，可以make了吧！但是...T T

提示文件丢失： `fatal error: bits/libc-header-start.h:没有那个文件或目录编译中断`

还好比较简单，这是因为gcc安装环境没有完善安装，执行以下一条指令即可：
```
sudo apt-get install gcc-multilib
```
这时候再**make**，终于可以了。linux-0.11源码终于编译完成了 T T

## 3. bochs模拟器运行linux0.11内核
编译好内核后，我们试一下启动bochs模拟器加载内核和挂在文件系统。
```shell
$ ./run
```
运行后，提示动态链接库libSM.so.6未找到。

查找一下我们本地的库：
```shell
$ ls -al /usr/lib/lib*
```
可以看到确实没有libSM.so.6的库。

对于这种库找不到的情况，我们可以通过这种方法安装：
```
$ sudo apt-get install libsm6:i386

# 其他库也类似安装
$ sudo apt-get install libx11-6:i386
$ sudo apt-get install libxpm4:i386
```

**Finally**，这时候再rum，终于可以运行linux-0.11内核了。

![abc](在ubuntu18-04上编译运行linux0-11/2.png)

接下来，自由地修改、编译linux-0.11的内核，在实践中加深对操作系统的理解吧！

Good luck!


## One More Thing
在使用linux安装一些软件和环境的时候，总是很容易出现各种报错。其实这些报错别人基本都遇到过了，仔细阅读提示信息，或者直接ctrl c + ctrl v 到google上找一下，大部分情况都是能找到比较好的解决方法的。

另外我们也要总结一下一些常见的问题比如版本不对、库缺失等等，了解常见的安装方式，如何查找安装软件(如apt-cache search xxx) ...这样我们就能慢慢地自己解决一些问题了。




[link1]:https://mooc.study.163.com/course/1000002004#/info
[link2]:https://blog.csdn.net/u014069939/article/details/90726175
[link3]:https://blog.csdn.net/littleflypig/article/details/79584349