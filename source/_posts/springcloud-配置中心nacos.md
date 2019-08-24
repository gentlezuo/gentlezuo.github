---
title: springcloud-配置中心nacos
date: 2019-07-1 22:55:01
tags: 
- springcloud
category: springcloud
---

nacos的配置功能比springcloud原生的要强大，更简洁。
<!--more-->

## 配置nacos

### 使用mysql持久化

- 创建一个mysql数据库，例如`create database nacos`
- 将`nacos/cofn/nacos-mysql.sql`中的sql语句在该数据库中执行。
- 修改`nacos/conf/application.properties`，在文件之后追加
```properites
spring.datasource.platform=mysql
db.num=1
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&serverTimezone=GMT
db.user=root
db.password=xxxx
```
- 启动nacos`startup.cmd`

### 解决不支持mysql8.x的问题
- clone 源码`git clone https://github.com/alibaba/nacos.git`
- 修改nacos-all的pom文件中的依赖,将mysq-conncetor-java和cglib的版本升级
```xml
<!-- JDBC libs -->
<dependency>
       <groupId>mysql</groupId>
       <artifactId>mysql-connector-java</artifactId>
       <version>8.0.15</version>
</dependency>
<dependency>
    <groupId>cglib</groupId>
    <artifactId>cglib-nodep</artifactId>
    <version>2.2</version> 
</dependency>
```
-在nacos-naming 项目下找到` com.alibaba.nacos.naming.healthcheck.MysqlHealthCheckProcessor`修改 `import com.mysql.jdbc.jdbc2.optional.MysqlDataSource` 为   `import com.mysql.cj.jdbc.MysqlDataSource;`
- 打包`mvn -Prelease-nacos clean install -U`，输出的项目根目录\distribution\targe下的nacos即可使用

### 登录问题
生成加密密码`BCryptPasswordEncoder().encode("密码")`，注意盐值是随机的，所以生成密码每次可能不一样。
```java
public class PasswordEncoderUtil {
    public static void main(String[] args) {
        System.out.println(new BCryptPasswordEncoder().encode("yourPawword"));
    }
}
```
插入用户以及对应的密码的散列值
```sql
INSERT INTO users (username, password, enabled) VALUES ('nacos', '$2a$10$EuWPZHzz32dJN7jexM34MOeYirDdFAZm2kuWj7VEOJhhZkDrxfvUu', TRUE);
INSERT INTO roles (username, role) VALUES ('nacos', 'ROLE_ADMIN');
```
 
## 依赖
```xml
<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
            <version>0.9.0.RELEASE</version>
</dependency>
```

## 配置文件
**bootstrap.yaml**，不能是application.yaml
```yaml
spring:
  application:
    name: service
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8844
        #配置文件的扩展名，支持yaml和properties
        file-extension: yaml
```
之所以需要配置 spring.application.name ，是因为它是构成 Nacos 配置管理 dataId字段的一部分。

在 Nacos Spring Cloud 中，`dataId` 的完整格式如下：

`${prefix}-${spring.profile.active}.${file-extension}`

`prefix` 默认为` spring.application.nam`e 的值，也可以通过配置项 `spring.cloud.nacos.config.prefix`来配置。

`spring.profile.active` 即为当前环境对应的 `profile`，详情可以参考 Spring Boot文档。 注意：当 spring.profile.active 为空时，对应的连接符 - 也将不存在，dataId 的拼接格式变成` ${prefix}.${file-extension}`

`file-exetension` 为配置内容的数据格式，可以通过配置项 `spring.cloud.nacos.config.file-extension` 来配置。目前只支持 `properties` 和` yaml` 类型

## 发布于获取配置
### 发布

方法一：在web界面中创建dataId后直接编辑

方法二：执行POST请求`curl -X POST "http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=service&group=DEFAULT_GROUP&content=useLocalCache=true`
### 获取

`curl -X GET http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=nacos.cfg.dataId&group=test`

## 启动类
```java
@RestController
@RequestMapping("/config")
@SpringBootApplication
public class ServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceApplication.class,args);
    }

 @Value("${useLocalCache:false}")
    private boolean useLocalCache;

    @RequestMapping("/get")
    public boolean get() {
        return useLocalCache;
    }
}
```
依旧可以使用`@RefreshScope`实现自动刷新，比springcloud原生 config简单很多。


## 集群搭建

与nacos配置中心搭建时类似

### nacos配置
- 修改`nacos/conf/applicaton.properties`,这里使两个数据库作为准备模式
```properties
spring.datasource.platform=mysql
db.num=2
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&serverTimezone=GMT
db.url.1=jdbc:mysql://127.0.0.1:3306/nacos2?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&serverTimezone=GMT
db.user=root
db.password=root
```

- 修改`nacos/conf/cluster.conf`，如果使用`localhost`或者`127.0.0.1`会出错；三台机器的`ip:端口`
```
10.xx.xx.23:8844
10.xx.xx.23:8845
10.xx.xx.23:8846
```
- 启动每一台机器`startup.cmd -m cluster`，以集群模式启动。

### 配置文件
```yaml
spring:
  application:
    name: service
  cloud:
    nacos:
      config:
	 #所有节点的地址
        server-addr: 127.0.0.1:8844,127.0.0.1:8845,127.0.0.1:8846
        file-extension: yaml
```