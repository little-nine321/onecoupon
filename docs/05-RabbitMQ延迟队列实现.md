# RabbitMQ 延迟队列实现

## 1. 说明

你现在的简历口径是 RabbitMQ，但牛券当前源码和课程文档里的延时消息实现是 RocketMQ。

如果要迁移到 RabbitMQ，延迟任务最常见的两种做法是：

1. `TTL + 死信交换机（DLX）`
2. `x-delayed-message` 插件

对你这个项目来说，可以这样理解：

- `预约提醒` 更接近“任意延迟时间触发”
- `超时关单` 更接近“固定超时时间触发”

所以通常建议：

- 预约提醒优先考虑 `x-delayed-message`
- 超时关单优先考虑 `TTL + DLX`

官方资料：

- [RabbitMQ TTL](https://www.rabbitmq.com/docs/4.0/ttl)
- [RabbitMQ Dead Letter Exchanges](https://www.rabbitmq.com/docs/dlx)
- [RabbitMQ delayed message plugin](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange)

## 2. 方案一：TTL + 死信交换机（DLX）

### 适用场景

更适合固定延迟档位，例如：

- 订单 15 分钟未支付自动关单
- 订单 30 分钟未支付自动关单
- 开售前 5 分钟、15 分钟、30 分钟提醒

### 基本原理

先把消息发到延迟队列，消息在这个队列里等待 TTL 到期。  
到期后，RabbitMQ 会把消息死信投递到目标交换机，再路由到真正的业务消费队列。

链路如下：

```text
生产者 -> 延迟交换机 -> 延迟队列(TTL) -> 死信交换机 -> 业务队列 -> 消费者
```

### Spring AMQP 配置示例

```java
@Configuration
public class RabbitDelayQueueConfig {

    public static final String ORDER_EVENT_EXCHANGE = "ticket.order.event.exchange";
    public static final String ORDER_DELAY_QUEUE = "ticket.order.delay.30m.queue";
    public static final String ORDER_RELEASE_QUEUE = "ticket.order.release.queue";
    public static final String ORDER_DELAY_ROUTING_KEY = "ticket.order.delay.30m";
    public static final String ORDER_RELEASE_ROUTING_KEY = "ticket.order.release";

    @Bean
    public DirectExchange orderEventExchange() {
        return new DirectExchange(ORDER_EVENT_EXCHANGE, true, false);
    }

    @Bean
    public Queue orderDelayQueue() {
        Map<String, Object> args = new HashMap<>();
        args.put("x-message-ttl", 30 * 60 * 1000);
        args.put("x-dead-letter-exchange", ORDER_EVENT_EXCHANGE);
        args.put("x-dead-letter-routing-key", ORDER_RELEASE_ROUTING_KEY);
        return new Queue(ORDER_DELAY_QUEUE, true, false, false, args);
    }

    @Bean
    public Queue orderReleaseQueue() {
        return new Queue(ORDER_RELEASE_QUEUE, true);
    }

    @Bean
    public Binding orderDelayBinding() {
        return BindingBuilder.bind(orderDelayQueue())
                .to(orderEventExchange())
                .with(ORDER_DELAY_ROUTING_KEY);
    }

    @Bean
    public Binding orderReleaseBinding() {
        return BindingBuilder.bind(orderReleaseQueue())
                .to(orderEventExchange())
                .with(ORDER_RELEASE_ROUTING_KEY);
    }
}
```

### 发送延时消息

```java
public void sendCloseOrderDelayMessage(OrderTimeoutEvent event) {
    rabbitTemplate.convertAndSend(
            RabbitDelayQueueConfig.ORDER_EVENT_EXCHANGE,
            RabbitDelayQueueConfig.ORDER_DELAY_ROUTING_KEY,
            event,
            message -> {
                message.getMessageProperties().setMessageId(String.valueOf(event.getOrderId()));
                message.getMessageProperties().setCorrelationId(String.valueOf(event.getOrderId()));
                return message;
            }
    );
}
```

### 消费超时关单消息

```java
@RabbitListener(queues = RabbitDelayQueueConfig.ORDER_RELEASE_QUEUE)
public void handleCloseOrder(OrderTimeoutEvent event,
                             Message message,
                             Channel channel) throws IOException {
    long deliveryTag = message.getMessageProperties().getDeliveryTag();
    try {
        orderService.closeIfUnpaid(event);
        channel.basicAck(deliveryTag, false);
    } catch (Exception ex) {
        channel.basicNack(deliveryTag, false, true);
    }
}
```

### 优点

- 不依赖额外插件
- RabbitMQ 原生能力即可实现
- 适合固定超时时间的订单类任务

### 缺点

- 更适合少量固定延迟档位，不适合大量任意延迟时间
- 如果一个队列里混很多延迟时间，维护成本和时效控制会更复杂

## 3. 方案二：`x-delayed-message` 插件

### 适用场景

更适合“任意 delay”场景，例如：

- 开售前 7 分钟提醒
- 开售前 12 分钟提醒
- 开售前 25 分钟提醒

这类场景和当前 RocketMQ 的 `delayTime` 更接近。

### 安装插件

```bash
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```

要注意两点：

- 插件版本要和 RabbitMQ 大版本匹配
- 插件官方明确说明，它更适合秒、分、小时级延迟，不适合长周期调度

### 交换机配置示例

```java
@Configuration
public class RabbitDelayPluginConfig {

    public static final String REMIND_DELAY_EXCHANGE = "ticket.remind.delay.exchange";
    public static final String REMIND_QUEUE = "ticket.remind.queue";
    public static final String REMIND_ROUTING_KEY = "ticket.remind";

    @Bean
    public CustomExchange remindDelayExchange() {
        Map<String, Object> args = new HashMap<>();
        args.put("x-delayed-type", "direct");
        return new CustomExchange(
                REMIND_DELAY_EXCHANGE,
                "x-delayed-message",
                true,
                false,
                args
        );
    }

    @Bean
    public Queue remindQueue() {
        return new Queue(REMIND_QUEUE, true);
    }

    @Bean
    public Binding remindBinding() {
        return BindingBuilder.bind(remindQueue())
                .to(remindDelayExchange())
                .with(REMIND_ROUTING_KEY)
                .noargs();
    }
}
```

### 发送任意延迟消息

```java
public void sendRemindDelayMessage(CouponTemplateRemindDelayEvent event, long delayMs) {
    rabbitTemplate.convertAndSend(
            RabbitDelayPluginConfig.REMIND_DELAY_EXCHANGE,
            RabbitDelayPluginConfig.REMIND_ROUTING_KEY,
            event,
            message -> {
                message.getMessageProperties().setHeader("x-delay", delayMs);
                message.getMessageProperties().setMessageId(
                        event.getUserId() + ":" + event.getCouponTemplateId()
                );
                return message;
            }
    );
}
```

### 消费提醒消息

```java
@RabbitListener(queues = RabbitDelayPluginConfig.REMIND_QUEUE)
public void handleRemind(CouponTemplateRemindDelayEvent event,
                         Message message,
                         Channel channel) throws IOException {
    long deliveryTag = message.getMessageProperties().getDeliveryTag();
    try {
        remindExecutor.execute(event);
        channel.basicAck(deliveryTag, false);
    } catch (Exception ex) {
        channel.basicNack(deliveryTag, false, true);
    }
}
```

### 优点

- 支持任意毫秒级延迟
- 更接近 RocketMQ 里的任意 `delayTime`
- 拓扑更简单，不需要为每个延迟档位建队列

### 缺点

- 依赖插件
- 官方插件明确提醒有设计限制，不适合长期调度和海量延迟消息
- 插件停用时，未投递的延迟消息会丢失

## 4. 这个项目里到底该怎么选

### 场景一：预约提醒

当前预约提醒来自：

```text
delayTime = startTime - remindTime
```

这本质上是任意延迟时间。

更合适的方案：

- 优先 `x-delayed-message` 插件

原因：

- 比 TTL 多队列方案更贴近当前 RocketMQ 思路
- 不需要为每个提醒时间额外建一个队列

### 场景二：超时关单

如果你们业务里是固定 15 分钟、30 分钟自动关单：

- 优先 `TTL + DLX`

原因：

- 不需要插件
- 固定延迟更好维护
- 对订单系统足够稳定和直接

## 5. 和当前文档里的两个亮点如何对应

### 对应亮点一：MQ 幂等消费

RabbitMQ 里同样可能发生：

- 重复投递
- 消费失败重试
- 网络抖动导致重复消费

所以无论你选 TTL+DLX 还是 delayed plugin，都要保留消费端幂等。

建议：

- 使用 `messageId`、`correlationId` 或自定义业务 Header 作为幂等 Key
- 消费成功后再 ack
- 消费失败时让消息重试或进入死信队列

### 对应亮点二：延时任务 + 状态流转

RabbitMQ 只负责“到点把消息送过来”，真正的可靠性仍然要靠业务层补齐：

- 任务先落库
- 消费端再次校验业务状态
- 线程池异步执行
- Redisson 延时队列回查任务是否丢失
- 分布式锁 + 事务保证状态流转一致性

所以如果你换成 RabbitMQ，业务骨架是不变的，换的是“延时触发入口”。

## 6. 对你这个项目的推荐口径

如果你要把项目继续包装成“景区订票项目”，建议这样说：

### 预约提醒

> 基于 RabbitMQ 延时消息实现开售提醒，到点后由消费者触发通知任务，并结合线程池 + Redisson 延时队列保证提醒任务的异步执行与可靠补偿。

### 超时关单

> 基于 RabbitMQ `TTL + DLX` 实现订单超时关闭，到期后异步触发关单逻辑，并结合分布式锁与事务完成订单状态回退和库存恢复。

## 7. 落地建议

如果你面试时不想把自己讲复杂，最稳的表达是：

1. `预约提醒` 用 RabbitMQ delayed plugin
2. `超时关单` 用 RabbitMQ TTL + DLX
3. `幂等、锁、事务、补偿` 仍然沿用当前文档里的思路

这样逻辑最顺，也最像真实项目演进路径。
