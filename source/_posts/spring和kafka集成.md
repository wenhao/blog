title: spring和kafka集成
date: 2015-10-02 10:27:35
categories: kafka
tags: [kafka]
toc: true
---

![kafka](/img/kafka-logo.png)

###创建项目

* 创建项目文件夹spring-kafka
* 生成项目结构`gradle init --type java-library`

<!-- more -->

###添加依赖build.gradle

```groovy
buildscript {
    repositories {
        mavenCentral()
        maven { url 'https://repo.spring.io/simple/ext-release-local' }
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:1.2.6.RELEASE")
        classpath 'org.apache.maven:maven-artifact:2.2.1'
        classpath 'org.apache.avro:avro-compiler:1.7.3'
        classpath "org.apache.avro:avro-gradle-plugin:1.7.2"
    }
}

apply plugin: 'java'
apply plugin: 'idea'
apply plugin: 'maven'
apply plugin: 'spring-boot'
apply plugin: 'avro-gradle-plugin'

repositories {
    mavenCentral()
    jcenter()
}

defaultTasks 'clean', 'build'

ext {
    avroTaskGroup = "Avro"
    avroSource = "schemas"
    avroDest = "target/generated-avro-sources/main/java"
}

dependencies {
    compile 'org.springframework.boot:spring-boot-starter-web'
    compile('org.springframework.integration:spring-integration-kafka:1.2.1.RELEASE') {
        exclude module: 'log4j'
        exclude module: 'jms'
        exclude module: 'jmxtools'
        exclude module: 'jmxri'
    }
    compile("log4j:log4j:1.2.15") {
        exclude module: 'mail'
        exclude module: 'jms'
        exclude module: 'jmx'
        exclude module: 'jmxtools'
        exclude module: 'jmxri'
    }
    compile "commons-logging:commons-logging:1.1.1"
    compile('org.apache.avro:avro:1.7.7') {
        exclude module: 'slf4j-log4j12'
    }
}

compileAvro.group = avroTaskGroup
compileAvro.description = "Generates Java code from avro schema"
compileAvro.source = avroSource
compileAvro.destinationDir = file(avroDest)

task cleanAvro(type: Delete) {
    group = avroTaskGroup
    description = "deletes generated avro code"
    delete avroDest
}

compileJava.dependsOn compileAvro

sourceSets {
    main {
        java {
            srcDir avroDest
        }
        resources {
            srcDir avroSource
        }
    }
}

mainClassName = 'org.github.wenhao.kafka.Application'

```

###spring-boot启动类

```java
package org.github.wenhao.kafka;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

```

###使用Avro序列化，创建schema

在项目根目录schemas文件夹下创建**user.avdl**, [Apache AVDL文档](http://avro.apache.org/docs/current/idl.html)

```java
@namespace("org.github.wenhao.kafka.avro")
protocol UserAvroProtocol{
    record UserAvro {
        string name;
        int age;
    }
 }
```

###配置kafka-outbound

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:int="http://www.springframework.org/schema/integration"
       xmlns:int-kafka="http://www.springframework.org/schema/integration/kafka"
       xmlns:task="http://www.springframework.org/schema/task"
       xsi:schemaLocation="http://www.springframework.org/schema/integration/kafka http://www.springframework.org/schema/integration/kafka/spring-integration-kafka.xsd
		http://www.springframework.org/schema/integration http://www.springframework.org/schema/integration/spring-integration.xsd
		http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/task http://www.springframework.org/schema/task/spring-task.xsd">


    <int:channel id="inputToKafka">
        <int:queue/>
    </int:channel>

    <int:poller default="true" id="default" fixed-rate="3" time-unit="MILLISECONDS" />

    <int-kafka:outbound-channel-adapter id="kafkaOutboundChannelAdapter"
                                        auto-startup="true"
                                        kafka-producer-context-ref="kafkaProducerContext"
                                        channel="inputToKafka"
                                        topic="test">
    </int-kafka:outbound-channel-adapter>

    <task:executor id="taskExecutor" pool-size="5" keep-alive="120" queue-capacity="500"/>

    <bean id="producerProperties" class="org.springframework.beans.factory.config.PropertiesFactoryBean">
        <property name="properties">
            <props>
                <prop key="topic.metadata.refresh.interval.ms">3600000</prop>
                <prop key="message.send.max.retries">5</prop>
                <prop key="send.buffer.bytes">5242880</prop>
            </props>
        </property>
    </bean>

    <bean id="kafkaReflectionEncoder" class="org.springframework.integration.kafka.serializer.avro.AvroReflectDatumBackedKafkaEncoder">
        <constructor-arg type="java.lang.Class" value="java.lang.String"/>
    </bean>

    <bean id="kafkaSpecificEncoder" class="org.springframework.integration.kafka.serializer.avro.AvroSpecificDatumBackedKafkaEncoder">
        <constructor-arg value="org.github.wenhao.kafka.avro.UserAvro" />
    </bean>

    <int-kafka:producer-context id="kafkaProducerContext" producer-properties="producerProperties">
        <int-kafka:producer-configurations>
            <int-kafka:producer-configuration broker-list="localhost:9092"
                                              key-class-type="java.lang.String"
                                              value-class-type="org.github.wenhao.kafka.avro.UserAvro"
                                              topic="test"
                                              key-encoder="kafkaReflectionEncoder"
                                              value-encoder="kafkaSpecificEncoder"/>
        </int-kafka:producer-configurations>
    </int-kafka:producer-context>
</beans>
```

###配置kafka-inbound

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:int="http://www.springframework.org/schema/integration"
       xmlns:int-kafka="http://www.springframework.org/schema/integration/kafka"
       xsi:schemaLocation=" http://www.springframework.org/schema/integration/kafka http://www.springframework.org/schema/integration/kafka/spring-integration-kafka.xsd
		http://www.springframework.org/schema/integration http://www.springframework.org/schema/integration/spring-integration.xsd
		http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <int:channel id="inputFromKafka">
        <int:queue/>
    </int:channel>

    <bean id="kafkaReflectionDecoder" class="org.springframework.integration.kafka.serializer.avro.AvroReflectDatumBackedKafkaDecoder">
        <constructor-arg type="java.lang.Class" value="java.lang.String"/>
    </bean>

    <bean id="kafkaSpecificDecoder" class="org.springframework.integration.kafka.serializer.avro.AvroSpecificDatumBackedKafkaDecoder">
        <constructor-arg value="org.github.wenhao.kafka.avro.UserAvro"/>
    </bean>

    <int-kafka:zookeeper-connect id="zookeeperConnect" zk-connect="localhost:2181" zk-connection-timeout="6000"
                                 zk-session-timeout="6000"
                                 zk-sync-time="2000"/>

    <bean id="kafkaConfiguration" class="org.springframework.integration.kafka.core.ZookeeperConfiguration">
        <constructor-arg ref="zookeeperConnect"/>
    </bean>

    <bean id="connectionFactory" class="org.springframework.integration.kafka.core.DefaultConnectionFactory">
        <constructor-arg ref="kafkaConfiguration"/>
    </bean>

    <int-kafka:message-driven-channel-adapter
            id="adapter"
            channel="inputFromKafka"
            connection-factory="connectionFactory"
            key-decoder="kafkaReflectionDecoder"
            payload-decoder="kafkaSpecificDecoder"
            max-fetch="100"
            auto-startup="true"
            topics="test"/>
</beans>
```

###注入kafka-outbound

```java
package org.github.wenhao.kafka.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.ImportResource;

@Configuration
@ImportResource("classpath:/kafka-outbound.xml")
public class KafkaOutboundConfiguraiton {
}

```

###注入kafka-inbound

```java
package org.github.wenhao.kafka.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.ImportResource;

@Configuration
@ImportResource("classpath:/kafka-inbound.xml")
public class KafkaInboundConfiguration {
}

```

###值对象

```java
package org.github.wenhao.kafka.domain;

public class User {
    private String name;
    private int age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}
```

###消息发送服务

```java
package org.github.wenhao.kafka.service;

import javax.annotation.Resource;

import org.github.wenhao.kafka.avro.UserAvro;
import org.github.wenhao.kafka.domain.User;
import org.springframework.integration.kafka.support.KafkaHeaders;
import org.springframework.integration.support.MessageBuilder;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageChannel;
import org.springframework.stereotype.Service;

@Service
public class ProducerService {

    @Resource(name = "inputToKafka")
    private MessageChannel inputToKafka;

    public void produce(User user) {
        UserAvro specificUser = UserAvro.newBuilder()
                .setName(user.getName())
                .setAge(user.getAge())
                .build();
        Message<UserAvro> message = MessageBuilder.withPayload(specificUser)
                .setHeader(KafkaHeaders.MESSAGE_KEY, "user")
                .setHeader(KafkaHeaders.TOPIC, "test")
                .build();
        inputToKafka.send(message);
    }

}
```

###消息接受服务

```java
package org.github.wenhao.kafka.service;

import javax.annotation.Resource;

import org.github.wenhao.kafka.avro.UserAvro;
import org.github.wenhao.kafka.domain.User;
import org.springframework.integration.channel.QueueChannel;
import org.springframework.messaging.Message;
import org.springframework.stereotype.Service;

@Service
public class ConsumerService {
    @Resource(name = "inputFromKafka")
    private QueueChannel inputFromKafka;

    public User receive() {
        Message<UserAvro> message = (Message<UserAvro>) inputFromKafka.receive();
        UserAvro userAvro = message.getPayload();

        User user = new User();
        user.setName(userAvro.getName().toString());
        user.setAge(userAvro.getAge());
        return user;
    }
}
```

###创建API

```java
package org.github.wenhao.kafka.api;

import static org.springframework.web.bind.annotation.RequestMethod.POST;

import org.github.wenhao.kafka.domain.User;
import org.github.wenhao.kafka.service.ConsumerService;
import org.github.wenhao.kafka.service.ProducerService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping(value = "/producer")
public class ProducerApi {

    private ProducerService producerService;
    private ConsumerService consumerService;

    @Autowired
    public ProducerApi(ProducerService producerService, ConsumerService consumerService) {
        this.producerService = producerService;
        this.consumerService = consumerService;
    }

    @RequestMapping(method = POST)
    public ResponseEntity<User> produce(@RequestBody User user) {
        producerService.produce(user);
        User receive = consumerService.receive();
        return ResponseEntity.ok(receive);
    }
}
```
