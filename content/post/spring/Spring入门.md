---
typora-root-url: ../../../static
title: "Spring入门"
date: 2019-11-05T08:28:36+08:00
draft: false
categories: ["Spring"]
tags: ["Spring体系结构"]
---

## Spring简介
Spring是一个轻量级的Java开发框架， **目的是为了解决企业级应用开发的业务逻辑层和其他各层的耦合问题。** 

## Spring体系结构
Spring的体系结构图如下：

![Spring体系结构][p0]

### Core Container
- Core模块：提供框架的基本组成部分，包括控制反转（Inversion of Control，IoC）和依赖注入（Denpendency Injection，DI）。
- Beans：提供了BeanFactory，是工厂模式的典型实现， **Spring将管理对象称为Bean。** 
- Context：建立在Core和Beans的基础上，提供一个 **框架式的对象访问方式，是访问定义和配置的任何对象的媒介。** 常用接口为ApplicationContext。
- SpEl： **提供强大的表达式语言去支持运行时查询和操作对象图。** 

### Test
此模支持使用 **JUnit或TestNG** 对Spring组件进行单元测试和集成测试。

### AOP、Aspects、Instrumentation和Messaging
- AOP： **提供了一个符合AOP要求的面向切面的编程实现，允许定义方法拦截器和切入点，将代码按照功能进行分离，以便干净的解构。** 
- Aspects：提供了与AspectJ的集成功能，AspectJ是一个功能强大且成熟的AOP框架。
- Instrumentation：提供了类植入（Instrumentation）支持和类加载器的实现，可以在特定的应用服务器中使用。
- Messaging：此模块是Spring 4.0以后新增的，提供了对消息传递体系结构和协议的支持。

### Data Access/Integration
- JDBC：提供了一个JDBC的抽象层，消除了繁琐的JDBC编码和数据库厂商特有的错误代码解析。
- ORM：为流行的 **对象关系映射（Object Relational Mapping）API提供集成层。**
此模块可以将O/R映射框架与Spring提供的所有其他功能结合使用，例如声明式事务管理功能。
- OXM：提供了一个支持对象/XML映射的抽象层实现，例如：JAXB、Castor等。
- JMS：指JAVA消息传递服务（Java Message Server），包含用于生产和使用消息的功能。
- Transactions：支持用于实现特殊接口和所有POJO类的编程和声明式事务管理。

### Web
- WebSocket：Spring 4.0以后新增的模块，提供了WebSocket和SockJS的实现。
- Servlet：也成为webmvc模块，包含用于Web应用程序的SpringMVC和REST Web Services实现。Spring MVC框架提供了领域模型代码和Web表单之间的清晰分离，并与Spring FrameWork的所有其他功能集成。
- Web：提供了基本的Web开发集成功能，例如：文件上传、使用一个Servlet监听器初始化一个IoC容器以及Web应用上下文。
- Portlet：类似于Servlet模块的功能，提供了Portlet环境下的MVC的实现。

## Spring目录结构
下载Spring Framework解压后，得到如下图所示的目录结构：

![Spring目录结构][p1]

其中，docs目录是Spring的API文档和开发规范；libs是Spring应用所需要的JAR包和源代码；schema是Spring应用所需要的schema文件，这些schema文件定义了Spring相关配置文件的约束。其中， **libs下有三类JAR文件：以RELEASE.jar结尾的是开发时所需要的JAR包；以RELEASE-source.jar结尾的时源文件的压缩包；以RELEASE-javadoc.jar结尾的是API文档的压缩包。**

***Notice：*** Spring框架依赖于Apache Commons Logging组件，需要配合commons-logging的JAR包才能使用。

## Spring入门程序
1. 首先项目导入Spring的四个基本的依赖包：Core,Beans,Context,SpEL。以及Spring所依赖的commons-logging包。
2. 创建一个POJO：Student。
3. 创建Spring的配置文件applicationContext.xml，在此文件中配置Student交给Spring管理。

### 编码
项目导入Spring基础包和commons-logging包、创建POJO、创建Spring配置文件后的结构图如下：

![入门程序结构][p2]

Student的代码如下：

	public class Student {
	    private Integer id,number;
	    private String name,sex;
	
	    public Student() {
	        System.out.println("哈哈哈，我被实例化辽~");
	    }
	
	    public Integer getId() {
	        return id;
	    }
	
	    public void setId(Integer id) {
	        this.id = id;
	    }
	
	    public Integer getNumber() {
	        return number;
	    }
	
	    public void setNumber(Integer number) {
	        this.number = number;
	    }
	
	    public String getName() {
	        return name;
	    }
	
	    public void setName(String name) {
	        this.name = name;
	    }
	
	    public String getSex() {
	        return sex;
	    }
	
	    public void setSex(String sex) {
	        this.sex = sex;
	    }
	
	    @Override
	    public String toString() {
	        return "Student{" +
	                "id=" + id +
	                ", number=" + number +
	                ", name='" + name + '\'' +
	                ", sex='" + sex + '\'' +
	                '}';
	    }
	}

applicationContext.xml的内容如下：

	<?xml version="1.0" encoding="UTF-8"?>
	<beans xmlns="http://www.springframework.org/schema/beans"
	       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
	    <!--将Student交给Spring管理-->
	    <bean class="pojo.Student" id="student" />
	</beans>

### 测试
Main类中写了一个测试方法，如下：

    public static void main(String[] args) {
        System.out.println("-------创建Spring容器开始-------");
        ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring/applicationContext.xml");
        System.out.println("-------创建Spring容器结束-------");
        Student student0=(Student)applicationContext.getBean("student");
        Student student1=(Student)applicationContext.getBean("student");
        student0.setName("GIAO！");
        System.out.println(student1);
    }

输出如下：

![测试结果][p3]

可以看到，当创建Spring容器的时候，Spring会自动实例化配置了的bean，并且此时利用容器得到的bean都是同一个实例（student0调用setName()后影响到了student1的name，从而得出student0和student1是同一个对象）。









[p0]:/media/20191105-1.png
[p1]:/media/20191105-2.png
[p2]:/media/20191105-3.png
[p3]:/media/20191105-4.png