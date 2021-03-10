---
layout:     post
title:      "Spring动态切换数据源"
subtitle:   " \" Exception\""
date:       2021-03-10 12:00:00
author:     "ysbao"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Java


---

> 千里之行，始于足下

#### 动态的数据源切换

maven引入：

```xml
<!-- https://mvnrepository.com/artifact/com.baomidou/dynamic-datasource-spring-boot-starter -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
    <version>3.2.1</version>
</dependency>
```

application.properties文件里把原先的数据源配置注释掉，写入新的配置：



```kotlin
#spring.datasource.url=jdbc\:oracle\:thin\:@127.0.0.1\:1521\:xe
#spring.datasource.username=zhaohy
#spring.datasource.password=oracle
#spring.datasource.driver-class-name=oracle.jdbc.OracleDriver

#set default datasource
spring.datasource.dynamic.primary=master
spring.datasource.dynamic.datasource.master.url=jdbc\:oracle\:thin\:@127.0.0.1\:1521\:xe
spring.datasource.dynamic.datasource.master.username=name1
spring.datasource.dynamic.datasource.master.password=xxx
spring.datasource.dynamic.datasource.master.driver-class-name=oracle.jdbc.OracleDriver

spring.datasource.dynamic.datasource.slave_1.url=jdbc\:oracle\:thin\:@127.0.0.1\:1521\:xe
spring.datasource.dynamic.datasource.slave_1.username=name2
spring.datasource.dynamic.datasource.slave_1.password=xxx
spring.datasource.dynamic.datasource.slave_1.driver-class-name=oracle.jdbc.OracleDriver
```

参考的是官网的例子



```kotlin
spring:
  datasource:
    dynamic:
      primary: master #设置默认的数据源或者数据源组,默认值即为master
      strict: false #设置严格模式,默认false不启动. 启动后在未匹配到指定数据源时候会抛出异常,不启动则使用默认数据源.
      datasource:
        master:
          url: jdbc:mysql://xx.xx.xx.xx:3306/dynamic
          username: root
          password: 123456
          driver-class-name: com.mysql.jdbc.Driver # 3.2.0开始支持SPI可省略此配置
        slave_1:
          url: jdbc:mysql://xx.xx.xx.xx:3307/dynamic
          username: root
          password: 123456
          driver-class-name: com.mysql.jdbc.Driver
        slave_2:
          url: ENC(xxxxx) # 内置加密,使用请查看详细文档
          username: ENC(xxxxx)
          password: ENC(xxxxx)
          driver-class-name: com.mysql.jdbc.Driver
          schema: db/schema.sql # 配置则生效,自动初始化表结构
          data: db/data.sql # 配置则生效,自动初始化数据
          continue-on-error: true # 默认true,初始化失败是否继续
          separator: ";" # sql默认分号分隔符
          
       #......省略
       #以上会配置一个默认库master，一个组slave下有两个子库slave_1,slave_2
```

这样就配置好了，master是默认数据源，slave_1是第二个数据源，名字可以随便取
 如果不指定数据源，则直接是master默认的，如虚指定数据源，则可去mybatis的mapper层加注解@DS（"slave_1"）即可，当然也可以加在service层的方法或者类上面。如果只是指定@DS则负载均衡的访问配置好的数据库，如果不是主从数据库还是不要用这个。

测试：
 controller:

```css
@RequestMapping("/test/test1.do")
    public void test1(HttpServletRequest request) {
        testService.test1();
    }
```

serviceImpl:

```jsx
public void test1() {
        List<Map<String, Object>> list = new ArrayList<Map<String,Object>>();
        Map<String, Object> paramsMap = new HashMap<String, Object>();
        list = testMapper.getUserList(paramsMap);
        System.out.println("===" + JSON.toJSONString(list));
        list = testMapper.getUserList1(paramsMap);
        System.out.println("1===" + JSON.toJSONString(list));
    }
```

mapper:

```dart
package com.zhaohy.app.dao;

import java.util.List;
import java.util.Map;

import com.baomidou.dynamic.datasource.annotation.DS;

public interface TestMapper {

    List<Map<String, Object>> getUserList(Map<String, Object> paramsMap);

    @DS("slave_1")
    List<Map<String, Object>> getUserList1(Map<String, Object> paramsMap);

}
```

xml:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.zhaohy.app.dao.TestMapper">
    
    <select id="getUserList" resultType="java.util.HashMap">
    select * from da_user
    </select>
    
    <select id="getUserList1" resultType="java.util.HashMap">
    select * from da_user
    </select>
</mapper>
```

运行效果：

![img](https:////upload-images.jianshu.io/upload_images/6445455-d96fb9893cfa8231.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

动态数据源测试效果图

从上图可以看到，因为两个数据库的da_user的条数不同，第一个查到了4条，第二个查到了9条，可见生效了。

springboot默认的数据源连接池是HikariDataSource，这个开源动态数据源框架也支持阿里的druid.

#### **集成druid**

maven引入druid



```xml
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>1.1.24</version>
        </dependency>
```

更改数据源配置：

```kotlin
#spring.datasource.url=jdbc\:oracle\:thin\:@127.0.0.1\:1521\:xe
#spring.datasource.username=zhaohy
#spring.datasource.password=oracle
#spring.datasource.driver-class-name=oracle.jdbc.OracleDriver

#set druid
spring.datasource.druid.stat-view-servlet.login-username=admin
spring.datasource.druid.stat-view-servlet.login-password=123456

#set default datasource
spring.datasource.dynamic.primary=master
spring.datasource.dynamic.datasource.master.url=jdbc\:oracle\:thin\:@127.0.0.1\:1521\:xe
spring.datasource.dynamic.datasource.master.username=zhaohy
spring.datasource.dynamic.datasource.master.password=oracle
spring.datasource.dynamic.datasource.master.driver-class-name=oracle.jdbc.OracleDriver

spring.datasource.dynamic.datasource.slave_1.url=jdbc\:oracle\:thin\:@39.100.143.84\:1521\:xe
spring.datasource.dynamic.datasource.slave_1.username=zhaohy
spring.datasource.dynamic.datasource.slave_1.password=oracle
spring.datasource.dynamic.datasource.slave_1.driver-class-name=oracle.jdbc.OracleDriver

spring.datasource.dynamic.datasource.master.druid.initial-size=3
spring.datasource.dynamic.datasource.master.druid.max-active=8
spring.datasource.dynamic.datasource.master.druid.min-idle=2
spring.datasource.dynamic.datasource.master.druid.max-wait=-1
spring.datasource.dynamic.datasource.master.druid.min-evictable-idle-time-millis=30000
spring.datasource.dynamic.datasource.master.druid.max-evictable-idle-time-millis=30000
spring.datasource.dynamic.datasource.master.druid.time-between-eviction-runs-millis=0
spring.datasource.dynamic.datasource.master.druid.validation-query=select 1 from dual
spring.datasource.dynamic.datasource.master.druid.validation-query-timeout=-1
spring.datasource.dynamic.datasource.master.druid.test-on-borrow=false
spring.datasource.dynamic.datasource.master.druid.test-on-return=false
spring.datasource.dynamic.datasource.master.druid.test-while-idle=true
spring.datasource.dynamic.datasource.master.druid.pool-prepared-statements=true
spring.datasource.dynamic.datasource.master.druid.filters=stat,wall
spring.datasource.dynamic.datasource.master.druid.share-prepared-statements=true
```

在springboot启动类上排除原生Druid的快速配置类。

```kotlin
package com.zhaohy.app;

import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.boot.web.servlet.ServletListenerRegistrationBean;
import org.springframework.context.annotation.Bean;

import com.alibaba.druid.spring.boot.autoconfigure.DruidDataSourceAutoConfigure;
import com.zhaohy.app.sys.filter.LoginProcessFilter;
import com.zhaohy.app.utils.OnLineCountListener;

@SpringBootApplication(exclude = DruidDataSourceAutoConfigure.class)
@MapperScan("com.zhaohy.app.dao")
public class App {

    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
        System.out.println("springboot started...");
    }

    @SuppressWarnings({ "rawtypes", "unchecked" })
    @Bean
    public FilterRegistrationBean myFilterRegistration() {
        FilterRegistrationBean regist = new FilterRegistrationBean(new LoginProcessFilter());
        // 过滤全部请求
        regist.addUrlPatterns("/*");//过滤url
        regist.setOrder(1);//过滤器顺序
        return regist;
    }
    
    @SuppressWarnings({ "rawtypes", "unchecked" })
    @Bean
    public ServletListenerRegistrationBean listenerRegist() {
        ServletListenerRegistrationBean srb = new ServletListenerRegistrationBean();
        srb.setListener(new OnLineCountListener());
        System.out.println("listener====");
        return srb;
    }
}
```

为什么要排除？

DruidDataSourceAutoConfigure在DynamciDataSourceAutoConfiguration之前，其会注入一个DataSourceWrapper，会在原生的spring.datasource下找url,username,password等。而我们动态数据源的配置路径是变化的。

这样，再次运行，也可以得到同样的结果。

注意事项：**一、排除原生的原生Druid的快速配置类**。 在启动类这边。@SpringBootApplication(exclude = DruidDataSourceAutoConfigure.class)

**二、对于SQL Server 连接时**。需要注意：在写表名时，需要加上数据库名.架构名.表名。否则会报找不到对应的数据库名称。

```mysql
SELECT t.empno,t.EmpName
FROM Hrm.dbo.hr_employee_rz t
```

原因：

dbo 和 Person 都是架构名，默认的架构都是以 dbo 开头的。

在不同的数据库之间调用数据时，我们可以使用“数据库名.构架名.表名”这种方式获取到其他库表数据。

当在同一个数据库中时就可以省略数据库名，只需要“构架名.表名”。

如果在同一个的数据库架构的情况下，只需要直接用表名就可以了，select * from 表。

如果存在架构有多种的情况下，就需要在调用中用“构架名.表名”，select * from 架构名.表。

他们起到识别功能，比方说表名相同都叫 a，但是一个是 dbo 架构的，一个是 Person 的，在调用过程中是不一样的：

select * from dbo.表

select * from person.表

不写架构名则默认为dbo。

**引用**：[springboot动态切换数据源两种方法示例](https://www.jianshu.com/p/932142703e25?utm_campaign=hugo&utm_content=note&utm_medium=seo_notes&utm_source=recommendation)

