# spring-cloud-demo-01-eureka

> 本Demo演示了Eureka服务端的搭建，以及消费者如何通过Eureka服务注册中心调用其他供应者提供的接口。

## 模块介绍

spring-cloud-demo-01-eureka模块分为三个子模块，分别为Eureka-Server(Eureka服务端)、Eureka-Provider(供应者)和Eureka-Consumer(消费者)。

## 一、Eureka服务注册与发现

### 1.Eureka-Server

1.1 添加依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
</dependencies>
```

1.2 在启动类上添加注解@EnableEurekaServer

```java
@EnableEurekaServer
@SpringBootApplication
public class SpringCloudDemo01EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringCloudDemo01EurekaServerApplication.class, args);
    }
}
```

1.3 添加配置文件

```yaml
server:
  port: 8761
eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false #该实例是否向EurekaServer注册自己，默认true
    fetch-registry: false  #该实例是否向Eureka服务器获取所有的注册信息表，默认true
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/  #配置 Eureka-Server 地址
```

### 2.Eureka-Provider

1.1 添加依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

1.2 添加配置文件

```yaml
server:
  port: 8761
eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false #该实例是否向EurekaServer注册自己，默认true
    fetch-registry: false  #该实例是否向Eureka服务器获取所有的注册信息表，默认true
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/  #配置 Eureka-Server 地址
```

## 启动Demo

- 1.启动Eureka-Server
- 2.启动Eureka-Provider
- 3.启动Eureka-Consumer

## 结果检验

打开[http://localhost:8761/](http://localhost:8761/)，将会看到Eureka的界面以及Eureka-Provider和Eureka-Consumer服务

## 二、生产者与消费者(RestTemplate)

### 1.Eureka-Provider

1.1 添加接口

```java
@RestController
public class ApiController {
    @RequestMapping("/hello/{name}")
    public String hello(@PathVariable String name) {
        System.out.println("I am provider , name :" + name);
        return "hello " + name;
    }
}
```

1.2 测试接口

![](https://pic.downk.cc/item/5e61c52598271cb2b8045f07.png)

### 2.Eureka-Consumer

1.1 添加接口

```java
@RestController
@AllArgsConstructor
public class ApiController {
    private DiscoveryClient discoveryClient;
    private RestTemplate restTemplate;

    @RequestMapping("/hello/{name}")
    public String hello(@PathVariable String name) {
        System.out.println("I am consumer , name :" + name);
        List<ServiceInstance> instances = discoveryClient.getInstances("provider");
        if (!instances.isEmpty()) {
            ServiceInstance serviceInstance = instances.get(0);
            return restTemplate.getForObject(serviceInstance.getUri().toString() + "/hello/" + name, String.class);
        }
        return "not find provider";
    }
}
```

1.2 测试接口

![](https://pic.downk.cc/item/5e61ffc398271cb2b8167015.png)

同时Eureka-Provider控制台打印出，说明能够通过Eureka-consumer的接口，调用Eureka-Provider的接口

```
I am provider , name :seepine
```



## 三、生产者与消费者(Feign)

### 1.Eureka-Consumer

1.1 添加Feign依赖
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

1.2 启动类添加注解(@EnableFeignClients)
```java
@EnableEurekaClient
@SpringBootApplication
@EnableFeignClients
public class SpringCloudDemo01EurekaConsumerApplication {
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudDemo01EurekaConsumerApplication.class, args);
    }

}
```

1.3 添加FeignService
```java
package com.seepine.demo.eureka.consumer.feign;

//填写要调用方，即生产者spring.application.name
@FeignClient("provider")
public interface RemoteApiService {
    @GetMapping("/hello/{name}")
    String sayHello(@PathVariable("name") String name);
}
```

1.4 添加接口

```java
@RestController
@AllArgsConstructor
public class ApiController {
    private RemoteApiService remoteApiService;

    @RequestMapping("/helloByFeign/{name}")
    public String helloByFeign(@PathVariable String name) {
        return remoteApiService.sayHello(name);
    }
}
```

1.5 测试接口

![](https://pic.downk.cc/item/5e6368d298271cb2b8d26e96.png)

同时Eureka-Provider控制台打印出，说明能够通过Eureka-consumer的接口，调用Eureka-Provider的接口

```
I am provider , name :seepine-feign
```

## 四、Eureka-Server增加用户验证

### 1.添加依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

### 2.添加配置
```yaml
spring:
  security:
    user:
      name: seepine
      password: 123456
```

### 3.打开[http://127.0.0.1:8761/](http://127.0.0.1:8761/)验证

![](https://pic.downk.cc/item/5e6706a098271cb2b87b83f9.png)

### 4.修改所有Eureka连接url
```
http://seepine:123456@${eureka.instance.hostname}:${server.port}/eureka/
```
