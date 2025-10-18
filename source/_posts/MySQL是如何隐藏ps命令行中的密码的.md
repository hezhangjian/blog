---
title: Mysql是如何隐藏ps命令行中的密码的
link: mysql-ps-password
date: 2021-04-26 19:35:38
tags:
  - MySQL
---

## 参考

- http://northernmost.org/blog/how-does-mysql-hide-the-command-line-password-in-ps/index.html

之前就在环境上`ps -ef`看到过`xxxxxx`的密码，一直没搞明白怎么回事，今天整理了一下，核心内容均来自于上述连接，作了一些额外的测试和查阅资料。

## 测试

### 运行Mysql实例

```bash
# 自己做的Mysql8的镜像
docker run ttbb/mysql:stand-alone
```

### 使用密码连接Mysql服务器

```bash
mysql -u hzj -p Mysql@123 -e "select 1"
```

### ps -ef查看

```
[root@91bcbd15a82e mysql]# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 07:34 ?        00:00:00 /usr/local/bin/dumb-init bash -vx /opt/sh/mysql/hzj/scripts/start.sh
root         8     1  0 07:34 ?        00:00:00 bash -vx /opt/sh/mysql/hzj/scripts/start.sh
root        17     1  0 07:34 ?        00:00:00 mysqld --daemonize --user=root
root        62     8  0 07:34 ?        00:00:00 tail -f /dev/null
root        63     0  0 07:34 pts/0    00:00:00 bash
root        98    63  0 07:37 pts/0    00:00:00 mysql -h 127.0.0.1 -u hzj -px xxxxxxx
root        99     0  1 07:37 pts/1    00:00:00 bash
root       122    99  0 07:37 pts/1    00:00:00 ps -ef
```

## Mysql隐藏密码原理
改写了`args`系统参数，demo如下
```c
//
// Created by 张俭 on 2021/4/26.
//
#include <stdio.h>
#include <unistd.h>
#include <string.h>

int main(int argc, char *argv[]) {
    int i = 0;
    pid_t mypid = getpid();
    if (argc == 1)
        return 1;
    printf("argc = %d and arguments are:\n", argc);
    for (i; i < argc; i++) {
        printf("%d = %s\n", i, argv[i]);
    }
    fflush(stdout);
    sleep(30);
    printf("Replacing first argument with x:es... Now open another terminal and run: ps p %d\n", (int)mypid);
    memset(argv[1], 'x', strlen(argv[1]));
    getc(stdin);
    return 0;
}

```

编译并运行

```bash
gcc password_hide.c
[root@c77dc365cd1a sh]# ./a.out abcd
argc = 2 and arguments are:
0 = ./a.out
1 = abcd
Replacing first argument with x:es... Now open another terminal and run: ps p 55

```
观测结果，开始看的确有明文密码
```
[root@c77dc365cd1a sh]# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 07:49 pts/0    00:00:00 bash
root        32     0  0 07:51 pts/1    00:00:00 bash
root        64     1  0 07:56 pts/0    00:00:00 ./a.out abcd
root        66    32  0 07:56 pts/1    00:00:00 ps -ef
```
经过30秒后，已经被复写
```
[root@c77dc365cd1a sh]# ps p 55
  PID TTY      STAT   TIME COMMAND
   55 pts/0    S+     0:00 ./a.out xxxx
```
## Mysql源码地址

mysql-server/client/mysql.cc line 2054

```
      if (argument) {
        char *start = argument;
        my_free(opt_password);
        opt_password = my_strdup(PSI_NOT_INSTRUMENTED, argument, MYF(MY_FAE));
        while (*argument) *argument++ = 'x';  // Destroy argument
        if (*start) start[1] = 0;
        tty_password = false;
      } else
        tty_password = true;
```

PS: 后面，我还在OSX上用go程序尝试修改参数，估摸go程序的args传入是值拷贝，修改完成之后args没有生效，看来这个黑科技只有c程序能使用呀。
