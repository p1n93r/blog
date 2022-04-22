---
typora-root-url: ../../../static
title: "SpringMVC优雅的去除参数的前后空格"
date: 2020-04-15T00:38:36+08:00
draft: false
categories: ["SpringMVC"]
---

## 需要解决的问题
1. SpringMVC利用参数绑定接收的前端的参数，需要去除参数的前后空格。
2. SpringMVC利用@RequestBody接收的Json数据需要去除键值对的前后空格。

## 问题一的解决方法
可以自定义一个web参数绑定初始化器，然后在RequestMappingHandlerAdapter中配置webBindingInitializer。具体步骤如下：

① 通过实现WebBindingInitializer自定义一个web参数绑定初始化器：

	/**
	 * @author P1n93r
	 * 对参数绑定进行扩展
	 */
	@Component
	public class CustomWebBindingInitializer implements WebBindingInitializer {
	    @Override
	    public void initBinder(WebDataBinder webDataBinder) {
	        // 对字符串的参数绑定进行扩展，去除前后空格
	        // 注册对于String类型参数对象的属性进行trim操作的编辑器,
	        // 构造参数代表：空串是否转为null，true则将空串转为null
	        webDataBinder.registerCustomEditor(String.class, new StringTrimmerEditor(false));
	
	        //可以在此继续扩展~，比如扩展日期参数绑定
	    }
	}

② 在springmvc的全局配置文件中配置步骤①定义的初始化器：

    <!--手动配置RequestMappingHandlerAdapter实现自定义扩展-->
    <bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter">
        <!--添加自定义参数绑定-->
        <property name="webBindingInitializer" ref="customWebBindingInitializer"/>
    </bean>

## 问题二的解决方法
可以自定义一个jackson的objectMapper，然后配置json消息转换器的objectMapper为自定义的objectMapper。具体的步骤如下：

① 通过继承ObjectMapper来自定义一个ObjectMapper：

	/**
	 * @author P1n93r
	 * 自定义的一个jsonObject，用于去除接收的json数据中存在的前后空格情况
	 */
	@Component
	public class CustomObjectMapper extends ObjectMapper {
	    /**
	     * 重写构造器
	     */
	    public CustomObjectMapper(){
	        /*
	            调用ObjectMapper的registerModule方法添加扩展点
	            此处使用匿名对象直接new SimpleModul添加扩展功能
	        */
	        registerModule(new SimpleModule() {
	            {
	                /*
	                    注册对于String类型值对象的反序列化器
	                    对于反序列化器直接new StdDeserializer的子类StdScalarDeserializer完成
	                */
	                addDeserializer(String.class, new StdScalarDeserializer<String>(String.class) {
	                    @Override
	                    public String deserialize(JsonParser jp, DeserializationContext context) throws IOException {
	                        return StringUtils.trim(jp.getValueAsString());
	                    }
	                });
	                // ... 也可自定义其他类型序列化和反序列化器，例如：蛋疼的日期类型...
	            }
	        });
	    }
	}

② 在springmvc的全局配置文件中配置json消息转换器，并设置其ObjectMapper为步骤①定义的ObjectMapper：

    <!--手动配置RequestMappingHandlerAdapter实现自定义扩展-->
    <bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter">
        <!--添加自定义参数绑定-->
        <property name="webBindingInitializer" ref="customWebBindingInitializer"/>
        <property name="messageConverters">
            <list>
                <!--先进行string处理-->
                <bean class="org.springframework.http.converter.StringHttpMessageConverter">
                    <constructor-arg index="0" value="UTF-8"/>
                </bean>
                <!--实现自定义Jackson消息转换，以完成以json形式对对象进行序列化和反序列化以及配置支持的media-type-->
                <bean class="org.springframework.http.converter.json.MappingJackson2HttpMessageConverter">
                    <property name="objectMapper" ref="customObjectMapper"/>
                    <property name="supportedMediaTypes">
                        <list>
                            <value>text/plain;charset=UTF-8</value>
                            <value>application/json;charset=UTF-8</value>
                        </list>
                    </property>
                </bean>
            </list>
        </property>
    </bean>

***Notice：*** 配置json消息转换器之前一定要先配置一个String消息转换器，否则控制器返回给前端的json数据会被一个双引号 `""` 包围。RequestMappingHandlerAdapter的配置也需要在注解驱动 `<mvc:annotation-driven />` 之前。



