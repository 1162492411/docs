---
title: "MapStruct简要指南"
date: 2022-11-13T15:55:32+08:00
draft: false
toc: true
categories:
  - 实战
tags:
  - Spring
  - 实战
  - 数据转换
---

  MapStruct主要用于对象转换
- 原理：大致原理与lombok类似，通过注解处理器，在代码编译时为带有MapStruct注解的接口生成实现类，这种实现方式在性能方面要高于基于反射一类对象转换组件，如BeanUtils。
- 使用方法：引入依赖，创建接口，在接口的方法中添加注解，编译代码即可生成完整的转换代码。
    下面我们以生活中常见的订单为例子进行讲解。
# 简单接入
引入依赖，创建接口，添加注解。我们以Maven为例，添加依赖包与编译插件
```xml  
<properties>
    <org.mapstruct.version>1.5.2.Final</org.mapstruct.version>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
</properties>
<dependencies>
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>${org.mapstruct.version}</version>
    </dependency>
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct-jdk8</artifactId>
        <version>${org.mapstruct.version}</version>
    </dependency>
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct-processor</artifactId>
        <version>${org.mapstruct.version}</version>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.12</version>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.13.1</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```
  下面我们创建几个实体类
```java    
@Data
public class Order{
    private Long id;
    private Long no;
    private Date createDate;
}
@Data
public class OrderRes {
    private Long id;
    private Long orderNo;
    private Date orderCreateDate;
}
@Mapper
public interface OrderMapper {
    OrderMapper INSTANCE = Mappers.getMapper(OrderMapper.class);
    @Mappings({
            @Mapping(source = "no", target = "orderNo")
    })
    OrderRes dbToRes(Order order);
}
```
  然后我们使用单元测试，查看代码运行效果
```java  
public class OrderMapperTest {
    @Test
    public void personDTOToPerson() {
        //准备入参
        OrderMapper orderMapper = OrderMapper.INSTANCE;
        Order order = new Order();
        order.setId(222L);
        order.setNo(1L);
        order.setCreateDate(new Date());
        //转换
        OrderRes orderRes = orderMapper.dbToRes(order);
        //判断出参
        Assert.assertEquals(order.getId(),orderRes.getId());
        Assert.assertEquals(order.getNo(),orderRes.getOrderNo());
        Assert.assertNull(orderRes.getOrderCreateDate());
    }
}
```
  从上面的例子可以看出，使用MapStruct定义一个对象转换器，分为以下几步
1. 创建一个对象转换接口，使用@Mapper注解
2. 定义转换方法，设置需要转换的对象作为参数，返回值是转换后的对象
3. 使用@Mapping注解方法，设置转换对应的属性，如果属性名相同，则不需要设置。
4. 接口中定义一个属性，使用Mappers.getMapper方获取对应的实现，方便使用。
    通过上面4步，就可以定义出一个对象转换器。
# 获取Mapper的方式
```java
//方式一:直接使用INSTANCE
@Mapper
public abstract class CarMapper {
    public static final CarMapper INSTANCE = Mappers.getMapper(CarMapper.class);
    
    @Mapping(source = "numberOfSeats", target = "seatCount")
    CarDto carToCarDto(Car car);
}
//方式二:交给DI托管,但这种在Mockito的时候似乎有问题
@Mapper(componentModel = "spring")
public interface CarInjectMapper {
    @Mapping(source = "numberOfSeats", target = "seatCount")
    CarDto carToCarDto(Car car);
}
```
# Mapping常用属性
```java
@Repeatable(Mappings.class)
@Retention(RetentionPolicy.CLASS)
@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
public @interface Mapping {
    /**
    * 指的是目标对象或者出参对象的某字段
    **/
    String target();
    /**
    * 指的是入参对象或者入参对象的某字段
    **/
    String source() default "";
    /**
    * 用于进行日期格式化,将Date转换为String
    **/
    String dateFormat() default "";
    String numberFormat() default "";
    /**
    * 用于指定出参对象的某字段的值为某一个固定的值
    * eg. target="name",constant="哈哈",那么出参对象的name字段的值就是"哈哈"
    **/
    String constant() default "";
    /**
    * 返回值会被赋给出参对象的某字段，注意expression中的值最好采用全类名
    * eg. target="createDate",expression="java(new java.util.Date())",那么出参对象的createDate字段的值就是此时此刻
    **/
    String expression() default "";
    String defaultExpression() default "";
    /**
    * 用于指定入参对象的某个字段是否不赋值给出参对象,
    * eg. source="product.name",ignore="true",那么入参对象的product对象的name字段将不会赋值给出参对象
    **/
    boolean ignore() default false;
    /**
    * 返回值会被赋给出参对象的某字段
    **/
    Class<? extends Annotation>[] qualifiedBy() default {};
    /**
    * 返回值会被赋给出参对象的某字段
    **/
    String[] qualifiedByName() default {};
    Class<?> resultType() default void.class;
    String[] dependsOn() default {};
    /**
    *
    **/
    String defaultValue() default "";
    NullValueCheckStrategy nullValueCheckStrategy() default NullValueCheckStrategy.ON_IMPLICIT_CONVERSION;
    NullValuePropertyMappingStrategy nullValuePropertyMappingStrategy() default NullValuePropertyMappingStrategy.SET_TO_NULL;
    Class<? extends Annotation> mappingControl() default MappingControl.class;
}
```
# 常用场景
## 基本的转换
```java
@Data
public class Order{
    private Long id;
    private Long no;
    private Date createDate;
}
@Data
public class OrderRes {
    private Long id;
    private Long orderNo;
    private Date orderCreateDate;
}
@Mapper
public interface OrderMapper {
    OrderMapper INSTANCE = Mappers.getMapper(OrderMapper.class);
    @Mappings({
            @Mapping(source = "no", target = "orderNo")
    })
    OrderRes dbToRes(Order order);
}
@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2022-09-06T22:43:33+0800",
    comments = "version: 1.4.2.Final, compiler: javac, environment: Java 1.8.0_262 (AdoptOpenJDK)"
)
public class OrderMapperImpl implements OrderMapper {

    @Override
    public OrderRes dbToRes(Order order) {
        if ( order == null ) {
            return null;
        }

        OrderRes orderRes = new OrderRes();

        orderRes.setOrderNo( order.getNo() );
        orderRes.setId( order.getId() );

        return orderRes;
    }
}
```
  @Mapper注解作用是：在build-time时，MapStruct会自动生成一个实现PersonMapper接口的类。
接口中定义的方法，在自动生成时，默认会将source对象（比如PersonDTO）中所有可读的属性拷贝到target（比如Person）对象中相关的属性，转换规则主要有以下两条：
1. 当target和source对象中属性名相同，则直接转换,部分情况下可以自动转换，日期Date转String需要在@Mapping中使用dateFormat进行格式化，long转int会出现精度丢失，包装类型转换为基础数据类型会进行null判断，详细的转换规则见官方文档
2. 当target和source对象中属性名不同，名字的映射可以通过@Mapping注解来指定。比如上面no映射到orderNo属性上。
## 多个入参对象转换为一个出参对象
  MapStruct支持将多个入参对象的属性合并到一个出参对象中。例如我们需要将下单用户的信息和订单信息整合到一个订单详情对象中
```java
@Data
public class Merchant {
    private Long id;
    private String name;
    private Byte sex;
}
@Data
public class Order{
    private Long id;
    private Long no;
    private Date createDate;
    private Long recyclerId;
}
@Data
public class OrderRes {
    private Long id;
    private Long orderNo;
    private Date orderCreateDate;
    private Long recyclerId;
    private String recyclerName;
}
@Mapper
public interface OrderMapper {
    OrderMapper INSTANCE = Mappers.getMapper(OrderMapper.class);
    @Mappings({
            @Mapping(source = "order.no", target = "orderNo"),
            //这里注意:出参对象中有id字段,入参对象有两个id字段,因此需要显式指定映射关系
            //如果出参对象中不需要id,可以先任意指定一个id映射然后添加ignore=true
            @Mapping(source = "order.id", target = "id"),
            @Mapping(source = "mer.id", target = "recyclerId"),
            @Mapping(source = "mer.name", target = "recyclerName")
    })
    OrderRes dbToRes(Order order,Merchant mer);
}
@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2022-09-06T22:43:33+0800",
    comments = "version: 1.4.2.Final, compiler: javac, environment: Java 1.8.0_262 (AdoptOpenJDK)"
)
public class OrderMapperImpl implements OrderMapper {

    @Override
    public OrderRes dbToRes(Order order, Merchant mer) {
        if ( order == null && mer == null ) {
            return null;
        }

        OrderRes orderRes = new OrderRes();

        if ( order != null ) {
            orderRes.setOrderNo( order.getNo() );
            orderRes.setId( order.getId() );
        }
        if ( mer != null ) {
            orderRes.setRecyclerId( mer.getId() );
            orderRes.setRecyclerName( mer.getName() );
        }

        return orderRes;
    }
}
public class OrderMapperTest {
    @Test
    public void personDTOToPerson() {
        //准备入参
        OrderMapper orderMapper = OrderMapper.INSTANCE;
        Order order = new Order();
        order.setId(222L);
        order.setNo(1L);
        order.setCreateDate(new Date());
        Merchant merchant = new Merchant();
        merchant.setId(333L);
        merchant.setName("张三");
        //转换
        OrderRes orderRes = orderMapper.dbToRes(order,merchant);
        //判断出参
        Assert.assertEquals(order.getId(),orderRes.getId());
        Assert.assertEquals(order.getNo(),orderRes.getOrderNo());
        Assert.assertNull(orderRes.getOrderCreateDate());
        Assert.assertEquals(merchant.getId(),orderRes.getRecyclerId());
        Assert.assertEquals(merchant.getName(),orderRes.getRecyclerName());
    }
}
```
## 将方法的返回值映射到字段
  有的时候，某个字段的值无法通过简单的映射来解决，需要经过一定的逻辑，这种情况下MapStrut也是支持的
```java
@Data
public class Order{
    private Long id;
    private Long no;
}
@Data
public class OrderRes {
    private Long id;
    private Long orderNo;
    private String orderDay;
    private Integer status;
    private Date createDate;
}
public enum EnumOrderStatus {
    UN_PAYED(1, "待支付"),
    PAYED(2, "已支付");
    //...
}
public class BUtil {
    public static String extractFromNo(Long input){
        String inputString = input.toString();
        return inputString.length() >= 8 ? inputString.substring(0,8) : "20220101";
    }
}
@Mapper
public interface OrderMapper {
    OrderMapper INSTANCE = Mappers.getMapper(OrderMapper.class);
    @Mappings({
        @Mapping(source = "no", target = "orderNo"),
        @Mapping(target = "orderDay", expression = "java(com.study.mapstruct.sample.methodReturnToField.BUtil.extractFromNo(order.getNo()))"),
        @Mapping(target = "status", expression = "java(com.study.mapstruct.sample.methodReturnToField.EnumOrderStatus.PAYED.getStatus())"),
        @Mapping(target = "createDate", expression = "java(new java.util.Date())")
    })
    OrderRes dbToRes(Order order);
}
@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2022-09-06T22:43:33+0800",
    comments = "version: 1.4.2.Final, compiler: javac, environment: Java 1.8.0_262 (AdoptOpenJDK)"
)
public class OrderMapperImpl implements OrderMapper {

    @Override
    public OrderRes dbToRes(Order order) {
        if ( order == null ) {
            return null;
        }

        OrderRes orderRes = new OrderRes();

        orderRes.setOrderNo( order.getNo() );
        orderRes.setId( order.getId() );

        orderRes.setOrderDay( com.study.mapstruct.sample.methodReturnToField.BUtil.extractFromNo(order.getNo()) );
        orderRes.setStatus( com.study.mapstruct.sample.methodReturnToField.EnumOrderStatus.PAYED.getStatus() );
        orderRes.setCreateDate( new java.util.Date() );

        return orderRes;
    }
}
public class OrderMapperTest {
    @Test
    public void personDTOToPerson() {
        //准备入参
        OrderMapper orderMapper = OrderMapper.INSTANCE;
        Order order = new Order();
        order.setId(222L);
        order.setNo(1L);

        //转换
        OrderRes orderRes = orderMapper.dbToRes(order);
        //判断出参
        Assert.assertEquals(order.getId(),orderRes.getId());
        Assert.assertEquals(order.getNo(),orderRes.getOrderNo());
    }
}
```
  关于这种需求，也有一些其他方式可以做到，详见[执行自定义方法](https://mapstruct.org/documentation/stable/reference/html/#invoking-custom-mapping-method)、[向Mapper添加自定义方法](https://mapstruct.org/documentation/stable/reference/html/#adding-custom-methods)、[调用其他映射器](https://mapstruct.org/documentation/stable/reference/html/#selection-based-on-qualifiers)、[映射前后执行自定义方法](https://mapstruct.org/documentation/stable/reference/html/#passing-context)，
当重名方法很多时，可以通过@Qualifer、@Named等注解来指定唯一方法
## 映射集合字段
  MapStruct支持对集合的映射，当入参对象中的某个字段为集合时，会遍历该集合，将该字段拷贝到出参对象的字段中,Java中常用的List、Map、Set、Steam都是支持的，MapStruct对集合接口类的默认实现见[官方文档-6.3](https://mapstruct.org/documentation/stable/reference/html/#implementation-types-for-collection-mappings)。
```java
@Data
public class Order{
    private Long id;
    private Long no;
    private List<String> saleItemNameList;
    private Set<String> subNoSet;
}
@Data
public class OrderRes{
    private Long id;
    private Long orderNo;
    private List<String> saleItemNames;
    private Set<String> subNoSet;
}
@Mapper
public interface OrderMapper {
    OrderMapper INSTANCE = Mappers.getMapper(OrderMapper.class);
    @Mappings({
            @Mapping(source = "no", target = "orderNo"),
            @Mapping(source = "saleItemNameList", target = "saleItemNames")
    })
    OrderRes dbToRes(Order order);
}
@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2022-09-06T22:43:33+0800",
    comments = "version: 1.4.2.Final, compiler: javac, environment: Java 1.8.0_262 (AdoptOpenJDK)"
)
public class OrderMapperImpl implements OrderMapper {

    @Override
    public OrderRes dbToRes(Order order) {
        if ( order == null ) {
            return null;
        }

        OrderRes orderRes = new OrderRes();

        orderRes.setOrderNo( order.getNo() );
        List<String> list = order.getSaleItemNameList();
        if ( list != null ) {
            orderRes.setSaleItemNames( new ArrayList<String>( list ) );
        }
        orderRes.setId( order.getId() );
        Set<String> set = order.getSubNoSet();
        if ( set != null ) {
            orderRes.setSubNoSet( new HashSet<String>( set ) );
        }

        return orderRes;
    }
}
public class OrderMapperTest {
    @Test
    public void personDTOToPerson() {
        //准备入参
        OrderMapper orderMapper = OrderMapper.INSTANCE;
        Order order = new Order();
        order.setId(222L);
        order.setNo(1L);
        order.setSaleItemNameList(Arrays.asList("商品1","商品2","商品3"));
        order.setSubNoSet(new HashSet<>(Arrays.asList("sub001","sub002")));
        //转换
        OrderRes orderRes = orderMapper.dbToRes(order);
        //判断出参
        Assert.assertEquals(order.getId(),orderRes.getId());
        Assert.assertEquals(order.getNo(),orderRes.getOrderNo());
    }
}
```
  在Map映射方面，MapStruct提供的功能较为有限，需要通过@MapMapping注解来完成
```java
public @interface MapMapping {

    String keyDateFormat() default "";

    String valueDateFormat() default "";

    String keyNumberFormat() default "";

    String valueNumberFormat() default "";

    Class<? extends Annotation>[] keyQualifiedBy() default { };

    String[] keyQualifiedByName() default { };
    
    Class<? extends Annotation>[] valueQualifiedBy() default { };
    
    String[] valueQualifiedByName() default { };
    
    Class<?> keyTargetType() default void.class;
    
    Class<?> valueTargetType() default void.class;
    
    NullValueMappingStrategy nullValueMappingStrategy() default NullValueMappingStrategy.RETURN_NULL;
    
    Class<? extends Annotation> keyMappingControl() default MappingControl.class;
    
    Class<? extends Annotation> valueMappingControl() default MappingControl.class;
}

public interface SourceTargetMapper {
    @MapMapping(valueDateFormat = "dd.MM.yyyy")
    Map<String, String> longDateMapToStringStringMap(Map<Long, Date> source);
}
```
## 映射前后执行自定义方法
  有的时候，在MapsStruct进行字段简单映射的基础上，我们还想要在映射前或映射后执行一些自定义的方法，这种也是支持的，可以通过[在映射中传递上下文参数](https://mapstruct.org/documentation/stable/reference/html/#passing-context)，借助@AfterMapping来实现。例如我们想要在映射后额外计算一下订单中的商品数量和是否包含子订单。
```java
@Data
public class Order{
    private Long id;
    private Long no;
    private List<String> saleItemNameList;
    private Set<String> subNoSet;
}
@Data
public class OrderRes{
    private Long id;
    private Long orderNo;
    private List<String> saleItemNames;
    private Integer saleItemCount;
    private Set<String> subNoSet;
    private Boolean containsSubNo;
}
@Mapper
public interface OrderMapper {
    OrderMapper INSTANCE = Mappers.getMapper(OrderMapper.class);

    @Mappings({
            @Mapping(source = "no", target = "orderNo"),
            @Mapping(source = "saleItemNameList", target = "saleItemNames")
    })
    OrderRes dbToRes(Order order);

    @AfterMapping
    default void quickReqToSearchModelAfter(Order ord, @MappingTarget OrderRes des) {
        //设置商品数量
        if(CollectionUtils.isEmpty(ord.getSaleItemNameList())){
            des.setSaleItemCount(0);
        }else{
            des.setSaleItemCount(ord.getSaleItemNameList().size());
        }
        //设置是否包含子订单
        //既然已经进行过字段映射了，其实也可以使用出参对象中的字段了
        des.setContainsSubNo(!CollectionUtils.isEmpty(des.getSubNoSet()));
    }

}
@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2022-09-07T22:07:41+0800",
    comments = "version: 1.4.2.Final, compiler: javac, environment: Java 1.8.0_262 (AdoptOpenJDK)"
)
public class OrderMapperImpl implements OrderMapper {

    @Override
    public OrderRes dbToRes(Order order) {
        if ( order == null ) {
            return null;
        }

        OrderRes orderRes = new OrderRes();

        orderRes.setOrderNo( order.getNo() );
        List<String> list = order.getSaleItemNameList();
        if ( list != null ) {
            orderRes.setSaleItemNames( new ArrayList<String>( list ) );
        }
        orderRes.setId( order.getId() );
        Set<String> set = order.getSubNoSet();
        if ( set != null ) {
            orderRes.setSubNoSet( new HashSet<String>( set ) );
        }

        quickReqToSearchModelAfter( order, orderRes );
 
        return orderRes;
    }
}
public class OrderMapperTest {
    @Test
    public void personDTOToPerson() {
        //准备入参
        OrderMapper orderMapper = OrderMapper.INSTANCE;
        Order order = new Order();
        order.setId(222L);
        order.setNo(1L);
        order.setSaleItemNameList(Arrays.asList("商品1","商品2","商品3"));
        order.setSubNoSet(new HashSet<>(Arrays.asList("sub001","sub002")));
        //转换
        OrderRes orderRes = orderMapper.dbToRes(order);
        //判断出参
        System.out.println(JSON.toJSONString(orderRes));
        Assert.assertEquals(order.getId(),orderRes.getId());
        Assert.assertEquals(order.getNo(),orderRes.getOrderNo());
    }
}
```
## 映射枚举
不知道怎么搞，官网链接有[枚举A的值映射到枚举B](https://mapstruct.org/documentation/stable/reference/html/#_mapping_enum_to_enum_types)
## 映射嵌套对象/对象引用
  对于嵌套对象的映射也是支持的，在@Mapping的source中使用入参对象中嵌套对象的名称即可
```java
@Data
public class Order{
    private Long id;
    private Long no;
    private Long recyclerId;
    private Merchant me;
}
@Data
public class Merchant {
    private Long id;
    private String name;
    private Byte sex;
}
@Data
public class OrderRes {
    private Long id;
    private Long orderNo;
    private Date orderCreateDate;
    private Long recyclerId;
    private String recyclerName;
    private Merchant merchantInfo;
}
@Mapper
public interface OrderMapper {
    OrderMapper INSTANCE = Mappers.getMapper(OrderMapper.class);

    @Mappings({
            @Mapping(source = "no", target = "orderNo"),
            @Mapping(source = "me.id", target = "recyclerId"),
            @Mapping(source = "me.name", target = "recyclerName"),
            @Mapping(source = "me", target = "merchantInfo")
    })
    OrderRes dbToRes(Order order);
}
@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2022-09-07T22:33:29+0800",
    comments = "version: 1.4.2.Final, compiler: javac, environment: Java 1.8.0_262 (AdoptOpenJDK)"
)
public class OrderMapperImpl implements OrderMapper {

    @Override
    public OrderRes dbToRes(Order order) {
        if ( order == null ) {
            return null;
        }

        OrderRes orderRes = new OrderRes();

        orderRes.setOrderNo( order.getNo() );
        orderRes.setRecyclerId( orderMeId( order ) );
        orderRes.setRecyclerName( orderMeName( order ) );
        orderRes.setMerchantInfo( order.getMe() );
        orderRes.setId( order.getId() );

        return orderRes;
    }

    private Long orderMeId(Order order) {
        if ( order == null ) {
            return null;
        }
        Merchant me = order.getMe();
        if ( me == null ) {
            return null;
        }
        Long id = me.getId();
        if ( id == null ) {
            return null;
        }
        return id;
    }

    private String orderMeName(Order order) {
        if ( order == null ) {
            return null;
        }
        Merchant me = order.getMe();
        if ( me == null ) {
            return null;
        }
        String name = me.getName();
        if ( name == null ) {
            return null;
        }
        return name;
    }
}
public class OrderMapperTest {
    @Test
    public void personDTOToPerson() {
        //准备入参
        OrderMapper orderMapper = OrderMapper.INSTANCE;
        Order order = new Order();
        order.setId(222L);
        order.setNo(1L);
        order.setRecyclerId(333L);
        Merchant merchant = new Merchant();
        merchant.setId(333L);
        merchant.setName("张三");
        merchant.setSex(new Byte("1"));
        order.setMe(merchant);
        //转换
        OrderRes orderRes = orderMapper.dbToRes(order);
        //判断出参
        System.out.println(JSON.toJSONString(orderRes));
        Assert.assertEquals(order.getId(),orderRes.getId());
        Assert.assertEquals(order.getNo(),orderRes.getOrderNo());
        Assert.assertNull(orderRes.getOrderCreateDate());
        Assert.assertEquals(merchant.getId(),orderRes.getRecyclerId());
        Assert.assertEquals(merchant.getName(),orderRes.getRecyclerName());
    }
}
```
## 批量调用已有的映射方法
  有的时候我们已经写好了单个入参对象转换为出参对象的方法，但是还需要一个批量的入参对象转换为出参对象的功能，这种情况下，直接List<出参对象> methodName(List<入参对象>  xxList)即可实现。
## 转换支持builder构建的对象
如果builder存在，mapstruct会使用builder构建对象（默认使用set方法），父类不支持，就只会转换当前对象。
解决方法是
1.让父类支持builder ，例如使用lombok的时候加上superBuilder注解
2.用下面方法指定mapstruct不用builder
```java
@Mapper(builder = @Builder(disableBuilder = true))
public interface TaskSearchMapping {

    TaskSearchMapping INSTANCE = Mappers.getMapper(TaskSearchMapping.class);

    TaskSearchModel dto2Model(TaskSearchDto dto);

    TaskSearchModel h5DtoModel(H5TaskSearchDto dto);
}
```
# 其他使用场景
- [将Map对象转换为Object对象](https://mapstruct.org/documentation/stable/reference/html/#adding-custom-methods)
- [通过Builder映射](https://mapstruct.org/documentation/stable/reference/html/#mapping-with-builders)
- [通过构造器映射](https://mapstruct.org/documentation/stable/reference/html/#mapping-with-constructors)
- [逆向映射](https://mapstruct.org/documentation/stable/reference/html/#inverse-mappings)
- [有条件的映射](https://mapstruct.org/documentation/stable/reference/html/#conditional-mapping)
- [关于异常](https://mapstruct.org/documentation/stable/reference/html/#conditional-mapping)

# 映射规则
- 字段名相同的情况下，如果字段类型相同，无需代码显式声明就可以自动映射；字段类型不同时，大部分也可以自动转换
- 字段名不同的情况下，可以通过@Mapping注解来直接映射关系
- 简单的字段和集合类字段都是可以直接映射的
- 支持将方法的返回值映射到一个字段上，并且方式有很多种
- 一个字段或对象可以进行多次映射
- 多个入参对象转换为一个出参对象，或者一个入参对象中存在多个或多层的嵌套对象，在@Mapping注解中使使用"入参名.xxx"就可以进行映射，也可以使用"."代替很多目标字段
- 多个入参对象转换为一个出参对象时，默认情况下只有所有入参对象都为空时才不会进行转换(当然，这点可以修改配置来解决)
- 在映射前后也可以执行自定义方法
# 参考文档
- [官方文档-v1.5.2](https://mapstruct.org/documentation/stable/reference/html/#Preface)
- [MapStruct使用手册](https://mofan212.gitee.io/posts/MapStruct-User-Manual/)
- [MapStruct原理解析](https://blog.csdn.net/datastructure18/article/details/120208842)
