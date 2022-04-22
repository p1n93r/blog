---
typora-root-url: ../../../static
title: "IDEA之Maven疑难杂症"
date: 2020-03-09T19:29:36+08:00
draft: false
categories: ["Other"]
tags: ["Maven"]
---

## 待解决的问题
1. IDEA的Maven项目从模板构建卡住的问题。
2. Maven下载依赖包提示SSL错误的问题。
3. Maven下载依赖包速度慢的问题。

## 优化步骤
如果使用Maven从模板创建一个项目，那么Maven会从外网下载一个叫做archetype-catalog.xml的文件，此文件大小8m左右，下载速度却只有20kb/s。所以解决方法是直接想办法下载这个xml文件，然后指定IDEA使用模板构建项目的时候，不从远程下载此xml文件，而使用本地的此xml文件。步骤如下：

1. 进入IDEA的Maven的全局设置（设置路径：Settings | Build, Execution, Deployment | Build Tools | Maven | Runner），设置其VM Options如图1。
2. 将<a href="/files/archetype-catalog.xml" download="">archetype-catalog.xml</a>文件保存在你的电脑的 `${user.home}/.m2/` 目录下（${user.home}意思是你的电脑的用户目录）。
3. 打开你的IDEA的Maven的全局配置文件setting.xml,添加阿里的Maven仓库镜像，如图2。

![图1 Maven参数设置][p0]
<center><h6>图1 Maven参数设置</h6></center>

![图2 Maven添加镜像][p1]
<center><h6>图2 Maven添加镜像</h6></center>

## 供复制内容
### VM Options
```
-Dmaven.wagon.http.ssl.insecure=true
-Dmaven.wagon.http.ssl.allowall=true
-Dmaven.wagon.http.ssl.ignore.validity.dates=true
-DarchetypeCatalog=internal
```

### 阿里镜像
	<mirror>
		<id>alimaven</id>
		<name>aliyun maven</name>
		<url>http://maven.aliyun.com/nexus/content/groups/public/</url>
		<mirrorOf>central</mirrorOf> 
	</mirror>




[p0]:/media/20200309-1.png
[p1]:/media/20200309-2.png