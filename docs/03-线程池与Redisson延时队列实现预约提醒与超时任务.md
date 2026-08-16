# 基于线程池 + Redisson 延时队列实现预约提醒与可靠消费

## 1. 先说结论

当前仓库里真正落地的是“预约提醒”能力，不是“超时关单”能力。

但这套模式完全可以迁移到超时关单，核心结构是一样的：

1. 主链路只做轻量校验和持久化
2. 通过延时 MQ 在未来某个时间点触发任务
3. 消费后把真正耗时的通知动作扔到线程池
4. 用 Redisson 延时队列做二次兜底，防止线程池任务丢失

所以你简历里写：

> 基于线程池 + Redisson 延时队列实现预约提醒、超时关单等任务的异步触发与可靠消费

这句话在“设计模式层面”是成立的，但要如实知道：当前源码直接实现的是“预约提醒”，超时关单需要按同样模式迁移扩展。

## 2. 相关源码位置

### 预约提醒主流程

- `code/onecoupon/engine/src/main/java/com/nageoffer/onecoupon/engine/service/impl/CouponTemplateServiceRemindImpl.java`

### 延时消息生产者

- `code/onecoupon/engine/src/main/java/com/nageoffer/onecoupon/engine/mq/producer/CouponTemplateRemindDelayProducer.java`

### 延时消息消费者

- `code/onecoupon/engine/src/main/java/com/nageoffer/onecoupon/engine/mq/consumer/CouponTemplateRemindDelayConsumer.java`

### 提醒任务执行器

- `code/onecoupon/engine/src/main/java/com/nageoffer/onecoupon/engine/service/handler/remind/CouponTemplateRemindExecutor.java`

### 提醒位图工具

- `code/onecoupon/engine/src/main/java/com/nageoffer/onecoupon/engine/toolkit/CouponTemplateRemindUtil.java`

### 相关 Redis Key

- `code/onecoupon/engine/src/main/java/com/nageoffer/onecoupon/engine/common/constant/EngineRedisConstant.java`

## 3. 预约提醒的数据建模

提醒主表是 `t_coupon_template_remind`，主键为：

- `user_id`
- `coupon_template_id`

这意味着：

- 一个用户针对一张模板，只保留一条提醒主记录。
- 如果同一用户对同一张券设置多个提醒时间/提醒方式，不新增多行，而是把配置合并进 `information`。

```sql
CREATE TABLE `t_coupon_template_remind`
(
    `user_id`            bigint(20) NOT NULL COMMENT '用户ID',
    `coupon_template_id` bigint(20) NOT NULL COMMENT '券ID',
    `information`        bigint(20) DEFAULT NULL COMMENT '存储信息',
    `shop_number`        bigint(20) DEFAULT NULL COMMENT '店铺编号',
    `start_time`         datetime DEFAULT NULL COMMENT '优惠券开抢时间',
    PRIMARY KEY (`user_id`, `coupon_template_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户预约提醒信息存储表';
```



### `information` 字段为什么用位图

位图工具 `CouponTemplateRemindUtil` 里写死了两件事：

1. 以 5 分钟为一个时间间隔
2. 每种提醒方式占用 12 个 bit

也就是说：

- 最多表示开抢前 60 分钟内的提醒
- 同时支持多种提醒方式，例如 App、Email

这套设计的好处是：

- 一条记录就能表示多种提醒配置
- 查询、取消时只需要做位运算
- 不需要为每个提醒时间单独插一行

## 4. 创建预约提醒时做了什么

逻辑在 `CouponTemplateServiceRemindImpl#createCouponRemind`。

### 第一步：校验模板是否存在、是否已开抢

先查询券模板：

1. 模板必须存在
2. `valid_start_time` 不能早于当前时间

这是为了防止：

- 给不存在的券创建提醒
- 给已经开始抢购的券创建提醒

### 第二步：写入提醒表

如果用户之前没有创建过提醒：

- 插入一条 `t_coupon_template_remind`
- 设置 `start_time`
- 计算 `information` 位图

如果之前已经创建过提醒：

1. 读取当前 `information`
2. 计算本次提醒对应的 bit
3. 如果位图已存在，直接报“已创建过提醒”
4. 否则用位运算把新的提醒合并进去

### 第三步：发送延时消息

创建提醒成功后，会构造一个 `CouponTemplateRemindDelayEvent`：

- `couponTemplateId`
- `userId`
- `type`
- `remindTime`
- `startTime`
- `delayTime`

其中最关键的是：

```java
delayTime = 开抢时间 - 提醒提前量
```

也就是：

- 比如门票 10:00 开售
- 用户设置“提前 15 分钟提醒”
- 那就发送一个“在 09:45 触发”的延时消息

## 5. RocketMQ 在这里承担什么角色

当前仓库用的是 RocketMQ 延时消息。

生产者 `CouponTemplateRemindDelayProducer` 会把消息发到提醒 Topic，并带上业务 Key：

```java
keys = userId + ":" + couponTemplateId
delayTime = messageSendEvent.getDelayTime()
```

它的作用不是“立刻通知”，而是：

- 在未来某个时间点唤醒这条提醒任务

这点对超时关单也是一样的。

如果迁移成订单系统：

- 下单成功时发一条“30 分钟后触发”的延时消息
- 到点后消费端检查订单是否仍未支付
- 若仍未支付，则执行关单

## 6. 消费端怎么把提醒任务异步化

延时消息到点后，`CouponTemplateRemindDelayConsumer` 会消费消息，然后调用：

```java
couponTemplateRemindExecutor.executeRemindCouponTemplate(...)
```

真正的提醒发送不在 MQ 线程里直接做，而是在 `CouponTemplateRemindExecutor` 里交给线程池。

线程池配置是：

```java
new ThreadPoolExecutor(
    CPU * 2,
    CPU * 4,
    60,
    TimeUnit.SECONDS,
    new SynchronousQueue<>(),
    new ThreadPoolExecutor.CallerRunsPolicy()
)
```

这么做的原因很明确：

- 发送 App/Email 通知属于 IO 密集型任务
- 不应该阻塞 MQ 消费线程
- 线程池能提高并发通知吞吐

## 7. 为什么还需要 Redisson 延时队列兜底

如果只把任务扔给线程池，会有一个风险：

> 消息已经被 MQ 消费成功了，但线程池任务还没执行完，应用就宕机了。

这时：

- MQ 不会再投这条消息
- 线程池里的提醒任务又丢了

这就是“消费成功但业务未完成”的典型半成功问题。

为了解决这个问题，项目又做了一层本地异步后的可靠兜底。

## 8. Redisson 延时队列是怎么兜底的

关键代码在 `CouponTemplateRemindExecutor#executeRemindCouponTemplate`。

### 提交线程池前先做两件事

1. 生成一个 Redis 检查 Key：

```java
COUPON_REMIND_CHECK_KEY = one-coupon_engine:coupon-remind-check:%s_%s_%d_%d
```

这个 Key 包含：

- 用户 ID
- 模板 ID
- 提前时间
- 提醒方式

2. 把这个 Key 放进 Redisson 延时队列，延迟 10 秒再可见：

```java
RBlockingDeque<String> blockingDeque = redissonClient.getBlockingDeque(...)
RDelayedQueue<String> delayedQueue = redissonClient.getDelayedQueue(blockingDeque)
stringRedisTemplate.opsForValue().set(key, dtoJson)
delayedQueue.offer(key, 10, TimeUnit.SECONDS)
```

### 线程池任务正常执行时

线程池执行真正的提醒发送：

- 如果是 App 提醒，调用 `sendAppMessageRemindCouponTemplate`
- 如果是 Email 提醒，调用 `sendEmailRemindCouponTemplate`

任务执行完成后，删除这个 Redis Key：

```java
stringRedisTemplate.delete(key)
```

### 兜底线程怎么检查任务是否丢了

系统启动时，内部类 `RefreshCouponRemindDelayQueueRunner` 会启动一个守护线程，持续监听 Redisson 阻塞队列：

```java
String key = blockingDeque.take();
if (stringRedisTemplate.hasKey(key)) {
    // 说明提醒任务未执行完成
    // 重新投递 MQ 消息
}
```

判断逻辑很简单：

- 如果 10 秒后这个 Key 还在
- 说明线程池任务大概率没有跑完，或者应用在执行前就挂了
- 于是重新组装消息并重新投递到 MQ

这样就形成了：

1. RocketMQ 负责“到点触发”
2. 线程池负责“并发执行耗时任务”
3. Redisson 延时队列负责“检查线程池任务是否丢失”

## 9. 为什么说它实现了“可靠消费”

这里的“可靠”不是 MQ 官方意义上的 exactly once，而是业务层面的“尽量不丢提醒”。

它的可靠性来自 3 层：

### 第一层：预约配置先落库

提醒请求先写 `t_coupon_template_remind`，所以任务配置不会因为进程重启而丢。

### 第二层：到点触发用延时 MQ

把“未来执行”交给 MQ，而不是自己在 JVM 内存里维护定时器。

### 第三层：线程池后再做 Redisson 二次兜底

防止“MQ 消费成功，但线程池任务丢了”的问题。

## 10. 取消提醒是怎么配合这套机制的

取消提醒逻辑在 `CouponTemplateServiceRemindImpl#cancelCouponRemind`。

核心点有 3 个：

1. 通过位图删除某个提醒时间/提醒方式
2. 用 `where information = oldInformation` 做乐观并发控制
3. 把取消信息写入布隆过滤器 `cancelRemindBloomFilter`

后续消费者在真正发提醒前，会先调用：

```java
couponTemplateRemindService.isCancelRemind(...)
```

如果用户已经取消提醒，即使 MQ 到点了，也不会真的再发通知。

所以这套系统不是“取消了就撤回 MQ 消息”，而是：

- 保留已发送的 MQ 延时消息
- 在消费侧二次校验是否已经取消

这是更常见、也更现实的做法。

## 11. 如何迁移到“超时关单”

当前源码没有直接实现超时关单，但完全可以复用这套模式。

### 迁移后的思路

1. 用户下单时，订单先写库，状态设为 `LOCKED/待支付`
2. 发送一条“30 分钟后触发”的延时消息
3. 延时消息到点后，消费者检查订单状态
4. 如果订单仍未支付，则执行关单
5. 关单后回退库存、回退票实例状态
6. 如果关单动作交给线程池异步执行，也可以继续用 Redisson 延时队列兜底

### 为什么简历里可以写“预约提醒、超时关单等任务”

因为这两个任务的技术骨架相同：

- 都是“未来某个时间点触发”
- 都需要异步执行
- 都需要失败兜底
- 都需要消费时再次校验当前状态

## 12. 面试时可以怎么总结

你可以直接说：

> 预约提醒这块我采用了“延时消息 + 线程池 + Redisson 延时队列补偿”的三段式设计。用户创建提醒后先落库，再发送延时消息，到点后消费者只负责触发任务，真正的通知动作交给线程池异步执行。为了避免 MQ 消费成功但线程池任务丢失的问题，我额外把检查 Key 放入 Redisson 延时队列，10 秒后回查任务是否完成，如果 Redis Key 仍存在就重新投递消息，从而保证提醒任务尽量不丢失。

## 13. 如果改成 RabbitMQ，需要修改的部分

这篇文档里真正需要替换的是“延时消息触发层”，不是“线程池 + Redisson 延时队列兜底层”。

需要替换的点：

- `CouponTemplateRemindDelayProducer` 里的 RocketMQ 延时发送，替换成 RabbitMQ 延时发布。
- `CouponTemplateRemindDelayConsumer` 里的 RocketMQ 消费监听，替换成 `@RabbitListener`。
- 如果你们 RabbitMQ 集群允许装插件，最接近当前 RocketMQ 任意延迟能力的做法是 `x-delayed-message` 插件。
- 如果不允许装插件，就用 `TTL + 死信交换机（DLX）` 实现固定档位延迟，例如 5 分钟、15 分钟、30 分钟。

建议怎么选：

- `预约提醒`：如果提醒时间是多个可变档位，优先用 RabbitMQ delayed message plugin。
- `超时关单`：如果通常只有固定超时时间，例如 15 分钟或 30 分钟，用 `TTL + DLX` 就够了。

不需要改的点：

- 提醒表 `t_coupon_template_remind`
- `information` 位图设计
- 取消提醒逻辑
- 布隆过滤器判断取消状态
- 线程池异步执行
- Redisson 延时队列回查 Redis Key 的兜底机制

也就是说，RabbitMQ 替换后，业务主链路仍然是：

1. 先落提醒配置
2. 再发延时消息
3. 消费时交给线程池
4. 用 Redisson 延时队列做本地异步任务补偿

可以配合阅读：

- [05-RabbitMQ延迟队列实现.md](D:\study\java\oneCoupon\docs\05-RabbitMQ延迟队列实现.md)
