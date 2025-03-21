---
title: fileno
date: 2020-09-24 19:33:36
tags: fileno
---

# 文件描述符

## 文件描述符的描述
```
a  file descriptor, a small, nonnegative integer for use in 
subsequent system calls (read(2), write(2), lseek(2), fcntl(2), etc.)  

 --from man 2.6.3
```
关键词:
nonnegative integer  非负整数
use in system calls     系统调用使用

当打开或者创建一个新的文件时，内核向进程返回一个文件描述符
用此文件描述符就能读写文件
linux一切皆文件，拿到文件描述符，就拿到了linux的钥匙
<!--more-->

## 标准文件描述符
标准输出文件描述符、标准输入文件描述符、标准错误文件描述符
```
/* Standard file descriptors.  */
#define STDIN_FILENO    0       /* Standard input.  */
#define STDOUT_FILENO   1       /* Standard output.  */
#define STDERR_FILENO   2       /* Standard error output.  */

---from <unistd.h>
```

## 文件描述符的限制 
使用ulimit -n 进行文件描述符设置当前进程
```
[para ~]$ ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 7271
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 65535           --> 文件描述符限制 65535个
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 4096
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```

设置永久生效最大文件描述符限制
```
修改 /etc/security/limits.conf

#End of file
root soft nofile 65535
root hard nofile 65535
para soft nofile 1024
para hard nofile 1024
* soft nofile 65535
* hard nofile 65535

[para ~]$ ulimit -n
1024
```

## 文件描述符唯一性
进程PCB中维护一个文件描述符表，该表的索引值从0开始
每个进程都有自己的PCB块，当然也有属于自己的文件描述符表
![Alt text](/images/fileno/fileno0.png)

此图由unix环境高级编程书籍而来，网上说  linux系统只使用i节点，而不使用v节点 ，知道方法再细研究此问题。

(1)进程拿到文件描述符fd --> 通过文件描述符表拿到对应文件描述符指针
(2)通过指针找到文件表的偏移量，再通过文件偏移找到当前文件指针的位置
(3)在通过i节点信息去文件系统进行具体操作

## 进程结构图
![Lena](/images/fileno/fileno1.png)

