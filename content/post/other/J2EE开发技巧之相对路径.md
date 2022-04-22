---
typora-root-url: ../../../static
title: "J2EE开发技巧之相对路径"
date: 2019-12-03T11:27:36+08:00
draft: false
categories: ["Other"]
tags: ["J2EE开发技巧"]
---

## 前言
在进行WEB开发时，如果一个项目比较大，分了很多的模块，那么必定项目的url也很多，如果url没有统一的规范，极易容易造成混乱，本文就是为了解决这个问题。主要思路要做到所有的视图页面内的连接都是相对于类似这种链接： `https://host:88/projectName/` ，其中的host就是域名，projectName就是你的项目名。如果我们能够做到所有的视图内的链接都是基于这个路径，一是有了一个统一的规范不容易出错，二是能够保证视图内的链接的点击不会出现死链接。

## 开发思路
1. 写一个自定义标签类，作用就是在html的 `<head/>` 标签内输出网站的basePath（例如 `<base href='http://localhost:8080/TheSecondLab_war_exploded/'/>` ）。
2. 在jsp页面中调用这个自定义标签（需要一个HttpServletRequst对象作为参数传进去），就可以做到输出basePath了，这样页面内的所有链接都是相对于这个basePath。

### 创建自定义标签类
我写的一个自定义标签类代码如下：

	public class BaseUrlTag extends SimpleTagSupport {
	    //为了得到basePath，需要一个HttpServletRequest对象
	    HttpServletRequest request;
	    //打印网站的basePath
	    @Override
	    public void doTag() throws JspException, IOException{
	        String path = request.getContextPath();
	        String basePath = "<base href='"+request.getScheme()+"://"+request.getServerName()+":"+request.getServerPort()+path+"/'/>";
	        getJspContext().getOut().write(basePath);
	    }
	    public void setRequest(HttpServletRequest request) {
	        this.request = request;
	    }
	}

### 创建tld文件
在项目的WEB-INF下创建与上面自定义标签类相关的tld文件，我写得tld文件如下：

	<!DOCTYPE taglib
	        PUBLIC "-//Sun Microsystems, Inc.//DTD JSP Tag Library 1.2//EN"
	        "http://java.sun.com/dtd/web-jsptaglibrary_1_2.dtd">
	<!-- 标签库描述符 -->
	<taglib xmlns="http://java.sun.com/JSP/TagLibraryDescriptor">
	    <tlib-version>1.0</tlib-version>
	    <jsp-version>1.2</jsp-version>
	    <short-name>bp</short-name>
	    <uri>/setBasePath</uri>
	    <tag>
	        <!-- 标签名 -->
	        <name>basePath</name>
	        <!-- 标签助手类(自定义的标签类) 放入类的全限定名 -->
	        <tag-class>com.mybatis.web.tag.BaseUrlTag</tag-class>
	        <!-- 标签的内容类型：empty表示空标签(使用空标签会报错)，jsp表示可以为任何合法的JSP元素 -->
	        <!--SimpleTagSupport不支持设置为JSP-->
	        <body-content>empty</body-content>
	        <attribute>
	            <!-- 属性名 -->
	            <name>request</name>
	            <!-- 是否为必填项 -->
	            <required>true</required>
	            <!--是否可以填jsp表达式  el表达式 -->
	            <rtexprvalue>true</rtexprvalue>
	            <type>javax.servlet.http.HttpServletRequest</type>
	        </attribute>
	    </tag>
	</taglib>

### 手动配置tld的位置
在项目的web.xml中配置上一步创建的tld的位置，防止系统提示找不到tld，配置的主要代码如下（上一步我创建的tld文件名为BasePathTag.tld）：

    <!--手动配置tld的位置（用于basePath的tld），不然提示找不到-->
    <jsp-config>
        <taglib>
            <taglib-uri>/setBasePath</taglib-uri>
            <taglib-location>/WEB-INF/tags/BasePathTag.tld</taglib-location>
        </taglib>
    </jsp-config>

### 在jsp中调用自定义标签
1. 首先jsp引入刚才自定义的标签，引入方式为： `<%@taglib prefix="bp" uri="/setBasePath" %>` 。
2. 其次在jsp页面的 `<head/>` （这是html的标记）标签内通过 `    <bp:basePath request="<%=request%>"/>` 调用刚才引入的自定义标签。
3. 明白本jsp页面内的所有超链接都是基于basePath来设计，比如 `<a href="test.do">` （test.do前没有 `/` ，因为我设计输出的basePath以 `/` 结尾）的物理路径为 `basePath/test.do` 。
4. 重启项目，访问jsp，查看渲染后的html代码，观察是否输出了basePath。

### 测试
访问调用了自定义标签的jsp页面，通过浏览器，按ctr+u可以查看渲染后的html源码，观察结果如下（输出了basePath）：

![测试结果][p0]

[p0]:/media/20191204-1.png