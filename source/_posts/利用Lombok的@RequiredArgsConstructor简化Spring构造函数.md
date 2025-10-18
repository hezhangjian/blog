---
title: 利用Lombok的@RequiredArgsConstructor简化Spring构造函数
link: lombok-required-args-constructor-spring
date: 2024-03-11 20:47:12
tags:
  - Java
  - Lombok
  - Spring
---

从Spring的新版本开始，推荐使用构造函数的注入方式，通过构造函数注入有很多优点，诸如不变性等等。同时在构造函数上，也不需要添加`@Autowire`
注解就可以完成注入

```java
// Before
public class ABC {
    private final A a;

    private final B b;

    private final C c;

    public ABC(@Autowire A a, @Autowire B b, @Autowire C c) {
        this.a = a;
        this.b = b;
        this.c = c;
    }
}

// After
public class ABC {
    private final A a;

    private final B b;

    private final C c;

    public ABC(A a, B b, C c) {
        this.a = a;
        this.b = b;
        this.c = c;
    }
}
```

但是，这种注入方式会导致变动代码的时候，需要同时修改field以及构造函数，在项目早期发展时期，这种变动显得有一些枯燥，再加上已经不需要`@Autowire`
注解。这时，我们可以用Lombok的@RequiredArgsConstructor来简化这个流程。

Lombok的`@RequiredArgsConstructor`会包含这些参数：

- 所有未初始化的 final 字段
- 被标记为 @NonNull 但在声明时未初始化的字段。

对于那些被标记为 @NonNull
的字段，还会生成一个显式的空检查（不过在Spring框架里这个没什么作用）。通过应用`@RequiredArgsConstructor`
，代码可以简化为如下模样，同时添加新的字段也不需要修改多行。

```java

@RequiredArgsConstructor
public class ABC {
    private final A a;

    private final B b;

    private final C c;
}
```
