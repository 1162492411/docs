---
title: "FeignAmbiguousMapping重复映射问题"
date: 2022-01-09T09:40:26+08:00
draft: false
toc: TRUE
categories:
  - 实战
tags:
  - Java
  - Feign
  - Feign实战
---

# 现象

  demo服务引入了多个依赖的业务系统(businessA/businessB/...)的sdk后，启动时FeignClient 报错 Ambiguous mapping 重复映射,报错日志如下:

> 2022-01-06 15:57:20.267 [main] ERROR [bootstrap,,,] org.springframework.boot.SpringApplication - Application startup failed
> org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'documentationPluginsBootstrapper' defined in URL 
> [jar:file:/Users/xxx/.m2/repository/io/springfox/springfox-spring-web/2.9.2/springfox-spring-web-2.9.2.jar!/springfox/documentation/spring/web/plugins/DocumentationPluginsBootstrapper.class]: 
>    Unsatisfied dependency expressed through constructor parameter 1; 
>    nested exception is org.springframework.beans.factory.UnsatisfiedDependencyException: 
>         Error creating bean with name 'webMvcRequestHandlerProvider' defined in URL 
>           [jar:file:/Users/zyg/.m2/repository/io/springfox/springfox-spring-web/2.9.2/springfox-spring-web-2.9.2.jar!/springfox/documentation/spring/web/plugins/WebMvcRequestHandlerProvider.class]:
>             Unsatisfied dependency expressed through constructor parameter 1; 
>             nested exception is org.springframework.beans.factory.BeanCreationException: 
>===============Error creating bean with name 'requestMappingHandlerMapping' defined in class path resource 
>               [org/springframework/boot/autoconfigure/web/WebMvcAutoConfiguration$EnableWebMvcConfiguration.class]: 
>                 Invocation of init method failed; nested exception is java.lang.IllegalStateException: 
>===================Ambiguous mapping. 
>===================Cannot map 'com.demo.client.feign.UserClient' method
> public abstract com.BusinessBService.DataResponse<java.util.List<com.xxx.response.UserVO>> com.business.b.sdk.api.IUserApi.searchDetail(com.business.b.request.UserDTO)
> to {[/user/detail],methods=[POST]}: There is already 'com.demo.client.BusinessBUserClient' bean method
> public abstract com.business.response.DataResponse<java.util.List<com.business.b.response.UserVO>> com.xx.api.IUserApi.searchDetail(com.business.b.request.UserDetailSearchDTO) mapped.
>  at org.springframework.beans.factory.support.ConstructorResolver.createArgumentArray(ConstructorResolver.java:749)
>  at org.springframework.beans.factory.support.ConstructorResolver.autowireConstructor(ConstructorResolver.java:189)
>  at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.autowireConstructor(AbstractAutowireCapableBeanFactory.java:1198)
>  at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBeanInstance(AbstractAutowireCapableBeanFactory.java:1100)
>  at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:511)
>  at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:481)
>  at org.springframework.beans.factory.support.AbstractBeanFactory$1.getObject(AbstractBeanFactory.java:312)
>  at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:230)
>  at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:308)
>  at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:197)
>  at org.springframework.beans.factory.support.DefaultListableBeanFactory.preInstantiateSingletons(DefaultListableBeanFactory.java:761)
>  at org.springframework.context.support.AbstractApplicationContext.finishBeanFactoryInitialization(AbstractApplicationContext.java:867)
>  at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:543)
>  at org.springframework.boot.context.embedded.EmbeddedWebApplicationContext.refresh(EmbeddedWebApplicationContext.java:124)
>  at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:693)
>  at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:360)
>  at org.springframework.boot.SpringApplication.run(SpringApplication.java:303)
>  at org.springframework.boot.SpringApplication.run(SpringApplication.java:1118)
>  at org.springframework.boot.SpringApplication.run(SpringApplication.java:1107)
>  at com.xxxx.DemoApplication.main(DemoApplication.java:31)

Maven依赖 :
* spring-boot : 1.5.18.RELEASE
* spring-boot-starter-web : 1.5.18.RELEASE
* spring-cloud-starter-feign : 1.4.6.RELEASE

# 业务代码

  报错内容表示是在demo服务中有两个不同服务的Client存在同样的URL,导致SpringMvc部分扫描时因为同一个url下存在多个Method/Handler而启动服务失败。
```java
//我们自己写的第一个Client
@FeignClient(name="BusinessAUserClient",url = "${business.a.address}")
public interface BusinessAUserFeignClient extends com.business.a.UserApi{
    //...
}

//我们第一个Client引用的父类(来自依赖的其他服务BusinessA)
@RequestMapping({"/user"})
public interface UserApi{
    @PostMapping("/detail")
    List<UserVo> searchDetail(UserDTO userDTO);
    
    @GetMapping("list")
    List<UserVo> getList(UserDTO userDTO);
}

//我们自己写的第二个Client
@FeignClient(name="BusinessBUserClient",url = "${business.b.address}")
public interface BusinessBUserFeignClient extends com.business.b.UserApi{
    //...
}

//我们第二个Client引用的父类(来自依赖的其他服务BusinessB)
@RequestMapping({"/user"})
public interface UserApi{
    @PostMapping("/detail")
    List<UserVo> searchDetail(UserDTO userDTO);
}
```

> 疑问1 : 为什么FeignClient会被SpringMVC扫描??
> 疑问2 : 平常我们自己写的各种@FeignClient的类是如何被Spring扫描到的
> 疑问3 : 如果是所有依赖服务的URL都被当做我们自己服务的URL进行扫描，那么每个服务都存在health/check接口，为什么这些地方不会报错

# Feign+SpringMVC扫描 - 相关代码定位

  首先我们怀疑一下是SpringMVC的扫描存在问题,从这方面入手,我们查看一下@RequestMapping的类是如何被处理的,省略中间过程,最终查找到如下代码
```java
public class RequestMappingHandlerMapping {

    /**
     * {@inheritDoc}
     * <p>Expects a handler to have either a type-level @{@link Controller}
     * annotation or a type-level @{@link RequestMapping} annotation.
     */
    @Override
    protected boolean isHandler(Class<?> beanType) {
        return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
                AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
    }
}
```
那么原因就是各个依赖的UserApi等类存在@RequestMapping注解,所以SpringMVC顺手进行了扫描

# Feign+SpringMVC扫描 - 解决思路
  既然根源在于被SpringMVC额外扫描了，那么我们可以想办法让其不被SpringMVC扫描(但是最好仍然交给Spring托管xxxFeignClient)

## 方案一 : 手动写URL
```java
//我们自己写的第一个Client(这个类不再继承com.business.a.UserApi)
@FeignClient(name="BusinessAUserClient",url = "${business.a.address}",path = "/user")
public interface BusinessAUserFeignClient{
    
    @PostMapping("/detail")
    List<UserVo> searchDetail(UserDTO userDTO);

    @GetMapping("/list")
    List<UserVo> getList(UserDTO userDTO);
    
}

//我们自己写的第二个Client(这个类可以仍然继承com.business.b.UserApi,也可以像BusinessAUserFeignClient一样手写url)
@FeignClient(name="BusinessBUserClient",url = "${business.b.address}")
public interface BusinessBUserFeignClient extends com.business.b.UserApi{
    //...
}

//我们第二个Client引用的父类(来自依赖的其他服务BusinessB)
@RequestMapping({"/user"})
public interface UserApi{
    @PostMapping("/detail")
    List<UserVo> searchDetail(UserDTO userDTO);
}
```
  这样一来,最多有一个url的Handler被SpringMVC扫描，服务可以正常启动，代码也可以正常调用这几个url。
  但是,这样一来,我们就无法享受到依赖服务SDK的好处了,如果依赖服务后期新增/改动了url/改动了url中的请求或相应类,我们这边都要进行相应修改;如果要求
依赖服务方将UserApi的类级别的@RequestMapping移动到各方法中也有些强人所难且不合理

## 方案二 : SpringMVC扫描时过滤
  既然问题出在SpringMVC扫描,那么如果我们修改isHandler()的逻辑,使其在遇到@FeignClient的类时忽略,也能达到我们想要的效果
```java
//代码拷贝自https://github.com/spring-cloud/spring-cloud-netflix/issues/466#issuecomment-257043631
import feign.Feign;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.web.WebMvcRegistrations;
import org.springframework.boot.autoconfigure.web.WebMvcRegistrationsAdapter;
import org.springframework.cloud.netflix.feign.FeignClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.AnnotationUtils;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping;

/**
 * 解决带有@FeignClient的类继承了类级别带有@RequestMapping的API类之后被springmvc扫描handlerMapping的问题
 * @author xxx
 */
@Configuration
@ConditionalOnClass({Feign.class})
public class FeignMappingDefaultConfiguration {
    @Bean
    public WebMvcRegistrations feignWebRegistrations() {
        return new WebMvcRegistrationsAdapter() {
            @Override
            public RequestMappingHandlerMapping getRequestMappingHandlerMapping() {
                return new FeignFilterRequestMappingHandlerMapping();
            }
        };
    }

    private static class FeignFilterRequestMappingHandlerMapping extends RequestMappingHandlerMapping {
        @Override
        protected boolean isHandler(Class<?> beanType) {
            return super.isHandler(beanType) && (AnnotationUtils.findAnnotation(beanType, FeignClient.class) == null);
        }
    }
}
```

## 方案三 : 官方解决方案
Spring官方(Spring-framework)相关开发经过讨论,最终给出的[解决方案](https://github.com/spring-projects/spring-framework/issues/22154#issuecomment-936906502)是
> 2021-10-7
> after some cross-team discussions, for the short term the issue will be addressed on the side of Spring Data REST and Spring Cloud.
> For Spring Framework 6.0, we will also address this in the Spring Framework by no longer considering a class with a 
> type-level @RequestMapping as a candidate for detection, unless there is also @Controller.
> 经过一些跨团队讨论，短期内该问题将在 Spring Data REST 和 Spring Cloud 方面解决。
> 对于 Spring Framework 6.0，我们还将在 Spring Framework 中解决这个问题，不再将具有类型级别的类@RequestMapping作为检测候选，除非也有@Controller.

具体代码如下

```java
public class RequestMappingHandlerMapping {
  
  //只将类级别有@Controller的进行扫描,这样改整体看合理但是不向前兼容
	protected boolean isHandler(Class<?> beanType) {
		return AnnotatedElementUtils.hasAnnotation(beanType, Controller.class);
	}
}
```





Spring-cloud-openfeign的[解决方案](https://github.com/spring-cloud/spring-cloud-openfeign/issues/547)是不支持@FeignClient+@RequestMapping,
具体代码为https://github.com/spring-cloud/spring-cloud-openfeign/commit/d6783a6f1ec8dd08fafe76ecd072913d4e6f66b9

```java
public class SpringMvcContract extends Contract.BaseContract
		implements ResourceLoaderAware {

	protected void processAnnotationOnClass(MethodMetadata data, Class<?> clz) {
			RequestMapping classAnnotation = findMergedAnnotation(clz,
					RequestMapping.class);
			if (classAnnotation != null) {
				LOG.error("Cannot process class: " + clz.getName()
						+ ". @RequestMapping annotation is not allowed on @FeignClient interfaces.");
				throw new IllegalArgumentException(
						"@RequestMapping annotation not allowed on @FeignClient interfaces");
   }
}   
```

