---
title: "Jackson实现入参多态"
date: 2022-11-13T15:55:32+08:00
draft: false
toc: true
categories:
  - 实战
tags:
  - Spring
  - 实战
---
# 需求
  在Spring项目中,我们在接收入参时,一般使用@RequestBody,将HTTP报文反序列化为具体的入参对象。但有时情况会略微复杂一些，需要将入参解析为不同的子类对象。这里我们以经典的学校人员管理系统为例，讲解如何实现该功能。
  具体的需求为：该系统中的人员类型分为学生/老师，在信息录入接口中需要根据人员类型,执行不同的信息录入方法。
  在各序列化框架中调研后，发现Jackson中的@JsonTypeInfo可以用来进行多态的序列化和反序列化，该注解作用在接口/类上，被用来开启多态类型的处理，对基类/接口和子类/实现类都有效。
# 代码实现
这里我们先给出一个简单方案。
```java
// 定义一个抽象的父类,在这里指定入参与子类的映射关系
import com.fasterxml.jackson.annotation.JsonSubTypes;
import com.fasterxml.jackson.annotation.JsonTypeInfo;
import lombok.*;
@Data
@AllArgsConstructor
@NoArgsConstructor
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type", visible = true,
        defaultImpl = InfoReqBaseParam.class, include = JsonTypeInfo.As.EXISTING_PROPERTY)
@JsonSubTypes({
        @JsonSubTypes.Type(value = StudentInfoParam.class, name = "code_student"),
        @JsonSubTypes.Type(value = TeacherInfoParam.class, name = "code_teacher")
})
public class InfoReqBaseParam {
    private String type;
}
// 定义子类
import lombok.*;
@Data
@EqualsAndHashCode(callSuper = true)
@AllArgsConstructor
@NoArgsConstructor
public class StudentInfoParam extends InfoReqBaseParam{
    private String name;
    private String no;
}
//定义子类
import lombok.*;
@Data
@EqualsAndHashCode(callSuper = true)
@AllArgsConstructor
@NoArgsConstructor
public class TeacherInfoParam extends InfoReqBaseParam{
    private String name;
    private Integer classNumber;
}
//使用转换换后的子类
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;
import org.springframework.util.StringUtils;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import java.util.Objects;
@RestController
@RequestMapping("demo")
@Slf4j
public class IDemoController implements IDemoApi{
    @Override
    public void receive(@RequestBody InfoReqBaseParam req) throws JsonProcessingException {
        log.info("接收到入参:{}",new ObjectMapper().writeValueAsString(req));
        if(Objects.isNull(req) || StringUtils.isEmpty(req.getType())){
            return;
        }
        switch (req.getType()){
            case "code_student" :
                log.info("mockStudentMethod,object type:{}", req.getClass());
                break;
            case "code_teacher" :
                log.info("mockTeacherMethod,object type:{}", req.getClass());
                break;
            default: log.info("no method match"); break;
        }
    }
}
```
  这样，我们就可以通过在入参中为InfoReqBaseParam的type指定不同的值，来自动转换为不同的子类：type=code_student时将入参转换为StudentInfoParam.class;type=code_teacher时将入参转换为TeacherInfoParam.class;默认情况下将入参转换为InfoReqBaseParam。这个转换关系我们在@JsonTypeInfo与@JsonSubTypes中进行维护
# 入参解析代码改进
上述方案已经基本满足我们的需求,但是在实际的业务场景中，也许会出现子类对象也有type字段，为了解决这种情况，我们可以通过如下方式来满足需求：我们将type字段放在另外一个大对象中，根据大对象中的reqType字段，来将入参转换为reqParam的不同子类。
```java
//定义父类
import com.fasterxml.jackson.annotation.JsonTypeInfo;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
@Data
@AllArgsConstructor
@NoArgsConstructor
public class InfoReqDto<T extends InfoReqBaseParam> {
    private String reqType;
    @JsonTypeInfo(
            use = JsonTypeInfo.Id.NAME,
            property = "reqType",
            visible = true,
            defaultImpl = com.study.jackson.sample.resolveBySelf.InfoReqBaseParam.class,
            include = JsonTypeInfo.As.EXTERNAL_PROPERTY
    )
    private T reqParam;
}
//定义映射关系
import com.fasterxml.jackson.annotation.JsonSubTypes;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@JsonSubTypes({
        @JsonSubTypes.Type(value = StudentInfoParam.class, name = "code_student"),
        @JsonSubTypes.Type(value = TeacherInfoParam.class, name = "code_teacher")
})
public class InfoReqBaseParam {

}
// 定义子类
import lombok.*;
@Data
@EqualsAndHashCode(callSuper = true)
@AllArgsConstructor
@NoArgsConstructor
public class StudentInfoParam extends InfoReqBaseParam {

    private String name;

    private String no;

}
// 定义子类
import lombok.*;
@Data
@EqualsAndHashCode(callSuper = true)
@AllArgsConstructor
@NoArgsConstructor
public class TeacherInfoParam extends InfoReqBaseParam {

    private String name;

    private Integer classNumber;

}
//使用
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;
import org.springframework.util.StringUtils;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Objects;

@Slf4j
@RestController
@RequestMapping("")
public class IOtherController implements IOtherApi {

    @Override
    public void receiveDto(@RequestBody InfoReqDto req) throws JsonProcessingException {
        log.info("接收到入参:{}",new ObjectMapper().writeValueAsString(req));
        if(Objects.isNull(req) || StringUtils.isEmpty(req.getReqType())){
            return;
        }
        switch (req.getReqType()){
            case "code_student" :
                log.info("mockStudentMethod,object type:{}", req.getReqParam().getClass());
                break;
            case "code_teacher" :
                log.info("mockTeacherMethod,object type:{}", req.getReqParam().getClass());
                break;
            default: log.info("no method match"); break;
        }
    }
}
```
# 子类映射代码改进
以上两种方式已经可以使用在实际业务场景中了，但是仍然存在一个不太优雅的地方：每次增加一个子类，就要修改@JsonSubTypes，这违反了“开闭原则”(对扩展开放、对修改封闭)，那么有什么方法可以改进吗？有以下两种方式可以实现我们想要的效果
## 方式一：扫描子类并注册
现在我们是对@JsonSubTypes进行硬编码的，只要想办法将其改为动态编码即可,我们可以拆成三步实现：在子类添加某个注解；使用工具扫描包含有该注解的类，生成一个class列表；将该列表注册到Jackson中
```java
//这里只给出第三步的demo代码
ObjectMapper om = new ObjectMapper();
classList.stream().foreach(item -> om.registerSubTypes(item));
```
## 方式二：自定义TypeIdResolver
  仍然借助Jackson的@JsonTypeInfo，使用JsonTypeInfo.Id.CUSTOM策略，然后自定义解析规则(思路来自[网友实现](https://www.jianshu.com/p/684794b02d19))
```java
//在抽象父类添加该代码
@JsonTypeInfo(use = JsonTypeInfo.Id.CUSTOM, property = "type")
@JsonTypeIdResolver(JacksonTypeIdResolver.class)
//自定义一个TypeIdResolver
public class JacksonTypeIdResolver implements TypeIdResolver {private JavaType baseType;

    @Override
    public void init(JavaType javaType) {
        baseType = javaType;}


    @Override
    public String idFromValue(Object o) {return idFromValueAndType(o, o.getClass());}/**
     * 序列化时填充什么对象
     */
    @Override
    public String idFromValueAndType(Object o, Class<?> aClass) {//有出现同名类时可以用这个来做区别
        JsonTypeName annotation = aClass.getAnnotation(JsonTypeName.class);if (annotation != null) {return annotation.value();}
        String name = aClass.getName();
        String[] splits = StringUtils.split(name, ".");
        String className = splits[splits.length - 1];return className;}/**
     * idFromValueAndType 是序列化的时候告诉序列化器怎么生成标识符
     * <p>
     * typeFromId是反序列化的时候告诉序列化器怎么根据标识符来识别到具体类型，这里用了反射,在程序启动时，把要加载的包通过Reflections加载进来
     */
    @Override
    public JavaType typeFromId(DatabindContext databindContext, String type) {
        Class<?> clazz = getSubType(type);
        if (clazz == null) {
            throw new IllegalStateException("cannot find class '" + type + "'");
        }
        return databindContext.constructSpecializedType(baseType, clazz);
    }

    public Class<?> getSubType(String type) {
        Reflections reflections = ReflectionsCache.getReflections();
        Set<Class<?>> subTypes = reflections.getSubTypesOf((Class<Object>) baseType.getRawClass());
        for (Class<?> subType : subTypes) {
            JsonTypeName annotation = subType.getAnnotation(JsonTypeName.class);if (annotation != null && annotation.value().equals(type)) {return subType;} else if (subType.getSimpleName().equals(type) || subType.getName().equals(type)) {return subType;}}return null;}

    @Override
    public String idFromBaseType() {return idFromValueAndType(null, baseType.getClass());}

    @Override
    public String getDescForKnownTypeIds() {return null;}

    @Override
    public JsonTypeInfo.Id getMechanism() {return JsonTypeInfo.Id.CUSTOM;}
}
//使用-父类定义
@Data
@JsonTypeInfo(use = JsonTypeInfo.Id.CUSTOM, property = "type")
@JsonTypeIdResolver(JacksonTypeIdResolver.class)
public class Animal {private String name;
}
//使用-子类(无需添加任何注解)
@Data
public class Dog extends Animal {
    private int age;
}
```
# 另一种思路-自定义参数解析器
  之前的思路都是在使用Jackson的前提下进行技术改造，那么对于不使用Jackson的项目，我们可以通过自行实现参数解析器来达到我们想要的效果，这里只给出来实现思路，具体代码自行实现：继承AbstractMessageConverterMethodArgumentResolver，覆写resolveArgument方法
```java
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.alibaba.fastjson.support.spring.FastJsonHttpMessageConverter;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.io.IOUtils;
import org.springframework.core.Conventions;
import org.springframework.core.MethodParameter;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.WebDataBinder;
import org.springframework.web.bind.support.WebDataBinderFactory;
import org.springframework.web.context.request.NativeWebRequest;
import org.springframework.web.method.support.ModelAndViewContainer;
import org.springframework.web.servlet.mvc.method.annotation.AbstractMessageConverterMethodArgumentResolver;

import javax.servlet.http.HttpServletRequest;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.util.Collections;
import java.util.stream.Stream;

/**
 * 自定义参数解析
 */
@Slf4j
public class xxxModelArgumentResolver extends AbstractMessageConverterMethodArgumentResolver {

    public SettlementApplyModelArgumentResolver() {
        super(Collections.singletonList(new FastJsonHttpMessageConverter()));
    }

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(xxxBaseModel.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest,
                                  WebDataBinderFactory binderFactory) throws Exception {
        SettlementApplyReqModel model = resolveModel(webRequest);
        parameter = parameter.nestedIfOptional();
        String name = Conventions.getVariableNameForParameter(parameter);
        WebDataBinder binder = binderFactory.createBinder(webRequest, model, name);
        if (model != null) {
            // 使用 Spring Validator 进行参数校验
            this.validateIfApplicable(binder, parameter);
            if (binder.getBindingResult().hasErrors() && this.isBindExceptionRequired(binder, parameter)) {
                throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
            }
        }
        mavContainer.addAttribute(BindingResult.MODEL_KEY_PREFIX + name, binder.getBindingResult());
        return model;
    }

    /**
     * 解析请求参数中的场景码，反序列化特定的 Model 类型
     */
    private SettlementApplyReqModel resolveModel(NativeWebRequest webRequest) throws IOException {
        HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
        String body = IOUtils.toString(request.getInputStream(), StandardCharsets.UTF_8);
        log.info("参数解析请求体：{}", body);
        JSONObject jsonObject = JSON.parseObject(body);
        Integer sceneCode = jsonObject.getInteger("sceneCode");
        AssertUtil.isNotNull(sceneCode, "场景码为必传参数！");
        //下面根据不同的sceneCode解析不同的代码
        return jsonObject.toJavaObject(xxxx.class);
    }

}
```