---
typora-root-url: ../../../static
title: "Arthas学习"
date: 2021-06-15T17:07:36+08:00
draft: false
categories: ["security"]
---

## 安装使用

**（1）安装**

```
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar --telnet-port 9998 --http-port -1
```

![image-20210928171455643](/media/image-20210928171455643.png)

**（2）退出Arthas**

直接运行 `quit` 或者 `exit` 命令。但是这不是退出 Arthas ，而是退出当前连接而已，下次还是可以继续连接 Arthas 。想完全退出 Arthas，需要使用 `stop` 或者 `shutdown` 命令；

## 常用命令

 **（1）查看仪表盘**

直接运行 `dashboard` ，这个主要是帮助我们发现性能问题：

![image-20210928172024450](/media/image-20210928172024450.png)

 **（2）修改日志级别**

直接输入 `logger` 命令即可查看logger信息，包括日志的位置，对于查看系统的日志位置非常方便：

![image-20210928173235967](/media/image-20210928173235967.png)

此外， `logger` 命令还可以用于修改日志级别：

| 参数   | 说明               |
| ---- | ------------------------------------------------------------ |
| -c   | 指定 `classloader` 的 hash ，默认为 `SystemClassLoader` ，例如: `logger -c 1654a892` |
| -l   | 设置日志级别，例如: `logger -n debug -c 1654a892 -l debug`   |
| -n   | 获取指定名称logger的信息，例如: `logger -n debug`          |

 **（3）下载内存**

可以使用 `heapdump` 命令下载程序的运行时内存，参数说明如下：

| 参数   | 说明                                                         |
| ------ | ------------------------------------------------------------ |
| -l     | 只dump内存中的live对象，例如: `heapdump -l /tmp/dump.hprof ` |
| [file] | 指定输出文件，例如: `heapdump /tmp/dump.hprof`               |

 **（4）反编译**

直接使用 `jad xxx.xxx.xxx.XXX` 来反编译某个类：

![image-20210928174846064](/media/image-20210928174846064.png)

 **（5）动态修改类**

这个功能相关的命令为： `retransform` ，其限制如下：

- 不允许新增加 `field` 或者 `method` ；
- 正在跑的函数，没有退出不能生效；

参数说明：

| 参数                   | 说明                                                         |
| ---------------------- | ------------------------------------------------------------ |
| -c                     | 指定classloader加载文件，例如: `retransform -c 327a647b /tmp/Test.class /tmp/Test\$Inner.class` |
| --classPattern [value] | 触发重新加载class文件，例如: `retransform --classPattern demo.*` |
| -d                     | 删除指定id替换的条目，例如: `retransform -d 1`               |
| --deleteAll            | 删除所有替换的条目，例如: `retransform --deleteAll`          |
| -l                     | 列出所有替换的条目，例如: `retransform -l`                   |
| [classfilePaths]       | 加载指定路径的class文件，例如: `retransform /tmp/Test.class` |

如果需要还原修改后的类：

```
retransform -d 1
retransform --classPattern xxx.xx.xx.XXX
```

 **（6）观察方法调用**

这个功能相关的命令为： `watch` ；相关参数和介绍如下：

| 参数     | 说明                             |
| ------------------- | ------------------------------------------ |
| *class-pattern*     | 类名表达式匹配                             |
| *method-pattern*    | 方法名表达式匹配                           |
| *express*           | 观察表达式                                 |
| *condition-express* | 条件表达式                                 |
| [b]                 | 在**方法调用之前**观察                     |
| [e]                 | 在**方法异常之后**观察                     |
| [s]                 | 在**方法返回之后**观察                     |
| [f]                 | 在**方法结束之后**(正常返回和异常返回)观察 |
| [E]                 | 开启正则表达式匹配，默认为通配符匹配       |
| [x:]                | 指定输出结果的属性遍历深度，默认为 1       |

注意点：

- `watch` 命令定义了4个观察事件点，即 `-b` 方法调用前，`-e` 方法异常后，`-s` 方法返回后，`-f` 方法结束后；
- 4个观察事件点 `-b`、`-e`、`-s` 默认关闭，`-f` 默认打开，当指定观察点被打开后，在相应事件点会对观察表达式进行求值并输出；
- 这里要注意`方法入参`和`方法出参`的区别，有可能在中间被修改导致前后不一致，除了 `-b` 事件点 `params` 代表方法入参外，其余事件都代表方法出参；
- 当使用 `-b` 时，由于观察事件点是在方法调用前，此时返回值或异常均不存在；

例如使用如下命令观察方法入参和返回的参数：

```
watch com.rebuild.web.commons.UrlSafe isTrusted "{params,returnObj}" -x 3
```

![image-20210930001103080](/media/image-20210930001103080.png)

一些案例场景可以查看：https://github.com/alibaba/arthas/issues/71

Arthas中Ognl默认支持的变量为如下：

```
ClassLoader loader;
Class<?> clazz;
ArthasMethod method;
Object target;
Object[] params;
Object returnObj;
Throwable throwExp;
boolean isBefore;
boolean isThrow;
boolean isReturn;
```

 **（7）观察方法调用路径**

`trace` 命令能主动搜索 `class-pattern`／`method-pattern` 对应的方法调用路径，渲染和统计整个调用链路上的所有性能开销和追踪调用链路。

参数说明如下：

| 参数名称            | 参数说明                             |
| ------------------- | ------------------------------------ |
| *class-pattern*     | 类名表达式匹配                       |
| *method-pattern*    | 方法名表达式匹配                     |
| *condition-express* | 条件表达式                           |
| [E]                 | 开启正则表达式匹配，默认为通配符匹配 |
| `[n:]`              | 命令执行次数                         |
| `#cost`             | 方法执行耗时                         |

注意：`watch` ， `stack` ， `trace` 这个三个命令都支持 `#cost` 。

例如使用如下命令查看某个函数的调用路径，但是貌似只能查看目标函数的栈顶调用路径，不能看到栈低的调用路径，也就是说，不能查看是谁调用到了目标函数：

```
trace *UrlSafe isTrusted -v
```

![image-20210930003032829](/media/image-20210930003032829.png)

 **（8）查看ClassLoader**

使用 `classloader` 命令可以查看classloader的继承树，urls，类加载信息；例如使用如下命令查看classloader统计信息：

```
classloader -l
```

![image-20210930004101314](/media/image-20210930004101314.png)

查看classloader的继承树：

```
classloader -t
```

![image-20210930004227581](/media/image-20210930004227581.png)

 **（9）内存编译**

使用 `mc` 命令可以编译 `.java` 文件生成 `.class` ；可以使用 `-c` 选项指定classloader：

```
mc -c 327a647b /tmp/Test.java
```

也可以通过`--classLoaderClass`参数指定ClassLoader：

```
mc --classLoaderClass org.springframework.boot.loader.LaunchedURLClassLoader /tmp/UserController.java -d /tmp
```

编译生成 `.class` 文件之后，可以结合 `retransform` 命令实现热更新代码。

注意：`mc` 命令有可能失败。如果编译失败可以在本地编译好 `.class` 文件，再上传到服务器。

 **（10）查看方法调用栈**

前面说过可以通过 `trace` 命令查看方法调用路径，但是这样只能得到从当前方法往下执行的调用路径，如果想得到完整的方法调用栈，需要使用 `stack` 命令：

```
stack *UrlSafe isTrusted
```

![image-20210930005256206](/media/image-20210930005256206.png)

一些参数说明如下：

| 参数名称            | 参数说明                             |
| ------------------- | ------------------------------------ |
| *class-pattern*     | 类名表达式匹配                       |
| *method-pattern*    | 方法名表达式匹配                     |
| *condition-express* | 条件表达式                           |
| [E]                 | 开启正则表达式匹配，默认为通配符匹配 |
| `[n:]`              | 执行次数限制                         |

## 注意事项

-  `JVM` 只能 attach 同样用户下的 java 进程，需要切换到目标 `JVM` 相同的用户下，才能使用，或者使用某个用户的身份执行命令： `java -jar arthas-boot.jar --telnet-port 9998 --http-port -1` ；

## 参考

- https://arthas.aliyun.com/doc/commands.html
