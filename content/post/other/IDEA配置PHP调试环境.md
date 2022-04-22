---
typora-root-url: ../../../static
title: "IDEA配置PHP调试环境"
date: 2021-10-07T17:05:36+08:00
draft: false
categories: ["Other"]
tags: ["Maven"]
---

我使用的是IDEA来进行调试php，需要开启两个插件，一个是： `PHP Built-in Web Server` ，还有一个是： `PHP Web Page` 。

其中 `PHP Built-in Web Server` 用于启动一个PHP Server，而 `PHP Web Page` 用于连接xdebug端口，进行调试。

**（1）配置Xdebug**

这个主要是在php中添加xdebug插件配置：

![image-20211007164141665](/media/image-20211007164141665.png)

然后在 `php.ini` 中添加如下配置：

```ini
[XDebug]
xdebug.profiler_output_dir="E:\Program Files\phpStudy\PHPTutorial\php\php-5.5.38\tmp\xdebug"
xdebug.trace_output_dir="E:\Program Files\phpStudy\PHPTutorial\php\php-5.5.38\tmp\xdebug"
zend_extension="E:\Program Files\phpStudy\PHPTutorial\php\php-5.5.38\ext\php_xdebug.dll"
xdebug.remote_log =D:/xdebug/log/xdebug.log
;是否允许Xdebug跟踪函数调用，跟踪信息以文件形式存储，默认值为0
xdebug.auto_trace=1
xdebug.remote_enable = On
xdebug.remote_handler = dbgp
xdebug.remote_port = 9001
xdebug.idekey = mykey
;解决ip变化的问题
xdebug.remote_connect_back = 1
;是否允许Xdebug跟踪函数参数，默认值为0
xdebug.collect_params=1
;是否允许Xdebug跟踪函数返回值，默认值为0
xdebug.collect_return=1
```

**（2）配置PHP Built-in Web Server**

配置这个主要是开启一个PHP Server：

![image-20211007164326025](/media/image-20211007164326025.png)

**（3）配置PHP Web Page**

如果是本地Debug，则不需要勾选映射，远程Debug则需要。

![image-20211007165533318](/media/image-20211007165533318.png)

**（4）测试**

先启动 `PHP Built-in Web Server` ，然后启动 `PHP Web Page` 。在某个PHP上打个断点，即可断到：

![image-20211007165659930](/media/image-20211007165659930.png)
