# 1.路由模式(精确匹配)

路由模式（Routing）的特点：

- 该模式的交换机为direct，意思为定向发送，精准匹配。
- 队列与交换机的绑定，不能是任意绑定了，而是要指定一个RoutingKey（路由key）
- 消息的发送方在向Exchange发送消息时，也必须指定消息的 RoutingKey。
- Exchange不再把消息交给每一个绑定的队列，而是根据消息的RoutingKey进行判断，只有队列的Routingkey与消息的Routing key完全一致，才会接收到消息。

生产者将消息发送到direct交换器，同时生产者在发送消息的时候会指定一个路由key，而在绑定队列和交换器的时候又会指定一个路由key，那么消息只会发送到相应routing key相同的队列，然后由监听该队列的消费者进行消费消息。模型如下图所示：


![](https://files.mdnice.com/user/34714/02ea74eb-d00a-45d6-8dfe-dfe970e2a373.png)

- 创建生产者

```
package com.olive;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;

/**
 * 生产者（路由模式）
 */
public class RoutorProducer {

    // 交换机名称
    private static final String EXCHANGE_NAME = "routing_exchange";

    public static void main(String[] args) throws Exception {
        // 1、创建连接
        Connection connection = ConnectionUtils.getConnection();
        // 2、创建通道（频道）
        Channel channel = connection.createChannel();
        // 3、发送消息，连续发3条
        for (int i = 0; i < 3; i++) {
            String routingKey = "";
            //发送消息的时候根据相关逻辑指定相应的routing key。
            switch (i) {
                case 0:  //假设i=0，为error消息
                    routingKey = "error";
                    break;
                case 1: //假设i=1，为info消息
                    routingKey = "info";
                    break;
                case 2: //假设i=2，为warning消息
                    routingKey = "warning";
                    break;
            }
            // 要发送的消息
            String message = "Hello World Message!!!~~~" + routingKey;
            // 消息发送 channel.basicPublish(交换机名称,路由key,消息其它属性,消息内容)
            channel.basicPublish(EXCHANGE_NAME, routingKey, null, message.getBytes("utf-8"));
            System.out.println("生产者发送的消息：" + message);
        }
        //释放资源
        channel.close();
        connection.close();
    }
}
```
- 创建消费者

**消费者1**

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
 * 消费者1（路由模式）
 */
public class RoutorCunsumer1 {

    // 队列名称
    private static final String QUEUE_NAME1 = "routing_queue1";
    // 交换机名称
    private static final String EXCHANGE_NAME = "routing_exchange";

    public static void main(String[] args) throws Exception {
        // 1、获取连接对象
        Connection connection = ConnectionUtils.getConnection();
        // 2、创建通道（频道）
        Channel channel = connection.createChannel();
        // 3、声明交换机(有则不创建,无则创建) channel.exchangeDeclare(交换机名字,交换机类型,是否持久化)
        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT, true);
        // 4、声明队列Queue。channel.queueDeclare(队列名称，是否持久化，是否独占本连接,是否自动删除,附加参数)
        channel.queueDeclare(QUEUE_NAME1, true, false, false, null);

        // 5、根据指定的routingKey绑定队列和交换机 channel.queueBind(队列名, 交换机名, 路由key)
        channel.queueBind(QUEUE_NAME1, EXCHANGE_NAME, "error");

        // 6、监听队列，接收消息
        DefaultConsumer defaultConsumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                //获取路由的key
                String routingKey = envelope.getRoutingKey();
                //获取交换机信息
                String exchange = envelope.getExchange();
                //获取消息信息
                String message = new String(body, "utf-8");
                System.out.println("路由Key:" + routingKey + ", 交换机名称:" + exchange + ", 消费者获取消息: " + message);
            }
        };
        channel.basicConsume(QUEUE_NAME1, true, defaultConsumer);
        //注意，消费者这里不建议关闭资源，让程序一直处于读取消息的状态
    }
}

```

**消费者2**

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
 * 消费者2（路由模式）
 */
public class RoutorCunsumer2 {

    // 队列名称
    private static final String QUEUE_NAME2 = "routing_queue2";
    // 交换机名称
    private static final String EXCHANGE_NAME = "routing_exchange";

    public static void main(String[] args) throws Exception {
        // 1、获取连接对象
        Connection connection = ConnectionUtils.getConnection();
        // 2、创建通道（频道）
        Channel channel = connection.createChannel();
        // 3、声明交换机(有则不创建,无则创建) channel.exchangeDeclare(交换机名字,交换机类型,是否持久化)
        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT, true);
        // 4、声明队列Queue。channel.queueDeclare(队列名称，是否持久化，是否独占本连接,是否自动删除,附加参数)
        channel.queueDeclare(QUEUE_NAME2, true, false, false, null);

        // 5、根据指定的routingKey绑定队列和交换机 channel.queueBind(队列名, 交换机名, 路由key)
        channel.queueBind(QUEUE_NAME2, EXCHANGE_NAME, "error");
        channel.queueBind(QUEUE_NAME2, EXCHANGE_NAME, "info");
        channel.queueBind(QUEUE_NAME2, EXCHANGE_NAME, "warning");

        // 6、监听队列，接收消息
        DefaultConsumer defaultConsumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                //获取路由的key
                String routingKey = envelope.getRoutingKey();
                //获取交换机信息
                String exchange = envelope.getExchange();
                //获取消息信息
                String message = new String(body, "utf-8");
                System.out.println("路由Key:" + routingKey + ", 交换机名称:" + exchange + ", 消费者获取消息: " + message);
            }
        };
        channel.basicConsume(QUEUE_NAME2, true, defaultConsumer);
    }
}
```

- 验证

分别先启动所有消费者，再启动生产者发送消息；在消费者对应的控制台可以查看到生产者发送对应routing key对应队列的消息；达到**按照需要接收**的效果。

消费者1绑定的交换机和队列的路由Key为error，所以只要生产者发送消息时带有error的routingKey它都能够获取到消息。

![](https://files.mdnice.com/user/34714/1dd91d55-9077-42b1-b7c3-9352b03f04d0.png)


消费者2绑定的交换机和队列的路由Key为error、info、warning，所以只要生产者发送消息时带有这3种的routingKey它都能够获取到消息。

![](https://files.mdnice.com/user/34714/1397d79d-38dd-4ff0-be6f-5fd083b53852.png)

从RabbitMQ管理后台也能看到对应的交换机，以及队列绑定情况

![](https://files.mdnice.com/user/34714/fc41807f-3d45-4c2a-a588-91e633ea4892.png)


- 总结

1. Routing模式需要将交换机设置为Direct类型。
2. Routing模式要求队列在绑定交换机时要指定routing key，当发送消息到交换机后，交换机会根据routing key将消息发送到对应的队列。

# 2.Topic模式(模糊匹配)

Topic类型与Direct相比，都是可以根据RoutingKey把消息路由到不同的队列。但是Topic类型的Exchange可以让队列在绑定Routing key的时候使用通配符进行匹配，也就是模糊匹配，这样与之前的模式比起来，它更加的灵活！

Topic主题模式的Routingkey一般都是有一个或多个单词组成，多个单词之间以”.”分割，例如： log.insert ，它的通配符规则如下：

`*`：匹配不多不少恰好1个词

`#`：匹配0或多个单词

简单举例：

```
log.*：只能匹配log.error，log.info等
log.#：能够匹配log.insert，log.insert.abc，log.news.update.abc等
```


![](https://files.mdnice.com/user/34714/4f242f60-0a74-4854-a4b7-451cd5bb853d.png)


![](https://files.mdnice.com/user/34714/9fa6c629-d9ff-4c31-9f1e-f19e0d52e92b.png)

图解：

1. 红色Queue：绑定的是usa.# ，因此凡是以`usa.`开头的routing key都会被匹配到

2. 黄色Queue：绑定的是#.news ，因此凡是以`.news`结尾的 routing key都会被匹配

- 创建生产者

```
package com.olive;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;

/**
 * 生产者（Topic主题模式）
 */
public class TopicProducer {

    // 交换机名称
    private static final String EXCHANGE_NAME = "topic_exchange";

    public static void main(String[] args) throws Exception {
        // 1、创建连接
        Connection connection = ConnectionUtils.getConnection();
        // 2、创建通道（频道）
        Channel channel = connection.createChannel();
        // 3、发送消息
        for (int i = 0; i < 4; i++) {
            String routingKey = "";
            //发送消息的时候根据相关逻辑指定相应的routing key。
            switch (i) {
                case 0:  //假设i=0，为select消息
                    routingKey = "log.select";
                    break;
                case 1: //假设i=1，为info消息
                    routingKey = "log.delete";
                    break;
                case 2: //假设i=2，为log.news.add消息
                    routingKey = "log.news.add";
                    break;
                case 3: //假设i=3，为log.news.update消息
                    routingKey = "log.news.update";
                    break;
            }
            // 要发送的消息
            String message = "Hello Message!!!~~~" + routingKey;
            // 消息发送 channel.basicPublish(交换机名称,路由key,消息其它属性,消息内容)
            channel.basicPublish(EXCHANGE_NAME, routingKey, null, message.getBytes("utf-8"));
            System.out.println("生产者发送的消息：" + message);
        }
        // 关闭资源
        channel.close();
        connection.close();
    }
}
```

- 创建消费者

**消费者1**

接收所有与`log.*`相匹配的路由key队列中的消息

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
 * 消费者（Topic模式）
 */
public class TopicConsumer1 {

    // 队列名称
    private static final String QUEUE_NAME1 = "topic_queue1";
    // 交换机名称
    private static final String EXCHANGE_NAME = "topic_exchange";

    public static void main(String[] args) throws Exception {
        // 1、获取连接对象
        Connection connection = ConnectionUtils.getConnection();
        // 2、创建通道（频道）
        Channel channel = connection.createChannel();
        // 3、声明交换机(有则不创建,无则创建) channel.exchangeDeclare(交换机名字,交换机类型,是否持久化)
        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.TOPIC, true);
        // 4、声明队列Queue channel.queueDeclare(队列名称，是否持久化，是否独占本连接,是否自动删除,附加参数)
        channel.queueDeclare(QUEUE_NAME1, true, false, false, null);

        // 5、根据指定的routingKey绑定队列和交换机,设置路由key channel.queueBind(队列名, 交换机名, 路由key)
        channel.queueBind(QUEUE_NAME1, EXCHANGE_NAME, "log.*");

        // 6、监听队列，接收消息
        DefaultConsumer defaultConsumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                //获取路由的key
                String routingKey = envelope.getRoutingKey();
                //获取交换机信息
                String exchange = envelope.getExchange();
                //获取消息信息
                String message = new String(body, "utf-8");
                System.out.println("路由Key:" + routingKey + ", 交换机名称:" + exchange + ", 消费者获取消息: " + message);
            }
        };
        channel.basicConsume(QUEUE_NAME1, true, defaultConsumer);

        //注意，消费者这里不建议关闭资源，让程序一直处于读取消息的状态
    }
}
```

**消费者2**

接收所有与`log.``#相匹配的路由key队列中的消息

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
 * 消费者（Topic模式）
 */
public class TopicConsumer2 {

    // 队列名称
    private static final String QUEUE_NAME2 = "topic_queue2";
    // 交换机名称
    private static final String EXCHANGE_NAME = "topic_exchange";

    public static void main(String[] args) throws Exception {
        // 1、获取连接对象
        Connection connection = ConnectionUtils.getConnection();
        // 2、创建通道（频道）
        Channel channel = connection.createChannel();
        // 3、声明交换机(有则不创建,无则创建) channel.exchangeDeclare(交换机名字,交换机类型,是否持久化)
        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.TOPIC, true);
        // 4、声明队列Queue。channel.queueDeclare(队列名称，是否持久化，是否独占本连接,是否自动删除,附加参数)
        channel.queueDeclare(QUEUE_NAME2, true, false, false, null);

        // 5、根据指定的routingKey绑定队列和交换机 channel.queueBind(队列名, 交换机名, 路由key)
        channel.queueBind(QUEUE_NAME2, EXCHANGE_NAME, "log.#");

        // 6、监听队列，接收消息
        DefaultConsumer defaultConsumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                //获取路由的key
                String routingKey = envelope.getRoutingKey();
                //获取交换机信息
                String exchange = envelope.getExchange();
                //获取消息信息
                String message = new String(body, "utf-8");
                System.out.println("路由Key:" + routingKey + ", 交换机名称:" + exchange + ", 消费者获取消息: " + message);
            }
        };
        channel.basicConsume(QUEUE_NAME2, true, defaultConsumer);
    }
}
```

分别先启动所有消费者，再使用生产者发送消息；在消费者对应的控制台可以查看到生产者发送对应routing key对应队列的消息；达到**按照需要接收**的效果。

消费者1的路由key匹配规则为`log.*`，所有该路由规则的绑定的队列应该只有2条信息。

![](https://files.mdnice.com/user/34714/6dcd73a6-f964-43d4-9064-36571cbe1351.png)

消费者2的路由key匹配规则为`log.#`，它能够匹配以log.开头的所有路由key，所有该路由规则的绑定的队列应该只有4条信息。

![](https://files.mdnice.com/user/34714/7868aee5-17b3-4e6f-9843-053c369661f1.png)

从RabbitMQ管理后台也能看到对应的交换机，以及队列绑定情况

![](https://files.mdnice.com/user/34714/ae53ece3-5fe0-44b0-95ef-443339880981.png)

- 总结

Topic主题模式需要设置类型为topic的交换机，交换机和队列进行绑定，并且指定通配符方式的routing key，当发送消息到交换机后，交换机会根据routing key将消息发送到对应的队列。

Topic主题模式可以实现Publish/Subscribe发布与订阅模式 和Routing路由模式的功能；只是Topic在配置routing key 的时候可以使用通配符，所以更加灵活。

```
来源
cnblogs.com/tanghaorong/p/14992330.html#_label0
```