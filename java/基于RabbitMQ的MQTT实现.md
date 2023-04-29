# 1.RabbitMQ mqtt协议开启

默认情况下RabbitMQ是不开启MQTT协议的，所以需要我们手动的开启相关的插件，而RabbitMQ的MQTT协议分为两种。

- rabbitmq_mqtt 提供与后端服务交互使用，对应端口1883

- rabbitmq_web_mqtt 提供与前端交互使用，对应端口15675

打开cmd窗口，进入RabbitMQ的sbin目录

开启`rabbitmq_mqtt`协议

```
rabbitmq-plugins enable rabbitmq_mqtt
```

![](https://files.mdnice.com/user/34714/aaeee68d-458f-44f4-8c9a-34fabd516a36.png)


开启`rabbitmq_web_mqtt`协议

```
rabbitmq-plugins enable rabbitmq_web_mqtt
```

![](https://files.mdnice.com/user/34714/30af003f-4713-4461-b314-005ef6289ad7.png)

重启RabbitMQ后，登录RabbitMQ管理后台

```
http://127.0.0.1:15672
```

![](https://files.mdnice.com/user/34714/750d6db8-3aa1-4011-9e6f-7f3c767c5d2e.png)


# 3.mqtt相关概念：

- Publisher（发布者）：消息的发出者，负责生产数据。发布者发送某个主题的数据给经纪人，发布者不知道订阅者。

- Subscriber（订阅者）：消息的订阅者，订阅经纪人管理的某个或者某几个主题。

- Broker（经纪人）：当经纪人接收到某个主题的数据时，将数据发送给这个主题的所有订阅者。

- Topic（主题）：可以理解为消息队列中的路由，订阅者订阅了主题之后，就可以收到发送到该主题的消息。

- Payload（负载）；可以理解为发送消息的内容。

- QoS（消息质量）：全称 Quality of Service，即消息的发送质量，主要有 QoS 0、QoS 1、QoS 2三个等级，下面分别介绍下：

  (1) QoS 0（Almost Once）：至多一次，只发送一次，会发生消息丢失或重复；

  (2) QoS 1（Atleast Once）：至少一次，确保消息到达，但消息重复可能会发生；

  (3) QoS 2（Exactly Once）：只有一次，确保消息只到达一次。

# 3.Spring整合mqtt

- 创建项目

pom.xml文件引入如下依赖

```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.olive</groupId>
	<artifactId>rabbitmq-mqtt-demo</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.7.7</version>
		<relativePath />
	</parent>
	<properties>
		<maven.compiler.target>1.8</maven.compiler.target>
		<maven.compiler.source>1.8</maven.compiler.source>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.integration</groupId>
			<artifactId>spring-integration-mqtt</artifactId>
		</dependency>
		<dependency>
			<groupId>org.eclipse.paho</groupId>
			<artifactId>org.eclipse.paho.client.mqttv3</artifactId>
			<version>1.2.1</version>
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
	</dependencies>
</project>
```
- mqtt连接配置类

```
package com.olive.config;

import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.mqtt.core.DefaultMqttPahoClientFactory;
import org.springframework.integration.mqtt.core.MqttPahoClientFactory;

@Configuration
public class MqttConfig {

	private static String servers[] = { "tcp://127.0.0.1:1883" };
	
	private static String username = "admin";
	
	private static String password = "admin123";

	@Bean
	public MqttConnectOptions getMqttConnectOptions() {
		MqttConnectOptions options = new MqttConnectOptions();
		// 设置是否清空session,这里如果设置为false表示服务器会保留客户端的连接记录，
		// 这里设置为true表示每次连接到服务器都以新的身份连接
		options.setCleanSession(true);
		// 设置连接的用户名
		options.setUserName(username);
		// 设置连接的密码
		options.setPassword(password.toCharArray());
		options.setServerURIs(servers);
		// 设置超时时间 单位为秒
		options.setConnectionTimeout(10);
		// 设置会话心跳时间 单位为秒 服务器会每隔1.5*20秒的时间向客户端发送心跳判断客户端是否在线，但这个方法并没有重连的机制
		options.setKeepAliveInterval(20);
		// 设置“遗嘱”消息的话题，若客户端与服务器之间的连接意外中断，服务器将发布客户端的“遗嘱”消息。
		//options.setWill("willTopic", WILL_DATA, 2, false);
		return options;
	}

	@Bean
	public MqttPahoClientFactory mqttClientFactory() {
		DefaultMqttPahoClientFactory factory = new DefaultMqttPahoClientFactory();
		factory.setConnectionOptions(getMqttConnectOptions());
		return factory;
	}
}
```

- 消息生产者配置类

```
package com.olive.config;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.integration.channel.DirectChannel;
import org.springframework.integration.mqtt.core.MqttPahoClientFactory;
import org.springframework.integration.mqtt.outbound.MqttPahoMessageHandler;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.MessageHandler;

@Configuration
public class MqttProducerConfig {

	public static final String CHANNEL_NAME_OUT = "mqttOutboundChannel";

	private static String clientId = "test_mqtt/producer";
	
	private static String topic = "test_mqtt_topic";

	@Autowired
	MqttPahoClientFactory mqttClientFactory;

	/**
	 * MQTT信息通道（生产者）
	 */
	@Bean(name = CHANNEL_NAME_OUT)
	public MessageChannel mqttOutboundChannel() {
		return new DirectChannel();
	}

	/**
	 * MQTT消息处理器（生产者）
	 */
	@Bean
	@ServiceActivator(inputChannel = CHANNEL_NAME_OUT)
	public MessageHandler mqttOutbound() {
		MqttPahoMessageHandler messageHandler = new MqttPahoMessageHandler(clientId, mqttClientFactory);
		messageHandler.setAsync(false);
		messageHandler.setDefaultTopic(topic);
		return messageHandler;
	}
}
```

- 消息监听器配置类

```
package com.olive.config;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.integration.channel.DirectChannel;
import org.springframework.integration.core.MessageProducer;
import org.springframework.integration.mqtt.core.MqttPahoClientFactory;
import org.springframework.integration.mqtt.inbound.MqttPahoMessageDrivenChannelAdapter;
import org.springframework.integration.mqtt.support.DefaultPahoMessageConverter;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.MessageHandler;
import org.springframework.messaging.MessagingException;

@Configuration
public class MqttListener {

    public static final String CHANNEL_NAME_IN = "mqttInboundChannel";

    private static String clientId = "test_mqtt/consumer";
    
    private static String listenTopic = "test_mqtt_topic";

    @Autowired
    MqttPahoClientFactory mqttClientFactory;

    /**
     * MQTT消息通道（消费者）
     */
    @Bean(name = CHANNEL_NAME_IN)
    public MessageChannel mqttInboundChannel() {
        return new DirectChannel();
    }

    /**
     * MQTT消息订阅绑定（消费者）
     */
    @Bean
    public MessageProducer inbound() {
        MqttPahoMessageDrivenChannelAdapter adapter = new MqttPahoMessageDrivenChannelAdapter(clientId,
        		mqttClientFactory, 
        		listenTopic);
        adapter.setCompletionTimeout(5000);
        adapter.setConverter(new DefaultPahoMessageConverter());
        //设置消息质量：0->至多一次；1->至少一次；2->只有一次
        adapter.setQos(1);
        adapter.setOutputChannel(mqttInboundChannel());
        return adapter;
    }

    /**
     * MQTT消息监听器（消费者）
     * MessageHandler: org.springframework:spring-messaging
     */
    @Bean
    @ServiceActivator(inputChannel = CHANNEL_NAME_IN)
    public MessageHandler handlerMessage() {
        return message -> {
            try {
                MessageHeaders messageHeaders = message.getHeaders();
            	   System.out.println("messageHeaders>>" + messageHeaders);
                String string = message.getPayload().toString();
                System.out.println("接收到消息：" + string);
            } catch (MessagingException e) {
                e.printStackTrace();
            }
        };
    }

}
```

`handlerMessage()`方法也可以独立出来，如下

```
package com.olive.handler;

import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageHandler;
import org.springframework.messaging.MessageHeaders;
import org.springframework.messaging.MessagingException;
import org.springframework.stereotype.Component;

@Component
public class CustomMessageHandler implements MessageHandler {

	@ServiceActivator(inputChannel = MqttListener.CHANNEL_NAME_IN)
	@Override
	public void handleMessage(Message<?> message) throws MessagingException {
		try {
			MessageHeaders messageHeaders = message.getHeaders();
			System.out.println("messageHeaders>>" + messageHeaders);
			String string = message.getPayload().toString();
			System.out.println("接收到消息：" + string);
		} catch (MessagingException e) {
			e.printStackTrace();
		}
	}

}
```

- 消息发送服务

```
package com.olive.service;

import org.springframework.integration.annotation.MessagingGateway;
import org.springframework.integration.mqtt.support.MqttHeaders;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.stereotype.Component;

import com.olive.config.MqttProducerConfig;

@Component
@MessagingGateway(defaultRequestChannel = MqttProducerConfig.CHANNEL_NAME_OUT)
public interface MqttSender {

	/**
	 * 发送信息到MQTT服务器
	 *
	 * @param data 发送的文本
	 */
	void sendToMqtt(String data);

	/**
	 * 发送信息到MQTT服务器
	 *
	 * @param topic   主题
	 * @param payload 消息主体
	 */
	void sendToMqtt(@Header(MqttHeaders.TOPIC) String topic, String payload);

	/**
	 * 发送信息到MQTT服务器
	 *
	 * qos: 0 至多一次，数据可能丢失 1 至少一次，数据可能重复 2 只有一次，且仅有一次，最耗性能
	 *
	 * @param topic   主题
	 * @param qos     服务质量
	 * @param payload 消息主体
	 */
	void sendToMqtt(@Header(MqttHeaders.TOPIC) String topic, @Header(MqttHeaders.QOS) int qos, 
			String payload);
}
```

- 验证

访问如下接口发送数据

`http://127.0.0.1:8080/mqtt?msg=mqttmessage`

`http://127.0.0.1:8080/mqtt2?msg=mqttmessage`

```
package com.olive.controller;

import javax.annotation.Resource;

import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import com.olive.service.MqttSender;

@RestController
public class MqttController {

    @Resource
    private MqttSender mqttSender;
    
    /**
     * 发送MQTT消息
     */
    @RequestMapping(value = "/mqtt", produces = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<String> sendMqtt(@RequestParam(value = "msg") String message) {
        System.out.println("生产MQTT消息: " + message);
        mqttSender.sendToMqtt(message);
        return new ResponseEntity<>("OK", HttpStatus.OK);
    }

    /**
     * 发送MQTT消息
     */
    @RequestMapping(value = "/mqtt2", produces = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<String> sendMqtt2(@RequestParam(value = "msg") String message) {
        System.out.println("生产MQTT消息：" + message);
        mqttSender.sendToMqtt("test_mqtt_topic", message);
        return new ResponseEntity<>("OK", HttpStatus.OK);
    }
}
```
