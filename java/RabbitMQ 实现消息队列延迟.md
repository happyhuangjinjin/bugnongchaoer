# 1.概述

要实现RabbitMQ的消息队列延迟功能，一般采用官方提供的 `rabbitmq_delayed_message_exchange`插件。但RabbitMQ版本必须是3.5.8以上才支持该插件，否则得用其**死信队列**功能。

# 2.安装RabbitMQ延迟插件

- 检查插件
使用`rabbitmq-plugins list`命令用于查看RabbitMQ安装的插件。

```
rabbitmq-plugins list
```

检查RabbitMQ插件安装情况

![](https://files.mdnice.com/user/34714/fe6c83a6-aa43-4ae6-b8aa-18bfe232f2d0.png)

- 下载插件

如果没有安装插件，则直接访问官网进行下载
```
https://www.rabbitmq.com/community-plugins.html
```

![](https://files.mdnice.com/user/34714/78b72b7f-e760-4d7f-84fd-3ad85fcdd54c.png)


![](https://files.mdnice.com/user/34714/c34c8834-e018-44b9-b88a-7e39f08bb101.png)

- 安装插件

下载后，将其拷贝到RabbitMQ安装目录的plugins目录；并进行解压，如：

```
E:\software\RabbitMQ Server\rabbitmq_server-3.11.13\plugins
```


![](https://files.mdnice.com/user/34714/f3f41d09-8d8d-4a6d-9840-fa14e75e3db7.png)


打开cmd命令行窗口，如果系统已经配置RabbitMQ环境变量，则直接执行以下的命令进行安装；否则需要进入到RabbitMQ安装目录的sbin目录。

```
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```

![](https://files.mdnice.com/user/34714/c5a12a0e-34fe-4f44-a674-1824d2dd380e.png)

# 3.实现RabbitMQ消息队列延迟功能

- pom.xml配置信息文件中，添加相关依赖文件

```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.olive</groupId>
	<artifactId>rabbitmq-spring-demo</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.7.7</version>
		<relativePath />
	</parent>
	<dependencies>
		<!--rabbitmq-->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-amqp</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		
	<dependency>
		    <groupId>org.eclipse.paho</groupId>
		    <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
		    <version>1.2.5</version>
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
</project>
```

- application.yml配置文件中配置RabbitMQ信息

```
server:
  port: 8080
spring:
  #给项目来个名字
  application:
    name: rabbitmq-spring-demo
  #配置rabbitMq 服务器
  rabbitmq:
    host: 127.0.0.1
    port: 5672
    username: admin
    password: admin123
    #虚拟host。可以不设置,使用server默认host；不同虚拟路径下的队列是隔离的
    virtual-host: /
```

- RabbitMQ配置类

```
package com.olive.config;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.CustomExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.HashMap;
import java.util.Map;

/**
 * RabbitMQ配置类
 **/
@Configuration
public class RabbitMqConfig {
	
	public static final String DELAY_EXCHANGE_NAME = "delayed_exchange";
	
	public static final String DELAY_QUEUE_NAME = "delay_queue_name";
	
	public static final String DELAY_ROUTING_KEY = "delay_routing_key";

	@Bean
	public CustomExchange delayExchange() {
		Map<String, Object> args = new HashMap<>();
		args.put("x-delayed-type", "direct");
		return new CustomExchange(DELAY_EXCHANGE_NAME, "x-delayed-message", true, false, args);
	}

	@Bean
	public Queue queue() {
		Queue queue = new Queue(DELAY_QUEUE_NAME, true);
		return queue;
	}

	@Bean
	public Binding binding(Queue queue, CustomExchange delayExchange) {
		return BindingBuilder.bind(queue).to(delayExchange).with(DELAY_ROUTING_KEY).noargs();
	}
}
```

- 发送消息

实现消息发送，设置消息延迟5s。

```
package com.olive.service;

import java.text.SimpleDateFormat;
import java.util.Date;

import org.springframework.amqp.AmqpException;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.core.MessagePostProcessor;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import com.olive.config.RabbitMqConfig;

/**
 * 消息发送者
 **/
@Service
public class CustomMessageSender {
	
	@Autowired
	private RabbitTemplate rabbitTemplate;

	public void sendMsg(String msg) {
		SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
		System.out.println("消息发送时间：" + sdf.format(new Date()));
		rabbitTemplate.convertAndSend(RabbitMqConfig.DELAY_EXCHANGE_NAME, 
				RabbitMqConfig.DELAY_ROUTING_KEY, 
				msg, 
				new MessagePostProcessor() {
					@Override
					public Message postProcessMessage(Message message) throws AmqpException {
						// 消息延迟5秒
						message.getMessageProperties().setHeader("x-delay", 5000);
						return message;
					}
				});
	}
}
```

- 接收消息

```
package com.olive.service;

import java.text.SimpleDateFormat;
import java.util.Date;

import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

import com.olive.config.RabbitMqConfig;

/**
 * 消息接收者
 **/
@Component
public class CustomMessageReceiver {
	
	@RabbitListener(queues = RabbitMqConfig.DELAY_QUEUE_NAME)
	public void receive(String msg) {
		SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
		System.out.println(sdf.format(new Date()) + msg);
		System.out.println("Receiver：执行取消订单");
	}
}
```

- 测试验证

```
package com.olive.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import com.olive.service.CustomMessageSender;

@RestController
public class DelayMessageController {
	
	@Autowired
	private CustomMessageSender customMessageSender;
	
	@GetMapping("/sendMessage")
	public String sendMessage() {
		// 发送消息
		customMessageSender.sendMsg("你已经支付超时，取消订单通知！");
		return "success";
	}

}
```

发送消息，访问

```
http://127.0.0.1:8080/sendMessage
```

查看控制台打印的信息

![](https://files.mdnice.com/user/34714/bb6d39f9-64b4-4a54-9332-d73869e8e474.png)

