---
title: Valgrind内存泄漏检测快速上手
link:
date: 2021-10-17 16:13:57
tags:
---

## 准备被测程序

通过`-g`编译程序来携带debug信息，这样子输出的错误信息就可以包含精确的行号。如果你可以承受程序运行缓慢，那么我们可以使用`-O0`来编译程序。如果使用`-O1`，那么输出的行号可能会不准确。不推荐使用`-O2`及以上，因为 `valgrind memcheck` 偶尔会报告不存在的未初始化值错误。

## 运行程序

如果平时这么运行

```bash
myprog arg1 arg2
```

就使用这个命令

```bash
valgrind --leak-check=yes myprog arg1 arg2
```

`Memcheck`是默认的`valgrind`工具，`--leak-check`打开了内存泄漏检测开关。

通过这个命令运行，大约会比平时运行慢20到30倍，并且使用更大的内存。Memcheck 将发出它检测到的内存错误和泄漏的信息。

## 解释`memcheck`的输出

使用样例的C程序，包含一个内存分配错误和内存溢出

```c
#include <stdlib.h>

void f(void)
{
   int* x = malloc(10 * sizeof(int));
   x[10] = 0;        // problem 1: heap block overrun
}                    // problem 2: memory leak -- x not freed

int main(void)
{
   f();
   return 0;
}
```

执行内存检测

```bash
gcc -g demo.c
valgrind ./a.out
```

输出信息如下

```
==145== Memcheck, a memory error detector
==145== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==145== Using Valgrind-3.17.0 and LibVEX; rerun with -h for copyright info
==145== Command: ./a.out
==145==
==145== Invalid write of size 4
==145==    at 0x401144: f (demo.c:6)
==145==    by 0x401155: main (demo.c:11)
==145==  Address 0x4a27068 is 0 bytes after a block of size 40 alloc'd
==145==    at 0x484086F: malloc (vg_replace_malloc.c:380)
==145==    by 0x401137: f (demo.c:5)
==145==    by 0x401155: main (demo.c:11)
==145==
==145==
==145== HEAP SUMMARY:
==145==     in use at exit: 40 bytes in 1 blocks
==145==   total heap usage: 1 allocs, 0 frees, 40 bytes allocated
==145==
==145== 40 bytes in 1 blocks are definitely lost in loss record 1 of 1
==145==    at 0x484086F: malloc (vg_replace_malloc.c:380)
==145==    by 0x401137: f (demo.c:5)
==145==    by 0x401155: main (demo.c:11)
==145==
==145== LEAK SUMMARY:
==145==    definitely lost: 40 bytes in 1 blocks
==145==    indirectly lost: 0 bytes in 0 blocks
==145==      possibly lost: 0 bytes in 0 blocks
==145==    still reachable: 0 bytes in 0 blocks
==145==         suppressed: 0 bytes in 0 blocks
==145==
==145== For lists of detected and suppressed errors, rerun with: -s
==145== ERROR SUMMARY: 2 errors from 2 contexts (suppressed: 0 from 0)
```

- 145 是进程号
- 输出的第一行，即上方的第六行(`Invalid write`)表明了错误的类型。这里程序往不归它拥有的内存中写入了信息
- 下方是错误的堆栈信息
- 代码地址如`0x401155`通常并不重要，但有时候定位奇怪的问题可能会很关键
- 一些错误信息会有第二段，用来描述所涉及的内存地址。上例指出写入的内存刚好超过 example.c 的第 5 行使用 malloc() 分配的块的末尾。

### 内存泄漏信息

```
==145== 40 bytes in 1 blocks are definitely lost in loss record 1 of 1
==145==    at 0x484086F: malloc (vg_replace_malloc.c:380)
==145==    by 0x401137: f (demo.c:5)
==145==    by 0x401155: main (demo.c:11)
```

堆栈可以表明什么内存泄露了，但`memcheck`并不能告诉你内存泄漏的原因

valgrind会提示两种内存泄漏

- "definitely lost": 绝对泄漏了内存，必须修复
- "probably lost":  程序可能泄漏了内存，也有可能是一些特定的指针操作（如：指针放到了堆中）

## 参考

https://www.valgrind.org/docs/manual/quick-start.html#quick-start.intro

https://www.valgrind.org/docs/manual/mc-manual.html#mc-manual.errormsgs
