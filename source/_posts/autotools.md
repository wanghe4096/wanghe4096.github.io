---
title: 用automake构建你的c项目
---

作为Linux下的程序开发人员，大家都一定遇到过Makefile, 用make命令来编译自己写的程序确实很方便。 一般情况下， 大家都是手工写一个简单的Makefile.例如
```makefile
CC		:= gcc
C_FLAGS := -Wall -Wextra

BIN		:= bin
SRC		:= src
INCLUDE	:= include
LIB		:= lib

LIBRARIES	:= $(LIB)/ncurses-6.1

ifeq ($(OS),Windows_NT)
EXECUTABLE	:= main.exe
else
EXECUTABLE	:= main
endif

all: $(BIN)/$(EXECUTABLE)

clean:
	-$(RM) $(BIN)/$(EXECUTABLE)

run: all
	./$(BIN)/$(EXECUTABLE)

$(BIN)/$(EXECUTABLE): $(SRC)/*
	$(CC) $(C_FLAGS) -I$(INCLUDE) -L$(LIB) $^ -o $@ $(LIBRARIES)
```


本文中， 介绍如何使用autoconf和automake工具来帮助我们自动生成符合自由软件惯例的makefile. 这样可以象常见的GNU程序一样， 只要使用"./configure", "make", "make install" 就可以安装l到inux系统。 我们常用的一些开源项目都是用这套工具来管理其项目构建过程的， 如redis、openssl、libcurl、nucurses等开源项目。 
本文描述了利用autotools 来编写一个hello world的程序过程。 


<!-- more -->

# 安装构建工具
## Macos 安装autotool 工具
  $ brew install automake 

## ubuntu linux 安装autotool 工具
  $ apt-get install automake

安装完成后， 会如下工具可用
- aclocal
- autoscan
- autoconf
- autoheader
- automake


# 工具使用顺序
  ```
  automake -->  aclocal --> autoconf --> autoheader --> automake

  ```
# 快速上手

## 第一步
创建一个hello目录，并在此目录下创一个hello.c文件， 其内容如下:
```c
#include<stdio.h> 
int main()
{
    printf("hello world!\n"); 
    return 0;
}

```
## 第二步
在hello目录下运行autoscan ， 生两个文件： autoscan.log configure.scan 

## 第三步
修改configure.scan 的文件名为configure.ac 
查看configure.ac 的内容：
```bash
#                                               -*- Autoconf -*-
# Process this file with autoconf to produce a configure script.

AC_PREREQ([2.69])
AC_INIT([FULL-PACKAGE-NAME], [VERSION], [BUG-REPORT-ADDRESS])
AC_CONFIG_SRCDIR([main.c])
AC_CONFIG_HEADERS([config.h])

# Checks for programs.
AC_PROG_CC

# Checks for libraries.

# Checks for header files.

# Checks for typedefs, structures, and compiler characteristics.

# Checks for library functions.

AC_OUTPUT

```

解读上面文件内容：
```bash
# 确保使用的是足够新的Autoconf版本。 如果用于创建configure的Autoconfi版本比version要早，就在标准错误输出打印一条错误消息并不会创建configure 
AC_PREREQ([2.69])

# 初始化软件包的基本信息，包括设置包的全称，版本号以及报告bug时需要用的邮箱
AC_INIT([FULL-PACKAGE-NAME], [VERSION], [BUG-REPORT-ADDRESS])


# 用来检测所指定的源码文件是否存在，来确定源码目录的正确性
AC_CONFIG_SRCDIR([main.c])

# 用于生成config.h文件， 以便autoheader使用
AC_CONFIG_HEADERS([config.h])

# Checks for programs.
AC_PROG_CC


# Checks for libraries.

# Checks for header files.

# Checks for typedefs, structures, and compiler characteristics.

# Checks for library functions.

# 创建输出的文件。 会在`configure.ac`的末尾调用本宏一次.
AC_OUTPUT
```

编辑configure.ac文件：
1. 修改AC_INIT里面的参数： 
```basj
    AC_INIT(hello, 1.0, wanghe4096@icloud.com)
```    
2. 添加宏AM_INIT_AUTOMAKE, 它是automake 所必备的宏。 
3. 在AC_OUTPUT后添加输出文件Makefile

修改后的结果

```bash
#                                               -*- Autoconf -*-
# Process this file with autoconf to produce a configure script.

AC_PREREQ([2.69])
AC_INIT(hello, 1.0, wanghe4096@icloud.com)
AC_CONFIG_SRCDIR([hello.c])
AC_CONFIG_HEADERS([config.h])

AM_INIT_AUTOMAKE
# Checks for programs.
AC_PROG_CC

# Checks for libraries.

# Checks for header files.

# Checks for typedefs, structures, and compiler characteristics.

# Checks for library functions.

AC_OUTPUT([Makefile])
```
# 第四步：
运行aclocal,生成一个aclocal.m4文件和一个缓冲文件夹autom4te.cache，该文件主要处理本地宏定义。 


# 第五步：
运行autoconf, 生成configure脚本
此时目录下的文件有
```
aclocal.m4     autom4te.cache autoscan.log   configure      configure.ac   hello          main.c
```
# 第六步：
运行autoheader, 它负责生成config.h.in。该工具通常会从"acconfig.h"文件中复制用户附加的符号定义， 因此此处没符号定义，所以不需要创建"acconfig.h"文件 
此时目录：
```bash
aclocal.m4     autom4te.cache autoscan.log   config.h.in    configure      configure.ac   hello          main.c

```

# 第七步： 
创建Makefile.am。这一步是创建makefile很重要的一步， automake 要用的脚本配置文件是Makefile.am， 用户需要自己创建相应的文件。 之后， automake工具转换成Makefile.in


Makefile.am内容如下：
```makefile
AUTOMAKE_OPTIONS=gnu
bin_PROGRAMS=hello
hello_SOURCES=hello.c
```

下面是对这个脚本文件的解释

  - AUTOMAKE_OPTIONS为设置automake的选项
    
    由于GNU对自己发布的软件有严格的规范， 比如必须附带许可证声明文件COPYING等， 否automake执行时会报错。 automake 提供了三种软件等级： foreign 、gnu和gnits ，让用户选择采用。 默认等级为gnu. foreign等级只检测必须的文件 

  - bin_PROGRAMS定义了要产生的执行文件名。 如果产生多个执行文件，每个文件用空格隔开。 
  - hello_SOURCES 定义“hello”这个执行程序所需要的原始文件。 如果“hello“这个程序是由多个原始文件所产生的， 则必须把它所用到的所有原始文件都列出来， 并用空格隔开。 例如： 若目标体“hello”需要用"hello.c" "sum.c" "hello.h" 三个依赖文件， 则定义

    hello_SOURCES hello.c sum.c hello.h 。 

  要注意的是， 如果要定义多个执行文件， 则对每个执行程序都要定义相应的file_SOURCES. 

  其次， 使用automake 对其生成的“configure.in"文件， 在这里使用选项“--add-missing ”可以让automake 自动添加有一些必须的脚本文件。 

# 第八步：
运行configure，在这一此中，通过运行自动配置设置文件configure，把Makefile.in变成了最终的Makefile. 

运行结果如下： 
```
checking for a BSD-compatible install... /usr/bin/install -c
checking whether build environment is sane... yes
checking for a thread-safe mkdir -p... ./install-sh -c -d
checking for gawk... no
checking for mawk... no
checking for nawk... no
checking for awk... awk
checking whether make sets $(MAKE)... yes
checking whether make supports nested variables... yes
checking for gcc... gcc
checking whether the C compiler works... yes
checking for C compiler default output file name... a.out
checking for suffix of executables...
checking whether we are cross compiling... no
checking for suffix of object files... o
checking whether we are using the GNU C compiler... yes
checking whether gcc accepts -g... yes
checking for gcc option to accept ISO C89... none needed
checking whether gcc understands -c and -o together... yes
checking whether make supports the include directive... yes (GNU style)
checking dependency style of gcc... gcc3
checking that generated files are newer than configure... done
configure: creating ./config.status
config.status: creating Makefile
config.status: creating config.h
config.status: executing depfiles commands
```

# 第九步：
  运行make 

```bash
/Applications/Xcode.app/Contents/Developer/usr/bin/make  all-am
gcc -DHAVE_CONFIG_H -I.     -g -O2 -MT hello.o -MD -MP -MF .deps/hello.Tpo -c -o hello.o hello.c
mv -f .deps/hello.Tpo .deps/hello.Po
gcc  -g -O2   -o hello hello.o
```


# 第十步
$ ./hello


# 完整的项目地址

[https://github.com/wanghe4096/hello](https://github.com/wanghe4096/hello)