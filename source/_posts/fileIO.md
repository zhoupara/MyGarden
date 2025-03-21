---
title: 文件IO--fileIO
date: 2020-10-12 10:58:48
tags:
---
# 文件IO简介 
1.文件IO也称不带缓冲的I/O(unbuffered)。不带缓冲指的是每个read, write都调用内核中的一个系统调用。
2.遵循POSIX相关标准，任何兼容POSIX标准的操作系统上都支持文件I/O。
3.读写文件时, 每次操作都会执行相关系统调用, 好处是直接读写实际文件，坏处是频繁的系统调用会增加系统开销。

<!--more-->

# 原子操作
--> 打开文件 + 移动到末尾
考虑一个进程，它将数据添加到一个文件尾端。早期的 unix版本并不支持open的O_APPEND选项，
```
if (lseek(fd, 0L, 2) < 0)
	err_sys("lseek error");
if (write(fd, buff, 100) != 100)
	err_sys("write error");
```
以上情况，若两个进程A和B对同一文件进行添加操作。每个进程都打开了该文件，但未使用O_APPEND标志。
每个进程都有它自己的文件表项，但是共享一个v节点表项。
1.假定进程A调用lseek，对于进程A的该文件的当前位置量设置为1500L,
2.然后内核切换进程使B进程运行。进程B执行lseek，也将该文件的当前位移量设置为1500L,
3.然后B调用write，它将B的该文件的当前位置量增至1600
4.因为文件的长度已经增加，所以内核对v节点中的当前长度更新为1600
5.然后内核又切换使用进程A恢复运行
6.当A调用write时，及从其当前文件位移量(1500)处写数据到文件中
这样就覆盖了进程B写到文件中的内容

这里的问题出在逻辑操作"定位到文件尾处，然后写"使用了两个分开的函数调用。
解决的方法是使用这两个操作对于进程而言成为一个原子操作。
unix提供了一种方法使这种操作成为原子操作，在打开文件时设置O_APPEND标志。
使内核每次对这种文件进行写之前，都将进程的当前位置量设置到该文件的尾端处，
于是在每次写之前就不再需要调用lseek

--> 创建临时文件
```
#include <stdio.h>
char *tmpnam(char *s);
描述 create a name for a temporary file

FILE *tmpfile(void);
描述 create a temporary file
```
以上两个函数都是创建一个临时文件。
tmpnam函数 得到一个临时文件名之后需要自己创建文件并打开
tmpfile函数 得到并打开一个临时文件(w+), 文件会自动删除当它关闭或者进程结束
对比两个函数，tmpfile函数操作原子化。

# 重定向 dup、dup2
## 函数 dup、dup2
```
#include <unistd.h>
int dup(int oldfd);  //uses the lowest-numbered unused descriptor for the new descriptor
int dup2(int oldfd, int newfd);  //makes newfd be the copy of oldfd, closing newfd first if necessary

//关闭 newfd前，注意：
//1.oldfd 是一个无效的文件描述符 调用失败,newfd不会关闭
//2.oldfd 是一个有效的文件描述符 且newfd和oldfd值一样，dup2什么也不做，直接返回newfd
返回值 
On success, these system calls return the new descriptor.  On error, -1 is returned, and errno is set appropriately.
```

## 输入重定向
1.命令 < 文件   将指定文件作为命令的输入设备
2.命令  << 分界符 表示从标准输入设备（键盘）中读入，直到遇到分界符才停止（读入的数据不包括分界符），这里的分界符其实就是自定义的字符串
```
命令 < 文件 
1.fd_file = open  O_RDONLY | O_EXCL 
2.fd_in = dup(0)
3.dup2(fd_file, 0)
操作
4.dup2(fd_in, 0)
5.close(fd_in)
6.close(fd_file)
```

## 输出重定向
1.命令 > 文件
3.命令 2> 文件  错误重定向 至文件
3.命令 >> 文件
4.命令  2>> 文件
5.命令 >> 文件 2>&1  // 命令 &>> 文件  // &>/dev/null

```
command > file  输出重定向 至file,以覆盖的方式
1.fd_file = open 有则清空无则创建 O_WRONLY | O_CREAT | O_TRUNC
2.fd_out = dup(1)
3.dup2(fd_file,1)
4.操作
5.dup2(fd_out,1)
6.close(fd_out)
7.close(fd_file)
```

```
command >>file 输出重定向 至file,以追加的方式 
1.fd_file = open 有则清空无则创建 O_WRONLY | O_CREAT | O_APPEND
2.fd_out = dup(1)  //复制标准输出 进行备份
3.dup2(fd_file,1)  //将标准输出的位置进行替换  
4.操作
5.dup2(fd_out,1)   //标准输出进行还原
6.close(fd_out)
7.close(fd_file)
```

```
命令 >> 文件 2>&1  以追加的方式，把正确输出和错误信息同时保存到同一个文件（file）中。
1.fd_file = open 有则清空无则创建 O_WRONLY | O_CREAT | O_APPEND
2.fd_out = dup(1)   fd_err = dup(2) //复制标准输出 进行备份
3.dup2(fd_file,1)   dup2(fd_file,2) //将标准输出的位置进行替换
4.操作
5.dup2(fd_out,1)  dup2(fd_err,2) //标准输出进行还原
6.close(fd_out) close(fd_err)
7.close(fd_file)
```

注意
1.dup使用的是系统调用IO，如果替换标准输入输出，而使用文件IO的话，会有缓冲区的问题
2.dup使用的时候，注意原子操作，其他人会操作文件描述符

## 命令 >> 文件 2>&1 的由来
命令 1>> 文件 2>&1   --->  2>&1   用文件描述符1修改文件描述符2，因此文件描述符2(错误输出)dup2(1,2) --> 错误输出和标准输出都输出至 文件中
```
[para dup]$ ls -l aaa &> file       &>  全部输出 至file
[para dup]$ ls -l aaa >> file 2>&1    先标准输出 输出至file 将1复制给1，共同输出至file
```

## 内容输出至 /dev/null
命令 >> /dev/null 2>&1 
命令 &>> /dev/null

![Lena](/images/fileIO01.png)

# 文件和设备
	--> /dev/console - 系统控制台,系统错误信息都输出到这里
    --> /dev/tty - 进程控制台，访问不同的物理设备。
    --> /dev/null - 空设备，向所有写这个设备的输出都将被丢弃
    --> /dev/zero 

