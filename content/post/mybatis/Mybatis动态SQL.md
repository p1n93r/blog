---
typora-root-url: ../../../static
title: "Mybatis动态SQL"
date: 2019-09-24T08:22:36+08:00
draft: false
categories: ["Mybatis"]
---

## 前言
Mybatis可以提供动态组装SQL语句的功能，称为动态SQL。

## 一个综合例子
这个例子缺少\<set\>元素，\<foreach\>元素

    <select id="selectAll" resultMap="selectAllResMap" parameterType="CustomQuerryStudentParam">
        select a.*,b.j2ee,b.android,b.sql,(b.j2ee+b.android+b.sql) as total_score
            from student a,student_score b
        <where>
            a.id=b.id
            <!--根据学号查询-->
            <if test="studentNumber!=null and studentNumber!=0">
                and a.student_number=#{studentNumber}
            </if>
            <!--根据姓名模糊查询-->
            <if test="studentName!=null and studentName!=''">
                <bind name="fuzzyName" value="'%'+studentName+'%'"/>
                and a.student_name like #{fuzzyName}
            </if>
            <!--根据班级查询-->
            <if test="studentClass!=null and studentClass!=0">
                and a.student_class=#{studentClass}
            </if>
            <!--排序-->
            order by
            <choose>
                <!--这里不能直接用${}将前台的数据存到这，会造成order by型sql注入-->
                <!--于是我对前台的数据进行数据对比，反正就是不能直接将前台的数据放这-->
                <when test="sort!=null and sort=='studentNumber'">a.student_number</when>
                <when test="sort!=null and sort=='studentClass'">a.student_class</when>
                <when test="sort!=null and sort=='sql'">b.`sql`</when>
                <when test="sort!=null and sort=='android'">b.android</when>
                <when test="sort!=null and sort=='j2ee'">b.j2ee</when>
                <otherwise>total_score</otherwise>
            </choose>
            <!--排序的方式（asc还是desc）-->
            <choose>
                <when test="isAsc!=null and isAsc==false">desc</when>
                <otherwise>asc</otherwise>
            </choose>
            <!--分页-->
            <if test="offset!=null">
                <if test="pageSize!=null and pageSize!=0">
                    limit #{offset},#{pageSize}
                </if>
            </if>
        </where>
    </select>

## \<if\>元素
if元素可以使得有条件的包含where字句的一部分，类似高级语言中的if语句，需注意的地方是： **if元素的test属性为判断条件，里面不能写&&和||，而是and和or，而且取值直接写标识符即可，无需用#和$符号**

## \<choose\>,\<when\>,\<otherkwise\>元素
类似高级语言的swich语句，当要多个条件选择一个时使用，同时还可以给个默认的条件——用\<otherwise\>元素即可

## \<where\>元素
where元素的好处如下：

1. 会自动在写where元素的地方输出一个where语句
2. 如果where里的所有条件都不满足，则mybatis会输出所有记录
3. 会自动去掉第一个输出条件的开头的and或者or
4. 无需考虑空格问题，比如两个查询条件之间的空格，mybatis会自动处理

## \<bind\>元素
主要用在模糊查询的拼接%符号的，模糊查询时，不要使用${}和%拼接，会造成sql注入。

## \<set\>元素
主要用在动态更新列，需要注意的是： **set元素内的多个if元素里的内容，需要以逗号结尾** ，因为set元素不会自动去掉第一个输出条件的开头的逗号，所以我们把逗号写在后面，\<set\>元素会自动去除其内的最后一个逗号。一个例子如下：

    <update id="updateOne" parameterType="Student">
        update student a,student_score b
        <set>
            <if test="studentNumber!=0">
                a.student_number=#{studentNumber},
            </if>
            <if test="studentClass!=0">
                a.student_class=#{studentClass},
            </if>
            <if test="studentName!=null and studentName!=''">
                a.student_name=#{studentName},
            </if>
            <if test="studentScore.j2ee!=0">
                b.j2ee=#{studentScore.j2ee},
            </if>
            <if test="studentScore.android!=0">
                b.android=#{studentScore.android},
            </if>
            <if test="studentScore.sql!=0">
                b.`sql`=#{studentScore.sql}
            </if>
        </set>
        <where>
            a.id=b.id
            <if test="id!=0">
                and a.id=#{id}
            </if>
        </where>
    </update>

## \<foreach\>元素
类似JSTL的\<c:foreach\>标签，一个例子如下：

    <!--批量删除-->
    <delete id="deleteByRange" parameterType="List">
        delete from student where id in
        <foreach collection="list" open="(" close=")" separator="," item="v">
            #{v}
        </foreach>
    </delete>

## \<trim\>元素
比较少用，主要作用是加前后缀，以及覆盖前后缀。