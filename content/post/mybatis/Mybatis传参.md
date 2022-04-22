---
typora-root-url: ../../../static
title: "Mybatis传参"
date: 2019-10-26T20:40:36+08:00
draft: false
categories: ["Mybatis"]
---

## 前言
有一次用到mybatis的动态sql中的foreach，发现foreach的属性collection填写的值有点问题。然后突然想到之前都是使用的都是用Map或JavaBean传参，没有使用过多参的情况，然后今天测试多参传递，之前只知道可以用#{0},#{1}或者再Mapper接口中形参声明中使用@Param，但是今天发现还可以使用#{param1},#{param2}这种，emmm，还发现了很多规则，都在下面了。

## 结论
- mybatis低版本：当Mapper接口的形参使用了@Param标记，那么映射文件中用#{0},#{1}...这种形式取值时，与@Param位置对应的那个#{}会被替换成@Param里的值。
- mybatis高版本：使用@Param注解，则只能通过@Parma的属性值以及param1,param2,param3...来取。
- 映射文件中使用#{param1}，#{param2}...的形式取值时，总是可以取到Mapper形参列表里的第一个、第二个...参数。
- 映射文件中可以使用 `_parameter` 。当Mapper接口形参列表只有一个值时，此参数就代表这个值；当形参存在多个值时，此参数就代表这几个形参形成的Map，其key为0,1,2...,param1,pram2,param3...，如果存在使用了@Param的情况，则对应位置的#{0},#{1}...会被替换成@Param里的值。并且利用_parameter.get()方式取值时，不能取0,1,2...，只能取param1,param2...,或者@Param标记设置的值。
- 当Mapper接口只有一个形参时，如果此形参是一个List，那么foreach的collection取值只能是list；如果此形参是一个数组，collection的取值只能是array；如果此形参是一个Map，则collection的取值只能是Map的key（对应的value为可迭代对象）；如果此形参用了@Param标记，则collection只能取@Param标记的值，或者_parameter.get('标记的值')。