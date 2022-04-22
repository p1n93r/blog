---
typora-root-url: ../../../static
title: "CewlKid: 1"
date: 2020-09-23T19:51:36+08:00
draft: false
tags: ["valnhub"]
categories: ["security"]
---

## 介绍
An intermediate boot2root. The name is a hint. The start is CTF but the end is real world and worth the effort. Created in Virtualbox. **Goal: Get the root flag.** 

地址：[CewlKid: 1][l0]

## virtualbox网络设置
首先需要保证攻击机和靶子在同一网段，能互通，于是设置通过virtualbox的全局设置来配置网络。配置过程如下（保证两者在同一个网段且能通外网）：

① virtualbox全局设定网络

![全局设置网络][p0]

② 攻击机和靶机设置网络

![虚拟机设置网络][p1]

## 主机发现
需要先找到靶机的ip，于是扫描当前攻击机的统一网段下的其他主机，通过nmap扫描，扫描过程与结果如下(可以发现靶机ip为10.0.2.5)：

![主机发现][p2]

## 端口扫描
找到靶机后，就需要扫描其端口发现开启了哪些服务，是什么操作系统，根据这些信息选择从哪些方向渗透。使用nmap扫描端口的过程和结果如下：

![扫描端口][p3]

其中的 `-A` 代表扫描综合扫描，会扫描端口的服务、操作系统等； `-sS` 代表半开扫描； `-T4` 代表扫描速度为4（0-6）。

## 0day搜索
可以看到靶机开了ssh服务、还有两个web服务。先打开80端口的web服务，发现是一个
服务器默认展示页，没有利用点。然后打开8080端口的web服务，发现是一个叫sitemagic的CMS，于是去搜索这个CMS的漏洞。搜索结果如下：

![sitemagic的0day][p4]

意思就是如果你有账号登录这个系统，且没有设置FileFilter则会存在这个shell上传漏洞。所以接下来就是如何进入这个系统。

## 从CMS系统到getshell
去网上搜索这个CMS的默认账号密码，发现是admin/admin，但是测试无法登录，于是选择进行爆破，通过cewl爬取网站，得到一个网站内关键字相关的爆破字典，过程如下：

![cewl生成爆破字典][p5]

然后使用burpsuit进行爆破(加载cewl生成的字典)，如下：

![burpsuit爆破][p6]

然后登陆CMS，找到网站的上传点。先利用weevely生成php的木马，如下：

![weevely生成木马][p7]

然后将木马上传，上传成功后如下：

![上传木马][p8]

然后通过weevely访问这个木马，得到shell如下：

![得到shell][p9]

然后开始乱翻，翻到了一个文件，通过执行其他用户的cat命令打开文件内容如下：

![得到ssh用户密码][p10]

用base64解码得到两个用户的ssh用户密码。




[l0]:https://www.vulnhub.com/entry/cewlkid-1,559/


[p0]:/media/2020-09-23-1.png
[p1]:/media/2020-09-23-2.png
[p2]:/media/2020-09-23-3.png
[p3]:/media/2020-09-23-4.png
[p4]:/media/2020-09-23-5.png
[p5]:/media/2020-09-23-6.png
[p6]:/media/2020-09-23-7.png
[p7]:/media/2020-09-23-8.png
[p8]:/media/2020-09-23-9.png
[p9]:/media/2020-09-23-10.png
[p10]:/media/2020-09-23-11.png