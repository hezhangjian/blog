---
title: 书写一个Linux内核模块
link:
date: 2021-03-11 19:34:52
tags:
---

## 参考

https://blog.sourcerer.io/writing-a-simple-linux-kernel-module-d9dc3762c234

# 编码

## C文件书写

首先，先书写一个C文件，命名为`kernel_first.c`

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Robert W. Oliver II");
MODULE_DESCRIPTION("A simple example Linux module.");
MODULE_VERSION("0.01");
static int __init

/**
 * Load时候触发的函数
 */
example_init(void) {
    printk(KERN_INFO
    "Hello, World!\n");
    return 0;
}

static void __exit

/**
 * Unload时候触发的函数
 */
example_exit(void) {
    printk(KERN_INFO
    "Goodbye, World!\n");
}

module_init(example_init);
module_exit(example_exit);
```

- 请注意使用printk而不是printf。 另外，printk与printf共享的参数不同。 例如，KERN_INFO是一个标志，用于声明应为此行设置日志记录的优先级，并且不带逗号。
  内核在printk函数中对此进行了分类，以节省堆栈内存。
- 在文件末尾，我们调用module_init和module_exit告诉内核哪些函数是在load时候执行，那些在unload的时候执行

## Makefile书写

```makefile
obj-m += kernel_first.o
all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

注 make前面应该是Tab键

## 测试

执行如下命令加载模块到内核`sudo insmod kernel_first.ko`执行`dmesg|grep -i hello`,将会看到Hello world的输出。接下来卸载内核模块
`sudo rmmod kernel_first`,接下来运行`dmesg`，你将会看到Goodbye world的输出
