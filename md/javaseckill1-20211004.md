> 本文为[Java高并发秒杀API之业务分析与DAO层](http://www.imooc.com/learn/587)课程笔记。

编辑器：IDEA

java版本：java8

## 介绍

**学习目标**：

<img src="https://gitee.com/hqinglau/img/raw/master/img/20211004142315.png" alt="image-20211004142313232" style="zoom:80%;" />

**秒杀业务**：

- 具有典型的“事务”特性

  > 事务的四大特性主要是：原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）、持久性（Durability）。 原子性是指事务是一个不可分割的工作单位，事务中的操作要么全部成功，要么全部失败

- 需求越来越常见

- 面试常问

**前提内容（链接可点）**：[J2EE](https://orzlinux.cn/blog/javaweb20210924.html)、[spring一](https://orzlinux.cn/blog/spring20210926.html)、[模拟一个简单的tomcat](https://orzlinux.cn/blog/tomcat20210926.html)、[SpringMVC和SSM](https://orzlinux.cn/blog/javaframe20210927.html)、[动态代理、IoC、AOP](https://orzlinux.cn/blog/javaAOP20211001.html)、[SpringBoot一](https://orzlinux.cn/blog/springboot20210928.html)、[SpringBoot二、Thymeleaf](https://orzlinux.cn/blog/springboot20210929.html)、[Java反射三四例](https://orzlinux.cn/blog/javareflect20210919.html)。

**具体学习**：

MySQL：表设计、SQL技巧、事务和行级锁。

MyBatis：DAO层设计开发、合理使用、与Spring整合。

Spring：IOC整合Service、声明式事务运用。

SpringMVC：Restful接口设计和使用、框架运作流程、Controller技巧。

前端：交互设计、Bootstrap、jQuery。

高并发：高并发点和分析、优化思路。

## 创建项目

![image-20211004145838853](https://gitee.com/hqinglau/img/raw/master/img/20211004145840.png)

创建得到目录如下：

![image-20211004151244385](https://gitee.com/hqinglau/img/raw/master/img/20211004151245.png)

下一步，右键项目名，添加java和resources、test文件夹，IDEA会给相应的提示：

![image-20211004151318865](https://gitee.com/hqinglau/img/raw/master/img/20211004151320.png)

之后目录为：

![image-20211004152327433](https://gitee.com/hqinglau/img/raw/master/img/20211004152328.png)



web.xml修改servlet版本：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
                      http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1" metadata-complete="true">
</web-app>
```

pom.xml依赖配置，详见注释：

```xml
<dependencies>
    <dependency>
        <!--测试：junit4-->
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.11</version>
        <scope>test</scope>
    </dependency>
    <!--日志：slf4j,log4j,logback,common-logging-->
    <!--slf4j是规范/接口-->
    <!--日志实现：log4j,logback,common-logging-->
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>1.7.12</version>
    </dependency>
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-core</artifactId>
        <version>1.1.1</version>
    </dependency>
    <!--实现slf4j接口并且整合-->
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.1.1</version>
    </dependency>

    <!--数据库相关依赖-->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>5.1.35</version>
        <!--驱动只有真正工作才会用到-->
        <scope>runtime</scope>
    </dependency>
    <!--数据库连接池-->
    <dependency>
        <groupId>c3p0</groupId>
        <artifactId>c3p0</artifactId>
        <version>0.9.1.2</version>
    </dependency>

    <!--DAO框架依赖：mybatis-->
    <dependency>
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis</artifactId>
        <version>3.5.2</version>
    </dependency>
    <!--mybatis 自身实现的spring整合依赖-->
    <dependency>
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis-spring</artifactId>
        <version>1.3.0</version>
    </dependency>
    <!--servlet web相关依赖-->
    <!--标签库-->
    <dependency>
        <groupId>taglibs</groupId>
        <artifactId>standard</artifactId>
        <version>1.1.2</version>
    </dependency>
    <dependency>
        <groupId>jstl</groupId>
        <artifactId>jstl</artifactId>
        <version>1.2</version>
    </dependency>
    <!--Jackson 是当前用的比较广泛的，用来序列化和反序列化 json 的 Java 的开源框架-->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.8.10</version>
    </dependency>

    <dependency>
        <groupId>javax.servlet</groupId>
        <artifactId>javax.servlet-api</artifactId>
        <version>3.1.0</version>
    </dependency>

    <!--spring相关依赖-->
    <!--spring核心依赖-->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-core</artifactId>
        <version>5.3.9</version>
    </dependency>

    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-beans</artifactId>
        <version>5.3.9</version>
    </dependency>
    <!--包扫描-->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>5.3.9</version>
    </dependency>
    <!--dao依赖-->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-jdbc</artifactId>
        <version>5.1.4.RELEASE</version>
    </dependency>
    <!--事务-->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-tx</artifactId>
        <version>5.1.4.RELEASE</version>
    </dependency>
    <!--spring web-->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-web</artifactId>
        <version>5.3.9</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
        <version>5.3.9</version>
    </dependency>
    <!--spring test相关依赖-->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-test</artifactId>
        <version>5.3.9</version>
    </dependency>
</dependencies>
```

## 秒杀业务分析

### 秒杀系统业务流程

核心在于**对库存的处理**。

<img src="https://gitee.com/hqinglau/img/raw/master/img/20211004164809.png" alt="image-20211004164748817" style="zoom:80%;" />



用户针对库存业务分析：

<img src="https://gitee.com/hqinglau/img/raw/master/img/20211004165349.png" alt="image-20211004165349207" style="zoom:80%;" />

购买行为：谁买、成功的时间、付款和发货信息。

事务：就像转账一样，要么扣钱加钱同时成功要么同时失败。

数据落地：MySQL VS NoSQL（关系型数据库和非关系型数据库）

### MySQL实现秒杀难点分析

竞争！

<img src="https://gitee.com/hqinglau/img/raw/master/img/20211004165827.png" alt="image-20211004165827317" style="zoom:80%;" />

解决竞争背后技术：事务+行级锁。

<img src="https://gitee.com/hqinglau/img/raw/master/img/20211004170026.png" alt="image-20211004170026645" style="zoom:50%;" />

行级锁：

一个用户A利用SQL语句：

```sql
update table set num=num-1 where id=10 and num>1
```

去修改库存：`id=10,name=xxx`的项。

同时另一个用户B也想用该语句修改库存，就需要等待，直到A成功了。

**难点：高效的处理竞争**。

### 秒杀功能

- 秒杀接口暴露
- 执行秒杀
- 相关查询

代码开发阶段：DAO设计编码、Service设计编码、Web设计编码

## DAO设计编码

创建数据库，两张表：

```shell
mysql> show tables;
+-------------------+
| Tables_in_seckill |
+-------------------+
| seckill           | 
| success_killed    |
+-------------------+
```

详细见下面的注释：

```sql
-- 数据库初始化脚本

-- 创建数据库
CREATE DATABASE seckill;
-- 使用数据库
use seckill;
-- 创建秒杀库存表
CREATE TABLE seckill(
    `seckill_id` bigint NOT NULL AUTO_INCREMENT COMMENT '商品库存id',
    `name` varchar(120) NOT NULL COMMENT '商品名称',
    `number` int NOT NULL COMMENT '库存数量',
    `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `start_time` timestamp NOT NULL COMMENT '秒杀开启时间',
    `end_time` timestamp NOT NULL COMMENT '秒杀结束时间',
    PRIMARY KEY (seckill_id),
    KEY idx_start_time(start_time),
    KEY idx_end_time(end_time),
    KEY idx_create_time(create_time)
)ENGINE=InnoDB AUTO_INCREMENT=1000 DEFAULT CHARSET=utf8 COMMENT="秒杀库存表";

-- 初始化数据
insert into seckill(name,number,start_time,end_time)
values
    ('1000秒杀iphone13',100,'2021-10-04 18:00:00','2021-10-05 18:00:00'),
    ('500秒杀iphone12',200,'2021-10-04 18:00:00','2021-10-05 18:00:00'),
    ('300秒杀iphone11',300,'2021-10-04 18:00:00','2021-10-05 18:00:00'),
    ('100秒杀iphone6',400,'2021-10-04 18:00:00','2021-10-05 18:00:00');

-- 秒杀成功明细表
-- 用户登录认证相关的信息
-- 需要明确用户身份、这里简化为一个字段
create table success_killed(
    `seckill_id` bigint NOT NULL COMMENT '秒杀商品id',
    `user_phone` bigint NOT NULL COMMENT '用户手机号',
    `state` tinyint NOT NULL DEFAULT -1 COMMENT '状态标识：-1无效，0成功，1已付款，2已发货',
    `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    PRIMARY KEY(seckill_id,user_phone),/*联合主键*/
    KEY idx_create_time(create_time)
)ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT="秒杀成功明细表"

-- 连接数据库控制台
-- mysql -uroot -p
```

加COMMENT是为了能方便地查看创建时的想法：

`SHOW CREATE TABLE` 展示的内容更加丰富，它可以查看表的存储引擎和字符编码；另外，还可以通过 \g 或者 \G 参数来控制展示格式。

```shell
mysql> show create table seckill\G
*************************** 1. row ***************************
       Table: seckill
Create Table: CREATE TABLE `seckill` (
  `seckill_id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '商品库存id',
  `name` varchar(120) NOT NULL COMMENT '商品名称',
  `number` int(11) NOT NULL COMMENT '库存数量',
  `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `start_time` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00' COMMENT '秒杀开启时间',
  `end_time` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00' COMMENT '秒杀结束时间',
  PRIMARY KEY (`seckill_id`),
  KEY `idx_start_time` (`start_time`),
  KEY `idx_end_time` (`end_time`),
  KEY `idx_create_time` (`create_time`)
) ENGINE=InnoDB AUTO_INCREMENT=1004 DEFAULT CHARSET=utf8 COMMENT='秒杀库存表'
```

### DAO实体和接口编码

目录如下：

![image-20211004212330041](https://gitee.com/hqinglau/img/raw/master/img/20211004212330.png)

**entity**：对应数据库，常规操作，具体看注释

SecKill.java - 对应seckill中的秒杀商品：

```java
/**
 * 对应数据库中的字段
 * mysql> select * from seckill;
 * +------------+------------------+--------+---------------------+---------------------+---------------------+
 * | seckill_id | name             | number | create_time         | start_time          | end_time            |
 * +------------+------------------+--------+---------------------+---------------------+---------------------+
 * |       1000 | 1000秒杀iphone13 |    100 | 2021-10-04 19:22:42 | 2021-10-04 18:00:00 | 2021-10-05 18:00:00 |
 * |       1001 | 500秒杀iphone12  |    200 | 2021-10-04 19:22:42 | 2021-10-04 18:00:00 | 2021-10-05 18:00:00 |
 * |       1002 | 300秒杀iphone11  |    300 | 2021-10-04 19:22:42 | 2021-10-04 18:00:00 | 2021-10-05 18:00:00 |
 * |       1003 | 100秒杀iphone6   |    400 | 2021-10-04 19:22:42 | 2021-10-04 18:00:00 | 2021-10-05 18:00:00 |
 * +------------+------------------+--------+---------------------+---------------------+---------------------+
 */
package cn.orzlinux.entity;

import java.util.Date;

public class SecKill {
    private long seckillId;
    private String name;
    private int number;
    private Date startTime;
    private Date endTime;
    private Date createTime;

    // getter setter toString
}
```

successkilled.java

```java
/**
 * mysql> show columns from success_killed;
 * +-------------+------------+------+-----+-------------------+-------+
 * | Field       | Type       | Null | Key | Default           | Extra |
 * +-------------+------------+------+-----+-------------------+-------+
 * | seckill_id  | bigint(20) | NO   | PRI | NULL              |       |
 * | user_phone  | bigint(20) | NO   | PRI | NULL              |       |
 * | state       | tinyint(4) | NO   |     | -1                |       |
 * | create_time | timestamp  | NO   | MUL | CURRENT_TIMESTAMP |       |
 * +-------------+------------+------+-----+-------------------+-------+
 */
package cn.orzlinux.entity;
import java.util.Date;

public class SuccessKilled {
    private long seckillId;
    private long userPhone;
    private short state;
    private Date createTime;

    // 为了能直接获取秒杀成功的商品对象
    private SecKill secKill;
    // getter setter toString
}
```

**DAO**：数据库要实现的方法接口

SecKillDao.java

```java
public interface SecKillDao {
    /***
     * 减库存
     * @param seckillId
     * @param killTime
     * @return 如果影响行数>1，表示更新的记录行数
     */
    int reduceNumber(long seckillId, Date killTime);

    /**
     * 根据Id查秒杀对象
     * @param seckillId
     * @return
     */
    SecKill queryById(long seckillId);

    List<SecKill> queryAll(int offset,int limit);
}
```

SuccessKilledDao.java

```java
public interface SuccessKilledDao {
    // 插入购买明细，可过滤重复
    // 通过联合唯一主键
    // 返回插入的行数
    int insertSuccessKilled(long seckillId,long userPhone);

    // 根据商品id和用户查询秒杀成功对象实体（带秒杀商品实体）
    // 原视频应该有误，这里仅凭一个商品Id得不到唯一的秒杀成功对象
    SuccessKilled queryByIdWithSeckill(long seckillId,long userPhone);
}
```

### MyBatis

<img src="https://gitee.com/hqinglau/img/raw/master/img/20211004212930.png" alt="image-20211004212930759" style="zoom:80%;" />

MyBatis特点：`参数+SQL=Entity/List`

SQL可以写在xml或者注解中。一般应该用xml

DAO接口：Mapper自动实现DAO接口（推荐）或者API方式

官方文档链接：[mybatis文档](https://mybatis.org/mybatis-3/zh/configuration.html)

#### 实现DAO编程

在resources文件夹下创建mybatis配置文件和mapper文件夹：

![image-20211004214437091](https://gitee.com/hqinglau/img/raw/master/img/20211004214437.png)

mybatis-config.xml配置如下：

DTD（Document Type Definition），全称为文档类型定义。具体头可以在官方文档例子中找到。

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>
    <!--配置全局属性-->
    <settings>
        <!--使用jdbc的getGeneratedKeys获取数据库自增主键-->
        <setting name="useGeneratedKeys" value="true"/>
        <!--使用列别名替换列名：默认true-->
        <!--select name as title from table-->
        <setting name="userColumn" value="true"/>
        <!--开启驼峰命名转换：Table(create_time) -> Entity(createTime)-->
        <setting name="namUnderscoreCamelCase" value="true"/>
    </settings>
</configuration>
```

#### mybatis整合spring

mybatis整合spring可以实现更少的配置：

- **别名**，如`resultType="cn.orzlinux.entity.SecKill"`可以简写为`SecKill`。

- **配置扫描**，如:

  ```xml
  <mapper resource="mapper/SecKillDao.xml"/>
  <mapper resource="mapper/SuccessKilledDao.xml"/>
  ...
  ```

  可以简化为自动配置扫描。

- **DAO实现**

  ```xml
  <bean id="ClubDao" class="...ClubDao"/>
  <bean id="Club2Dao" class="...Club2Dao"/>
  ...
  ```

  简化为自动实现DAO接口，自动注入spring容器。

- 依然具有足够的灵活性、定制SQL、自由传参、结果集自动赋值。

新建配置文件：

![image-20211004232637505](https://gitee.com/hqinglau/img/raw/master/img/20211004232637.png)

spring-dao.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd">
    <!--配置整合mybatis-->

    <!--配置数据库相关参数，一般在另一个properties文件配置:${url}-->
    <context:property-placeholder location="classpath:jdbc.properties"/>
    <!-- classpath:jdbc.properties具体内容：
    driver=com.mysql.jdbc.Driver
    usrl=jdbc:mysql://127.0.0.1:3306/seckill?userUnicode=true&characterEncoding=utf8
    username=root
    password= -->

    <!--数据库连接池-->
    <bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource">
        <!--配置连接池属性-->
        <property name="driverClass" value="${driver}"/>
        <property name="jdbcUrl" value="${url}"/>
        <property name="user" value="${username}"/>
        <property name="password" value="${password}"/>

        <!--c3p0连接池私有属性-->
        <property name="maxPoolSize" value="30"/>
        <property name="minPoolSize" value="10"/>
        <!--连接池在回收数据库连接时是否自动提交事务。如果为false，则会回滚未提交的事务，-->
        <!--如果为true，则会自动提交事务。（不建议使用）-->
        <property name="autoCommitOnClose" value="false"/>
        <!--获取连接超时时间-->
        <property name="checkoutTimeout" value="1000"/>
        <!--获取连接失败重试次数-->
        <property name="acquireRetryAttempts" value="2"/>
    </bean>

    <!--配置sqlSessionFactory对象-->
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <!--注入数据库连接池-->
        <property name="dataSource" ref="dataSource"/>
        <!--配置mybatis全局配置文件-->
        <property name="configLocation" value="classpath:mybatis-config.xml"/>
        <!--扫描entity包，使用别名 cn.orzlinux.entity.xxx -> xxx -->
        <property name="typeAliasesPackage" value="cn.orzlinux.entity;"/>
        <!--扫描sql配置文件，mapper需要的xml文件-->
        <property name="mapperLocations" value="classpath:mapper/*.xml"/>
    </bean>

    <!--配置扫描DAO接口包，动态实现DAO接口,并注入spring容器中-->
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <!--注入sqlSessionFactory-->
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
        <!--给出需要扫描DAO接口包-->
        <property name="basePackage" value="cn.orzlinux.dao"/>
    </bean>
</beans>
```



#### XML SQL

在使用mybatis 时我们sql是写在xml 映射文件中，如果写的sql中有一些特殊的字符的话，在解析xml文件的时候会被转义，但我们不希望他被转义，所以我们要使用`<![CDATA[ ]]>`来解决。

`<![CDATA[  ]]>` 是XML语法。在CDATA内部的所有内容都会被解析器忽略。

SecKillDao.xml：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="cn.orzlinux.dao.SecKillDao">
    <!--为DAO接口方法提供sql语句配置-->

    <!--int reduceNumber(long seckillId, Date killTime);-->
    <update id="reduceNumber">
        update
            seckill
        set
            number = number - 1
        where seckill_id = #{seckillId}
        and start_time <![CDATA[ <= ]]> #{killTime}
        and end_time >= #{killTime}
        and number > 0;
    </update>

    <!--SecKill queryById(long seckillId);-->
    <select id="queryById" parameterType="long" resultType="SecKill">
        select seckill_id,name,number,start_time,end_time,create_time
        from seckill where seckill_id=#{seckillId};
    </select>

    <!--List<SecKill> queryAll(int offset,int limit);-->
    <select id="queryAll" resultType="SecKill">
        select seckill_id,name,number,start_time,end_time,create_time
        from seckill
        order by create_time DESC
        limit ${offset},#{limit};
    </select>

</mapper>
```



SuccessKilledDao.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="cn.orzlinux.dao.SuccessKilledDao">
    <!--为DAO接口方法提供sql语句配置-->
    <!--int insertSuccessKilled(long seckillId,long userPhone);-->
    <insert id="insertSuccessKilled">
        <!-- 主键冲突，报错
        INSERT IGNORE 与INSERT INTO的区别就是INSERT IGNORE会忽略数
        据库中已经存在 的数据，如果数据库没有数据，就插入新的数据，如果有数
        据的话就跳过这条数据。-->
        insert ignore into success_killed(seckill_id,user_phone)
        values (#{seckillId,userPhone})
    </insert>
    
    <!--SuccessKilled queryByIdWithSeckill(long seckillId,long userPhone);-->
    <select id="queryByIdWithSeckill" resultType="SuccessKilled">
        <!-- 根据查询携带seckill实体
         如何告诉mybatis把结果映射到SuccessKilled同时映射seckill
         可以自由控制SQL-->
        select
            sk.seckill_id,
            sk.user_phone,
            sk.create_time,
            sk.state,
            s.seckill_id as "secKill.seckill_id", #开启了驼峰，会自动换
            s.name "secKill.name",
            s.number "secKill.number",
            s.start_time "secKill.start_time",
            s.end_time "secKill.end_time",
            s.create_time "secKill.create_time"
        from success_killed sk
        inner join seckill s on sk.seckill_id = s.seckill_id
        where sk.seckill_id = #{seckillId} and sk.user_phone = #{userPhone};
    </select>

</mapper>
```

### junit单元测试

#### SecKillDaoTest.java

```java
package cn.orzlinux.dao;

import cn.orzlinux.entity.SecKill;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

import javax.annotation.Resource;

import java.util.List;

import static org.junit.Assert.*;

/**
 * 配置spring和junit整合，junit启动时加载spring ioc容器
 * spring-test,junit
 */
@RunWith(SpringJUnit4ClassRunner.class)
//告诉junit spring配置文件
@ContextConfiguration({"classpath:spring/spring-dao.xml"})
public class SecKillDaoTest {
    // 注入DAO实现类依赖,resource注解去spring容器找其实现类
    @Resource
    private SecKillDao secKillDao;
    ...
}
```

在queryById的测试中一切正常：

```java
@Test
public void queryByIdTest() {
    long id =1000;
    SecKill secKill = secKillDao.queryById(id);
    System.out.println(secKill.getName());
    System.out.println(secKill);
    // output：1000秒杀iphone13
    //SecKill{seckillId=1000, name='1000秒杀iphone13',
    // number=100, startTime=Tue Oct 05 02:00:00 CST 2021,
    // endTime=Wed Oct 06 02:00:00 CST 2021,
    // createTime=Tue Oct 05 10:01:47 CST 2021}
}
```

但是在queryAllTest中出现了问题：

```java
@Test
public void queryAllTest() {
    List<SecKill> secKillList = secKillDao.queryAll(0,100);
    for(SecKill secKill:secKillList) {
        System.out.println(secKill);
    }
}
```

问题的原因在于：

```xml
<!--List<SecKill> queryAll(int offset,int limit);-->
<!--java没有保存形参的记录 queryAll(int offset,int limit); -> queryAll(arg0,arg1);-->
<!--byId可以是因为只有一个参数-->
<select id="queryAll" resultType="SecKill">
    select seckill_id,name,number,start_time,end_time,create_time
    from seckill
    order by create_time DESC
    limit ${offset},#{limit};
</select>
```

所以需要在SecKillDao.java里的函数那里配置参数Param：

```java
List<SecKill> queryAll(@Param("offset") int offset, @Param("limit") int limit);
```

问题解决，结果如下：

```shell
SecKill{seckillId=1000, name='1000秒杀iphone13', number=100, startTime=Tue Oct 05 02:00:00 CST 2021, endTime=Wed Oct 06 02:00:00 CST 2021, createTime=Tue Oct 05 10:01:47 CST 2021}
SecKill{seckillId=1001, name='500秒杀iphone12', number=200, startTime=Tue Oct 05 02:00:00 CST 2021, endTime=Wed Oct 06 02:00:00 CST 2021, createTime=Tue Oct 05 10:01:47 CST 2021}
SecKill{seckillId=1002, name='300秒杀iphone11', number=300, startTime=Tue Oct 05 02:00:00 CST 2021, endTime=Wed Oct 06 02:00:00 CST 2021, createTime=Tue Oct 05 10:01:47 CST 2021}
SecKill{seckillId=1003, name='100秒杀iphone6', number=400, startTime=Tue Oct 05 02:00:00 CST 2021, endTime=Wed Oct 06 02:00:00 CST 2021, createTime=Tue Oct 05 10:01:47 CST 2021}
```

减库存测试：

```java
@Test
public void reduceNumberTest() {
    Date killTime = new Date();
    int updateCount = secKillDao.reduceNumber(1000L,killTime);
    System.out.println(updateCount);
    // 1:表示更改了一条记录
}
```

测试记录，查看这里的确少了一件：

![image-20211005022443276](https://gitee.com/hqinglau/img/raw/master/img/20211005022450.png)

分析一下mybatis的行为：

```shell
# 从c3p0连接池拿到链接，没有被spring托管
02:18:42.793 [main] DEBUG o.m.s.t.SpringManagedTransaction - JDBC Connection [com.mchange.v2.c3p0.impl.NewProxyConnection@1cd629b3] will not be managed by Spring

# SQL
02:18:42.818 [main] DEBUG c.o.dao.SecKillDao.reduceNumber - ==>  Preparing: update seckill set number = number - 1 where seckill_id = ? and start_time <= ? and end_time >= ? and number > 0; 

# 参数传递
02:18:42.882 [main] DEBUG c.o.dao.SecKillDao.reduceNumber - ==> Parameters: 1000(Long), 2021-10-05 02:18:42.442(Timestamp), 2021-10-05 02:18:42.442(Timestamp)

# 参数返回
02:18:42.897 [main] DEBUG c.o.dao.SecKillDao.reduceNumber - <==    Updates: 1
```

同理另一个dao SuccessKilledDao也进行测试。

```java
package cn.orzlinux.dao;

import cn.orzlinux.entity.SuccessKilled;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

import javax.annotation.Resource;

import static org.junit.Assert.*;

@RunWith(SpringJUnit4ClassRunner.class)
//告诉junit spring配置文件
@ContextConfiguration({"classpath:spring/spring-dao.xml"})
public class SuccessKilledDaoTest {
    // 注入DAO实现类依赖,resource注解去spring容器找其实现类
    @Resource
    private SuccessKilledDao successKilledDao;
    @Test
    public void insertSuccessKilled() {
        long id = 1000L;
        long phone = 12345678901L;
        int insertCount = successKilledDao.insertSuccessKilled(id,phone);
        System.out.println(insertCount);
        // 输出1
        // 再运行一次输出 0，联合主键的效果
    }

    @Test
    public void queryByIdWithSeckill() {
        long id = 1000L;
        long phone = 12345678901L;
        SuccessKilled successKilled = successKilledDao.queryByIdWithSeckill(id,phone);
        System.out.println(successKilled);
        System.out.println(successKilled.getSecKill());
        // SuccessKilled{seckillId=1000, userPhone=12345678901, state=-1,
        //              createTime=Tue Oct 05 10:33:43 CST 2021}
        // SecKill{seckillId=1000, name='1000秒杀iphone13', number=99,
        //          startTime=Tue Oct 05 02:00:00 CST 2021, endTime=Wed Oct 06 02:00:00 CST 2021, createTime=Tue Oct 05 10:01:47 CST 2021}
    }
}
```



![image-20211005023447515](https://gitee.com/hqinglau/img/raw/master/img/20211005023447.png)

还有一点就是更改秒杀成功的状态，更改为0，也就是秒杀成功：

> 状态标识：-1无效，0成功，1已付款，2已发

```xml
<insert id="insertSuccessKilled">
    <!-- 主键冲突，报错
    INSERT IGNORE 与INSERT INTO的区别就是INSERT IGNORE会忽略数
    据库中已经存在 的数据，如果数据库没有数据，就插入新的数据，如果有数
    据的话就跳过这条数据。-->
    insert ignore into success_killed(seckill_id,user_phone,state)
    values (#{seckillId},#{userPhone},0)
</insert>
```

