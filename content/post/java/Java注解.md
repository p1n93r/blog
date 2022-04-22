---
typora-root-url: ../../../static
title: "Java注解"
date: 2021-01-01T21:28:36+08:00
draft: false
categories: ["Java"]
---

## 元注解
元注解的作用就是 **注解其他注解** ，Java5.0定义了以下4个标准元注解，它们被用来对其他注解类型作说明：

- @Target
- @Retention
- @Documented
- @Inherited

### @Target
@Target说明了注解所修饰的对象范围，使用@Target可以更加明确其修饰的目标。其取值（ElementType）有如下：

- CONSTRUCTOR：用于描述构造器。
- FIELD：用于描述域。
- LOCAL_VARIABLE：用于描述局部变量。
- METHOD：用于描述方法。
- PACKAGE：用于描述包。
- PARAMETER：用于描述参数。
- TYPE：用于描述类、接口（包括注解类型）或enum声明。

### @Retention
@Retention定义了该注解被保留的时间长短，可以对注解的“生命周期”限制。 **表示需要在什么级别保留该注释信息，用于描述注解的生命周期（被描述的注解在什么范围内有效）** ，其取值（唯一取值）有如下：

- SOURCE：在源文件中有效（源文件保留）
- CLASS：在class文件有效（class保留）
- RUNTIME：在运行时有效（运行时保留）

### @Documented
@Documented表明这个注解应该被javadoc工具记录。默认javadoc是不包括注解的，但是如果声明注解的时候指定了@Documented，那么它会被javadoc工具处理。 **是一个标记注解，没有成员。**

### @Inherited
@Inherited也是一个标记注解，没有成员。描述某个被注解的类型是被继承的。

如果一个使用了@Inherited修饰的annotation类型被用于一个class，则这个annotation将被用于该class的子类。

**注意** ：@Inherited annotation类型是被标注过的class的子类所继承。类并不从它所实现的接口继承annotation，方法并不从它所重载的方法继承annotation。

## 自定义注解
- 使用@interface自定义注解时，自动继承java.lang.annotation.Annocation接口，由编译程序自动完成其他细节。
- 定义自定义注解时，不能继承其他注解或接口。
- 自定义注解中的每一个方法实际上是声明一个配置参数，方法名就是参数的名称，返回值类型就是参数的类型。可以通过default来声明参数的默认值。
- 参数类型只能是：基本数据类似、String类型、Class类型、enum类型、Annotation类型以及上述所有类型的数组。
- 自定义注解的格式：public @interface 注解名 {定义体}