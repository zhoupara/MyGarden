---
title: IO相关函数 func_io 
date: 2020-09-30 14:51:53
tags:
---
# 标准IO和系统IO的效率
1.本文所讲的系统IO为 UNIX环境下的系统IO函数
2.标准IO 创建了一个缓冲区，当刷新缓冲区的后，在调用系统IO进行IO操作。
3.标准IO兼容了各种版本的系统IO
4.标准IO的 缓冲区(行缓冲、全缓冲，不带缓冲)
5.当且仅当标准输入和标准输出并不涉及交互作用设备时，他们才是全缓冲
<!--more-->

# 系统调用函数 
-->open,close,lseek,read,write

## open
```
int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode);

头文件：
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

参数
const char *pathname  要打开或创建的文件的名字
int flags
    O_RDONLY 只读
    O_WRONLY 只写
    O_RDWR 可读可写
可选
    O_APPEND  每次写时都加到文件的尾端
    O_CREAT    如此文件不存在则创建它，需与第三参数mode,说明新文件的存取全蝎
        umask --> ~umask | mode -->权限
    O_EXCL  如果同时指定了O_CREAT, 而文件已经存在，则报错，可测试一个文件是否存在，
            如果不存在则创建此文件成文一个原子操作
    O_TRUNC 如果文件存在，而且为只读或只写成功打开，则将其长度截短为0
    O_NOCTTY 如果pathname指的是终端设备，则不将此设备分配为此进程的控制终端
    O_NONBLOCK 如果pathname指的是一个FIFO、一个块特殊文件或一个字符特殊文件，则此选项为此
            文件的本次打开操作和后续的I/O操作设置非阻塞方式

返回值
```
fopen 和 open 对比
```
FILE *fopen(const char *path, const char *mode);
int open(const char *pathname, int flags);
r    只读 | 文件首                                        -->O_RDONLY  
r+   读写 | 文件首                                        -->O_RDWR
w    有则清空，无则创建 | 只写 | 文件首                     -->O_WRONLY | O_CREAT | O_TRUNC
w+   读写 有则清空，无则创建 | 文件首                       -->O_RDWR | O_CREAT | O_TRUNC
a    有则清空，无则创建 | 追加 | 读写 | 写在文件尾           -->O_RDWR | O_CREAT | O_TRUNC | O_APPEND
a+   读写  有则清空，无则创建 读在文件头、写在文件尾          -->O_RDWR | O_CREAT | O_TRUNC
```

## close
```
#include <unistd.h>
int close(int fd);
```
## read
```
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t count);

返回值：
On success, the number of bytes read is returned (zero indicates end of file), and the file position is  advanced  by this  number.
```
## write
```
#include <unistd.h>
ssize_t write(int fd, const void *buf, size_t count);

返回值：
On success, the number of bytes written is returned (zero indicates nothing was written).  On error, -1 is returned, and errno is set appropriately.
```
## lseek
```
#include <sys/types.h>
#include <unistd.h>
off_t lseek(int fd, off_t offset, int whence);
参数：
int whence SEEK_SET | SEEK_CUR | SEEK_END
返回值：
returns the resulting offset location as measured in bytes from the beginning of the file.  On error, the  value  (off_t) -1  is  returned  and errno is set to indicate the error.
```

# 标准IO操作函数
--> fopen、fclose、
--> getc、putc、fgetc、fputc、getchar
--> gets、puts、fgets、fputs
-->fread、fwrite
--> ftell、fseek、rewind

## fopen 、fclose
```
#include <stdio.h>
FILE *fopen(const char *path, const char *mode);
r    只读 | 文件首
r+   读写 | 文件首
w    有则清空，无则创建 | 只写 | 文件首
w+   读写 有则清空，无则创建 | 文件首
a    有则清空，无则创建 | 追加 | 读写 | 写在文件尾
a+   读写  有则清空，无则创建 读在文件头、写在文件尾

#include <stdio.h>
int fclose(FILE *fp);
```
## fgetc、getc、getchar、fputc、putc、putchar
```
#include <stdio.h>
int fgetc(FILE *stream);
int getc(FILE *stream);
int getchar(void);   ---> equivalent to getc(stdin)
getc 与 fgetc 区别
getc 宏定义  --> _IO_getc
返回值 return the character read as an unsigned char cast to an int or EOF on end of file or error.
-----------------------------------------------------
int fputc(int c, FILE *stream);
int putc(int c, FILE *stream);
int putchar(int c);
返回值 return the character written as an unsigned char cast to an int or EOF on error.
```

## fgets、gets、 fput、puts
```
#include <stdio.h>
char *fgets(char *s, int size, FILE *stream);  读取size-1大小的字符长度 第size 用'\0'补充
char *gets(char *s);  有漏洞，慎用
返回值 return s on success, and NULL on error or when end of file occurs while no characters have been read.
-------------------------------------------
int fputs(const char *s, FILE *stream);  writes the string s to stream, without its terminating null byte ('\0').将字符串的'\0'去掉输出
int puts(const char *s);  writes the string s and a trailing newline to stdout.
返回值 return a nonnegative number on success, or EOF on error
```

## fread、fwrite
```
#include <stdio.h>
size_t fread(void *ptr, size_t size, size_t nmemb, FILE *stream);
size_t fwrite(const void *ptr, size_t size, size_t nmemb, FILE *stream);
返回值 返回读取的次数
return the number of items successfully read or written
If an error occurs, or the end-of-file is reached, the return value is  a  short  item  count
```

## ftell、fseek、rewind
```
#include <stdio.h>
int fseek(FILE *stream, long offset, int whence);  移动位置
long ftell(FILE *stream);    返回文件指针位置
void rewind(FILE *stream);   equivalent (void) fseek(stream, 0L, SEEK_SET)
```

## fseeko、ftello
```
#include <stdio.h>
int fseeko(FILE *stream, off_t offset, int whence); 移动位置
off_t ftello(FILE *stream); 返回文件指针位置

On many architectures both off_t and long are 32-bit types, but compilation with
#define _FILE_OFFSET_BITS 64
makefile:
CFLAGS+=-D_FILE_OFFSET_BITS=64

CONFORMING TO
       SUSv2, POSIX.1-2001.

fseek(stream,0,SEEK_END);       
```

# 附加函数
## getline
```
#define _GNU_SOURCE
#include <stdio.h>
ssize_t getline(char **lineptr, size_t *n, FILE *stream);
makefile
CFLAGS+=-D_GNU_SOURCE
```

## 临时文件
```
#include <stdio.h>
char *tmpnam(char *s);
FILE *tmpfile(void);
```

## fflush
```
#include <stdio.h>
int fflush(FILE *stream);
返回值
Upon successful completion 0 is returned.  Otherwise, EOF is returned and  errno  is  set  to  indicate  the
error.
```