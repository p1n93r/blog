---
typora-root-url: ../../../static
title: "Mybatis级联查询"
date: 2019-09-17T11:19:36+08:00
draft: false
categories: ["Mybatis"]
---

## 模拟需求
- 一对一关系：一个学生（Student）有一个地址（Address），一个地址只能被一个学生拥有。
- 一对多关系：一个学生可以有多个订单（Order），一个订单只能被一个学生拥有。
- 多对多关系：一个订单可以有多个商品（Item），同时一个商品可以被多个订单拥有。

根据上面的需求定义以下实体：

- Student:`int id,age;String name,grade,country;Address address;List<Order> orderList;`
- Address:`int id;String province,city,detail_address;`
- Order:`int id,item_id,sid`;

**由于Order的记录只存储Item的id，所以暂时不写Item实体，用item_id代表一个Item对象。**

## 说明
mybatis中通过`<resultMap>`的子元素`<association>`来处理一对一级联查询。`<association>`子元素通常含有以下属性：

1. property：指定映射到实体类的对象属性。
2. column：指定表中的字段（当通过执行两个sql语句来实现级联查询时用到）。
3. javaType：指定映射到实体类对象属性的类型。
4. select：指定引入嵌套查询的子sql语句（一般和column结合使用）。

mybatis中通过`<resultMap>`的子元素`<collection>`来处理一对多级联查询。`<collection>`子元素通常含有以下属性：

1. property：指定映射到实体类的对象属性。
2. column：指定表中的字段。
3. ofType：指定实体的对象属性的类型。
4. select：指定引入嵌套查询的子sql语句。

级联查询有以下三种方法：

1. 执行一条sql语句，将属于子表的部分映射到实体的对象属性的属性上。
2. 执行两条sql语句，将嵌套子查询结果映射到实体的对象属性的属性上。
3. 执行一条sql语句，将查询结果映射到一个包含父表和子表字段的实体的属性上。

### 方法一
一个resultMap的例子如下：

    <!--执行一条sql语句-->
    <resultMap id="oneToOneResMap" type="com.mybatis.pojo.Student">
        <id property="id" column="id"/>
        <result property="age" column="age"/>
        <result property="name" column="name"/>
        <result property="grade" column="grade"/>
        <result property="country" column="country"/>
        <!--一对一的级联查询-->
        <association property="address" javaType="com.mybatis.pojo.Address">
            <id property="id" column="addr_id"/>
            <result property="province" column="province"/>
            <result property="city" column="city"/>
            <result property="detail_address" column="detail_address"/>
        </association>
        <!--一对多级联查询-->
        <collection property="orderList" ofType="com.mybatis.pojo.Order">
            <id property="id" column="oid"/>
            <result property="item_id" column="item_id"/>
        </collection>
    </resultMap>

使用此resultMap的`<select>`元素为：

    <!--定义一个select语句片段-->
    <sql id="selectSql">
        select a.*,b.province,b.city,b.detail_address,c.id as oid,c.item_id from student a,address b,`order` c where a.addr_id=b.id and c.sid=a.id
    </sql>
    <!--查询所有student记录-->
    <select id="selectAll" resultMap="oneToOneResMap">
        <include refid="selectSql"/>
    </select>

### 方法二
一个resultMap的例子如下：

    <!--执行两条sql语句-->
    <resultMap id="oneToOneResMap1" type="java.util.Map">
        <id property="id" column="id"/>
        <result property="age" column="age"/>
        <result property="name" column="name"/>
        <result property="grade" column="grade"/>
        <result property="country" column="country"/>
        <!--一对一的级联查询-->
        <association property="address" column="addr_id" javaType="com.mybatis.pojo.Address" select="com.mybatis.dao.AddressDao.selectById"/>
        <!--一对多级联查询-->
        <collection property="orderList" column="id" ofType="com.mybatis.pojo.Order" select="com.mybatis.dao.OrderDao.selectByOid"/>
    </resultMap>

namespace为com.mybatis.dao.OrderDao，且id为selectByOid的`<select>`元素为：

    <select id="selectByOid" parameterType="Integer" resultType="com.mybatis.pojo.Order">
        select * from `order` where sid=#{id}
    </select>

使用此resultMap的`<select>`元素为：

    <!--根据id查询一条student记录(利用Map存储结果集)-->
    <select id="selectById" parameterType="Integer" resultMap="oneToOneResMap1">
        select * from student where id=#{id}
    </select>

### 方法三
这个比较简单，就是写一个实体类，这个实体类包含了各个级联表的各个属性。然后就像正常使用resultMap一样，将resultMap的type属性设置为此实体类即可。此方法不太好，不做研究。
