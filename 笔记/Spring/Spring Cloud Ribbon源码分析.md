# Spring Cloud Ribbon源码分析

## 学习源码的目的

* 快速定位框架出现的问题
* 学习大牛的设计思想，潜移默化的提升自己的设计能力
* 学习各种API的使用

## Ribbon的作用

在分布式集群的部署下，客户端需要通过一种负载均衡的方式去调用服务端，Ribbon封装了各种负载均衡的算法，便于开发者使用。

## @LoadBalanced注解原理分析

表面功能： `http://service-id ->http://host:port`

```JAVA
@LoadBalanced
@Bean
public RestTemplate restTemplate(){
    return new RestTemplate();
}
```

@LoadBalanced注解

```JAVA
@Qualifier
public @enum LoadBalanced(){
    
}
```

`LoadBalancerAutoConfiguarion`  LoadBalance自动装配类

给添加了`@LoadBalanced`注解的`RestTemplate`添加`LoadBalancerInterceptor`拦截器，请求被Ribbon接管了，请求会被Riibon干预

## LoadBalancerInterceptor

默认采用 `ZoneAvoidanceRule`Rule实现负载均衡（底层使用轮询的算法）

## Rule.Choose原理分析

## 服务列表加载过程分析

