---
typora-root-url: ../../../static
title: "J2EE开发技巧之字符编码过滤器"
date: 2019-12-10T11:04:36+08:00
draft: false
categories: ["Other"]
tags: ["J2EE开发技巧"]
---

## 前言
j2ee开发过程中，经常会遇到编码问题，比如从前台传到后台的数据如果是中文的话，后台接收到的却是乱码；后台返回给前台的数据是中文，前台显示的却是乱码等等。。。为了解决这个问题，可以自己写一个字符编码过滤器，对每一个request和response设置UTF-8编码。（如果使用SpringMVC框架，可以使用SpringMVC自带的编码过滤器，后面也会贴出来）

## 开发过滤器
只需要在项目中创建如下过滤器即可：

	@WebFilter(filterName = "EncodingFilter",value = "/*")
	public class EncodingFilter implements Filter {
	    public void destroy() {
	    }
	
	    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain) throws ServletException, IOException {
	        //一个简单的字符编码（防止中文乱码）
	        HttpServletRequest request= (HttpServletRequest) req;
	        HttpServletResponse response= (HttpServletResponse)resp;
	        request.setCharacterEncoding("UTF-8");
	        response.setCharacterEncoding("UTF-8");
	        response.setContentType("text/html;charset=UTF-8");
	        chain.doFilter(req, resp);
	    }
	
	    public void init(FilterConfig config) throws ServletException {
	
	    }
	
	}

## SpringMVC的编码过滤器
SpringMVC框架自带编码过滤器，可以在web.xml里配置，主要配置如下：

    <!--编码-->
    <filter>
        <description>字符集过滤器</description>
        <filter-name>characterEncodingFilter</filter-name>
        <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
        <init-param>
            <description>字符集编码</description>
            <param-name>encoding</param-name>
            <param-value>UTF-8</param-value>
        </init-param>
        <init-param>
            <param-name>forceEncoding</param-name>
            <param-value>true</param-value>
        </init-param>
    </filter>
    <filter-mapping>
        <filter-name>characterEncodingFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>