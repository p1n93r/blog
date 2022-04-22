---
typora-root-url: ../../../static
title: "Pebble安全研究"
date: 2021-06-15T17:07:36+08:00
draft: false
categories: ["security"]
---

## 简介

Pebble是一款Java  模板引擎，开发他的灵感来自于Twig。它具有很强的模板延续性和易于阅读的语法；出于安全性考虑，它内置自动转义功能，并包括对国际化的集成支持。它支持模板引擎中最常见的语法，其中变量替换使用完成。通常，在模板引擎中，可以包含任意的 Java 表达式。假设想将名为 `name` 的变量放在模板中并大写，这时可以使用 `{{name.toUpperCase()}}` 。

这个模板引擎Github才800多star，比较小众。

## 基本使用

当然，项目需要引入jar包：

```xml
<dependency>
    <groupId>io.pebbletemplates</groupId>
    <artifactId>pebble-spring-boot-starter</artifactId>
    <version>3.1.5</version>
</dependency>
```

首先准备一个模板文件： `templates\home.html` ，如下所示：

```html
<!DOCTYPE html>
<p>{{content}}</p>
<h1>{{user.name}}</h1>
</html>
```

然后使用如下代码进行模板渲染：

```java
PebbleEngine engine = new PebbleEngine.Builder().build();
PebbleTemplate compiledTemplate = engine.getTemplate("templates/home.html");
Writer writer = new StringWriter();
Map<String, Object> context = new HashMap<>();
User user = new User();
user.setName("p1n93r");
context.put("user", user);
context.put("content", "This is a basic test");
compiledTemplate.evaluate(writer, context);
String output = writer.toString();
System.out.println(output);
```

运行后，输出如下所示：

```html
<!DOCTYPE html>
<p>This is a basic test</p>
<h1>p1n93r</h1>
</html>
```

我们跟进 `com.mitchellbosecke.pebble.template.PebbleTemplateImpl#evaluate()` ,通过debug可以看到context中可以被访问的各种对象：

![image-20210809191442501](/media/image-20210809191442501.png)

我们尝试使用模板的表达式访问一下这些对象：

```html
<!DOCTYPE html>
<p>{{content}}</p>
<h1>{{user.name}}</h1>
<h1>{{template}}}</h1>
<h1>{{_context}}}</h1>
<h1>{{locale}}</h1>
</html>
```

结果如下：

```html
<!DOCTYPE html>
<p>This is a basic test</p>
<h1>p1n93r</h1>
<h1>com.mitchellbosecke.pebble.template.PebbleTemplateImpl@5e025e70}</h1>
<h1>com.mitchellbosecke.pebble.template.GlobalContext@1fbc7afb}</h1>
<h1>zh_CN</h1>
</html>
```

其中， `GlobalContext` 就是一个 `HashMap` 的子类，只能调用 `get()` 方法， `Map` 里面的值就是前面分析的context中存在的可被表达式调用的几个对象。

 `PebbleTemplateImpl` 类存在很多公开方法，但是目前暂时未找到可以利用的。 `Locale` 类也是，没找到可被利用的方法。

小节一下：

**非springboot-start集成下，Pebble默认情况下存在三个对外暴露的对象：GlobalContext、PebbleTemplate和Locale** ，但是暂时没发现可以被利用的点。

## 历史漏洞

我们知道，对于SSTI常用的沙箱逃逸手法有如下方式：

```java
{{ variable.getClass().forName('java.lang.Runtime').getRuntime().exec('ls -la') }}
```

在Pebble v3.0.9之前，会对 `getClass` 方法进行黑名单验证：

```java
  /**
   * com.mitchellbosecke.pebble.attributes.MemberCacheUtils
   */
  private Method findMethod(Class<?> clazz, String name, Class<?>[] requiredTypes, String filename,
      int lineNumber, EvaluationOptions evaluationOptions) {
    if (!evaluationOptions.isAllowGetClass() && name.equals("getClass")) {
      throw new ClassAccessException(lineNumber, filename);
    }
```

所以，我们可以使用大小写绕过：

![image-20210810100133895](/media/image-20210810100133895.png)

Pebble官方在v3.0.9进行了修复：

![image-20210810100310762](/media/image-20210810100310762.png)

后续发现（v3.1.1之前），可以不使用 `getClass()` 即可获取 `Class` 对象。发现 `java.lang.Integer` 类存在静态变量：TYPE，这个变量的类型就是 `Class` 类型的。所以可以通过如下方式获取 `Class` 对象，并加载 `java.lang.Runtime` 类：

```java
{{ (1).TYPE.forName('java.lang.Runtime')... }}
```

这个漏洞在v3.1.1进行了修复，修复手法和 `FreeMarker` 一样。在 `unsafeMethods.properties` 中添加了大量黑名单。

## 安全机制

最新版（v3.1.5）中，发现是在 `com.mitchellbosecke.pebble.attributes.methodaccess.BlacklistMethodAccessValidator#isMethodAccessAllowed()` 存在黑名单，改变了以往的黑名单机制：

```java
// 类黑名单
@Override
public boolean isMethodAccessAllowed(Object object, Method method) {
  boolean methodForbidden = object instanceof Class
      || object instanceof Runtime
      || object instanceof Thread
      || object instanceof ThreadGroup
      || object instanceof System
      || object instanceof AccessibleObject
      || this.isUnsafeMethod(method);
  return !methodForbidden;
}
```

方法黑名单如下（ `com.mitchellbosecke.pebble.attributes.methodaccess.BlacklistMethodAccessValidator` ）：

```java
private static final String[] FORBIDDEN_METHODS = {"getClass",
    "wait",
    "notify",
    "notifyAll"};
```

此外，需要注意的是，Pebble可以配置成不进行黑名单验证：

```java
// 配置NoOpMethodAccessValidator，不进行黑名单验证（包括类黑名单）
PebbleEngine engine = new PebbleEngine.Builder().methodAccessValidator(new NoOpMethodAccessValidator()).build();
```

如果配置这样，最新版本也可以直接getshell了：

![image-20210810103745155](/media/image-20210810103745155.png)

## 参考

[https://research.securitum.com/server-side-template-injection-on-the-example-of-pebble/](https://research.securitum.com/server-side-template-injection-on-the-example-of-pebble/)


