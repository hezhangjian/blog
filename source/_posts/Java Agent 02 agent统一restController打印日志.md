---
title: Java Agent 02 agent统一restController打印日志
link:
date: 2020-12-31 18:05:43
tags:
---

### Step1 加入SpringWeb的依赖

```xml
<!-- spring 依赖 -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>5.2.9.RELEASE</version>
            <scope>provided</scope>
        </dependency>
```

### Step2 书写一个RestController

```java
package com.github.hezhangjian.demo.agent.test.controller;

import com.github.hezhangjian.demo.agent.test.module.rest.CreateJobReqDto;
import com.github.hezhangjian.demo.agent.test.module.rest.CreateJobRespDto;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

/**
 * @author hezhangjian
 */
@Slf4j
@RestController
public class TestController {

    /**
     * @param jobId
     * @param createJobReqDto
     * https://docs.aws.amazon.com/iot/latest/apireference/API_CreateJob.html
     * curl -X PUT -H 'Content-Type: application/json' 127.0.0.1:8080/jobs/111 -d '{"description":"description"}' -iv
     * @return
     */
    @PutMapping(path = "/jobs/{jobId}")
    public ResponseEntity<CreateJobRespDto> createJob(@PathVariable("jobId") String jobId, @RequestBody CreateJobReqDto createJobReqDto) {
        final CreateJobRespDto jobRespDto = new CreateJobRespDto();
        createJobReqDto.setDescription("description");
        createJobReqDto.setDocument("document");
        return new ResponseEntity<>(jobRespDto, HttpStatus.CREATED);
    }

}

```

ReqDto:

> ```java
> package com.github.hezhangjian.demo.agent.test.module.rest;
> 
> import lombok.Data;
> 
> /**
>  * @author hezhangjian
>  */
> @Data
> public class CreateJobReqDto {
> 
>     private String description;
> 
>     private String document;
> 
>     public CreateJobReqDto() {
>     }
> 
> }
> 
> ```
>
> 

RespDto:

```java
package com.github.hezhangjian.demo.agent.test.module.rest;

import lombok.Data;

/**
 * https://docs.aws.amazon.com/iot/latest/apireference/API_CreateJob.html
 * @author hezhangjian
 */
@Data
public class CreateJobRespDto {

    private String description;

    private String jobArn;

    private String jobId;

    public CreateJobRespDto() {
    }
}

```

### Step3 在AgentTransformer中织入切面的逻辑

```java
package com.github.hezhangjian.demo.agent;

import com.github.hezhangjian.demo.agent.interceptor.RestControllerInterceptor;
import net.bytebuddy.agent.builder.AgentBuilder;
import net.bytebuddy.asm.Advice;
import net.bytebuddy.description.type.TypeDescription;
import net.bytebuddy.dynamic.DynamicType;
import net.bytebuddy.matcher.ElementMatchers;
import net.bytebuddy.utility.JavaModule;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PatchMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestMapping;

/**
 * @author hezhangjian
 */
public class AgentTransformer implements AgentBuilder.Transformer {

    private static final Logger log = LoggerFactory.getLogger(AgentTransformer.class);

    @Override
    public DynamicType.Builder<?> transform(DynamicType.Builder<?> builder, TypeDescription typeDescription, ClassLoader classLoader, JavaModule javaModule) {
        try {
            //包名放在com.github.hezhangjian.demo.agent.test.controller下视为controller代码
            if (typeDescription.getTypeName().startsWith("com.github.hezhangjian.demo.agent.test.controller")) {
                final Advice advice = Advice.to(RestControllerInterceptor.class);
                return builder.visit(advice
                        .on(ElementMatchers.isAnnotatedWith(RequestMapping.class)
                                .or(ElementMatchers.isAnnotatedWith(GetMapping.class))
                                .or(ElementMatchers.isAnnotatedWith(PostMapping.class))
                                .or(ElementMatchers.isAnnotatedWith(PutMapping.class))
                                .or(ElementMatchers.isAnnotatedWith(DeleteMapping.class))
                                .or(ElementMatchers.isAnnotatedWith(PatchMapping.class))));
            }
        } catch (Exception e) {
            log.error("error is ", e);
        }
        return builder;
    }

}
```
### Step4 先添加一个打印日志工具类

Interceptor中不能出现和controller一样的字段，我们先写一个工具类用来agent打印日志

```java
package com.github.hezhangjian.demo.agent.util;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.Marker;

/**
 * @author hezhangjian
 */
public class AgentUtil {

    private static final Logger log = LoggerFactory.getLogger(AgentUtil.class);

    /**
     * Log a message at the TRACE level.
     *
     * @param msg the message string to be logged
     */
    public static void trace(String msg) {
        log.trace(msg);
    }

    /**
     * Log a message at the TRACE level according to the specified format
     * and argument.
     * <p/>
     * <p>This form avoids superfluous object creation when the logger
     * is disabled for the TRACE level. </p>
     *
     * @param format the format string
     * @param arg    the argument
     */
    public static void trace(String format, Object arg) {
        log.trace(format, arg);
    }

    /**
     * Log a message at the TRACE level according to the specified format
     * and arguments.
     * <p/>
     * <p>This form avoids superfluous object creation when the logger
     * is disabled for the TRACE level. </p>
     *
     * @param format the format string
     * @param arg1   the first argument
     * @param arg2   the second argument
     */
    public static void trace(String format, Object arg1, Object arg2) {
        log.trace(format, arg1, arg2);
    }

    /**
     * Log a message at the TRACE level according to the specified format
     * and arguments.
     * <p/>
     * <p>This form avoids superfluous string concatenation when the logger
     * is disabled for the TRACE level. However, this variant incurs the hidden
     * (and relatively small) cost of creating an <code>Object[]</code> before invoking the method,
     * even if this logger is disabled for TRACE. The variants taking {@link #trace(String, Object) one} and
     * {@link #trace(String, Object, Object) two} arguments exist solely in order to avoid this hidden cost.</p>
     *
     * @param format    the format string
     * @param arguments a list of 3 or more arguments
     */
    public static void trace(String format, Object... arguments) {
        log.trace(format, arguments);
    }

    /**
     * Log an exception (throwable) at the TRACE level with an
     * accompanying message.
     *
     * @param msg the message accompanying the exception
     * @param t   the exception (throwable) to log
     */
    public static void trace(String msg, Throwable t) {
        log.trace(msg, t);
    }

    /**
     * Log a message at the DEBUG level.
     *
     * @param msg the message string to be logged
     */
    public static void debug(String msg) {
    }

    /**
     * Log a message at the DEBUG level according to the specified format
     * and argument.
     * <p/>
     * <p>This form avoids superfluous object creation when the logger
     * is disabled for the DEBUG level. </p>
     *
     * @param format the format string
     * @param arg    the argument
     */
    public static void debug(String format, Object arg) {
    }

    /**
     * Log a message at the DEBUG level according to the specified format
     * and arguments.
     * <p/>
     * <p>This form avoids superfluous object creation when the logger
     * is disabled for the DEBUG level. </p>
     *
     * @param format the format string
     * @param arg1   the first argument
     * @param arg2   the second argument
     */
    public static void debug(String format, Object arg1, Object arg2) {
    }

    /**
     * Log a message at the DEBUG level according to the specified format
     * and arguments.
     * <p/>
     * <p>This form avoids superfluous string concatenation when the logger
     * is disabled for the DEBUG level. However, this variant incurs the hidden
     * (and relatively small) cost of creating an <code>Object[]</code> before invoking the method,
     * even if this logger is disabled for DEBUG. The variants taking
     * {@link #debug(String, Object) one} and {@link #debug(String, Object, Object) two}
     * arguments exist solely in order to avoid this hidden cost.</p>
     *
     * @param format    the format string
     * @param arguments a list of 3 or more arguments
     */
    public static void debug(String format, Object... arguments) {
    }

    /**
     * Log an exception (throwable) at the DEBUG level with an
     * accompanying message.
     *
     * @param msg the message accompanying the exception
     * @param t   the exception (throwable) to log
     */
    public static void debug(String msg, Throwable t) {
    }

    /**
     * Log a message at the INFO level.
     *
     * @param msg the message string to be logged
     */
    public static void info(String msg) {
    }

    /**
     * Log a message at the INFO level according to the specified format
     * and argument.
     * <p/>
     * <p>This form avoids superfluous object creation when the logger
     * is disabled for the INFO level. </p>
     *
     * @param format the format string
     * @param arg    the argument
     */
    public static void info(String format, Object arg) {
    }

    /**
     * Log a message at the INFO level according to the specified format
     * and arguments.
     * <p/>
     * <p>This form avoids superfluous object creation when the logger
     * is disabled for the INFO level. </p>
     *
     * @param format the format string
     * @param arg1   the first argument
     * @param arg2   the second argument
     */
    public static void info(String format, Object arg1, Object arg2) {
    }

    /**
     * Log a message at the INFO level according to the specified format
     * and arguments.
     * <p/>
     * <p>This form avoids superfluous string concatenation when the logger
     * is disabled for the INFO level. However, this variant incurs the hidden
     * (and relatively small) cost of creating an <code>Object[]</code> before invoking the method,
     * even if this logger is disabled for INFO. The variants taking
     * {@link #info(String, Object) one} and {@link #info(String, Object, Object) two}
     * arguments exist solely in order to avoid this hidden cost.</p>
     *
     * @param format    the format string
     * @param arguments a list of 3 or more arguments
     */
    public static void info(String format, Object... arguments) {
    }

    /**
     * Log an exception (throwable) at the INFO level with an
     * accompanying message.
     *
     * @param msg the message accompanying the exception
     * @param t   the exception (throwable) to log
     */
    public static void info(String msg, Throwable t) {
    }
    
    /**
     * Log a message at the WARN level.
     *
     * @param msg the message string to be logged
     */
    public static void warn(String msg) {
    }

    /**
     * Log a message at the WARN level according to the specified format
     * and argument.
     * <p/>
     * <p>This form avoids superfluous object creation when the logger
     * is disabled for the WARN level. </p>
     *
     * @param format the format string
     * @param arg    the argument
     */
    public static void warn(String format, Object arg) {
    }

    /**
     * Log a message at the WARN level according to the specified format
     * and arguments.
     * <p/>
     * <p>This form avoids superfluous string concatenation when the logger
     * is disabled for the WARN level. However, this variant incurs the hidden
     * (and relatively small) cost of creating an <code>Object[]</code> before invoking the method,
     * even if this logger is disabled for WARN. The variants taking
     * {@link #warn(String, Object) one} and {@link #warn(String, Object, Object) two}
     * arguments exist solely in order to avoid this hidden cost.</p>
     *
     * @param format    the format string
     * @param arguments a list of 3 or more arguments
     */
    public static void warn(String format, Object... arguments) {
    }

    /**
     * Log a message at the WARN level according to the specified format
     * and arguments.
     * <p/>
     * <p>This form avoids superfluous object creation when the logger
     * is disabled for the WARN level. </p>
     *
     * @param format the format string
     * @param arg1   the first argument
     * @param arg2   the second argument
     */
    public static void warn(String format, Object arg1, Object arg2) {
    }

    /**
     * Log an exception (throwable) at the WARN level with an
     * accompanying message.
     *
     * @param msg the message accompanying the exception
     * @param t   the exception (throwable) to log
     */
    public static void warn(String msg, Throwable t) {
    }

    /**
     * Log a message at the ERROR level.
     *
     * @param msg the message string to be logged
     */
    public static void error(String msg) {
    }

    /**
     * Log a message at the ERROR level according to the specified format
     * and argument.
     * <p/>
     * <p>This form avoids superfluous object creation when the logger
     * is disabled for the ERROR level. </p>
     *
     * @param format the format string
     * @param arg    the argument
     */
    public static void error(String format, Object arg) {
    }

    /**
     * Log a message at the ERROR level according to the specified format
     * and arguments.
     * <p/>
     * <p>This form avoids superfluous object creation when the logger
     * is disabled for the ERROR level. </p>
     *
     * @param format the format string
     * @param arg1   the first argument
     * @param arg2   the second argument
     */
    public static void error(String format, Object arg1, Object arg2) {
    }

    /**
     * Log a message at the ERROR level according to the specified format
     * and arguments.
     * <p/>
     * <p>This form avoids superfluous string concatenation when the logger
     * is disabled for the ERROR level. However, this variant incurs the hidden
     * (and relatively small) cost of creating an <code>Object[]</code> before invoking the method,
     * even if this logger is disabled for ERROR. The variants taking
     * {@link #error(String, Object) one} and {@link #error(String, Object, Object) two}
     * arguments exist solely in order to avoid this hidden cost.</p>
     *
     * @param format    the format string
     * @param arguments a list of 3 or more arguments
     */
    public static void error(String format, Object... arguments) {
    }

    /**
     * Log an exception (throwable) at the ERROR level with an
     * accompanying message.
     *
     * @param msg the message accompanying the exception
     * @param t   the exception (throwable) to log
     */
    public static void error(String msg, Throwable t) {
    }

}

```

### Step5 书写Interceptor切入方法

```java
package com.github.hezhangjian.demo.agent.interceptor;

import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.github.hezhangjian.demo.agent.util.AgentJacksonUtil;
import com.github.hezhangjian.demo.agent.util.AgentUtil;
import net.bytebuddy.asm.Advice;
import org.springframework.http.ResponseEntity;

import java.lang.reflect.Method;

/**
 * @author hezhangjian
 */
public class RestControllerInterceptor {

    private static final ThreadLocal<Long> costThreadLocal = new ThreadLocal<>();

    /**
     * #t Class名 ex: com.github.hezhangjian.demo.agent.test.controller.TestController
     * #m Method名 ex: createJob
     * #d Method描述 ex: (Ljava/lang/String;Lcom/github/hezhangjian/demo/agent/test/module/rest/CreateJobReqDto;)Lorg/springframework/http/ResponseEntity;
     * #s 方法签名 ex: (java.lang.String,com.github.hezhangjian.demo.agent.test.module.rest.CreateJobReqDto)
     * #r 返回类型 ex: org.springframework.http.ResponseEntity
     *
     * @param signature
     */
    @Advice.OnMethodEnter
    public static void enter(@Advice.Origin("#t #m") String signature) {
        AgentUtil.info("[{}]", signature);
    }

    /**
     * @param method 方法名
     * @param args
     * @param result
     * @param thrown
     */
    @SuppressWarnings("rawtypes")
    @Advice.OnMethodExit(onThrowable = Throwable.class)
    public static void exit(@Advice.Origin Method method, @Advice.AllArguments Object[] args,
                            @Advice.Return Object result, @Advice.Thrown Throwable thrown) {
        AgentUtil.debug("method is [{}]", method);
        final ArrayNode arrayNode = AgentJacksonUtil.createArrayNode();
        for (Object arg : args) {
            arrayNode.add(AgentJacksonUtil.toJson(arg));
        }
        final ObjectNode objectNode = AgentJacksonUtil.createObjectNode();
        objectNode.set("args", arrayNode);
        ResponseEntity responseEntity = (ResponseEntity) result;
        AgentUtil.info("status code is [{}] args is [{}] result is [{}]", responseEntity.getStatusCode(), objectNode, responseEntity.getBody());
    }


}
```

curl命令调用查看效果

```
2020-12-31,17:52:01,528+08:00(6121):INFO{}[http-nio-8080-exec-1#39]com.github.hezhangjian.demo.agent.util.AgentUtil.info(AgentUtil.java:167)-->[com.github.hezhangjian.demo.agent.test.controller.TestController createJob]
2020-12-31,17:52:01,544+08:00(6137):INFO{}[http-nio-8080-exec-1#39]com.github.hezhangjian.demo.agent.util.AgentUtil.info(AgentUtil.java:200)-->status code is [201 CREATED] args is [{"args":["\"111\"","{\"description\":\"description\",\"document\":\"document\"}"]}] result is [CreateJobRespDto(description=null, jobArn=null, jobId=null)]
```
