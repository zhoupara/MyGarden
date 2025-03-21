---
title: 标准IO--stdIO 
date: 2020-10-12 10:51:14
tags:
---
# 标准IO简介
1.标准I/O是ANSI C建立的一个标准I/O模型, 是一个标准函数包和stdio.h头文件中的定义, 不依赖系统内核, 所以移植性强。
2.遵循ANSI C相关标准。需要开发环境中有标准I/O库, 标准I/O就可以使用。
3.在文件IO的基础上封装文件描述符并维护了缓冲机制。
4.fopen函数 打开一个文件, 并且建立了一个流缓冲(用户空间，读写模式下建立两个缓冲区), 还创建了一个包含文件和缓冲区相关信息的FILE结构体, 从而先读写缓冲区, 必要时候再访问实际文件。

<!--more-->

注：
（Linux 中使用的是glibc, 它是标准C库的超集。不仅包含ANSI C中定义的函数, 还包括POSIX标准中定义的函数。因此, Linux 下既可以使用标准I/O, 也可以使用文件I/O）

# 标准IO的格式化
```
#include <stdio.h>
--标准输出
int printf(const char *format, ...);	-->标准输出
int fprintf(FILE *stream, const char *format, ...);  --> 指定的流
int sprintf(char *str, const char *format, ...);	--> 指定str(缓冲区)

--标准输出
int scanf(const char *format, ...);
int fscanf(FILE *stream, const char *format, ...);
int sscanf(const char *str, const char *format, ...);
```
格式字符串的一般形式：
[标志][输出最小宽度][.精度][长度]类型
其中方括号[]为可选项
## 类型
类型字符用以表示输出数据的类型
格式字符	意义
d		以十进制形式输出带符号整数(正数不输出符号)
o		以八进制形式输出无符号整数(不输出前缀0)
x,X		以十六进制形式输出无符号整数(不输出前缀Ox)
u		以十进制形式输出无符号整数
f		以小数形式输出单、双精度实数
e,E		以指数形式输出单、双精度实数
g,G		以%f或%e中较短的输出宽度输出单、双精度实数
c		输出单个字符
s		输出字符串

## 标志
标志字符为'-'、'+'、'#'和空格
标志 	意义
'-'		结果左对齐，右边填空格
'+'		输出符号(正号或负号)
空格		输出值为正时冠以空格，为负时冠以负号
'#'		对c、s、d、u类无影响；
		对o类，在输出时加前缀o；
		对x类，在输出时加前缀0x；
		对e、g、f 类当结果有小数时才给出小数点。

## 输出最小宽度
用十进制整数来表示输出的最少位数。若实际位数多于定义的宽度，则按实际位数输出，若实际位数少于定义的宽度则补以空格或0。

## 精度
精度格式符以“.”开头，后跟十进制整数。本项的意义是：如果输出数字，则表示小数的位数；如果输出的是字符，则表示输出字符的个数；若实际位数大于所定义的精度数，则截去超过的部分。

## 长度
长度格式符为h、l两种，h表示按短整型量输出，l表示按长整型量输出。

# 行缓冲和全缓冲(fflush) --> 面试题
```
#include <stdio.h>
#include <unistd.h>

int main(int argc, char *argv[])
{
    pid_t pid;

    printf("BEGIN");  
    
    //fflush(stdout);  --> 注释后 printf("BEGIN"); 打印了两遍
    pid = fork();
    if(pid < 0)
        return -1; 

    if(pid == 0) //子
        printf("[%d]: child\n", getpid());
    else
        printf("[%d]: parent\n", getpid());

    printf("END");
    wait();

    return 0;
}
执行 结果，BEGIN 打印两次 END 打印两次
[para fork_fflush]$ ./fork_fflush 
BEGIN[19918]: parent
BEGIN[19919]: child
ENDEND[para fork_fflush]$ 
```
标准I/O提供了三种类型的缓存：
缓存可由标准IO例程自动的刷新，或者可以调用函数fflush刷新一个流
## 全缓存
1)当填满标准IO缓存后才进行实际IO操作
2)对于驻在磁盘上的文件通常是有标准IO库实施全缓存的

## 行缓存
1)当在输入和输出中遇到新行符时, 标准IO库执行实际IO操作
2)当流涉及一个终端时(例如标准输入和标准输出), 典型地使用行缓存
3)标准IO库用来收集每一行的缓存的长度固定的, 只要填满了缓存, 即使没有写一个新行符, 也进行IO操作
4)任何时候只要通过标准输入输出库要求从一个不带缓存的流, 或者一个行缓存的流得到输入数据, 那么就会造成刷新所有行缓存输出流

## 不带缓存
1)标准IO库不对字符进行缓存, 如果用标准IO函数写若干字符到不带缓存的流中, 
2)则相当于用write系统调用函数将这些字符写至相关的打开文件上
3)标准出错流stderr通常是不带缓存的。

## ANSI C要求下列缓存特征:
1)当且仅当标准输入和标准输出并不涉及交互作用设备时，他们才是全缓存的
2)标准出错绝不会是全缓存的

## 程序问题
因为printf是标准IO函数，默认行缓冲(遇到换行符才会刷新流)
若没有进行缓冲区刷新(fflush), "BEGIN"在fork之前，并没有输出到标准输出, 
在fork的时候, 会将缓冲区的内容也复制了一份，
等程序到了 printf("[%d]: child\n", getpid()) 从缓冲区将"BEGIN"和"[%d]: child\n"进行一起打印输出

## fflush 刷新缓存区
```
#include <stdio.h>
int fflush(FILE *stream);
返回值
Upon successful completion 0 is returned.  Otherwise, EOF is returned and  errno  is  set  to  indicate  the
error.
```

# 文件描述符和FILE结构体指针转换
```
#include <stdio.h>
FILE *fdopen(int fd, const char *mode);
```