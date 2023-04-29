# 1.工作模式

工作模式也被称为任务模型（Task Queues）。当消息处理比较耗时的时候，可能生产消息的速度会远远大于消息的消费速度。长此以往，消息就会堆积越来越多，无法及时处理。此时就可以使用 work 模型：让多个消费者绑定到一个队列，共同消费队列中的消息。队列中的消息一旦消费，就会消失，因此任务是不会被重复执行。

> 这种模式只有一个生产者Producer，一个用于存储消息的队列 Queue、多个消费者Consumer用于接收消息。


![](https://files.mdnice.com/user/34714/e1267c55-4650-4c59-92ff-4557984b953b.png)

工作队列模式的特点有三：

- 一个生产者，一个队列，多个消费者同时竞争消息
- 任务量过高时可以提高工作效率
- 消费者获得的消息是无序的

 ## 1.1. 创建生产者
 
 生产者向队列中发送10条消息
 
```
package com.olive;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;

/**
 * 生产者（工作模式）
 */
public class WorkerProducer {

    /**队列名称*/
    private static final String QUEUE_NAME = "work_queue";

    public static void main(String[] args) throws Exception {
        // 1、创建连接
        Connection connection = ConnectionUtils.getConnection();
        // 2、创建通道
        Channel channel = connection.createChannel();
        // 3、声明队列 queueDeclare(队列名称,是否持久化,是否独占本连接,是否自动删除,附加属性参数)
        channel.queueDeclare(QUEUE_NAME, true, false, false, null);

        // 4、发送10条消息
        for (int i = 1; i <= 10; i++) {
            String msg = "Hello World RabbitMQ!!!" + i;
            System.out.println("生产者发送消息：" + msg);
            // basicPublish(交换机名称-""表示不用交换机,队列名称或者routingKey, 消息的属性信息, 消息内容的字节数组);
            channel.basicPublish("", QUEUE_NAME, null, msg.getBytes());
        }
        //释放资源
        channel.close();
        connection.close();
    }
}
```

## 1.2. 创建消费者

创建两个消费者WorkerConsumer1和WorkerConsumer2

- WorkerConsumer1.java
```
package com.olive;

import java.io.IOException;

import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.Envelope;

/**
 * 消费者1（工作模式）
 */
public class WorkerConsumer1 {

    /**队列名称*/
    private static final String QUEUE_NAME = "work_queue";

    public static void main(String[] args) throws Exception {
        // 1、获取连接对象
        Connection connection = ConnectionUtils.getConnection();
        // 2、创建通道（频道）
        Channel channel = connection.createChannel();

        // 3、创建队列Queue,如果没有一个名字叫work_queue的队列，则会创建该队列，如果有则不会创建.
        // 这里可有可无,但是发送消息是必须得有该队列，否则消息会丢失
        channel.queueDeclare(QUEUE_NAME, true, false, false, null);

        // 4、监听队列，接收消息
        DefaultConsumer defaultConsumer = new DefaultConsumer(channel) {
            // handleDelivery(消费者标识, 消息包的内容, 属性信息(生产者的发送时指定), 读取到的消息)
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                System.out.println("消费者获取消息：" + new String(body));
                // 模拟消息处理延时，加个线程睡眠时间
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };
        // basicConsume(队列名称, 是否自动确认, 回调对象)
        channel.basicConsume(QUEUE_NAME, true, defaultConsumer);
        //注意，消费者这里不建议关闭资源，让程序一直处于读取消息的状态
    }
}
```
- WorkerConsumer2.java
```
package com.olive;

import java.io.IOException;

import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.Envelope;

/**
 * 消费者2（工作模式）
 */
public class WorkerConsumer2 {

    /**队列名称*/
    private static final String QUEUE_NAME = "work_queue";

    public static void main(String[] args) throws Exception {
        // 1、获取连接对象
        Connection connection = ConnectionUtils.getConnection();
        // 2、创建通道（频道）
        Channel channel = connection.createChannel();

        // 3、创建队列Queue,如果没有一个名字叫work_queue的队列，则会创建该队列，如果有则不会创建.
        // 这里可有可无,但是发送消息是必须得有该队列，否则消息会丢失
        channel.queueDeclare(QUEUE_NAME, true, false, false, null);

        // 4、监听队列，接收消息
        DefaultConsumer defaultConsumer = new DefaultConsumer(channel) {
            // handleDelivery(消费者标识, 消息包的内容, 属性信息(生产者的发送时指定), 读取到的消息)
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                System.out.println("消费者获取消息：" + new String(body));
                // 模拟消息处理延时，加个线程睡眠时间
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };
        // basicConsume(队列名称, 是否自动确认, 回调对象)
        channel.basicConsume(QUEUE_NAME, true, defaultConsumer);
        //注意，消费者这里不建议关闭资源，让程序一直处于读取消息的状态
    }
}
```

> 消费者2与消费者1的代码逻辑是一模一样的

## 1.3. 验证

首先分别启动两个消费者**（注意这里一定要先启动消费者）**


![](https://files.mdnice.com/user/34714/4637ec48-f592-4755-b543-a1198c17f4df.png)

从RabbitMQ管理后台查看，已经创建了`work_queue`队列。

启动生产者，分别查看消费者1与消费者2的控制台的打印信息

消费者1`WorkerConsumer1`

![](https://files.mdnice.com/user/34714/2522bc9e-6940-47dd-8850-825d780e517a.png)

消费者2`WorkerConsumer2`


![](https://files.mdnice.com/user/34714/dc567685-826b-4f34-9b3c-c8376b14214b.png)

从两个消费者控制台的打印结果看，两个消费者消费的消息像是轮询方式消费的。

- 轮询分发(round-robin)

上面实现的就是轮询分发的方式。

> 现象：消费者1处理完消息之后，消费者2才能处理，它两这样轮着来处理消息，直到消息处理完成，这种方式叫轮询分发(round-robin)，结果就是不管两个消费者谁忙，**数据总是你消费一个我消费一个**，不管消费者处理数据的性能，此时autoAck = true。

```
/**
* @param queue 队列名称
* @param autoAck 是否自动发送确认，true自动确认，表示接收完消息后，自动将消息在队列中移除；false手动发送ack确认消息
* @param callback 回调对象
*/
String basicConsume(String queue, boolean autoAck, Consumer callback) throws IOException;
```

注意：autoAck属性设置为true，表示消息自动确认。消费者在消费时消息的确认模式可以分为：**自动确认和手动确认**。

自动确认：在队列中的消息被消费者读取之后会自动从队列中删除。不管消息是否被消费者消费成功，消息都会删除。

手动确认：当消费者读取消息后，消费端需要手动发送ACK用于确认消息已经消费成功了（也就是需要自己编写代码发送ACK确认），如果设为手动确认而没有发送ACK确认，那么消息就会一直存在队列中（前提是进行了持久化操作），后续就可能会造成消息重复消费，如果过多的消息堆积在队列中，还可能造成内存溢出，**手动确认消费者在处理完消息之后要及时发送ACK确认给队列**。

使用轮询分发的方式会有一个明显的缺点，例如，消费者1处理数据的效率很慢，消费者2处理数据的效率很高，正常情况下消费者2处理的数据应该多一点才对，而轮询分发则不管你的性能如何，反正就是每次处理一个消息，对于这种情况可以使用公平分发的方式来解决。

- 公平分发(fair dipatch)

要实现**公平分发**，需要做如下修改：

1. 消费者：保证消息一次只分发一次
2. 消费者：关闭自动确认，并且手动发送ACK给队列


![](https://files.mdnice.com/user/34714/1b7ff584-da61-4d41-8a49-467adb600a01.png)


修改后再次运行，由于消费者1设置处理完一个消息后睡眠2秒，而消费者2为1秒，所以期望输出的结果为：消费者2处理消息的速度大概是消费者1的两倍左右，结果如下。

消费者1

![](https://files.mdnice.com/user/34714/071be610-5354-4591-98fc-ab2630af03d9.png)

消费者2

![](https://files.mdnice.com/user/34714/7a3a7d2b-8016-4935-bb63-b2d5e4f62bf7.png)


# 2.发布订阅模式

发布订阅模式（Publish/Subscribe）：该模式需要涉及到交换机了，也可以称它为广播模式，消息通过交换机广播到所有与其绑定的队列中。

一个消费者将消息首先发送到交换机上（这里的交换机类型为fanout），然后交换机绑定到多个队列，这样每个发到fanout类型交换器的消息会被分发到所有的队列中，最后被监听该队列的消费者所接收并消费。如下图所示：

![](https://files.mdnice.com/user/34714/e7d64dfc-6894-436f-868a-889bb0c93379.png)

- 创建生产者

```
package com.olive;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;

/**
 * 生产者(发布订阅模式)
 */
public class PubSubProducer {

    // 交换机名称
    private static final String EXCHANGE_NAME = "fanout_exchange";

    public static void main(String[] args) throws Exception {
        // 1、创建连接
        Connection connection = ConnectionUtils.getConnection();
        // 2、创建通道
        Channel channel = connection.createChannel();
        // 3、连续发送10条消息
        for (int i = 1; i <= 10; i++) {
            String msg = "Hello World RabbitMQ!!!~~~" + i;
            System.out.println("生产者发送的消息：" + msg);
            //basicPublish(交换机名称[默认Default Exchage],路由key[简单模式可以传递队列名称],消息其它属性,发送的消息内容)
            channel.basicPublish(EXCHANGE_NAME, "", null, msg.getBytes());
        }
        //关闭资源
        channel.close();
        connection.close();
    }
}
```

- 创建消费者

由于从这里开始涉及到交换机了，使用这里介绍一下四种交换机的类型：

1. direct（直连）：消息中的路由键（RoutingKey）如果和 Bingding 中的 bindingKey 完全匹配，交换器就将消息发到对应的队列中。是基于完全匹配、单播的模式。

2. fanout（广播）：把所有发送到fanout交换器的消息路由到所有绑定该交换器的队列中，fanout 类型转发消息是最快的。

3. topic（主题）：通过模式匹配的方式对消息进行路由，将路由键和某个模式进行匹配，此时队列需要绑定到一个模式上。匹配规则：

> ① RoutingKey 和 BindingKey 为一个 点号 '.' 分隔的字符串。 比如: stock.usd.nyse；可以放任意的key在routing_key中，当然最长不能超过255 bytes。

> ② BindingKey可使用 * 和 # 用于做模糊匹配：*匹配一个单词，#匹配0个或者多个单词；

4. headers：不依赖于路由键进行匹配，是根据发送消息内容中的headers属性进行匹配，除此之外headers交换器和direct交换器完全一致，但性能差很多，目前几乎用不到了。

**消费者1**

注意：在发送消息前，RabbitMQ服务器中必须的有队列，否则消息可能会丢失，如果还涉及到交换机与队列绑定，那么就得先声明交换机、队列并且设置绑定的路由值(Routing Key)，以免程序出现异常，由于本例所有的声明都是在消费者中，所以我们首先要启动消费者。如果RabbitMQ服务器中已经存在了声明的队列或者交换机，那么就不在创建，如果没有则创建相应名称的队列或者交换机。

```
package com.olive;

import java.io.IOException;

import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.Envelope;

/**
 * 消费者1（发布订阅模式）
 */
public class PubSubConsumer1 {

    // 队列名称
    private static final String QUEUE_NAME1 = "fanout_queue1";
    // 交换机名称
    private static final String EXCHANGE_NAME = "fanout_exchange";

    public static void main(String[] args) throws Exception {
        // 1、获取连接对象
        Connection connection = ConnectionUtils.getConnection();
        // 2、创建通道（频道）
        Channel channel = connection.createChannel();

        /* 3、声明交换机
         * exchange  参数1：交换机名称
         * type      参数2：交换机类型
         * durable   参数3：交换机是否持久化
         */
        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.FANOUT, true);

        // 4、声明队列Queue queueDeclare(队列名称，是否持久化，是否独占本连接,是否自动删除,附加参数)
        channel.queueDeclare(QUEUE_NAME1, true, false, false, null);

        // 5、绑定队列和交换机 queueBind(队列名, 交换机名, 路由key[交换机的类型为fanout ，routingKey设置为""])
        channel.queueBind(QUEUE_NAME1, EXCHANGE_NAME, "");

        // 6、监听队列，接收消息
        DefaultConsumer defaultConsumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                //获取交换机信息
                String exchange = envelope.getExchange();
                //获取消息信息
                String message = new String(body, "utf-8");
                System.out.println("交换机名称:" + exchange + ",消费者获取消息: " + message);
            }
        };
        channel.basicConsume(QUEUE_NAME1, true, defaultConsumer);

        //注意，消费者这里不建议关闭资源，让程序一直处于读取消息的状态
    }
}
```

**消费者2**

消费者1基本一样，只是队列名称不同

```
package com.olive;

import java.io.IOException;

import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.Envelope;

/**
 * 消费者2（发布订阅模式）
 */
public class PubSubConsumer2 {

	// 队列名称
	private static final String QUEUE_NAME2 = "fanout_queue2";
	// 交换机名称
	private static final String EXCHANGE_NAME = "fanout_exchange";

	public static void main(String[] args) throws Exception {
		// 1、获取连接对象
		Connection connection = ConnectionUtils.getConnection();
		// 2、创建通道（频道）
		Channel channel = connection.createChannel();
		// 3、声明交换机,如果没有名称为EXCHANGE_NAME的交换机则创建，有则不创建
		channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.FANOUT, true);
		// 4、声明队列Queue。channel.queueDeclare(队列名称，是否持久化，是否独占本连接,是否自动删除,附加参数)
		channel.queueDeclare(QUEUE_NAME2, true, false, false, null);
		// 5、绑定队列和交换机。channel.queueBind(队列名, 交换机名, 路由key[fanout交换机的routingKey设置为""])
		channel.queueBind(QUEUE_NAME2, EXCHANGE_NAME, "");
		// 6、监听队列，接收消息
		DefaultConsumer defaultConsumer = new DefaultConsumer(channel) {
			@Override
			public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
					byte[] body) throws IOException {
				// 获取交换机信息
				String exchange = envelope.getExchange();
				// 获取消息信息
				String message = new String(body, "utf-8");
				System.out.println("交换机名称:" + exchange + ",消费者获取消息: " + message);
			}
		};
		channel.basicConsume(QUEUE_NAME2, true, defaultConsumer);
		// 注意，消费者这里不建议关闭资源，让程序一直处于读取消息的状态
	}
}
```

- 验证

首先分别启动所有消费者，然后使用生产者发送消息；在每个消费者对应的控制台可以查看到生产者发送的所有消息；达到**广播**的效果。

消费者1

![](https://files.mdnice.com/user/34714/a6682f47-98a4-46e4-bff2-a280dcbb76e7.png)

消费者2

![](https://files.mdnice.com/user/34714/865ec147-37c7-4ae9-b7da-c5f4bbce2f7d.png)


执行完测试代码后，在RabbitMQ的管理后台找到Exchanges选项卡，点击`fanout_exchange`交换机，可以查看到如下的绑定：

![](https://files.mdnice.com/user/34714/a4221f9e-47a8-4521-9102-14f9a7d1f6ad.png)

![](https://files.mdnice.com/user/34714/7c0c8758-81c9-43f6-919e-e024b2fa1971.png)

`fanout_exchange`是代码中定义的交换机的名称；`fanout_queue1`和`fanout_queue2`是代码中消费者1和消费者2定义的两个队列的名称

- 总结

发布订阅模式引入了交换机的概念，所以相对前面的类型更加灵活广泛一些。这种模式需要设置类型为fanout的交换机，并且将交换机和队列进行绑定，当消息发送到交换机后，交换机会将消息发送到所有绑定的队列，最后被监听该队列的消费者所接收并消费。发布订阅模式也可以叫广播模式，不需要RoutingKey的判断。

**发布订阅模式与工作队列模式的区别：**

1. 工作队列模式不用定义交换机，而发布/订阅模式需要定义交换机。
2. 发布/订阅模式的生产方是面向交换机发送消息，工作队列模式的生产方是面向队列发送消息(底层使用默认交换机)。
3. 发布/订阅模式需要设置队列和交换机的绑定，工作队列模式不需要设置，实际上工作队列模式会将队列绑定到默认的交换机 。

```
来源
cnblogs.com/tanghaorong/p/14992330.html#_label0
```

