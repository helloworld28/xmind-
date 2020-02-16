# Spring Cloud Gateway

## 定义 

This project provides an API Gateway built on top of the Spring Ecosystem, including: Spring 5, Spring Boot 2 and Project Reactor. Spring Cloud Gateway aims to provide a simple, yet effective way to route to APIs and provide cross cutting concerns to them such as: security, monitoring/metrics, and resiliency

大概的意思就是它是基于Spring 5 SpringBoot2 及Reactor (Webflux) 构建的，提供接口的路由，可能对接口的安全，监控，弹性扩展。

- Built on Spring Framework 5, Project Reactor and Spring Boot 2.0
- Able to match routes on any request attribute.
- Predicates and filters are specific to routes.
- Hystrix Circuit Breaker integration.
- Spring Cloud DiscoveryClient integration
- Easy to write Predicates and Filters
- Request Rate Limiting
- Path Rewriting

相当于反向代理

## 三大组件概念：

* route ：路由配置

* Predicate : 类似java 8里的function Predicate ,就是通过分析请求里的各个参数进行匹配判断

  > cookie,head, host,method,path, query,RemoteAddr

* Filter : 相当于Servlet哪个Filter 一样的，对请求的前后进前Filter 可以进行内容修改

  > addRespondHeader DedupeResponseHeader Hystrix PrefixPath PreserveHostHeader RequestRateLimiter RedirectTo RemoveHopByHopHeadersFilter RemoveRequestHeader RewritePath SecureHeaders StripPrefix Retry Default Filters
  >
  > 



## 流程图

![Spring Cloud Gateway Diagram](https://raw.githubusercontent.com/spring-cloud/spring-cloud-gateway/master/docs/src/main/asciidoc/images/spring_cloud_gateway_diagram.png)

https://dzone.com/articles/microservices-with-spring-boot-spring-cloud-gatewa

集成服务发现及负载均衡的例子

```
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
      routes:
        - id: account-service
          uri: lb://account-service
          predicates:
            - Path=/account/**
          filters:
            - RewritePath=/account/(?<path>.*), /$\{path}
        - id: customer-service
          uri: lb://customer-service
          predicates:
            - Path=/customer/**
          filters:
            - RewritePath=/customer/(?<path>.*), /$\{path}
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/order/**
          filters:
            - RewritePath=/order/(?<path>.*), /$\{path}
        - id: product-service
          uri: lb://product-service
          predicates:
            - Path=/product/**
          filters:
            - RewritePath=/product/(?<path>.*), /$\{path}
```

