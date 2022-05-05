---
title: "Intermediate_Representation"
date: 2022-05-05T14:08:52+08:00
lastmod: 2022-05-05T14:08:52+08:00
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

# Intermediate Representation

# Compilers & Static Analyzers

编译器（Compiler）可以将源码翻译成机器码； 整个翻译过程分为：

1. 词法分析器（Scanner）：使用词法分析（Lexical Analysis）根据正则表达式（Regualar Expression）进行分析，如果检查通过则生成Tokens；
2. 语法分析器（Parser）：使用语法分析（Syntax Analysis）根据上下文无关文法（Context-Free Grammar）进行分析，如果检查通过则生成AST（抽象语法树）；
3. 语义分析器（Type Checker）：使用语义分析（Semantic Analysis）根据属性文法（Attribute Grammar）进行分析，如果检查通过则生成Decorated AST；
4. 转换器（Translator）：将Decorated AST转换为IR（Intermediate Representation），通常为三地址码（3AC，3 Address Code），静态程序分析通常会将IR转换为CFG（Controll Flow Graph）进行下一步分析；
5. 代码生成器（Code Generator）：经过优化后，将IR生成机器码（Machine Code）；

图示如下所示：

![Untitled](/media/28940079-a514-4b3e-afb7-74ccbc917698-0-1731743.png)

# AST vs IR

针对一段代码：

```
do i = i+1; while(a[i]<v);
```

写出其AST和IR的表现形式，AST表现形式：

![Untitled](/media/28940079-a514-4b3e-afb7-74ccbc917698-1-1731743.png)

IR表现形式：

![Untitled](/media/28940079-a514-4b3e-afb7-74ccbc917698-2-1731743.png)

很明显的可以看出两者的区别：

（1）AST的特性：

- high-level并且贴合Grammar Structure；
- 通常依赖不同的语言；
- 适用于快速类型检查；
- 缺少控制流信息；

（2）IR的特性：

- low-level并且贴合机器码（和汇编很像）；
- 通常不依赖于语言（于此便可以使用不同的前端，把不同的Source Code翻译成统一的IR）；
- 结构比较紧凑和均匀；
- 包含控制流信息；
- 通常被认为是静态程序分析的基础；

# IR: Three-Address Code(3AC)

## 3-Address Code(3AC)

三地址码（3AC）的特征：一条指令的右侧最多有一个运算符，每一个3AC最多包含3个address，每种类型的指令都有对应的3AC；

例如如下语句：

```
t2 = a + b + 3
```

将其转换成3AC：

```
t1 = a + b
t2 = t1 + 3
```

对于如上的3AC中，其对应的address如下，每个指令的address都没超过3个：

- Name: a , b
- Constant: 3
- Compiler-generated temporary: t1, t2

## 一些常见的3AC形式

```
x = y bop z
x = uop y
x = y
goto L
if x goto L
if x rop y goto L
```

对应上面的一些符号解释如下：

- x, y, z：addresses
- bop：二进制算术或逻辑运算
- uop：一元运算（减号、取反、强制转换）
- L：表示程序位置的标签
- rop：关系运算符（>、<、==、>=、<= 等）
- goto L：无条件跳转
- if ... goto L：条件跳转

# 3AC in Real Static Analyzer

## Soot的IR: Jimple

Soot是一个流行的Java静态程序分析框架，一些资料如下：

- [https://github.com/Sable/soot](https://github.com/Sable/soot)
- [https://github.com/Sable/soot/wiki/Tutorials](https://github.com/Sable/soot/wiki/Tutorials)

Soot的IR（Intermediate Representation）叫做Jimper，也是三地址码形式（3AC）；

（1）例如如下do...while循环代码（Java Source Code）：

![Untitled](/media/28940079-a514-4b3e-afb7-74ccbc917698-3-1731743.png)

翻译成Jimper的形式：

![Untitled](/media/28940079-a514-4b3e-afb7-74ccbc917698-4-1731743.png)

（2）例如如下的函数调用（Method Call）代码：

![Untitled](/media/28940079-a514-4b3e-afb7-74ccbc917698-5-1731743.png)

翻译成Jimper的形式（foo函数）：

![Untitled](/media/28940079-a514-4b3e-afb7-74ccbc917698-6-1731743.png)

翻译成Jimper的形式（main函数）：

![Untitled](/media/28940079-a514-4b3e-afb7-74ccbc917698-7-1731743.png)

（3）例如如下类（Class）代码：

![Untitled](/media/28940079-a514-4b3e-afb7-74ccbc917698-8-1731743.png)

翻译成Jimper的形式：

![Untitled](/media/28940079-a514-4b3e-afb7-74ccbc917698-9-1731743.png)

# Static Single Assignment(SSA)

这一部分李老师说是选学部分，比较老，但是比较经典，这部分紧紧作为普及；SSA即为静态单一分配（Static Single Assignment）；

SSA 中的所有赋值都是有不同名称的变量，详细解释如下：

- 每个定义需要给定一个新的名字；
- 将新名称传播给后续使用；
- 每个变量都只有一个定义；

例如如下图是SSA的具体形式（左边是3AC，右边是SSA）：

![Untitled](/media/28940079-a514-4b3e-afb7-74ccbc917698-10-1731743.png)

既然每个变量只有一个定义，那么如果遇到了control flow merges，该如何处理？例如如下是一个control flow merges，x现在不确定到底是x0还是x1：

![Untitled](/media/28940079-a514-4b3e-afb7-74ccbc917698-11-1731743.png)

如果是SSA，那么就会引入一个**`∅`** 方法，这个方法用于在merge节点时进行值选择，**`∅**(x0, x1)` 操作中，如果控制流传递true条件，则选择x0，反之亦反；例如如上的例子，经过SSA的处理，会变成如下形式：

![Untitled](/media/28940079-a514-4b3e-afb7-74ccbc917698-12-1731743.png)

 （1）为啥使用SSA：

- 流量信息被间接纳入唯一的变量名：可能有助于提供一些更简单的分析，例如，非敏感数据流分析可以通过SSA增强部分敏感数据流分析的精度；
- Define-and-Use pairs是显式的：实现更有效的数据facts存储和传播一些按需任务；一些优化任务在 SSA 上表现更好（例如，条件常数传播，全局值编号）

（2）为啥不使用SSA：

- SSA可能会引入过多的phi函数；
- 翻译成机器码时可能会存在效率低下的问题（因为复制操作）；

# Control Flow Analysis

- 通常指的是创建控制流图（CFG，Control Flow Graph）；
- CFG 是静态程序分析的基本结构；
- CFG 中的节点可以是单独的3地址码，或（通常）基本块（BB，Basic Block）；

一个将3地址码转换为CFG的图示如下：

![Untitled](/media/28940079-a514-4b3e-afb7-74ccbc917698-13-1731743.png)

# Basic Blocks(BB)

BB（Basic Blocks）是最大的连续的3地址码，且这个最大的连续3地址码存在如下性质：

- 只能在开头输入，例如block中的第一条指令；
- 只能在结尾退出，例如block中的最后一条指令；

图示如下：

![Untitled](/media/28940079-a514-4b3e-afb7-74ccbc917698-14-1731743.png)

计算一个程序中的连续3AC的Basic Blocks的算法如下：

- 程序P的第一条指令是一个leader；
- 有条件跳转、无条件跳转的目标指令是一个leader（确保唯一出口）；
- 有条件跳转、无条件跳转紧接着的下一条指令是一个leader；
- 一个BB就是一个leader到下一个leader之前的所有指令；

接下来看到一个例子，如下程序的3AC：

```
(10) if p == q goto (12)
(8) b = x + a
(9) c = 2a - b
(1) x = input
(2) y = x - 1
(3) z = x * y
(4) if z < x goto (7)
(5) p = x / y
(6) q = p + y
(7) a = q
(11) goto (3)
(12) return
```

首先我们需要识别出程序中的所有leader；看到前面算法，知道程序的第一条指令就是一个leader，所以目前已知leader：(1) ；

然后跳转指令的目标指令是leader，那么就知道存在这些leader：(3)、(7)、(12)；

最后跳转指令紧接着的下一条指令也是leader，那么就知道存在如下leader：(5)、(11)、(12)；

总而言之，现在的leader有：(1)、(3)、(7)、(12)、(5)、(11)、(12)，那么根据“一个BB就是一个leader到下一个leader之前的所有指令”可以得到所有的BB如下：

![Untitled](/media/28940079-a514-4b3e-afb7-74ccbc917698-15-1731743.png)

也就是：

![Untitled](/media/28940079-a514-4b3e-afb7-74ccbc917698-16-1731743.png)

# Control Flow Graphs(CFG)

控制流图（CFG）存在如下性质：

- CFG的节点都是一些BB（Basic Blocks）；
- 块A到块B之间如果存在边（Edge），需要满足条件：（1）块A的最后一条指令是条件/无条件跳转指令，并且跳转指令的目标是块B的第一条指令；（2）当块A和块B顺序上是紧紧连着的，此时如果块A的最后一条指令是无条件跳转指令，即使跳转指令的目标指令在块B内，块A和块B之间也不应该存在边（Edge）；

图示如下：

![Untitled](/media/28940079-a514-4b3e-afb7-74ccbc917698-17-1731743.png)

根据图示其实也就很自然的会把跳转的Label替换成BB，也就是将上图变成下面的样子，把一个BB看做一个单元：

![Untitled](/media/28940079-a514-4b3e-afb7-74ccbc917698-18-1731743.png)

使用Edge按照前面说的规则将BB连接起来后，通常还会添加两个节点：Entry和Exit；关于这两个节点的性质如下：

- 它们不对应于可执行的 IR；
- Entry存在一条Edge指向一个BB，这个BB包含整个IR的第一条指令，也就是这个BB是整个程序的第一个BB；
- 某个BB存在一条Edge指向Exit，这个BB包含整个IR的最后一条指令，也就是这个BB是整个程序的最后一个BB；

图示如下：

![Untitled](/media/28940079-a514-4b3e-afb7-74ccbc917698-19-1731743.png)



