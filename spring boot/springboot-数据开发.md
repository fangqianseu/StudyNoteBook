# 基础访问

## 数据源的自动配置

### 1、导入JDBC场景

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-jdbc</artifactId>
</dependency>
```

![image.png](https://img-1252710297.cos.ap-nanjing.myqcloud.com/img/20210110164417.png)

默认使用 HikariDataSource 数据源

**为什么导入JDBC场景，官方不导入驱动？**

+ 官方不知道我们接下要操作什么数据库
+ 要自己导入数据库驱动

### 分析自动配置

#### 1、自动配置的类

- DataSourceAutoConfiguration ： 数据源的自动配置 数据库连接池

  + 修改数据源相关的配置：**spring.datasource**

  - **数据库连接池的配置，是自己容器中没有DataSource才自动配置的**
  - 底层配置好的连接池是：**HikariDataSource**

- DataSourceTransactionManagerAutoConfiguration： 事务管理器的自动配置

- JdbcTemplateAutoConfiguration： **JdbcTemplate的自动配置，可以来对数据库进行crud**

- - 配置项：  “spring.jdbc“

- JndiDataSourceAutoConfiguration： jndi的自动配置
- XADataSourceAutoConfiguration： 分布式事务相关的

## 使用Druid数据源

### 功能

+ 数据库连接池
+ StatViewServlet
  + 提供监控信息展示的html页面
  + 提供监控信息的JSON API
+ StatFilter
  + 用于统计监控信息；如SQL监控、URI监控

###  引入druid-starter

```
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>1.1.17</version>
        </dependency>
```

### 分析自动配置

- 扩展配置项 **spring.datasource.druid**
- 自动配置类 默认导入了以下配置项
  - DruidSpringAopConfiguration.**class**,  监控SpringBean的；配置项：**spring.datasource.druid.aop-patterns**
  - DruidStatViewServletConfiguration.**class**, 监控页的配置：**spring.datasource.druid.stat-view-servlet；默认开启**
  -  DruidWebStatFilterConfiguration.**class**, web监控配置；**spring.datasource.druid.web-stat-filter；默认开启**
  - DruidFilterConfiguration.**class**}) 所有Druid自己filter的配置

### 默认配置

```java
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/db_account
    username: root
    password: 123456
    driver-class-name: com.mysql.jdbc.Driver

    druid:
      aop-patterns: com.atguigu.admin.*  #监控SpringBean
      filters: stat,wall     # 底层开启功能，stat（sql监控），wall（防火墙）

      stat-view-servlet:   # 配置监控页功能
        enabled: true
        login-username: admin
        login-password: admin
        resetEnable: false

      web-stat-filter:  # 监控web
        enabled: true
        urlPattern: /*
        exclusions: '*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*'

      filter:
        stat:    # 对上面filters里面的stat的详细配置
          slow-sql-millis: 1000
          logSlowSql: true
          enabled: true
        wall:
          enabled: true
          config:
            drop-table-allow: false
```

## 整合 Mybatis

### 引入依赖

```xml
<dependency>
  <groupId>org.mybatis.spring.boot</groupId>
  <artifactId>mybatis-spring-boot-starter</artifactId>
  <version>2.1.4</version>
</dependency>
```

![image.png](https://img-1252710297.cos.ap-nanjing.myqcloud.com/img/20210110191933.png)

步骤：

+ 导入mybatis官方starter

+ 在application.yaml中指定Mapper配置文件的位置，以及指定全局配置文件的信息 

  ```yaml
  mybatis:
    configuration:
      map-underscore-to-camel-case: true
    mapper-locations: classpath:mapper/*Mapper.xml   # 注意 必须指定mapper.xml文件的地址
  ```

+ 编写mapper接口：标注 @Mapper 注解 

  如果需要指定扫描的包，可以使用 `@MapperScan("com.fq.dao")` 注解，不用使用 @Mapper 注解

  ```java
  @Mapper
  public interface UserMapper {
      User getById(Integer id);
  
      @Select({"select * from t_user where name=#{name}"})
      User getByName(String name);
  }
  ```

+ 编写sql映射文件并绑定 mapper 接口

  ```xml
  <?xml version="1.0" encoding="UTF-8" ?>
  <!DOCTYPE mapper
          PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
          "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
  <mapper namespace="com.fq.springboot.dao.UserMapper">
      <select id="getById" resultType="com.fq.springboot.dao.User">
          select * from  t_user where  id=#{id}
      </select>
  </mapper>
  ```




