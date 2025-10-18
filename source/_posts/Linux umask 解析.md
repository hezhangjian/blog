---
title: linux umask 解析
link: linux-umask-analyze
date: 2022-11-08 19:20:53
tags:
  - Linux
---

什么是**umask**， umask即user file-creation mask. 用来控制最终创建文件的权限。

umask是进程级属性，通常是由**login shell**设置，可以通过系统调用**umask()**或者命令**umask permission**来修改，通过umask命令来查询，linux内核版本4.7之后，还可以通过**cat /proc/self/status|grep -i umask** 查询，示例如下

```bash
hezhangjian:~/masktest $ umask
0022
hezhangjian:~/masktest $ umask 0077
hezhangjian:~/masktest $ umask
0077
hezhangjian:~/masktest $ umask 0022
hezhangjian:~/masktest $ umask
0022
hezhangjian:~/masktest $
```

一般来说，umask的系统默认值在**/etc/login.defs** 中设置

```bash
hezhangjian:~ $cat /etc/login.defs|grep -i umask
#	UMASK		Default "umask" value.
# UMASK is the default umask value for pam_umask and is used by
# 022 is the "historical" value in Debian for UMASK
# If USERGROUPS_ENAB is set to "yes", that will modify this UMASK default value
UMASK		022
# Other former uses of this variable such as setting the umask when
```

- 最常见的默认的umask值是022，目录权限755，文件权限644
- 077 的 umask 适用于私有的系统，则其他用户无法读取或写入您的数据。

针对标准函数**open**来说，最终写入磁盘的权限位是由mode参数和用户的文件创建掩码(umask)执行按位与操作而得到。

假设当**umask**为0022时，创建一个具有0666权限的文件，就会进行运算决定文件的最终权限，先对掩码取非，再和指定的权限进行binary-And操作，如图所示

![linux-umask-analyze](Images/linux-umask-analyze.png)

示例代码如下

```c
#include <stdlib.h>
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    int fd;
    if (argc != 2) {
        fprintf(stderr, "usage: %s <file>", argv[0]);
        exit(1);
    }
    fprintf(stdout, "create file %s", argv[1]);
    fd = open(argv[1], O_WRONLY | O_CREAT | O_TRUNC, 0666);
    if (fd == -1) {
        perror("open");
        exit(1);
    }
    close(fd);
}
```

结果如下，权限644，符合预期

```
ll
total 8
drwxr-xr-x  2 hezhangjian hezhangjian 4096 Nov  8 06:25 .
drwxr-xr-x 15 hezhangjian hezhangjian 4096 Nov  8 06:18 ..
-rw-r--r--  1 hezhangjian hezhangjian    0 Nov  8 06:25 my.txt
```
