MQ（Message Queue）是一种消息队列技术，它允许应用程序通过队列来异步交换消息。在分布式系统中，MQ通常用于解耦不同的服务或组件，提高系统的可伸缩性和可靠性。以下是MQ的一些核心概念：

- 消息（Message）：消息是MQ中传递的数据单元，它可以是任何类型的数据，通常包含一个负载（payload）和一组属性，如优先级、类型、目的地等。
- 队列（Queue）：队列是存储消息的容器，按照先进先出（FIFO）的原则管理消息。生产者将消息发送到队列，消费者从队列中接收消息。
- 生产者（Producer）：生产者是发送消息到队列的实体。它创建消息并将其发布到队列中。
- 消费者（Consumer）：消费者是从队列中接收消息的实体。它可以订阅一个或多个队列，并处理队列中的消息。
- 消息代理（Message Broker）：消息代理是实现消息队列功能的服务器软件。它负责接收、存储、转发消息，并确保消息的可靠传递。常见的消息代理有RabbitMQ、Apache Kafka、ActiveMQ、Amazon SQS等。

MQ的使用场景非常广泛，包括异步处理、应用解耦、流量削峰、数据同步等。通过使用MQ，系统可以更好地处理高峰期的负载，提高系统的整体性能和稳定性
**重点：通过队列来异步交互信息**

## mq 的实现 RabbitMQ
官网： https://www.rabbitmq.com/tutorials/tutorial-one-java
特点：使用 Erlang 开发 、 支持消息协议有：AMQP，XMPP，SMTP，STOMP 、可靠性强

### WorkQueue 
WorkQueue 是 mq 的中的一种机制针对的是消费者和队列之间的关系，特点：
1. 一个队列绑定多个消费者
2. 一个消息只能被一个消费者消费
3. 可以通过控制消费者预取消息的配置来实现能者多劳

### 交换机
  它负责接收生产者发送的消息，并根据一定的规则将这些消息路由到零个或多个队列中。交换机不存储消息，它只是在消息到达时根据绑定（binding）和路由键（routing key）将消息推送到合适的队列。

RabbitMQ中有几种不同类型的交换机，它们各自有不同的路由策略：
- 扇出交换机（Fanout Exchange）也叫广播交换机：这种交换机会将消息广播到所有绑定到它的队列，而不关心路由键。它适用于发布/订阅模式的场景。
 
- 直连交换机（Direct Exchange）：这种交换机会将消息路由到那些绑定键（binding key）与消息的路由键（routing key）完全匹配的队列。它是基于精确匹配的路由。

- 主题交换机（Topic Exchange）：这种交换机允许消息根据路由键的模式进行路由。绑定键可以包含特殊字符*（匹配一个单词）和#（匹配零个或多个单词）。它适用于根据特定模式路由消息的场景。

#### Fanout 交换机
Fanout 是RabbbitMQ 交换机的一种类型，特点：
1. 将消息发送给所有和他绑定的队列上
````java

@Configuration
public class FanoutExchangeConfig {

    @Bean
    public FanoutExchange fanoutExchange() {
        // 创建交换机
        return new FanoutExchange("hmall.fanout");
    }


    @Bean
    public Queue fanoutQueue1() {
        return new Queue("fanout.queue1");
    }

    @Bean
    public Queue fanoutQueue2() {
        return new Queue("fanout.queue2");
    }

    @Bean
    public Binding fanoutQueueBinding1(Queue fanoutQueue1, FanoutExchange fanoutExchange) {
        return BindingBuilder.bind(fanoutQueue1).to(fanoutExchange);
    }

    @Bean
    public Binding fanoutQueueBinding2(Queue fanoutQueue2, FanoutExchange fanoutExchange) {
        return BindingBuilder.bind(fanoutQueue2).to(fanoutExchange);
    }
}
````
#### Direct交换机
1. 将消息路由到 routerKey 和 BindingKey 一样的交换机上
2. 队列可以绑定对个BindingKey
3. 如果所有队列的BindingKey 都和RouterKey 一样就可以实现 Fanout交换机的效果
````java
 @RabbitListener(bindings = @QueueBinding(
            value = @Queue(name = "direct.queue1"),
            exchange = @Exchange(name = "hmall.direct", type = ExchangeTypes.DIRECT),
            key = {"red", "bleu"} // 设置多个BindingKey
    ))
    public void onDirect2Msg1(String msg) {
        System.out.println("消费者 1 接收的消息：【" + msg + "】,时间：" + LocalDateTime.now());
    }

````


#### Topic 交换机
1. 将消息路由到 routerKey 和 BindingKey 一样的交换机上
2. 队列可以绑定对个BindingKey
3. 如果所有队列的BindingKey 都和RouterKey 一样就可以实现 Fanout交换机的效果
4. 指定 RouterKey的时候必须使用 `.` 作为间隔符来声明 RouterKey ， RouterKey必须是多段的
5. 支持在BindingKey上使用通配符
    - `#`  配置0段或者任意段
    - `*`  能且只能匹配一段
 ````java
    @RabbitListener(bindings = @QueueBinding(
            value = @Queue(name = "topic.queue1"),
            exchange = @Exchange(name = "hmall.topic", type = ExchangeTypes.TOPIC),
            key = {"china.#"} // 匹配所有一china 开头的RouterKey 
    ))
    public void onDirect2Msg1(String msg) {
        System.out.println("消费者 1 接收的消息：【" + msg + "】,时间：" + LocalDateTime.now());
    }

````
**重点使用 @RabbitListener 声明队列、交换机、BindingKey以及他们之间的对应关系**
````java
@RabbitListener 表示监听队列
@QueueBinding( 
value @Queue  // 声明队列  主要属性：name 设置队列名字
exchange @Exchange // 声明交换机  主要的属性：name 设置交换机名字 type：设置交换机类型
key       String | List<String> 设置 BindingKey 
)
**/

@RabbitListener(bindings = @QueueBinding(
            value = @Queue(name = "队列名"),
            exchange = @Exchange(name = "交换机名" ,type = ExchangeTypes.DIRECT ),
            key = {"bindingKey", "bindingKey"}
    ))
    public void onDirect2Msg2(String msg) {....}
````

### 消息转换器
消息转换器其实就是 Java 对象序列化成字节数组、反序列化将字节数组转换为 Java 对象
在 RabbitMQ 底层使用 JDK 默认的消息转换器，使用默认的消息转换器有一些问题：
- 数据体积过大
- 有安全漏洞
- 可读性差
更为为 Jackson2JsonMessageConverter 消息转换器 （ 将Java对象转换为字节数组，然后解析成 json 格式 ）
步骤：
1. 导入 JackJson 坐标
```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>
````
2. 将 Jackson2JsonMessageConverter  注入到 IOC 容器中，RabbitMQ 底层就会自动使用这个消息转换器了 
````java
   /**
     * 使用 JackJson 来作为消息转换器
     *
     * @return
     */
    @Bean
    public MessageConverter jackson2JsonMessageConverter() {
        // 创建转换器
        Jackson2JsonMessageConverter messageConverter = new Jackson2JsonMessageConverter();
        // 设置每条消息有自己的 id
        messageConverter.setCreateMessageIds(true);
        return messageConverter;
    }
````
注意点：这消息转换需要同时配置给消费者和发送者。