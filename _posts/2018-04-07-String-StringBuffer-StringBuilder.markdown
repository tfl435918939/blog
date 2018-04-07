---
layout:     post
title:      "谈谈String,StringBuilder,StringBuffer"
subtitle:   " \"人生就像一场马拉松，跑到最后才是赢家。\""
date:       2018-04-07 16:34:00
author:     "Leon"
header-img: "img/2018-04-03-posts-bg.jpg"
catalog: true
tags:
    - Java
---

> “Anyway, anyhow. ”


### 谈谈String,StringBuilder,StringBuffer。

String字符串是常量。String对象的值是放在JVM方法区的常量池里面的(方法区存放类的信息：版本、字段、方法、接口；常量池：符号引用和字面量；静态变量)。修改String对象的值，就相当于重新new一个String对象出来。频繁修改String对象的值可能会触发JVM的GC，影响系统的性能。因此需要频繁修改字符串的值，我们可以使用StringBuilder和StringBuffer类。

StringBuffer是线程安全的。修改StringBuffer的值不会产生新的对象。StringBuffer有一个字符串缓冲区，这个缓冲区可以安全地应用在多线程当中，只有调用特定的方法才能改变缓冲区的值。

StringBuilder 是非线程安全的。StringBuilder被设计出来，就是简易替代StringBuffer的。在单线程的情况下，StringBuilder会更快，但是它不是线程安全的。它和StringBuffer的方法基本相同。

### String中加号连接符是怎么执行的。
String 对象的字符串拼接其实是被 JVM 解释成了 StringBuffer 对象的拼接。在特定的情况下，在 JVM 眼里，这个``String S1 = “This is only a” + “ simple” + “test”; ``其实就是：``String S1 = “This is only a simple test”;``也就是说利用+直接连接两个字符串常量，虚拟机会直接把这两个字符串连接起来看成一个字符串。

### String为什么是不可变性？
在java的世界里，String是作为类出现的，核心的一个域就是一个char数组，内部就是通过维护一个不可变的char数组，来向外部输出的。
这是jdk一段String类定义，首先类是final的，表明类不可被继承；核心域是private final的，final表明这个引用所指向的内存地址不会改变，但这还不足说明value[]是不可变的；因为引用所指向的内存的值有可能发生变化，但是jdk是不会让这样的事情发生的。private 保证这个域对外部来说是不可见的，这还不够，对value还要进行 保护性拷贝。参数是一个char数组引用，它并没有把这个数组引用直接赋值给实例对象的value成员变量，而是通过一个Arrays.copyOf的方式拷贝一个数组再给到对象的成员变量。

首先类相关的信息肯定是放在方法区的，堆中放一些实例对象，程序计数器始终指向下一条将要执行的指令，虚拟机栈和本地方法栈分别是用来于普通方法和本地方法的。

着重说一下虚拟机栈，它是线程私有的，描述的是java方法执行的内存模型：每个方法在执行的同时创建一个栈帧，用于存放局部变量表，操作数栈，方法入口，动态链接等。
<img class="shadow" src="blog/img/JVM内存模型.jpg" width="260">

局部变量表用来存放一些基本数据类，和引用。操作数栈的话，是用来作运算用的，打个比方

```
int a=1;
int b =2;
int c =a+b;
```
JVM会先把a,b的值压入到操作数栈保存起来，等到程序计数器执行加法指令的时候，再把a,b从栈中pop出来。重点就在于a,b值是从哪里压入到栈中的，如果没有``char[] val = value;/* avoid getfield opcode */``那么接下来要遍历的就是value数组了，value毫无疑问是堆中的数据，也就是每一次遍历会经历，数据由堆中取出，进入栈帧，再由栈帧压入到操作数栈，最后pop出来进行运算。

如果执行了上述代码，情况就大不相同了，val就是一个局部变量，它会被存放在局部变量表中，接来下的运算，就是局部变量表到操作数栈了，属于一个栈帧内的数据转移。JVM为了避免频繁进行堆栈数据转移，将值复制到本地变量一次，以避免在接下来的几行中循环的每一次迭代，从堆中多次取下的字段值。
