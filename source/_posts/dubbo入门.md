---
title: dubbo入门
date: 2019-05-26 23:45:10
tags: 
- dubbo
- 分布式
category: dubbo
---

# dubbo入门

## 简介
dubbo是一个高性能的rpc框架。它提供了三大核心能力：面向接口的远程方法调用，智能容错和负载均衡，以及服务自动注册和发现。

![结构图](/dubbo入门/dubbo.png)

dubbo的流程：  
初始化阶段：服务提供方注册服务，服务消费方订阅服务   
当服务消费方想要调用服务时，会区注册中心寻找服务的信息，然后通过网络连接服务提供方，远程调用方法，服务提供方调用方法后将结果返回给服务消费方。
<!--more-->
监控中心可以监控整个提供服务提供方和消费方。并不是必须的

## 使用

dubbo使用很简单，了解的dubbo的启动需要那些调节安，那么使用也就简单了，只需要服务提供方暴露服务给注册中心，服务消费方远程调用服务即可。

不要忘记引入先关的依赖，我这里使用的是：  

```xml
<dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>dubbo</artifactId>
            <version>2.6.5</version>
        </dependency>
        <!--操作zookeeper的依赖-->
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-framework</artifactId>
            <version>2.12.0</version>
        </dependency>
```

### API配置

需求：消费方想要调用一个hello方法，调用服务提供方的方法，得到结果   

```java
public interface Hello {
    String hello(String name);
}

public class JavaHello implements Hello {
    @Override
    public String hello(String name) {
        String res="java:hello "+name;
        System.out.println(res);
        return res;
    }
}

```

服务提供方：
```java
package provider;

public class Provider1 {
    public static void main(String[] args) throws IOException {

        //服务实现，可以提供服务的对象
        Hello javaHello=new JavaHello();

        //配置当前应用
        ApplicationConfig applicationConfig=new ApplicationConfig();
        applicationConfig.setName("hello-provider");

        //连接注册中心，注册中心可以有多种，比如zookeeper，redis等等
        RegistryConfig registryConfig =new RegistryConfig();
        registryConfig.setAddress("zookeeper://127.0.0.1:2181");
        registryConfig.setUsername("aaa");
        registryConfig.setPassword("bbb");

        //服务提供方协议配置，这里使用的是dubbo，具体的协议使用范围不同，可以参考官方文档

        ProtocolConfig protocolConfig=new ProtocolConfig();
        protocolConfig.setName("dubbo");
        protocolConfig.setPort(22880);
        protocolConfig.setThreads(10);

        //暴露服务配置
        // 注意：ServiceConfig为重对象，内部封装了与注册中心的连接，以及开启服务端口

        ServiceConfig<Hello> serviceConfig=new ServiceConfig<Hello>();
        serviceConfig.setApplication(applicationConfig);
        serviceConfig.setRegistry(registryConfig);
        serviceConfig.setProtocol(protocolConfig);

        serviceConfig.setInterface(Hello.class);
        serviceConfig.setRef(javaHello);
        serviceConfig.setVersion("1.0.0");

        //服务暴露及注册
        serviceConfig.export();
        System.in.read();
    }
}
```

服务消费方
```java
package consumer;
public class Consumer1 {

    public static void main(String[] args) {
        // 当前应用配置
        ApplicationConfig application = new ApplicationConfig();
        application.setName("hello-consumer");

        // 连接注册中心配置
        RegistryConfig registry = new RegistryConfig();
        registry.setAddress("zookeeper://127.0.0.1:2181");
        registry.setUsername("aaa");
        registry.setPassword("bbb");

        // 注意：ReferenceConfig为重对象，内部封装了与注册中心的连接，以及与服务提供方的连接


        //引用远程服务
        ReferenceConfig<Hello> referenceConfig =new ReferenceConfig<Hello>();
        referenceConfig.setApplication(application);
        referenceConfig.setRegistry(registry); // 多个注册中心可以用setRegistries()
        referenceConfig.setInterface(Hello.class);
        referenceConfig.setVersion("1.0.0");

        Hello javaHello=referenceConfig.get();
        System.out.println(javaHello.hello("zzx"));
    }
}

```

观察结果发现服务提供发打印了一条消息，而服务消费方也打印了一条消息。

观察注册中心：
```
[zk: localhost:2181(CONNECTED) 6] ls /dubbo/api.interfaces.Hello
[consumers, configurators, routers, providers]
[zk: localhost:2181(CONNECTED) 7] ls /dubbo/api.interfaces.Hello/providers
[dubbo%3A%2F%2F172.17.0.1%3A22880%2Fapi.interfaces.Hello%3Fanyhost%3Dtrue%26application%3Dhello-provider%26dubbo%3D2.0.2%26generic%3Dfalse%26interface%3Dapi.interfaces.Hello%26methods%3Dhello%26pid%3D23265%26revision%3D1.0.0%26side%3Dprovider%26threads%3D10%26timestamp%3D1558959362483%26version%3D1.0.0]

```
可以看见在/dubbo/api.interfaces.Hello节点下有[consumers, configurators, routers, providers]这几个节点：消费订阅者，配置，路由，服务提供者 

在`/dubbo/api.interfaces.Hello/providers`节点下，将节点内容进行url解码后是：    
```
dubbo://172.17.0.1:22880/api.interfaces.Hello?anyhost=true&application=hello-provider&dubbo=2.0.2&generic=false&interface=api.interfaces.Hello&methods=hello&pid=23265&revision=1.0.0&side=provider&threads=10&timestamp=1558959362483&version=1.0.0
```

这里包含了，服务提供方的联系方式，提供的服务的接口等等各种信息。

显而易见，在consumers下也有会存储一些信息。

还有一些特殊的场景，比如方法级设置，点对点直连等等。

当然它的功能不仅仅这么简单，但是一般不这么使用，而是与Spring集成。


### 与Spring集成

服务提供方
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans" xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">
    <!--<dubbo:application name="demo-consumer"/>
    <dubbo:registry group="aaa" address="zookeeper://127.0.0.1:2181"/>
    <dubbo:reference id="demoService" check="false" interface="org.apache.dubbo.samples.basic.api.DemoService"/>
-->
    <context:component-scan base-package="impl"/>
    <dubbo:application name="consumer_1"/>
    <dubbo:registry  address="zookeeper://127.0.0.1:2181"/>
    <dubbo:protocol  name="dubbo" port="20890"/>
    <dubbo:reference  interface="com.zzx.service.UserService" id="userService"/>

</beans>

```

### 注解配置
dubbo也可是使用注解配置，需要配置的信息与上面一致，只是实现方式不同而已。


可以很清楚的看见也就是将上面的ConfigXXX的内容写在的xml标签中，而事实上在配置在xml中，在解析xml后，dubbo也会将创建ConfigXXX对象，写入配置信息。只不过写在配置文件中更加便捷，易于修改。


## 原理浅析

```java
Hello javaHello=referenceConfig.get();
javaHello.hello("zzx");
```
从上面可以看见实际上调用的核心是这一段代码，那么它的具体流程是怎样的?

1. javaHello不是“纯粹”的Hello对象，而是一个代理对象
2. 在代理对象中拦截的这个方法
3. 在拦截方法后，将这些接口，方法，参数，服务消费方的一些信息封装成了一条消息，将这些消息发送个从注册中心知道的服务提供方的地址
4. 服务提供放收到消息后，根据接口，方法，参数等信息，生成一个代理对象，执行方法
5. 服务提供方将执行后的结果封装成一条信息，发送给服务消费方，服务消费方得到结果，返回

当然，dubbo的设计远不止这么简单，具有很多层次，很多功能。这里只是简单说明原理。


## 参考

[dubbo文档](http://dubbo.apache.org/zh-cn/)