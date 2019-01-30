---
layout: post
title: Spring Cloud Stream的基本用法
description: 以RabbitMQ为例，介绍Spring Cloud Stream的基本用法。
category: blog
---

## 使用案例：基于RabbitMQ

### 1. Maven Dependency

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
    <version>1.3.0.RELEASE</version>
</dependency>
```

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
        HashMap<String, Object> payloadMap = new HashMap<String, Object>() {{
            put("id", id);
            put("type", type);
        }};

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

