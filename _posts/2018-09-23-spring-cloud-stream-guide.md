---
layout: post
title: Spring Cloud Stream的基本用法
description: 以RabbitMQ为例，介绍Spring Cloud Stream的基本用法。
category: blog
---

## 基本概念

- Spring Cloud Stream本质上就是整合了Spring Boot和Spring Integration，实现了一套轻量级的消息驱动的微服务框架。目前只支持RabbitMQ、Kafka的自动化配置。
- Spring Cloud Stream构建的应用程序与消息中间件之间是通过绑定器`Binder`相关联的，绑定器对于应用程序而言起到了隔离作用，应用程序不需要知晓消息中间件的通信细节，只需要知道Binder给应用程序提供的概念（Channel）去实现即可。

### Binder

- 通过定义绑定器作为中间层，完美地实现了应用程序与消息中间件细节之间的隔离。通过向应用程序暴露统一的 `Channel`通道，使得应用程序不需要再考虑各种不同的消息中间件实现。当我们需要升级消息中间件，或是更换其他消息中间件产品时，我们要做的就是更换它们对应的 `Binder`绑定器而不需要修改任何Spring Boot的应用逻辑。



## 使用案例：基于RabbitMQ

### 1. Maven Dependency

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
    <version>1.3.0.RELEASE</version>
</dependency>
```

- 该依赖包是Spring Cloud Stream对RabbitMQ支持的封装，其中包含了对RabbitMQ的自动化配置等内容。

### 2. MQ Configuration

```properties
# output(exchange) setting: automatically created when running the application
# exchange name: log_parsed
spring.cloud.stream.bindings.log_parsed_output.content-type=application/json
spring.cloud.stream.bindings.log_parsed_output.destination=log_parsed

# input(queue) setting
# queue name: log_rendered.consumer_a
spring.cloud.stream.bindings.log_rendered_input.content-type=application/json
spring.cloud.stream.bindings.log_rendered_input.destination=log_rendered
spring.cloud.stream.bindings.log_rendered_input.group=consumer_a
```

### 2. Channel Interface

```java
import org.springframework.cloud.stream.annotation.Input;
import org.springframework.cloud.stream.annotation.Output;
import org.springframework.messaging.MessageChannel;

public interface MqChannel {

    // Value is the binder name in configuration file
    String LOG_PARSED_OUTPUT = "log_parsed_output";
    String LOG_RENDERED_INPUT = "log_rendered_input";

    // bind a output channel named "log_parsed_output"
    @Output(LOG_PARSED_OUTPUT)
    MessageChannel output();

    @Input(LOG_RENDERED_INPUT)
    MessageChannel input();
}
```

### 3. Producer/Consumer

```java
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;

import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.annotation.Output;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;
import java.util.HashMap;

@Component
@EnableBinding(MqChannel.class)
public class MqHelper {

    @Resource
    @Output(MqChannel.LOG_PARSED_OUTPUT)
    private MessageChannel outputChannel;

    public void notifyLogParsed(String id, LogType type) {

        // assemble message contents
        Map<String, Object> payloadMap = new HashMap<>(3);
        payloadMap.put("id", id);
        payloadMap.put("type", type);

        String json;
        try {
            json = new ObjectMapper().writeValueAsString(payloadMap);
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }

        // send message
        outputChannel.send(MessageBuilder.withPayload(json).build());
    }

    /**
     * Method 1: use POJO to receive message 
     */
    @StreamListener(MqChannel.LOG_RENDERED_INPUT)
    public void listenLogRenderResult1(PdfRenderRes res) {
        // process result
    }
    
    /**
     * Method 2: use byte array to receive message 
     */
    @StreamListener(MqChannel.LOG_RENDERED_INPUT)
    public void listenPdfRenderResult2(byte[] payload) throws IOException {
        PdfRenderRes msg = new ObjectMapper().readValue(payload, PdfRenderRes.class);
        // process result
    }
}
```

- `@EnableBinding`：该注解用来指定一个或多个定义了`@Input`或 `@Output`注解的接口，以此实现对消息通道（Channel）的绑定。

- `@StreamListener`：该注解主要定义在方法上，作用是将被修饰的方法注册为消息中间件上数据流的事件监听器，注解中的属性值对应了监听的消息Channel名称。