# Spring Cloud 学习

## Spring Boot 和 Spring Cloud的区别

* Spring boot 敏捷开发工具，专注于单个个体微服务
* Spring cloud 微服务之间的协调治理框架，将为一个Spring boot项目合并管理，spring cloud是基于Spring boot的。
* 比喻：整个微服务相当于是一个班级，其中每个spring boot 项目就是每一个普通的同学，spring cloud相当于是班长。

## 	Spring Cloud 的组件

* 注册中心：Eureka
* 网关：Spring Cloud Gateway
* 服务调用：Feign
* 负载均衡：Ribbon
* 配置中心：Spring Cloud config
* 服务隔离：Histrxy

## Spring Cloud和Dubbo的对比

| 核心要素     | Dubbo                   | Spring Cloud                 |
| ------------ | ----------------------- | ---------------------------- |
| 服务注册中心 | Zookeeper、Redis、Nacos | Eureka、Nacos                |
| 服务调用方式 | RPC                     | Rest API                     |
| 服务网关     | 无                      | Spring Cloud GateWay         |
| 分布式配置   | Nacos                   | Spring Cloud Config、Nacos   |
| 服务降级熔断 | 不完善                  | Spring Cloud Netflix Hystrix |
| 消息总线     | 无                      | Spring Cloud Bus             |



## Spring Cloud的优缺点

* 缺点
  * 技术栈非常多，学习成本高
  * 在公司中可能不同的模块由不同的部门开发，不同的部门使用的开发语言不同，Spring Cloud对多语言的支持不足
  * 代码侵入性强

## Spring Cloud中开发需要关注的问题

* 网关路由
* 负载均衡
* 熔断降级
* 链路追踪

## 微服务2.0

Service Mesh服务网关

sidecar service mesh代理，可以认为是一个代理层

开发人员只用关注业务的开发即可，不用关注过多的微服务技术细节



## 微服务化架构

![image-20220507152013756](https://s2.loli.net/2022/05/07/Z42L7p5QkiXRIEU.png)





## Spring Cloud Ribbon



## Spring Cloud Eureka

### Eureka的特性

:question:如何判断一个服务不可用

​	:star:  服务提供者每隔30S发送一个心跳请求，Eureka Server记录一个`Map<String,List<InstanceInfo>>`, Key->服务Id，Value->实例列表，在Eureka中有一个定时任务，每隔`60S`检测实例最后一次心跳时间和当前时间相差大于90S，如果大于了，就把实例从可用实例的Map中剔除。剔除的前提是Eureka没有开启自我保护机制。

:question:Eureka的自我保护机制

​	

:question:Eureka的信息存储原理

###  Eureka心跳机制

![image-20220509100241010](https://s2.loli.net/2022/05/09/tZybLUlYdcHAxjK.png)



### Eureka存储结构

![image-20220509101823493](https://s2.loli.net/2022/05/09/ENM9TPb5Gk7ZnhO.png)

### Eureka多级缓存

![image-20220509102540548](https://s2.loli.net/2022/05/09/n9pbl5rS8OuB2RV.png)

## Spring Cloud OpenFeign

> 屏蔽了网络通信的所有细节，直接面向接口开发，让远程调用的像调用本地方法一样简单

### OpenFeign特性

:star: Gzip压缩特性

```properties
feign.compression.request.enable=true
feign.compression.request.mime-type=application/json
feign.compression.request.min-request-size=2048
```

:star:日志功能

```java
@Configuration
public FeignClientConfiguration{
    @Bean
   	Logger.level level(){
        return Logger.level.FULL;
    }
}

@FeignClient(name="goods-service",configuration=FeignClientConfiguration.class)
public interface GoodsFeignClient{
    
}
```

:star:底层通信框架（sun.net.www.protocol.http.HttpURLConnection）可以替换底层的通信组件

```properties
feign.httpclient.enable=fasle
feign.okhttp.enable=true #使用OkHttp Client作为底层通信组件
```

### OpenFeign原理

:star:原理（作为Http的一种代理）

![image-20220509165515880](https://s2.loli.net/2022/05/09/ancCdpkf82gHtKy.png)

## Spring Cloud Nacos
