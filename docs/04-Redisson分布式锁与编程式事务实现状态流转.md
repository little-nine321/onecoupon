# 基于 Redisson 分布式锁 + 编程式事务实现状态流转

## 1. 先说结论

当前仓库里真正落地的是：

- `创建结算单并锁定优惠券`
- `支付后核销优惠券`
- `退款后回退优惠券`

源码中对应的方法分别是：

- `createPaymentRecord`
- `processPayment`
- `processRefund`

它们共同使用了两层保护：

1. **Redisson 分布式锁**：防止同一张券被并发重复操作
2. **TransactionTemplate 编程式事务**：保证多表更新要么一起成功，要么一起回滚

如果迁移到景区订票口径，你可以把它理解成：

- 锁定订单
- 核销门票/确认支付
- 退款回退库存与票实例状态

## 2. 相关源码位置

### 核心服务

- `code/onecoupon/engine/src/main/java/com/nageoffer/onecoupon/engine/service/impl/UserCouponServiceImpl.java`

### 相关表

- `code/onecoupon/engine/src/main/java/com/nageoffer/onecoupon/engine/dao/entity/UserCouponDO.java`
- `code/onecoupon/engine/src/main/java/com/nageoffer/onecoupon/engine/dao/entity/CouponSettlementDO.java`

### 锁 Key

- `code/onecoupon/engine/src/main/java/com/nageoffer/onecoupon/engine/common/constant/EngineRedisConstant.java`

其中结算锁 Key 是：

```java
one-coupon_engine:lock:coupon-settlement:%d
```

也就是：

- 一张券一个锁
- 同一张券相关的锁定、支付、退款操作共用同一把锁

## 3. 相关状态设计

### 用户券实例状态 `t_user_coupon.status`

- `0`：未使用
- `1`：锁定中
- `2`：已使用
- `3`：已过期
- `4`：已撤回

### 结算单状态 `t_coupon_settlement.status`

- `0`：锁定
- `1`：已取消
- `2`：已支付
- `3`：已退款

你可以把这两张表的职责理解成：

- `t_user_coupon`：这张券当前还能不能被再次使用
- `t_coupon_settlement`：这张券在某个订单维度下走到了哪个业务阶段

## 4. 为什么这里只靠事务不够

如果没有分布式锁，只靠事务，仍然会出现并发问题。

例如两次并发请求同时操作同一张券：

1. 请求 A 查询到券是 `未使用`
2. 请求 B 也查询到券是 `未使用`
3. 两边都进入事务
4. 两边都尝试创建结算单、修改状态

这时就会出现：

- 重复锁定
- 重复支付
- 状态覆盖

所以：

- **锁**解决“同一资源同时被多个线程处理”
- **事务**解决“本次处理里多步更新必须保持原子性”

两者不是替代关系，而是配合关系。

## 5. 创建结算单并锁定优惠券是怎么做的

方法位置：

- `UserCouponServiceImpl#createPaymentRecord`

### 第一步：先拿分布式锁

```java
RLock lock = redissonClient.getLock(
    String.format(LOCK_COUPON_SETTLEMENT_KEY, requestParam.getCouponId())
);
boolean tryLock = lock.tryLock();
```

如果拿锁失败，直接报错：

- 说明这张券正在被别的请求处理

这一步主要防止：

- 用户连续点击两次提交
- 订单服务重复调用
- 不同节点同时处理同一张券

### 第二步：先做非事务校验

进入事务之前，会先做一批业务校验：

1. 查询 `t_coupon_settlement` 是否已经存在状态为 `锁定` 或 `已支付` 的记录
2. 查询 `t_user_coupon` 是否存在
3. 校验券是否过期
4. 校验当前状态是否为 `未使用`
5. 校验订单金额和优惠规则是否匹配

这样设计的原因是：

- 这些都是纯校验，不需要占用事务资源
- 事务范围越小，锁竞争越低

### 第三步：进入编程式事务

项目没有把整个方法都加 `@Transactional`，而是用：

```java
transactionTemplate.executeWithoutResult(status -> {
    ...
});
```

原因是：

- 只把真正需要原子提交的部分包进事务
- 缩小事务范围

### 第四步：事务内完成两件事

#### 1. 插入结算单

```java
CouponSettlementDO couponSettlementDO = CouponSettlementDO.builder()
    .orderId(requestParam.getOrderId())
    .couponId(requestParam.getCouponId())
    .userId(currentUserId)
    .status(0)
    .build();
couponSettlementMapper.insert(couponSettlementDO);
```

表示：

- 当前这张券已经被这张订单锁住了

#### 2. 修改券实例状态

```java
update t_user_coupon
set status = LOCKING
where id = ?
  and user_id = ?
  and status = UNUSED
```

这里用了条件更新，而不是先查后改。

好处是：

- 它本身就带有 CAS 味道
- 只有当前状态确实是 `未使用` 才允许改成 `锁定中`

### 第五步：提交成功后移出可用缓存

锁定成功后，会把这张券从用户可用券 ZSet 里移除，避免继续出现在“可用列表”里。

## 6. 支付后核销是怎么做的

方法位置：

- `UserCouponServiceImpl#processPayment`

这一步你可以迁移成：

- 支付成功确认
- 核销门票
- 履约成功确认

### 流程

1. 先拿同一把分布式锁
2. 再进入 `TransactionTemplate`
3. 先更新结算单状态：`0 -> 2`
4. 再更新券实例状态：`1 -> 2`

### 结算单状态推进

```java
update t_coupon_settlement
set status = 2
where coupon_id = ?
  and user_id = ?
  and status = 0
```

含义：

- 只有当前是“锁定”状态，才允许改成“已支付”

### 券实例状态推进

```java
update t_user_coupon
set status = USED
where id = ?
  and user_id = ?
  and status = LOCKING
```

含义：

- 只有当前是“锁定中”，才允许改成“已使用”

这就是典型的状态机条件更新。

它的价值在于：

- 即使上层出现重复请求
- 只要状态不满足条件，就不会重复推进状态

## 7. 退款回退是怎么做的

方法位置：

- `UserCouponServiceImpl#processRefund`

### 流程

1. 先拿同一把分布式锁
2. 进入 `TransactionTemplate`
3. 更新结算单状态：`2 -> 3`
4. 更新券实例状态：`2 -> 0`
5. 事务提交后，把券重新放回用户可用缓存

### 结算单状态回退

```java
update t_coupon_settlement
set status = 3
where coupon_id = ?
  and user_id = ?
  and status = 2
```

含义：

- 只有已支付的结算单才能退款

### 券实例状态回退

```java
update t_user_coupon
set status = UNUSED
where id = ?
  and user_id = ?
  and status = USED
```

含义：

- 只有已使用的券，才能回退成未使用

### 为什么事务提交后还要回写缓存

退款回退之后，这张券又重新可用了，所以要把它重新放回用户可用券列表缓存。

这一步放在事务外面是可以接受的，因为：

- 数据库状态才是最终真相
- 缓存只是加速层

如果缓存更新失败，也可以后续补偿或通过失效重建恢复。

## 8. 为什么说这是“分布式锁 + 编程式事务”的组合

你可以把这套实现拆成三层：

### 第一层：资源级互斥

`RLock lock = redissonClient.getLock(...)`

作用：

- 同一张券同一时刻只允许一个请求处理

### 第二层：事务内多表原子更新

`transactionTemplate.executeWithoutResult(...)`

作用：

- 结算单表和券实例表要么一起成功，要么一起失败

### 第三层：状态机条件更新

`where status = oldStatus`

作用：

- 即使并发校验漏过一层，最终更新时也只能按合法状态迁移

这三层叠加后，系统就能比较稳地保证状态一致性。

## 9. 如果要扩展成“超时关单”怎么做

当前代码没有直接写出“超时关单”接口，但按照现有设计，很容易补上。

思路如下：

1. 下单时先调用 `createPaymentRecord`，把状态改成：
   - 结算单：`锁定`
   - 券实例：`锁定中`
2. 发送一条延时消息，例如 30 分钟后触发
3. 到点后消费端检查：
   - 结算单是否仍为 `锁定`
   - 订单是否仍未支付
4. 若是，则加同一把分布式锁
5. 事务内执行：
   - 结算单：`0 -> 1`
   - 券实例：`1 -> 0`
6. 把券放回可用缓存

所以你简历里写“超时关单等任务”的依据就是：

- 当前代码已经具备完整的锁定、支付、退款回退机制
- 只差一个到期触发入口
- 业务骨架已经是现成的

## 10. 面试时怎么讲最稳

你可以直接这样说：

> 状态流转这块我没有只靠数据库事务，而是做了三层保护。第一层是按券实例维度加 Redisson 分布式锁，保证同一张券不会被并发重复锁定或重复退款；第二层是用 TransactionTemplate 把结算单和券实例状态更新放进同一个事务，确保多表原子提交；第三层是在 update 语句里带上旧状态条件，把状态推进做成有限状态机，例如只允许未使用变锁定、锁定变已使用、已使用变未使用，从而保证并发场景下的状态一致性。

## 11. 如果改成 RabbitMQ，需要修改的部分

这篇文档的核心代码其实和 MQ 品牌关系不大。

真正不变的部分：

- `Redisson` 分布式锁
- `TransactionTemplate` 编程式事务
- `t_user_coupon` 和 `t_coupon_settlement` 的双表状态流转
- `where status = oldStatus` 这种条件更新

真正需要改的是“谁来触发这些方法”：

- 如果是支付成功后核销，从 RocketMQ 消费者改为 RabbitMQ 消费者触发 `processPayment`。
- 如果是退款成功后回退，从 RocketMQ 消费者改为 RabbitMQ 消费者触发 `processRefund`。
- 如果要做超时关单，则由 RabbitMQ 的延时消息在到期后触发“锁定 -> 取消、券状态回退”的逻辑。

RabbitMQ 版本里建议额外注意：

- 支付、退款、超时关单这些消息同样要做消费端幂等。
- 触发状态流转的方法前，仍然建议按 `couponId` 或 `orderId + couponId` 维度加分布式锁。
- 如果 RabbitMQ 消息重复投递，最终还是要靠“幂等 + 锁 + 条件更新”三层保证一致性。

所以，如果改成 RabbitMQ，这篇文档对应的结论可以简化成：

> MQ 只负责触发状态流转，真正保证一致性的仍然是 Redisson 分布式锁、编程式事务和状态机条件更新。

可以配合阅读：

- [05-RabbitMQ延迟队列实现.md](D:\study\java\oneCoupon\docs\05-RabbitMQ延迟队列实现.md)

## 12. 映射到景区购票系统时应该怎么理解

这篇文档里的双表模型，**不能直接等同成“标准订单表 + 标准门票表”**，更准确的映射是：

- `t_user_coupon`：更接近 **门票实例表 / 用户持有门票表**
- `t_coupon_settlement`：更接近 **订单与门票的关联表 / 票务结算表**

### `t_user_coupon` 对应什么

如果迁移到景区购票系统，`t_user_coupon` 可以理解成：

- 用户买到手的一张具体门票
- 这张门票当前是否可用
- 这张门票是否已被占用、已核销、已过期

也就是说，它记录的是“**门票实例本身的状态**”。

### `t_coupon_settlement` 对应什么

如果迁移到景区购票系统，`t_coupon_settlement` 更适合理解成：

- 哪一笔订单占用了哪一张门票
- 当前订单关联的票务处理走到了哪个阶段
- 是待处理、已确认、已取消，还是已退款

也就是说，它记录的是“**订单和门票之间的业务关系与流转状态**”。

### 为什么不能直接说它就是订单表

因为当前源码里的 `t_coupon_settlement` 只有：

- `order_id`
- `coupon_id`
- `user_id`
- `status`

它缺少完整订单域常见字段，比如：

- 订单号
- 订单金额
- 支付渠道
- 支付时间
- 关单时间
- 游客信息
- 场次信息

所以它更像“**订单关联记录/结算记录**”，而不是一个完整的订单主表。

### 如果你的票务系统只想保留两张核心表

可以把它改讲成：

- 一张 **订单表**：记录 `待支付 / 已取消 / 已支付`
- 一张 **门票表**：记录 `未使用 / 已核销 / 已过期`

这时本篇文档能提供的主要价值是：

- 如何用分布式锁防止同一张票被并发重复处理
- 如何用编程式事务保证多表状态同时提交
- 如何用 `where status = oldStatus` 做状态机条件更新

### 如果你想沿用本篇文档的双表思路

那最稳的口径是：

- `t_user_coupon -> 门票实例表`
- `t_coupon_settlement -> 订单门票关联表`

不要直接说：

- `t_coupon_settlement -> 订单表`

这样讲更贴近源码，也更经得起面试追问。
