---
title: "PointerAnalysis"
date: 2022-05-29T20:40:57+08:00
lastmod: 2022-05-29T20:40:57+08:00
draft: false
keywords: []
description: ""
tags: []
categories: []
typora-root-url: ../../../static
author: "P1n93r"

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: false
toc: true
autoCollapseToc: false
postMetaInFooter: false
hiddenFromHomePage: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false

# You unlisted posts you might want not want the header or footer to show
hideHeaderAndFooter: false

# You can enable or disable out-of-date content warning for individual post.
# Comment this out to use the global config.
#enableOutdatedInfoWarning: false

flowchartDiagrams:
  enable: false
  options: ""

sequenceDiagrams: 
  enable: false
  options: ""

---

# PointerAnalysis

# Motivation

如果我们使用CHA创建CallGraph，我们知道CHA仅仅考虑Class Hierarchy（类继承关系），那么正对如下程序分析，因为Number存在三个子类，那么调用 `n.get()` 方法的时候，就会产生三条调用边，其中有两条是假的调用边，导致最终分析的结果是一个不准确的结果：

![Untitled](/media/Untitled-3828124.png)

而如果通过指针分析，那么就可以清楚的知道变量n指向的对象，由此只会产生一条调用边，此时分析的结果就是一个precise的结果：

![Untitled](/media/Untitled-1-3828124.png)

# Intriduction to Pointer Analysis

- 是静态程序分析的基础：用于计算一个指针可以指向的内存地址；
- 对于OO语言而言，用于计算一个变量或者对象的field可以指向哪个对象；
- 可被认为是一个may-analysis：在over-approximation的约束下，计算一个指针可能指向的对象集合；
- 拥有超过40多年的研究历史，并且至今还是一个热门研究领域；

一个指针分析的例子如下，经过指针分析，可以知道各个变量或者对象的field的指向关系：

![Untitled](/media/Untitled-2-3828124.png)

## Pointer Analysis and Alias Analysis

两者很相关，但是又有如下不同：

- Pointer Analysis：分析一个指针可以（may）指向那些对象；
- Alias Analysis：分析两个指针是否能指向同一个对象；

Aliase信息可以从指针指向关系中推导出；

如下程序中，变量p和q指向相同的对象，则称p和q是aliase，但是x和y不是aliase：

![Untitled](/media/Untitled-3-3828124.png)

# Key Factors in Pointer Analysis

指针分析是一个复杂的系统，许多因素会影响指针分析的精度和效率，各种影响因素如下：

![Untitled](/media/Untitled-4-3828124.png)

## Heap Abstraction

- 即堆抽象，解决的一个问题就是：如何对程序中的Heap Memory进行建模；
- 在动态执行时，程序会因为循环和递归等产生无穷无尽的对象（heap object）；
- 使用静态程序分析时，因为模拟动态执行会产生无穷无尽的对象，程序分析无法终止，所以就引出一种堆抽象技术，把无穷无尽的对象抽象成有限个数的对象；

例如使用堆抽象技术，把左边无穷无尽的对象，抽象成右边有限个数的对象：

![Untitled](/media/Untitled-5-3828124.png)

堆抽象有很多分支，课程中后续会讲解的就是Allocation sites（调用点的抽象）这个分支：

![Untitled](/media/Untitled-6-3828124.png)

## Allocation-Site Abstraction

- 是目前使用最广泛的堆抽象技术；
- 通过程序中的allocation sites进行对象建模；
- 通过程序中的每个创建点（allocation site）构建一个抽象对象；

如下程序中，通过创建点构建堆抽象，把三次循环的对象抽象成一个allocation site抽象对象：

![Untitled](/media/Untitled-7-3828124.png)

## Context Sensitivity

主要解决的问题：如何构建calling context？

当使用上下文不敏感的指针分析时（Context insensitive），不同位置的相同函数调用，会被merge成一个数据流，从而造成精度丢失，但是如果使用上下文敏感（Context sensitive）的指针分析，对于不同位置的相同函数调用会根据不同的调用上下文（calling context）进行区分，然后分别分析每个context，图示如下：

![Untitled](/media/Untitled-8.png)

## Flow Sensitivity

主要解决的问题：如何构建控制流？

所谓流敏感，就是在分析过程中，会遵循程序的执行流，即会遵循程序中语句的执行顺序，在程序的每一个statement都会维护当前程序指针指向的关系映射，而流非敏感则是相反的；如下图所示，左边蓝色的是流敏感分析的结果，右边是流非敏感的分析结果，会产生一个false positive：

![Untitled](/media/Untitled-9.png)

## Analysis Scope

主要解决的问题：分析程序中的那一部分？

比较好理解，一个就是全部分析，一个就是分析关心的某部分：

![Untitled](/media/Untitled-10.png)

# Concerned Statements

- 现代程序语言中，有很多类型的statements，比如 `if-else` ， `switch-case` ， `for/while/do-while` 等等，但是对于指针分析，我们并不用关心全部类型的statements，那些不会直接影响指针的statements会被忽略；
- 我们只关注那些影响指针的statements；

在Java中的指针主要有如下四类，我们关注的有如下加粗的三种：

![Untitled](/media/Untitled-11.png)

其中，需要注意的是 `Array element: array[i]` ，这个地方意思就是忽略数组的索引，把整个数组抽象为一个整体的、单独的对象，这个地方其实很重要，我阅读GadgetInspector源码时，发现作者没处理 `Array element` 的指针分析，我修改了这部分的源码，并且忽略数组索引，效果提高了不少（当然，误报也多了，但是为了safe-approximation，可以认为效果更好了）；

同时，Java中可以影响指针的5种statements如下，后续的指针分析几乎就是围绕着这几种statements进行分析：

![Untitled](/media/Untitled-12.png)

如果遇到了连续调用，一般会先转换成3AC的形式再进行分析：

![Untitled](/media/Untitled-13.png)
