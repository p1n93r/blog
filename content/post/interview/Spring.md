---
typora-root-url: ../../../static
title: "Spring"
date: 2020-06-11T16:04:36+08:00
categories: ["interview"]
draft: false
---

## 什么是Spring的IOC？
IOC（Inversion of Controller）即 **控制反转** 。反转获得依赖对象的过程，获得依赖对象的过程由自身管理变为了由IOC容器主动注入。例如，系统在未引入IOC容器之前，对象A依赖于对象B，那么必须主动去创建对象B，此时控制权在自己手上，软件系统引入IOC容器之后，对象A与对象B之间失去了直接联系，IOC会主动创建一个对象B注入到对象A需要的地方，对象A获得依赖对象B的过程，由主动变为了被动，控制权反转过来了。依赖注入(DI)和控制反转(IOC)是从不同角度描述的同一件事情，就是指 **通过引入IOC容器，利用依赖关系注入的方式，实现对象之间的解耦** 。

## 说说Spring的几个通知以及相关注解？
1. 前置通知(Before)：在方法执行之前执行的通知，使用@Before注解进行装配。
2. 后置通知(After)：无论方法是否执行成功，都在方法执行后执行的通知，使用@After注解进行装配。
3. 返回通知(After-Retuning)：仅当方法执行成功完成后执行的通知，使用@AfterReturing注解进行装配。
4. 异常通知(After-Throwing)：在方法抛出异常退出后执行的通知，使用@AfterThrowing注解进行装配。
5. 环绕通知(Around)：在方法执行之前以及之后执行的通知。使用@Around注解进行装配。

## 说说@Resource、@Autowired和@Qulifier注解？
- @Resource：默认按照by name的方式进行自动注入，是J2EE提供的，它有两个属性：name和type。当使用name属性的时候，按照by name的方式进行注入；当使用type属性的时候，按照by bype的方式进行注入；如果两个都缺省，则使用反射机制按照by name的方式进行注入。
- @Autowired：默认按照by type的方式进行自动注入，可以修饰构造器、setter方法、属性和任意名称/任意参数的普通方法。
- @Qulifier：当存在同个类型的多个bean时，利用@Autowired注入则无法确认注入哪一个，此时可以配合使用@Qulifier注解来确切的注入哪个bean来消除歧义。

## 说说AOP、SpringAOP和AspectJ？
- AOP：AOP是一种程序设计范式，该范式以一种称为切面(aspect)的语言构成基础，切面是一种新的模块化机制，用来描述分散在对象、类或方法中的横切关注点。
- SpringAOP：是一个使用动态代理的AOP框架，不会改变字节码，每次运行时在内存中创建一个临时AOP增强对象，在特定的切点做了增强处理，并回调原对象方法。
- AspectJ：是一个使用静态代理的AOP框架，在编译阶段会将切面织入到字节码中，生成增强后的AOP代理类，运行时就是得到增强后的AOP对象。

## 如何理解"横切关注"这个概念？
横切关注是会影响到 **整个** 应用程序的关注功能，它跟正常的业务逻辑是正交的，没有必然的联系，但是几乎所有的业务逻辑都会涉及到关注功能。通常，日志、事物、安全性等关注就是横切关注点。

## 如何理解AOP中的连接点(JoinPoint)、切点(PointCut)、增强(Advice)、引介(Introdution)、织入(Weaving)和切面(Aspect)这概念? 
- 连接点(JoinPoint)：程序执行的某个特定位置，一个类或者一段程序代码中具有一些边界性质的特定点就是连接点。Spring仅支持方法的连接点。
- 切点(PointCut)：如果说连接点相当于数据中的记录，那么切点相当于查询条件，一个切点可以匹配多个连接点。SpringAOP负责解析切点所设定的查询条件，找到对应的连接点。
- 增强(Advice)：增强(通知)是织入到目标类连接点上的一段代码。
- 引介(Introdution)：引介是一种特殊的增强，它为类添加一些属性和方法。
- 织入(Weaving)：将通知添加到目标类具体的连接点上的过程。
- 切面(Aspect)：由切点和通知(引介)组成，包括了对横切关注点功能的定义，也包括了对连接点的定义。

## 说说@Scope的五个作用域？
缺省的spring bean的作用域是singleton。

1. singleton：bean在每个SpringIOC容器中只有一个实例。
2. prototype：一个bean的定义可以存在多个实例。
3. request：每次http请求都会创建一个bean。该作用域仅在基于web的spring应用下有效。
4. session：在一个http session中，一个bean定义对应一个实例。该作用域仅在基于web的spring应用下有效。
5. global-session：在一个全局的http session中，一个bean定义对应一个实例。该作用域仅在基于web的spring应用下有效。

## 选择使用Spring框架的原因？
1. 使用它的IOC功能，在解耦上达到了配置级别。
2. 使用它对数据库访问事物事物相关的相关。