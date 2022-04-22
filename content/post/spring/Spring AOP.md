---
typora-root-url: ../../../static
title: "Spring AOP"
date: 2020-09-02T14:40:36+08:00
draft: false
categories: ["Spring"]
---

## Spring AOP术语和流程
**Spring AOP是一种基于方法的AOP，它只能应用于方法上。** AOP的术语如下：

- 连接点（join point）：对应的是具体被拦截的对象，在Spring中指的是某个方法，AOP通过动态代理技术把它织入对应的流程中。
- 切点（point cut）：有时候切面不单单应用于单个方法，可能是多个类的不同方法，这是时，可以通过正则式和指示器的规则去定义，从而适配连接点。切点就是提供这样一个功能的概念。也就是 **匹配连接点的定义，一个切点可以匹配多个连接点。**
- 通知（advice）：织入到目标类连接点上的一段代码。分为前置通知(before advice)、后置通知(after advice)、环绕通知(around advice)、正常返回通知(afterReturning advice)和异常通知(afterThrowing advice)。
- 目标对象（target）：被代理对象。
- 织入（weaving）：通过动态代理技术，为原有服务对象生成代理对象，然后将与切点定义匹配的连接点拦截，并按照约定将各类通知织入约定流程的过程。
- 引入（introduction）：是一种特殊的增强，为类添加一些属性和方法。
- 切面（aspect）：一个可以定义切点、各类通知和引入的内容，Spring AOP将通过它的信息来增强Bean的功能或者将对应的方法织入流程。

## AOP开发
以下都是基于注解的Spring AOP：

- 使用@Aspect注解在类上表示这是一个切面。
- 使用@Pointcut("正则式")注解在切面的方法上，表示这是一个切点。
- 使用@Before("切点或者正则式"),@After("切点或者正则式"),@Around("切点或者正则式"),@AfterReturning("切点或者正则式"),@AfterThrowing("切点或者正则式")注解在方法上分别表示对应的通知。

一个例子如下：

	@Aspect
	public class UserAspect {
	
	    @Pointcut("execution(* com.example.springdemo.pojo.*.say(..))")
	    public void pointCut(){}
	
	    @Before("pointCut()")
	    public void before(){
	        System.out.println("我是前缀通知嗷~~~~");
	    }
	}

### 引入
一个例子如下：

    /**
     * 引入;相当于为TestService指定实现一个父接口UserValidator，其实现类为UserValidatorImpl
     */
    @DeclareParents(value = "com.example.springdemo.service.TestService+",
            defaultImpl = UserValidatorImpl.class)
    private UserValidator validator;

这样就可以将TestService向上强转成UserValidator类型对象，然后当成此类似对象来使用，达到增强的目的。

### 通知获取参数
1. 在通知中加入正则式，例如 `@Before("pointCut() && args(user)")` 代表将连接点名称为user的参数传递过来。
2. 在非环绕通知的形参中声明JoinPoint类型的形参，环绕通知中声明ProceedingJoinPoint类型的形参。

## 多个切面
Spring可以支持多个切面的运行，在组织多个切面的运行顺序时，可以使用@Order(数字)来设定切面的运行顺序。数字越大越早运行。同时也可以实现Order接口来实现规定运行顺序。
