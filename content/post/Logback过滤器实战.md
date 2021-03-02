---
title: "Logback过滤器实战"
date: 2020-12-29T15:45:02+08:00
draft: true
categories:
  - 实战
  - 组件
  - 日志组件
  - Logback
tags:
  - 实战
  - 组件
  - 日志组件
  - Logback
---



# 过滤器介绍

logback 过滤器基于三元逻辑，允许它们组装或者链接在一起组成一个任意复杂的过滤策略，它们在很大程度上受到 Linux iptables 的启发。

## 过滤器如何生效

在过滤器链中，发生日志事件时，每个过滤器的decide(ILoggingEvent event)方法依次被调用，该方法返回[`FilterReply`](https://logback.qos.ch/xref/ch/qos/logback/core/spi/FilterReply.html) 枚举值中的一个， `DENY`, `NEUTRAL` 或者 `ACCEPT`：返回deny时，该日志事件会被丢弃；返回NEUTRAL时，会交给下一个过滤器处理，如果当前过滤器是最后一个过滤器，会正常处理该日志事件；返回ACCEPT时，会正常处理该日志事件，不再经过后续的过滤器。

## 过滤器的分类

在 logback-classic 中，有两种类型的过滤器，regular 过滤器以及 turbo 过滤器.

* Regular 过滤器继承自 [`Filter`](https://logback.qos.ch/xref/ch/qos/logback/core/filter/Filter.html) 抽象类；当Appender被调用时，Regular过滤器也一起被调用；本质上它由一个单一的 `decide()` 方法组成，接收一个 `ILoggingEvent` 实例作为参数。常见的Regular过滤器的子类有以下几种：
  * LevelFilter ： 级别过滤器，如果日志事件的级别与配置的级别相等，过滤器会根据配置的 `onMatch` 与 `onMismatch` 属性，接受或者拒绝事件
  * ThresholdFilter : 临界值过滤器，过滤掉低于指定临界值的日志。当日志级别等于或高于临界值时，过滤器返回NEUTRAL；当日志级别低于临界值时，日志会被拒绝
  * EvaluatorFilter ： 求值过滤器，根据EventEvaluator的表达式执行后的返回值进行过滤。默认的EventEvaluator为JaninoEventEvaluator，可以配置一段最终返回布尔值的java代码，但是需要额外引入[Janino 类库](http://docs.codehaus.org/display/JANINO/Home)依赖，也存在使用groovy脚本的GEventEvaluator，可以配置一段最终返回布尔值的groovy代码，但是需要额外引入groovy依赖

* TurboFIlter过滤器继承自 [`TurboFilter`](https://logback.qos.ch/xref/ch/qos/logback/classic/turbo/TurboFilter.html) 抽象类；它与日志上下文绑定，每发生一次日志事件时Turbo过滤器都会被调用；它会过滤所有的日志请求，并且TurboFIlter的方法中提供了丰富的可访问信息用来进行控制和改写，且性能高于RegularFilter（TurboFilter在作用时机是在创建ILoggingEvent之前，RegularFilter的作用时机是在创建ILoggingEvent之后），常见的TurboFilter过滤器的子类有以下几种：
  * MDCFilter ： 若MDC域中存在指定的key-value，则进行记录，否者不作记录
  * MarkerFilter ： 针对带有指定标记的日志，进行记录或者不记录
  * DynamicThresholdFilter ： 动态版的ThresholdFilter，根据MDC域中是否存在某个键，该键对应的值是否相等，可实现日志级别动态切换
  * DuplicateMessageFilter ： 根据配置的日志重复次数，实现不记录多余的重复的日志，内部维护了一个格式化之前的日志内容的缓存

# 过滤器实战

## 对loggerName支持正则表达式

### 思路一 ： 基于求值过滤器JaninoEventEvaluator

引入依赖

```xml
　<dependency>
   <groupId>org.codehaus.janino</groupId>
   <artifactId>janino</artifactId>
   <version>2.5.16</version>
 </dependency>
```

修改logback.xml

```xml
    <!-- INFO日志 -->
    <appender name="logCenter" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <File> /opt/logs/data_collector/metrics_acs_settlement_log/mt_logCenter.log</File>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>/opt/logs/data_collector/metrics_acs_settlement_log/mt_logCenter_%d{yyyy-MM-dd}.log</fileNamePattern>
        </rollingPolicy>
        <encoder>
            <pattern>xxxxx</pattern>
            <charset>UTF-8</charset>
        </encoder>
        <filter class="ch.qos.logback.core.filter.EvaluatorFilter">        
          <evaluator> 
            <matcher>   
              <Name>odd</Name>   
              <regex>.*XMDT.*?(?=\b)</regex>   
            </matcher>  
            <expression>odd.matches(formattedMessage)</expression>  
          </evaluator>
          <OnMatch>ACCEPT</OnMatch>  
            <OnMismatch>DENY</OnMismatch>
        </filter> 
    </appender>
    <!-- 根配置 -->
    <root level="INFO">
        <appender-ref ref="logCenter" />
    </root>
```

ps ：该思路来自[博客园](https://www.cnblogs.com/tianzhiyun/p/6495511.html)

### 思路二 ： 基于自定义Filter

自定义Filter

```java
package com.champbay.core.log;

import java.util.regex.Matcher;
import java.util.regex.Pattern;

import org.apache.commons.lang3.StringUtils;

import ch.qos.logback.classic.Level;
import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.core.filter.Filter;
import ch.qos.logback.core.spi.FilterReply;

public class MyLogbackFilter extends Filter<ILoggingEvent> {

    private Pattern pattern;

    private String logPattern;

    public String getLogPattern() {
        return logPattern;
    }

    public void setLogPattern(String logPattern) {
        this.logPattern = logPattern;

        pattern = Pattern.compile(logPattern);
    }

    //对loggerName进行正则表达式的过滤，loggerName来自于ILoggingEvent暴露的属性
    @Override
    public FilterReply decide(ILoggingEvent event) {
        if(pattern == null)
            return FilterReply.DENY;

        String loggerName = event.getLoggerName();
        if(StringUtils.isBlank(loggerName)) {
            return FilterReply.DENY;
        }

        Matcher matcher = pattern.matcher(loggerName);
        if(matcher.matches()) {
            Level level = event.getLevel();
            if(level.isGreaterOrEqual(Level.DEBUG)) {
                return FilterReply.ACCEPT;
            } else {
                return FilterReply.DENY;
            }
        } else {
            return FilterReply.DENY;
        }
    }
}
```

修改logback.xml

```xml
 <appender name="myAppender" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <filter class="com.champbay.core.log.MyLogbackFilter">
          <!-- 自定义logPattern属性，接收作用于loggerName的正则表达式 -->
            <logPattern>com.champbay\..*\.mapper\..*</logPattern>
        </filter>
        <file>aaa.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>aaa-%d{yyyy-MM-dd}.log</fileNamePattern>
        </rollingPolicy>
        <encoder>
            <pattern>xxxxxxxxxxxxxxxxxx</pattern>
        </encoder>
    </appender>

    <logger name="com.champbay" level="DEBUG" additivity="false">
        <appender-ref ref="myAppender" />
    </logger>
```

Ps : 该思路来自于[全贝博客](http://blog.champbay.com/2019/11/13/%E5%9C%A8logback%E4%B8%AD%E5%A2%9E%E5%8A%A0%E6%AD%A3%E5%88%99%E8%A1%A8%E8%BE%BE%E5%BC%8F%E6%94%AF%E6%8C%81/)

## 动态修改日志级别



