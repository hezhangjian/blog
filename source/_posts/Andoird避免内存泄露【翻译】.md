---
title: Android避免费内存泄露【翻译】
link:
date: 2016-09-08T15:44:36
tags:
---
原文：http://android-developers.blogspot.sg/2009/01/avoiding-memory-leaks.html

Android中堆的内存是有限的，你应当使用尽量小的内存。因为Android能在内存中保存越多的应用，对于用户来说，切换应用就会十分的迅速。相当多的内存泄漏的原因是因为：保持了一个对context的长引用(long-lived)。
    在Android中，Context可以用来做许多事情，不过大部分是用来加载和获取资源。这就是为什么所有的视图组件在构造方法里面需要context作为参数的原因。有两种Context，Activity和Application，通常使用第一个。比如:
```java
TextView label =new TextView(this);
```
        
这代表着views持有一个对于activity的引用。所以，如果你泄漏了Context(你保留了一个对它的引用，在GC的时候保护了它)。如果你不够小心的话，泄漏掉整个Activity是很容易的。
    当屏幕的显示方向发生变化时，系统将会(默认)销毁现在的Activity，维护它的一些数据然后新建一个Activity。如果这样做的话，Android 将会重新读取UI。想象一下，你在你的应用里面使用了一个大bitmap，你不想每次旋转的时候都重新加载它。最简单的方法就是把它放在一个静态域里面。

```java
        private static Drawable sBackground;
        
        @Override
        Protected void onCreate(Bundle state){
            super.onCreate(state);
            TextView label = new TextView(this);
            label.setText("Leaks are bad");
            if(sBackground == null){
                sBackground = getDrawable(R.drawable.hezhangjian);
            }
            label.setBackgroundDrawable(sBackground);
            setContentView(label);
        }   
```


上述代码很快但是是错误的，它泄漏了在第一次屏幕旋转之前的Activity。当一个Drawable连系在view上时，view就像被设置为drawable的回调一样。对于上面的代码，就是说drawable持有了对于textview的引用，而textview又持有了activity的引用。这就把Activity泄漏掉了。当Activity被销毁的时候，把drawable的回调设置为null。
    有两种简单的方式去避免内存泄漏。第一个就是避免context超范围的使用。例子上展示的是静态引用但是内部类以及他们持有的对外部类的明确引用也同样危险。第二个解决方法，利用Application context，这个context的存活时间跟你的应用一样久。如果打算保持一个长久的对象，就用 Context.getApplicationContext() 或者 Activity.getApplication()。
