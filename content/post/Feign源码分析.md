---
title: "Feign源码分析"
date: 2022-01-17T21:50:40+08:00
draft: false
toc: TRUE
categories:
  - Feign
tags:
  - Java
  - Feign
  - 源码分析
---

# Feign源码分析-初始化流程和执行流程

## 概述

Feign`是一个面向对象的`http`客户端, 这里主要介绍`Feign`是如何初始化的, 并如何进行`http`请求的.
省略部分不重要的代码. 剥离了`hystrix`和`ribbon`的相关逻辑.

Feign大致经历了以下的发展历程

> Feign : Netflix开源维护的声明式REST客户端
>
> spring-cloud-feign ：Spring-Cloud-1.x基于Feign，支持SpringMvc注解
>
> OpenFeign ：Netflix不再维护Feign，交由社区维护，更名为OpenFeign
>
> spring-cloud-openfeign ：Spring-Cloud-2.x基于OpenFeign，支持SpringMvc注解

## 使用方式

### 1.引入框架依赖

```xml
<dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

### 2.开启feign功能

```java
//在启动类添加@EnableFeignClients注解
@EnableFeignClients
@SpringBootApplication
public class ConsumerApplication {
    //....       
}
```

### 3.定义业务Client

```java
//业务client
@FeignClient(value = "businessA-client",url = "http://localhost:8081/service-a.address", configuration=BusinessAFeignConfiguration.class)
public interface BusinessAFeignClient {

    @GetMapping("/v1/user/{id}")
    ResponseEntity<User> getUserById(@PathVariable Long id, @RequestParam(required = false) String name);

    @PostMapping("/v1/user")
    ResponseEntity<User> createUser(@RequestBody User user);

}

//业务client的feign配置类
public class BusinessAFeignConfiguration {

    @Bean
    Retryer feignRetryer() {
        return Retryer.NEVER_RETRY;
    }

    @Bean
    Request.Options requestOptions() {
        int ribbonReadTimeout = 2000;
        int ribbonConnectionTimeout = 500;
        return new Request.Options(ribbonConnectionTimeout, ribbonReadTimeout);
    }
}
```

### 4.使用业务Client

```java
@RestController
public class FeignController {
  
  	//注入AFeignClient
    @Autowired
    private BusinessAFeignClient aFeignClient;

    @GetMapping("/v1/user/query")
    public ResponseEntity<User> queryUser(Long id,String name) {
        //调用AFeignClient的方法
        User user = aFeignClient.getUserById(id,name);
        return ResponseEntity.ok(user);
    }

}
```

## 初始化流程



### 流程概览

在feign的调用过程中，针对不同的服务为其创建一个上下文，我们可以将这个上下文理解为沙箱。执行调用所需要的资源是从各自的沙箱环境中取。整体的职责分工如下图所示

![image-20220117220527611](https://gitee.com/1162492411/pic/raw/master/image-20220117220527611.png)

### 扫描注册子流程

​    通过前面的DEMO可以发现，使用 Feign 最核心的应该就是 @EnableFeignClients 和 @FeignClient 这两个注解，@FeignClient 加在客户端接口类上，@EnableFeignClients 加在启动类上，就是用来扫描加了 @FeignClient 接口的类。我们研究源码就从这两个入口开始。

```java
@Import(FeignClientsRegistrar.class) // 引入 FeignClientsRegistrar
public @interface EnableFeignClients {
  ...
}
```

`@EnableFeignClients`注解引入了`FeignClientsRegistrar`, 类实现了`ImportBeanDefinitionRegistrar`接口, 然后重写了`registerBeanDefinitions`方法.
说明加入到了`Spring`生命周期的管理中.

```java
// org.springframework.cloud.openfeign.FeignClientsRegistrar
// 实现了 ImportBeanDefinitionRegistrar 接口, 注册到 Spring 生命周期中
class FeignClientsRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware, EnvironmentAware {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
        // 1. 注册全局默认feign配置
        registerDefaultConfiguration(metadata, registry);
        // 2. 创建 FeignClientFactoryBean 注入到 Spring 容器中
        registerFeignClients(metadata, registry);
    }
}
```

#### 注册全局默认feign配置

registerDefaultConfiguration()方法比较简单，这里直接给出文字版逻辑:

* 获取 EnableFeignClients 中的属性
* 注册一个 default.类名.FeignClientSpecification 的 Bean, 作为默认Feign配置

#### 注册FeignClientFactoryBean

逻辑如下

* 准备扫描路径列表 : 根据 @EnableFeignClients 的 clients 属性或者 value、basePackages、basePackageClasses 属性初始化 basePackages 变量
* 准备扫描器，并添加过滤，只扫描 @FeignClient 注解修饰的类
* for循环basePackages
  * 扫描包下的所有被 @FeignClient 注解修饰的接口
  * 获取@FeignClient注解上的值
  * 获取name : 按 contextId、value、name、serviceId 的优先级顺序获取 name, 用来做下面 bean name 的前缀${name}
  * 注册feignConfigraution到spring core ：注册为${name}.FeignClientSpecification
  * 注册FeignClientFactoryBean到spring core : com.xxx.AClient(别名 ${name}FeignClient)

#### 扫描注册一图流

![image-20220117220742515](https://gitee.com/1162492411/pic/raw/master/Feign-源码分析-注册流程一图流.png)

### 生成动态代理子流程

​    FeignClientFactoryBean 实现了 FactoryBean 接口，当一个Bean实现了 FactoryBean 接口后，Spring 会先实例化这个工厂，然后调用 getObject() 创建真正的Bean。

```java
class FeignClientFactoryBean implements FactoryBean<Object>, InitializingBean, ApplicationContextAware {
 @Override
    public Object getObject() throws Exception {
        //核心代码
        FeignContext context = applicationContext.getBean(FeignContext.class);
        Feign.Builder builder = feign(context);
        Client client = getOptional(context, Client.class);
        builder.client(client);
        Targeter targeter = get(context, Targeter.class);
        return targeter.target(this, builder, context, new HardCodedTarget<>(
                this.type, this.name, url));
    }
}
```

#### 核心组件

在该子流程中,涉及到以下核心组件

- FeignContext : 作用是隔离不同服务的各种feign组件,创建一个沙箱环境
- Feign.Builder : 作用是准备好每个FeignClient所需要的各种feign配置组件(借助sring存储在不同的springContext),设置了 Logger、Encoder、Decoder、Contract....，并读取配置文件中 feign.client.* 相关的配置
- Client : 作用是完成http请求，目前有以下实现类
  - Default ： 默认实现，使用HttpURLConnection
  - Proxied : https实现，使用HttpURLConnection
  - FeignBlockingLoadBalancerClient ： 负载均衡实现，使用ribbon load balance
  - RetryableFeignBlockingLoadBalancerClient ：负载均衡+重试实现，使用spring cloud load balance
- Target : 作用是桥接不同的动态代理实现类,现在有`DefaultTargeter`和`HystrixTargeter`
- Feign : 作用是生成代理类并借助java生成代理关系,目前有ReflectiveFeign，**在这里注入了properties**
- Map<Method,MethodHandler> dispatchMap : 实际请求时通过该数据进行路由

#### 生成代理一图流

![image-20220117220849995](https://gitee.com/1162492411/pic/raw/master/Feign-源码分析-生成代理流程.png)

#### 代理关系核心代码

```java
//1.Feign.newInstance()具体逻辑
public class ReflectiveFeign extends Feign {
   public <T> T newInstance(Target<T> target) {
    //feign自己的内部关系 
    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
    //借助java反射维护的关系
    List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();
	  //在factory.create中将fein的invocationHandler注入进去
    InvocationHandler handler = factory.create(target, methodToHandler);
    //借助java反射实现代理拦截效果
    T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(),
        new Class<?>[] {target.type()}, handler);
    for (DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
      defaultMethodHandler.bindTo(proxy);
    }
    return proxy;
  }
}
//factory.create()具体逻辑
public interface InvocationHandlerFactory {
  //调用这里
  InvocationHandler create(Target target, Map<Method, MethodHandler> dispatch);

  interface MethodHandler {

    Object invoke(Object[] argv) throws Throwable;
  }

  static final class Default implements InvocationHandlerFactory {
    //最终调用到这里
    @Override
    public InvocationHandler create(Target target, Map<Method, MethodHandler> dispatch) {
      return new ReflectiveFeign.FeignInvocationHandler(target, dispatch);
    }
  }
}
```

## 处理请求流程

​    feign的设计思路是在初始化流程中生成业务Client和业务Client的动态代理类的绑定关系,那么当代码调用到业务Client方法时,便会转向调用代理类的方法。

#### 处理请求核心代码

上文分析到最终在ReflectiveFeign中生成了代理，那么处理请求时就会调用到该类的invoke()

```java
static class FeignInvocationHandler implements InvocationHandler {
  //这个在初始化流程中准备好了关系
  private final Map<Method, MethodHandler> dispatch;
 @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      return dispatch.get(method).invoke(args);
    }
}  
```

​    在dispatch中注册进的MethodHandler实际为SynchronousMethodHandler

```java
final class SynchronousMethodHandler implements MethodHandler{
    @Override
  public Object invoke(Object[] argv) throws Throwable {
    /**
    1.根据请求参数构建请求模板 RequestTemplate，这个时候会处理参数中的占位符、拼接请求参数、处理body中的参数等等。
    2.将 RequestTemplate 转成 Request : 
      2.1 用 RequestInterceptor 处理请求模板
      2.2 用 Target（HardCodedTarget）处理请求地址，拼接上服务名前缀
      2.3 调用 RequestTemplate 的 request 方法获取到 Request 对象
    3. 调用 LoadBalancerFeignClient 的 execute 方法来执行请求并得到请求结果 Response
    4. 得到 Response 后，就使用解码器 Decoder 解析响应结果，返回接口方法定义的返回类型
    */
  }
}
```

#### 处理请求一图流

![image-20220117221035060](https://gitee.com/1162492411/pic/raw/master/Feign-源码分析-处理请求一图流.png)

![image-20220117221221585](https://gitee.com/1162492411/pic/raw/master/Feign-源码分析-处理请求概览.png)
