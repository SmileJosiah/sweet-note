# Apahce ShardingSphere学习

## 基本概念

### 什么是Sharding Sphere

>Apache ShardingSphere is an open source ecosystem of distributed databases, including three independent products: JDBC, Proxy & Sidecar (Planning). It adopts a plugin-oriented (or plugabble) architecture and expands the original databases’ features list thanks to components.

> Apache ShardingSphere是一套开源的分布式数据库中间件解决方案组成的生态圈，它游由Sharding-JDBC、Sharding-Proxy和Sharding-Sidcar组成。
>
> ShardingSphere定位为关系数据库中间件。

### 什么是分库分表

:question:在项目中出现了单表数据超过了千万级别的数据量，甚至是过亿的数据量的时候，再去对数据库执行CRUD操作，如何解决数据库性能的瓶颈？

常规的解决方案有两种：

* 堆硬件（提高数据库服务器的资源配置），这种方式不科学也不常用，因为业务数据会源源不断的产生，但是服务器的配置不可能无限的提升
* 分库分库
  * 分库 将一个DB,按照某种维度拆分为多个Db
  * 分表 将一张大表，拆分为多张小表存储

#### 分库分表的方式

:star2: 两种方式：垂直拆分、水平拆分

:bookmark_tabs:商品表（MySQL）

|  id  | 商品名称 | 价格（元） | 商品数量 |   描述信息   |  产地  |
| :--: | :------: | :--------: | :------: | :----------: | :----: |
|  1   |  娃哈哈  |     2      |    10    | 农夫山泉产品 |  成都  |
|  2   | 可口可乐 |     3      |    25    |     糖水     | 芝加哥 |
| ...  |   ...    |    ...     |   ...    |     ...      |  ...   |
| 9999 | 雀巢咖啡 |     5      |    30    |   咖啡饮料   |  北京  |

1. 垂直拆分

   * 垂直分表

     * 将一张表的一部分字段，拆分开存储在另外一张表中

       * 商品基本信息表

         | id   | 商品名称 | 价格 | 数量 |
         | ---- | -------- | ---- | ---- |
         | 1    | 娃哈哈   | 2    | 10   |
         | 2    | 可口可乐 | 3    | 25   |
         | ...  | ...      | ...  | ...  |
         | 9999 | 雀巢咖啡 | 5    | 30   |

       * 商品描述信息表

         | id   | 描述         | 产地   |
         | ---- | ------------ | ------ |
         | 1    | 农夫山泉产品 | 成都   |
         | 2    | 糖水         | 芝加哥 |
         | ...  | ...          | ...    |
         | 9999 | 咖啡饮料     | 北京   |

   * 垂直分库

     * 按照不同的业务将分为多个数据库，专库专用，比如可以在电商系统中可以将`用户信息管理`的数据专门存放在`用户库`	中，将`订单管理相关的信息`存放在`订单库`中
     * ![image-20220731212404817](C:\Users\0.0\AppData\Roaming\Typora\typora-user-images\image-20220731212404817.png)

2. 水平拆分

   * 水平分库

     ![image-20220731213042124](C:\Users\0.0\AppData\Roaming\Typora\typora-user-images\image-20220731213042124.png)

   * 水平分表（解决单表数据过大的问题）

#### 分库分表的应用和带来的问题

1. 应用
   * 在数据库设计的时候考虑垂直分库和垂直分表
   * 随着数据库数据量增加，不要马上考虑做水平切分，首先考虑缓存处理，读写分离，使用索引等等方式，如果这些方式都不能根本的解决问题了，再考虑水平分库分表
2. 带来的问题  
   * 跨节点连接查询问题（分页、排序）
   * 多数据源管理问题

## Sharding-JDBC

>定位为轻量级Java框架，在Java的JDBC层提供的额外服务。它使用客户端直连数据库，以jar包形式提供服务，无需额外部署和依赖，可理解为增强版的JDBC驱动，完全兼容JDBC和各种ORM框架

### 特性

* 适用于任何基于JDBC的ORM框架，如：JPA,Hibernate，Mybatis,Spring JDBC Template或者使用JDBC
* 支持任何第三方的数据连接池，如：DBCP，C3P0,Druid，HirkariCP等
* 支持任意实现JDBC规范的数据，目前支持MySQL，Oracle，SQLServer以及任何遵循SQL92的标准数据库
* **主要的目的是：简化分库分表之后对数据的相关操作**

### 水平分表

1. 搭建环境

   * Spring Boot 2.2.1+Mybatis Plus+Sharding JDBC+Druid连接池

   * `pom.xml`

     ```XML
     <dependency>
         <groupId>org.apache.shardingsphere</groupId>
         <artifactId>sharding-jdbc-spring-boot-starter</artifactId>
         <version>4.0.0-RC1</version>
     </dependency>
     ```

2. 按照水平分表的方式，创建数据库和数据库表

   * 创建数据库`course_db`
   * 在数据库创建两张表 `course_1`和`course_2`
   * 约定规则：如果添加课程Id是偶数，把数据添加到`course_1`,如果是奇数，把数据添加到`course_2`

3. DDL

   ```SQL
   
   ```

4. 配置Sharding-JDBC分片策略

5. 测试结果

### 水平分库

1. 创建两个数据 `db_1`和`db_2`
   :triangular_ruler: userId为偶数添加到 `db_1`数据库，为奇数数据添加到`db_2`数据库

    :straight_ruler:cid为偶数数据添加到`course_1`表，为奇数添加到`course_2`表

2. 配置数据库分片规则

   

3. 测试结果

