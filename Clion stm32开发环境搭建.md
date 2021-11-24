# Clion stm32开发环境搭建

> https://zhuanlan.zhihu.com/p/145801160

## 准备

### 软件：

OpenOCD：https://gnutoolchains.com/arm-eabi/openocd/

arm-none-eabi-gcc：https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm/downloads

## 一些名词

**工具链：**

> https://www.cnblogs.com/lvdongjie/p/6835048.html

**bin文件夹：**bin即binary，该文件夹存放二进制文件

## 踩的一些坑

### 4k屏如何字体太小怎么办？

修改对象软件的dpi设置，右键属性兼容性高dpi设置为系统（增强）

DPI：Dots Per Inch, 每英寸点数

### Board config file如何编写？

<openocd>\share\openocd\scripts\里有官方提供的cfg文件，只需要选择target和interface，复制来用即可

```
# 最简单的配置方法

# 选择下载器
source [find interface/stlink.cfg]
transport select hla_swd
# 选择板子
source [find target/stm32f1x.cfg]
adapter speed 10000
```

### Debug模式用不了？

2021.10.14已解决

> 参考：https://youtrack.jetbrains.com/issue/CPP-23587

原报错信息：

```
Unexpected command line argument: Files\OpenOCD\OpenOCD-20200729-0.10.0\share\op
enocd\scripts
```

原因是openOCD的script目录名称中不允许出现空格，而Program Files文件夹中有空格

- 方法一：

  > Use the following notations:
  >
  > - For "**C:\Program Files**", use "**C:\PROGRA~1**"
  > - For "**C:\Program Files (x86)**", use "**C:\PROGRA~2**"
  >
  > Thanks @lit for your ideal answer in below comment:
  >
  > > Use the environment variables **%ProgramFiles%** and **%ProgramFiles(x86)%**

- 方法二：把openocd移到不含空格的文件夹（有效）

记得修改文件夹后要把系统Path变量对应修改，然后重启clion

### printf()函数的重定向问题

Keil软件里直接重写fputc()

> ```c
> int fputc (int ch, FILE *f)
> 
> {
> 
> 	(void)HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, 1000);
> 
> 	return ch;
> 
> }
> ```
>
> 其中的FILE定义在stdio.h头文件中，所以需要在项目中包含这个头文件，但是经过测试发现，Keil里面包含的是MDK\ARM\ARMCC\include这个目录下的stdio.h，而在Clion中是不会链接到这个文件的。因此如果在Clion中也按之前的方法进行重定向，会发现printf没有任何输出。
>
> 在Clion中链接的是GNU-Tools-ARM-Embedded\arm-none-eabi\include里面的stdio.h

创建retarget.c和retarget.h（注意一定得是.c文件）

retarget.h

```c
#ifndef SELF_BALANCE_CAR_RETARGET_H
#define SELF_BALANCE_CAR_RETARGET_H

#include "stm32f1xx_hal.h"
#include <sys/stat.h>
#include <stdio.h>

// C和C++对于函数名称的处理不同，所以为了方便给C++调用，需要加入extern "C"，但是源文件本身是C不需要该代码，所以就加入了#ifdef __cplusplus
#ifdef __cplusplus
extern "C" {
#endif
void RetargetInit(UART_HandleTypeDef *huart);
#ifdef __cplusplus
}
#endif


int _isatty(int fd);

int _write(int fd, char *ptr, int len);

int _close(int fd);

int _lseek(int fd, int ptr, int dir);

int _read(int fd, char *ptr, int len);

int _fstat(int fd, struct stat *st);

#endif //SELF_BALANCE_CAR_RETARGET_H
```

retarget.c

```c
#include "retarget.h"

#include <_ansi.h>
#include <_syslist.h>
#include <errno.h>
#include <sys/time.h>
#include <sys/times.h>
#include <retarget.h>
#include <stdint.h>

#if !defined(OS_USE_SEMIHOSTING)

#define STDIN_FILENO  0
#define STDOUT_FILENO 1
#define STDERR_FILENO 2

UART_HandleTypeDef *gHuart;

void RetargetInit(UART_HandleTypeDef *huart) {
    gHuart = huart;

    /* Disable I/O buffering for STDOUT stream, so that
     * chars are sent out as soon as they are printed. */
    setvbuf(stdout, NULL, _IONBF, 0);
}

int _isatty(int fd) {
    if (fd >= STDIN_FILENO && fd <= STDERR_FILENO)
        return 1;

    errno = EBADF;
    return 0;
}

int _write(int fd, char *ptr, int len) {
    HAL_StatusTypeDef hstatus;

    if (fd == STDOUT_FILENO || fd == STDERR_FILENO) {
        hstatus = HAL_UART_Transmit(gHuart, (uint8_t *) ptr, len, HAL_MAX_DELAY);
        if (hstatus == HAL_OK)
            return len;
        else
            return EIO;
    }
    errno = EBADF;
    return -1;
}

int _close(int fd) {
    if (fd >= STDIN_FILENO && fd <= STDERR_FILENO)
        return 0;

    errno = EBADF;
    return -1;
}

int _lseek(int fd, int ptr, int dir) {
    (void) fd;
    (void) ptr;
    (void) dir;

    errno = EBADF;
    return -1;
}

int _read(int fd, char *ptr, int len) {
    HAL_StatusTypeDef hstatus;

    if (fd == STDIN_FILENO) {
        hstatus = HAL_UART_Receive(gHuart, (uint8_t *) ptr, 1, HAL_MAX_DELAY);
        if (hstatus == HAL_OK)
            return 1;
        else
            return EIO;
    }
    errno = EBADF;
    return -1;
}

int _fstat(int fd, struct stat *st) {
    if (fd >= STDIN_FILENO && fd <= STDERR_FILENO) {
        st->st_mode = S_IFCHR;
        return 0;
    }

    errno = EBADF;
    return 0;
}


#endif //#if !defined(OS_USE_SEMIHOSTING)
```

然后把syscalls.c中重复定义的函数注释掉即可

### 新建的文件没办法被cmake识别？

新建source文件时选择.c/.h文件，若选择.cpp则无法识别

### undefined reference to

如果main文件是.c，则无法引用c++，但是反过来，c++文件可以引用c语言的，所以我们需要把main.c改为main.cpp

### 如何改成c++模式？

> https://blog.csdn.net/PoJiaA123/article/details/81632106

有时候遇到某个文件内没有代码提示功能，可以在Clion中点击Tools->CMake->Reset Cache and Reload Project

## Clion创建Doxygen注释

> https://www.jetbrains.com/help/clion/creating-and-viewing-doxygen-documentation.html#create-doxygen-comments
>
> ## Creating comments from scratch﻿
>
> To create a Doxygen comment from scratch:
>
> 1. Type one of the following symbols: `///`, `//!`, `/**` or `/*!` and press Enter.
>
> 2. You will get a stub to fill with the documentation text:
>
>    ![Generate comments](https://resources.jetbrains.com/help/img/idea/2021.2/cl_doxygen_stub.png)

Doxygen的注释规范

> https://www.jianshu.com/p/9464eca6aefe
