# 1.概述

SpringCloud Stream框架抽象出了三个最基础的概念来对各种消息中间件提供统一调用：

- Destination Binders: 负责集成外部消息系统的组件。

- Destination Binding: 由Binders创建的，负责沟通外部消息系统、消息发送者和消息消费者的桥梁。

- Message: 消息发送者与消息消费者沟通的简单数据结构。


![](https://files.mdnice.com/user/34714/2eb59e7c-9b0b-4fa4-9dc0-1a2ee2b1f084.png)


# 2.创建生产者项目

创建项目`rabbitmq-stream-sender`

- pom.xml添加依赖

```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.olive</groupId>
	<artifactId>rabbitmq-stream-sender</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.7.7</version>
		<relativePath />
	</parent>
	<dependencies>
		<!--rabbitMQ相关-->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-stream</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-stream-rabbit</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-stream-binder-rabbit</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
	</dependencies>
	<build>
		<plugins>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-compiler-plugin</artifactId>
				<configuration>
					<source>1.8</source>
					<target>1.8</target>
				</configuration>
			</plugin>
		</plugins>
	</build>
	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>2021.0.6</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>
</project>
```

- application.yml 配置

```
spring:
  cloud:
    stream:
      bindings:
        output:
          destination: rabbitmqExchange
          content-type: text/plain
          group: stream
  rabbitmq:
    username: admin
    password: admin123
    port: 5672
    host: 127.0.0.1
    virtual-host: /

server:
  port: 8081
```

- 生产者

```
package com.olive.service;

import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.messaging.Source;
import org.springframework.stereotype.Component;

@Component
@EnableBinding(Source.class)
public class MessageProducer {

}
```

- 发送消息接口

```
package com.olive.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.messaging.Source;
import org.springframework.integration.support.MessageBuilder;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class MessageController {
	
	@Autowired
	private Source source;
	
	@GetMapping("/api/send")
	public String send(String message) {
		MessageBuilder<String> messageBuilder = MessageBuilder.withPayload(message);
		source.output().send(messageBuilder.build());
		return "success" ;
	}
}
```

# 3.创建消费者者项目

创建项目`rabbitmq-stream-revicer`

- pom.xml添加依赖

```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.olive</groupId>
  <artifactId>rabbitmq-stream-revicer</artifactId>
  <version>0.0.1-SNAPSHOT</version>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.7.7</version>
		<relativePath />
	</parent>
	<dependencies>
		<!--rabbitMQ相关-->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-stream-rabbit</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
	</dependencies>
	<build>
		<plugins>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-compiler-plugin</artifactId>
				<configuration>
					<source>1.8</source>
					<target>1.8</target>
				</configuration>
			</plugin>
		</plugins>
	</build>
	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>2021.0.6</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>
</project>
```

- application.yml 配置

```
spring:
  cloud:
    stream:
      bindings:
        input:
          destination: rabbitmqExchange
          content-type: text/plain
          group: stream
  rabbitmq:
    username: admin
    password: admin123
    port: 5672
    host: 127.0.0.1
    virtual-host: /

server:
  port: 8082
```

- 消费者

```
package com.olive.service;

import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.cloud.stream.messaging.Sink;
import org.springframework.stereotype.Component;

@Component
@EnableBinding(Sink.class)
public class MessageConsumer {
	
	@StreamListener(Sink.INPUT)
	public void process(Object message) {
		System.out.println("received message: " + message);
	}
	
}
```

# 4. 验证

- 分别启动`rabbitmq-stream-sender`和`rabbitmq-stream-revicer`项目

- 访问

```
http://127.0.0.1:8081/api/send?message=hello world stream
```

![](https://files.mdnice.com/user/34714/54685c98-aa0f-442b-a593-7a0a2739820a.png)

```
参考：
blog.csdn.net/admin522043032/article/details/124877134
blog.csdn.net/Extraordinarylife/article/details/114208673
www.cnblogs.com/wessonshin/p/12602072.html
```

