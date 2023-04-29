# 1.安装erlang

**下载`otp_win64_25.3.exe`**

```
https://www.erlang.org/downloads
```

**erlang安装完成，需要配置erlang环境变量**

```
ERLANG_HOME=E:\software\Erlang OTP

PATH=%PATH%;%ERLANG_HOME%\bin;
```

# 2.安装RabbitMQ

**下载`rabbitmq-server-3.11.13.exe`**

```
https://www.rabbitmq.com/download.html
```

**进入安装目录下sbin目录，安装并运行服务**

```
安装服务: rabbitmq-service.bat install
删除服务：rabbitmq-service.bat remove
启动服务：rabbitmq-service.bat start
停止服务: rabbitmq-service.bat stop
```

**安装管理插件**

安装RabbitMQ的管理插件，方便在浏览器端管理RabbitMQ

管理员身份打开cmd，进入`E:\software\RabbitMQ Server\rabbitmq_server-3.11.13\sbin`目录，运行

```
rabbitmq-plugins.bat enable rabbitmq_management
```

**执行结果**

![](https://files.mdnice.com/user/34714/1495e097-9720-474a-9ad4-5c5f0259387e.png)

**重启一下RabbitMQ**

![](https://files.mdnice.com/user/34714/fe05f585-5fa7-4091-b514-b9751948b831.png)

**启动成功，登录RabbitMQ**

访问地址`http://127.0.0.1:15672/`；初始账号和密码`guest/guest`。

![](https://files.mdnice.com/user/34714/ecf0cf5c-b7e2-4da8-b944-3244a73803b4.png)

![](https://files.mdnice.com/user/34714/e439f207-e166-46dc-9338-7e33b92bac2d.png)

# 3. RabbitMQ常用五种模式

# 3.1. 简单模式

该模式是个一对一模式，只有一个生产者Producer（用于生产消息），一个队列Queue（用于存储消息），一个消费者Consumer （用于接收消息）。


![](https://files.mdnice.com/user/34714/3d35308d-d714-4a92-8bbe-e417ad6fd187.png)

> 注：简单模式也用到了交换机，使用的是默认的交换机(AMQP default)。

- 创建项目`rabbitmq-learn`

pom.xml引入以下依赖
```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.olive</groupId>
	<artifactId>rabbitmq-learn</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<properties>
        <maven.compiler.target>1.8</maven.compiler.target>
        <maven.compiler.source>1.8</maven.compiler.source>
    </properties>

	<dependencies>
		<!-- mq的依赖 -->
		<dependency>
			<groupId>com.rabbitmq</groupId>
			<artifactId>amqp-client</artifactId>
			<version>5.16.0</version>
		</dependency>
		<!-- 日志处理 -->
		<dependency>
			<groupId>org.slf4j</groupId>
			<artifactId>slf4j-log4j12</artifactId>
			<version>1.7.21</version>
		</dependency>
		<dependency>
			<groupId>log4j</groupId>
			<artifactId>log4j</artifactId>
			<version>1.2.17</version>
		</dependency>
	</dependencies>
</project>
```
RabbitMQ官方提供了`amqp-client`java客户端连接RabbitMQ Server；仓库地址如下

```
https://github.com/rabbitmq/rabbitmq-java-client
```

- RabbitMQ连接的工具类

```
package com.olive;

import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

/**
 * 封装连接工具类
 */
public class ConnectionUtils {
	
    public static Connection getConnection() throws Exception {
        // 1.定义连接工厂
        ConnectionFactory factory = new ConnectionFactory();
        // 2.设置服务器地址
        factory.setHost("127.0.0.1");
        // 3.设置协议端口号
        factory.setPort(5672);
        // 4.虚拟主机名称;默认为 /
        factory.setVirtualHost("/");
        // 5.设置用户名称
        factory.setUsername("admin");
        // 6.设置用户密码
        factory.setPassword("admin123");
        // 7.创建连接
        Connection connection = factory.newConnection();
        return connection;
    }
}
```
在RabbitMQ管理后台创建admin用户；可以使用默认的guest用户。

![](https://files.mdnice.com/user/34714/79e4611d-0d93-4a07-a8c5-7cbca28d1a82.png)

- 创建生产者

生产者负责创建消息并且将消息发送至指定的队列中，简单分为5步：

**创建连接 ——> 创建通道 ——> 创建（声明）队列 ——> 发送消息 ——> 关闭资源**

```
package com.olive;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;

/**
 * 生产者（简单模式）
 */
public class SimpleProducer {

    /**队列名称*/
    private static final String QUEUE_NAME = "simple_queue";

    public static void main(String[] args) throws Exception {
        // 1、获取连接
        Connection connection = ConnectionUtils.getConnection();
        // 2、创建通道（频道）
        Channel channel = connection.createChannel();
        // 3、声明（创建）队列
        /*
         * queue      参数1：声明通道中对应的队列名称
         * durable    参数2：是否定义持久化队列,当mq重启之后队列还在
         * exclusive  参数3：是否独占本次连接，为true则只能有一个消费者监听这个队列
         * autoDelete 参数4：是否自动删除队列，如果为true表示没有消息也没有消费者连接自动删除队列
         * arguments  参数5：队列其它参数（额外配置）
         */
        channel.queueDeclare(QUEUE_NAME, true, false, false, null);

        // 4.发送消息
        /*
         * exchange   参数1：交换机名称，如果没有指定则使用默认Default Exchange
         * routingKey 参数2：队列名称或者routingKey，如果指定了交换机就是routingKey路由key,简单模式可以传递队列名称
         * props      参数3：消息的配置信息
         * body       参数4：要发送的消息内容
         */
        String msg = "Hello World RabbitMQ!!!";
        System.out.println("生产者发送的消息：" + msg);
        channel.basicPublish("", QUEUE_NAME, null, msg.getBytes());

        //关闭资源
        channel.close();
        connection.close();
    }
}
```

- 创建消费者

消费者实现和生产者实现过程差不多，但是没有关闭通道和连接，因为消费者要一直等待随时可能发来的消息，大致分为如下3步：

**获取连接 ——> 创建通道 ——> 监听队列，接收消息**

```
package com.olive;

import java.io.IOException;

import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.Envelope;

/**
 * 消费者（简单模式）
 */
public class SimpleConsumer {

    /**队列名称*/
    private static final String QUEUE_NAME = "simple_queue";

    public static void main(String[] args) throws Exception {
        // 1、获取连接对象
        Connection connection = ConnectionUtils.getConnection();
        // 2、创建通道（频道）
        Channel channel = connection.createChannel();

        // 3. 创建队列Queue,如果没有一个名字叫simple_world的队列，则会创建该队列，如果有则不会创建.
        // 这里可有可无,但是发送消息是必须得有该队列，否则消息会丢失
        channel.queueDeclare(QUEUE_NAME, true, false, false, null);

        // 4、监听队列，接收消息
        DefaultConsumer defaultConsumer = new DefaultConsumer(channel) {
            /*
             *  handleDelivery回调方法，当收到消息后，会自动执行该方法
             *  consumerTag 参数1：消费者标识
             *  envelope    参数2：可以获取一些信息，如交换机，路由key...
             *  properties  参数3：配置信息
             *  body        参数4：读取到的消息
             */
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                System.out.println("消费者获取消息：" + new String(body));
            }
        };
        /*
         * queue    参数1：队列名称
         * autoAck  参数2：是否自动确认,true表示自动确认接收完消息以后会自动将消息从队列移除。否则需要手动ack消息
         * callback 参数3：回调对象，在上面定义了
         */
        channel.basicConsume(QUEUE_NAME, true, defaultConsumer);

        //注意，消费者这里不建议关闭资源，让程序一直处于读取消息的状态
    }
}
```

- 验证测试

运行生产者的代码，表示向队列中发送消息。

![](https://files.mdnice.com/user/34714/6ad3ddb1-ce90-4bc2-a0e0-10449c8b84d6.png)

查看RabbitMQ控制台中的Queues内容

![](https://files.mdnice.com/user/34714/a120f860-0b61-470f-b765-924e3fce3f12.png)

启动消费者，消费RabbitMQ队列中的消息。

![](https://files.mdnice.com/user/34714/2623921c-6fd2-4ff4-bac2-5b3d0412dfa1.png)

在通过RabbitMQ控制台查看Queues的内容；发现消息已经被消费

![](https://files.mdnice.com/user/34714/7b73e7b8-18ee-4798-822a-37cef4a59d2e.png)

> 简单模式的不足之处：该模式是一对一，一个生产者向一个队列中发送消息，一个消费者从绑定的队列中获取消息，这样耦合性过高，如果有多个消费者想消费队列中信息就无法实现了。

```
参考
www.cnblogs.com/zwh0910/p/16056182.html#autoid-5-0-0
www.cnblogs.com/tanghaorong/p/14992330.html#_label0
```