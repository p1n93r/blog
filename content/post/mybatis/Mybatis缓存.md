---
typora-root-url: ../../../static
title: "Mybatis缓存"
date: 2019-10-26T16:24:36+08:00
draft: false
categories: ["Mybatis"]
---

## 概述
Mybatis主要有一级缓存和二级缓存，mybatis默认开启一级缓存，需要手动开启二级缓存。两者特性如下：

1. 一级缓存：是相对于同一个SqlSession而言；缓存存储介质为内存；**一级缓存刷新的标志是SqlSession执行commit。**
2. 二级缓存：是相对于同一个namespace的mapper而言；存储介质可以为磁盘，所以要缓存的实体需要实现Serializable接口；**SqlSession提交或者关闭后，二级缓存才会生效，二级缓存刷新的标志不是SqlSession执行commit。**

## 一级缓存
mybatis默认就是开启并使用的一级缓存，同一个SqlSession而言，执行多次查询，只会执行一次sql语句，并将结果存储到缓存中，后面的查询都是从缓存中拿取。一个测试方法例子如下：

    @Test
    public void test1(){
        SqlSession sqlSession = factory.openSession();
        SellMapper mapper = sqlSession.getMapper(SellMapper.class);
        System.out.println(mapper.selectAll("asc").size());
		//mapper.deleteById(2);
        //sqlSession.commit();
        System.out.println(mapper.selectAll("asc").size());
    }

查看执行日志如下：

	DEBUG [main] - Opening JDBC Connection
	DEBUG [main] - Created connection 2147046752.
	DEBUG [main] - Setting autocommit to false on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@7ff95560]
	DEBUG [main] - ==>  Preparing: select id,place,spring,summer,autumn,winter,(spring+summer+autumn+winter) as total from sell order by total asc 
	DEBUG [main] - ==> Parameters: 
	DEBUG [main] - <==      Total: 24
	24
	24

可以看到，第二次执行查询操作，没有再次执行sql语句了，总共只执行了一次sql语句，第二次的取值是从缓存中拿到的。那么两次查询之间执行一次commit操作呢（虽然没有真正的对数据库做出修改）？将上面测试方法中的`sqlSession.commit();`的注释去掉，再次执行得到结果如下：

	DEBUG [main] - Opening JDBC Connection
	DEBUG [main] - Created connection 2147046752.
	DEBUG [main] - Setting autocommit to false on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@7ff95560]
	DEBUG [main] - ==>  Preparing: select id,place,spring,summer,autumn,winter,(spring+summer+autumn+winter) as total from sell order by total asc 
	DEBUG [main] - ==> Parameters: 
	DEBUG [main] - <==      Total: 24
	24
	DEBUG [main] - ==>  Preparing: select id,place,spring,summer,autumn,winter,(spring+summer+autumn+winter) as total from sell order by total asc 
	DEBUG [main] - ==> Parameters: 
	DEBUG [main] - <==      Total: 24
	24

可以看到，**虽然此时没有对数据库做出任何修改，但是只要执行了commit操作，就会刷新缓存，**执行了两次sql语句。那么我们在两次查询之间做出对数据库的更改的操作，但是不commit呢？将上面测试方法中的`mapper.deleteById(2);`的注释去掉，再次执行得到结果如下：

	DEBUG [main] - Opening JDBC Connection
	DEBUG [main] - Created connection 2147046752.
	DEBUG [main] - Setting autocommit to false on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@7ff95560]
	DEBUG [main] - ==>  Preparing: select id,place,spring,summer,autumn,winter,(spring+summer+autumn+winter) as total from sell order by total asc 
	DEBUG [main] - ==> Parameters: 
	DEBUG [main] - <==      Total: 24
	24
	DEBUG [main] - ==>  Preparing: delete from sell where id=? 
	DEBUG [main] - ==> Parameters: 2(Integer)
	DEBUG [main] - <==    Updates: 0
	DEBUG [main] - ==>  Preparing: select id,place,spring,summer,autumn,winter,(spring+summer+autumn+winter) as total from sell order by total asc 
	DEBUG [main] - ==> Parameters: 
	DEBUG [main] - <==      Total: 24
	24

可以看到，还是执行了两次sql语句，缓存被刷新，**所以虽然没有执行commit操作，但是执行的是对数据库进行更新的操作，都会刷新缓存（因为这些操作的元素默认启用了属性：flushCache=true）。**

## 二级缓存
二级缓存需要我们手动开启，它是相对于namespace相同的mapper而言，被缓存的实体需要实现serializable接口，因为缓存存储介质可能是磁盘。开启二级缓存的步骤如下：

1. 在mybatis的全局配置文件中设置：`<setting name="cacheEnabled" value="true" />` （默认为true，可以不配置）。
2. 在需要启动二级缓存的映射文件中添加元素：`<cache/>` 。
3. 在映射文件的select元素中添加属性：`useCache=true` （默认为true，可以不配置）。
4. 在映射文件的insert、update、delete元素中添加属性：`flushCache=true` （默认为true，可以不配置）。

***注意：*** 二级缓存是在SqlSession提交或者关闭后才会生效，未提交之前，如果使用同一个SqlSession再次进行查询，使用的是一级缓存，提交之后，虽然用的是同一个SqlSession，但是用的是二级缓存。

步骤2中的cache元素有如下表所示属性：

|属性|说明|
|----|----|
|eviction|清除缓存的策略，取值有LRU，FIFO ，SOFT，WEAK|
|flushInterval|刷新缓存的时间间隔，单位为毫秒。 默认情况是不设置，也就是没有刷新间隔，缓存仅仅会在调用语句时刷新。|
|readOnly|返回对象是否只可读，取值可以为true或者false。|
|size|存储结果对象或列表的引用大小，默认值是 1024。|

其中的eviction的取值说明如下表：

|eviction取值|说明|
|-----------|----|
|LRU|最近最少使用：移除最长时间不被使用的对象。|
|FIFO|先进先出：按对象进入缓存的顺序来移除它们。|
|SOFT|软引用：基于垃圾回收器状态和软引用规则移除对象。|
|WEAK|弱引用：更积极地基于垃圾收集器状态和弱引用规则移除对象。|

### 测试分析
一个测试代码如下：

    @Test
    public void test2() throws FileNotFoundException {
        SqlSession sqlSession = factory.openSession();
        SellMapper mapper = sqlSession.getMapper(SellMapper.class);
        System.out.println(mapper.selectAll("asc").size());
        //sqlSession.commit();
        System.out.println(mapper.selectAll("asc").size());
        //mapper.deleteById(1);
        //sqlSession.commit();
        sqlSession.close();

        //创建另一个sqlSession
        SqlSession sqlSession1 = factory.openSession();
        SellMapper mapper1 = sqlSession1.getMapper(SellMapper.class);
        System.out.println(mapper1.selectAll("asc").size());
        sqlSession1.close();
    }

其中的第一个SqlSession在提交之前进行了两次查询，因为二级缓存只有提交或者关闭后才能生效，所以在第一个SqlSession提交之前，二级缓存是没有生效的，所以第一个SqlSession的第二个查询使用的是一级缓存。第一个SqlSession提交后，二级缓存生效，所以第二个SqlSession的查询使用的是二级缓存。日志输出如下：

	DEBUG [main] - Cache Hit Ratio [mapper.SellMapper]: 0.0
	DEBUG [main] - Opening JDBC Connection
	DEBUG [main] - Created connection 502800944.
	DEBUG [main] - Setting autocommit to false on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@1df82230]
	DEBUG [main] - ==>  Preparing: select id,place,spring,summer,autumn,winter,(spring+summer+autumn+winter) as total from sell order by total asc 
	DEBUG [main] - ==> Parameters: 
	DEBUG [main] - <==      Total: 24
	24
	DEBUG [main] - Cache Hit Ratio [mapper.SellMapper]: 0.0
	24
	DEBUG [main] - Resetting autocommit to true on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@1df82230]
	DEBUG [main] - Closing JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@1df82230]
	DEBUG [main] - Returned connection 502800944 to pool.
	DEBUG [main] - Cache Hit Ratio [mapper.SellMapper]: 0.3333333333333333
	24

可以看到，第二次查询的Cache Hit Ratio为0，但是又没有执行sql语句，所以此时是用一级缓存中取的值。第三次的Cache Hit Ratio就为0.333，代表总共请求了3次缓存，有一次成功了（第一次没有缓存，第二次缓存没有生效，第三次成功取得缓存）。接下来仅将上面测试代码中的两次查询之间的commit语句的注释去掉再次执行，日志输出如下：

	DEBUG [main] - Cache Hit Ratio [mapper.SellMapper]: 0.0
	DEBUG [main] - Opening JDBC Connection
	DEBUG [main] - Created connection 502800944.
	DEBUG [main] - Setting autocommit to false on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@1df82230]
	DEBUG [main] - ==>  Preparing: select id,place,spring,summer,autumn,winter,(spring+summer+autumn+winter) as total from sell order by total asc 
	DEBUG [main] - ==> Parameters: 
	DEBUG [main] - <==      Total: 24
	24
	DEBUG [main] - Cache Hit Ratio [mapper.SellMapper]: 0.5
	24
	DEBUG [main] - Resetting autocommit to true on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@1df82230]
	DEBUG [main] - Closing JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@1df82230]
	DEBUG [main] - Returned connection 502800944 to pool.
	DEBUG [main] - Cache Hit Ratio [mapper.SellMapper]: 0.6666666666666666
	24

可以看到，第二次查询的缓存命中率（二级缓存）为0.5，所以第二次查询就已经获取到了二级缓存中的值。接下来仅将`mapper.deleteById(1)` 及紧接着它下面的 `sqlSession.commit()` 的注释去掉，得到如下日志输出：

	DEBUG [main] - Cache Hit Ratio [mapper.SellMapper]: 0.0
	DEBUG [main] - Opening JDBC Connection
	DEBUG [main] - Created connection 502800944.
	DEBUG [main] - Setting autocommit to false on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@1df82230]
	DEBUG [main] - ==>  Preparing: select id,place,spring,summer,autumn,winter,(spring+summer+autumn+winter) as total from sell order by total asc 
	DEBUG [main] - ==> Parameters: 
	DEBUG [main] - <==      Total: 24
	24
	DEBUG [main] - Cache Hit Ratio [mapper.SellMapper]: 0.0
	24
	DEBUG [main] - ==>  Preparing: delete from sell where id=? 
	DEBUG [main] - ==> Parameters: 1(Integer)
	DEBUG [main] - <==    Updates: 0
	DEBUG [main] - Committing JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@1df82230]
	DEBUG [main] - Resetting autocommit to true on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@1df82230]
	DEBUG [main] - Closing JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@1df82230]
	DEBUG [main] - Returned connection 502800944 to pool.
	DEBUG [main] - Cache Hit Ratio [mapper.SellMapper]: 0.0
	DEBUG [main] - Opening JDBC Connection
	DEBUG [main] - Checked out connection 502800944 from pool.
	DEBUG [main] - Setting autocommit to false on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@1df82230]
	DEBUG [main] - ==>  Preparing: select id,place,spring,summer,autumn,winter,(spring+summer+autumn+winter) as total from sell order by total asc 
	DEBUG [main] - ==> Parameters: 
	DEBUG [main] - <==      Total: 24
	24
	DEBUG [main] - Resetting autocommit to true on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@1df82230]
	DEBUG [main] - Closing JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@1df82230]
	DEBUG [main] - Returned connection 502800944 to pool.

可以看到，第二次查询的二级缓存命中率为0，但是从一级缓存中取到了值（此时二级缓存还未生效）。第三次查询时命中率还是为0，并且执行了一遍sql。因为所有开启了二级缓存的映射文件中的insert、delete、update元素默认开启了flushCache=true，所以执行了`mapper.deleteById(1)` 会刷新二级缓存，所以第三次查询只能再次执行一次sql。