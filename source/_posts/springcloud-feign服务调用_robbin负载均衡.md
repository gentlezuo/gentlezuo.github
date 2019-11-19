---
title: springcloud-服务调用-robbin负载均衡
date: 2019-06-30 13:57:56
tags: 
- springcloud
category: springcloud
---

## 背景

微服务的远程调用问题。
<!--more-->
比如在java代码中,如果想要调用一个服务，是否必须依赖服务提供方的具体的实现？

如果依赖具体实现，那么服务调用方就必须知道所有的服务提供方的实现细节，服务调用方变得臃肿；而且如果服务提供方有了变化，那么服务的消费方也必须重新打包部署。

是否可以只依赖接口？不依赖具体实现。

在dubbo中，可以只依赖于接口，然后dubbo在服务调用的时候会生成一个代理，根据接口的class和version等信息生成代理去调用远程的服务。

当服务提供方的实现细节有变化时，服务调用方并不知情，也不需要重启服务。这是一种优秀的解决方法。

在spring cloud中又是怎么解决这个问题的呢？

## feign

### 依赖

```xml
<parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.5.RELEASE</version>
</parent>
<dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-dependencies</artifactId>
        <version>Greenwich.RELEASE</version>
        <type>pom</type>
        <scope>import</scope>
</dependency>
<dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
        <version>2.1.0.RELEASE</version>
</dependency>
```

### java代码

接口
```java
@FeignClient(value = "client-service")
public interface HelloFeignService {
    @GetMapping("/hello")
    public String hello(@RequestParam(value = "name") String name);
}
```

服务提供方的具体实现:一个名叫client-service的客户端的controller
```java
@RestController
public class HelloServiceController {
    @GetMapping("/hello")
    public String test(@RequestParam(value = "name") String name) {
        return "test " + name;
    }
}
```

服务调用方
```java
    //只需要注入接口，调用接口的方法就行了
    @Autowired
    private HelloFeignService HelloFeignService;
    public String test(String name) {
        return HelloFeignService.hello(name);
    }
```

```java
@SpringBootApplication
@EnableDiscoveryClient
//启用feign功能
@EnableFeignClients
public class ConsumerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConsumerApplication.class, args);
    }
}
```
使用feign，feign认为客户端需要有一个名字，用于确定哪一个服务，也就是`@FeignClient(value = "client-service")`中的client-service，当然这里也可以写uri。

然后服务调用方调用的时候只需要依赖这个接口，根据这个接口上的注解的信息知道应该找哪一个服务（从注册中心如[eureka](https://gentlezuo.github.io/2019/06/29/springcloud-注册中心eureka/)、[consul](https://gentlezuo.github.io/2019/06/30/springcloud-%E6%B3%A8%E5%86%8C%E4%B8%AD%E5%BF%83consul/)获得服务提供方的地址），调用哪一个uri对应的方法。当服务实现细节变化时，服务调用方并不需要改变。


## robbin

robbin是一个负载均衡的工具。

它提供的负载均衡策略：
- 轮询
- 随机
- 响应时间加权策略：每30秒计算一次服务器的响应时间，以响应时间作为权重，响应时间越短的服务器被选中的概率越大。
- 重试：如果调用失败，使用的负载均衡策略进行重试，注意重试的负载均衡策略默认是随机。
- 区域权衡策略：复合判断server所在区域的性能和server的可用性选择server
- BestAvailableRule：选择并发量最小的服务
- AvailabilityFilteringRule：过滤掉连接失败的服务和并发量高的服务


### 自定义
在configuration中配置bean即可，例如
```java
@Bean
public IRule myRule() {
    //return new RoundRobinRule(); // 显式的指定使用轮询算法
    return new RandomRule();//随机
    //return new RetryRule();//重试
    //return new WeightedResponseTimeRule();//响应时间权重
}
```


## 参考

[https://github.com/Netflix/ribbon/wiki/Working-with-load-balancers](https://github.com/Netflix/ribbon/wiki/Working-with-load-balancers)    
[https://cloud.spring.io/spring-cloud-netflix/2.0.x/single/spring-cloud-netflix.html](https://cloud.spring.io/spring-cloud-netflix/2.0.x/single/spring-cloud-netflix.html)   
[https://blog.csdn.net/wo18237095579/article/details/83384134](https://blog.csdn.net/wo18237095579/article/details/83384134)