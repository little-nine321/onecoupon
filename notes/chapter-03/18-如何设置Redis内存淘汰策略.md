# 第 18 小节：如何设置 Redis 内存淘汰策略

## 新手先读

本节没有复杂业务代码，重点是理解 Redis 内存满了以后会删哪些 key。不要把淘汰策略理解成“缓存过期时间”，过期时间是 key 自己到点失效，淘汰策略是 Redis 内存不够时被迫清理。

对 oneCoupon 来说，热门优惠券模板和用户券缓存访问频率高，冷门数据访问少，所以本节推荐从“保留热点数据”的角度理解 LFU。

## 1. 本节目标

本节不引入新的 Java 代码，重点学习 Redis 内存达到上限后的淘汰策略。学习目标：

- 理解 Redis 为什么需要 `maxmemory` 和 `maxmemory-policy`。
- 掌握 Redis 常见内存淘汰策略。
- 理解 `noeviction`、`volatile-lru`、`allkeys-lru`、`volatile-lfu`、`allkeys-lfu`、`volatile-ttl` 等策略差异。
- 理解 LRU 和 LFU 的基本原理。
- 理解牛券项目为什么更适合选择 `volatile-lfu`。
- 会在本地 Redis 查看和临时修改内存淘汰策略。
- 会结合牛券项目缓存 Key 判断哪些 Key 应该设置过期时间。

教程路径：

```text
D:\study\java\oneCoupon\第③章节：分发&引擎服务\《牛券oneCoupon优惠券系统设计》第18小节：如何设置Redis内存淘汰策略？.html
```

对应分支：

```text
本小节未发现独立代码分支。
建议结合上一节目标分支阅读：origin/20240827_dev_coupon-template-querypv-v2_cache_ding.ma
```

说明型 diff 附录：

```text
D:\study\java\oneCoupon\notes\diffs\18-如何设置Redis内存淘汰策略.diff
```

## 2. 教程核心内容提炼

Redis 是内存数据库，内存不是无限的。当 Redis 使用内存超过配置的最大值时，需要决定如何处理新写入或旧数据。

核心配置：

```text
maxmemory
maxmemory-policy
maxmemory-samples
```

教程结论是：牛券项目里的优惠券模板、空值缓存等 Key 大多有明确过期时间，同时业务存在明显冷热数据。为了在内存吃紧时优先保留访问频率更高的数据，推荐使用：

```text
volatile-lfu
```

## 3. 为什么本节没有代码 diff

本节讲的是 Redis 服务端配置和缓存淘汰原理，不是项目 Java 代码改造。

它和前面几节的关系是：

- 第 16 小节把优惠券模板写入 Redis Hash，并设置过期时间。
- 第 17 小节增加空值缓存，并设置 30 分钟 TTL。
- 第 18 小节讨论当 Redis 内存不足时，这些有 TTL 的 Key 应该如何被淘汰。

所以本节更像“缓存运维与架构策略”文档。

## 4. Redis 内存淘汰触发条件

Redis 不会无条件淘汰 Key。通常需要满足：

1. 配置了 `maxmemory`。
2. Redis 当前内存使用超过 `maxmemory`。
3. 执行写命令或可能增加内存的命令。
4. 根据 `maxmemory-policy` 选择淘汰方式。

查看配置：

```redis
CONFIG GET maxmemory
CONFIG GET maxmemory-policy
CONFIG GET maxmemory-samples
```

查看内存：

```redis
INFO memory
```

## 5. 常见淘汰策略

| 策略 | 淘汰范围 | 淘汰规则 |
| --- | --- | --- |
| `noeviction` | 不淘汰 | 内存不足时写入报错 |
| `allkeys-random` | 所有 Key | 随机淘汰 |
| `volatile-random` | 设置了过期时间的 Key | 随机淘汰 |
| `allkeys-lru` | 所有 Key | 淘汰最近最少使用的 Key |
| `volatile-lru` | 设置了过期时间的 Key | 淘汰最近最少使用的 Key |
| `allkeys-lfu` | 所有 Key | 淘汰最近最不常用的 Key |
| `volatile-lfu` | 设置了过期时间的 Key | 淘汰最近最不常用的 Key |
| `volatile-ttl` | 设置了过期时间的 Key | 优先淘汰最早过期的 Key |

`allkeys` 和 `volatile` 的区别：

- `allkeys`：所有 Key 都可能被淘汰。
- `volatile`：只有设置了过期时间的 Key 才可能被淘汰。

如果使用 `volatile-*`，但 Redis 中没有任何设置 TTL 的 Key，内存满时仍可能写入失败。

## 6. 牛券为什么推荐 volatile-lfu

牛券项目中的缓存有几个特点：

- 优惠券模板有自然有效期。
- 模板缓存可以设置到活动结束时过期。
- 空值缓存有短 TTL。
- 有些券是大促热点券，访问频率明显高。
- 有些券创建后访问很少。

因此，较合理的策略是：

```text
volatile-lfu
```

原因：

- `volatile` 限定只淘汰设置了过期时间的 Key，避免误删没有 TTL 的关键数据。
- `lfu` 更关注访问频率，适合热点优惠券场景。
- 相比 LRU，LFU 更不容易被短时间批量扫描污染。

示例：

```text
热门券 A：过去 1 小时被访问 10 万次
冷门券 B：刚刚被访问 1 次
```

LRU 可能因为 B 最近被访问而保留 B；LFU 更倾向保留访问频率高的 A。

## 7. LRU 基础理解

LRU 全称是 `Least Recently Used`，最近最少使用。

简单 Java 示例：

```java
public class LruCache<K, V> extends LinkedHashMap<K, V> {

    private final int capacity;

    public LruCache(int capacity) {
        super(capacity, 0.75f, true);
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}
```

使用：

```java
LruCache<String, String> cache = new LruCache<>(2);
cache.put("A", "1");
cache.put("B", "2");
cache.get("A");
cache.put("C", "3");
```

最终更可能淘汰 `B`，因为 `A` 最近被访问过。

Redis 中的 LRU 不是维护完整链表，而是近似 LRU：通过抽样选择最近最少使用的 Key，降低内存和 CPU 成本。

## 8. LFU 基础理解

LFU 全称是 `Least Frequently Used`，最近最不常用。

它关注访问频率：

```text
A 被访问 1000 次
B 被访问 3 次
C 被访问 1 次
```

内存不足时，LFU 更倾向淘汰 `C`。

Redis 的 LFU 也不是精确统计每个 Key 的完整访问次数，而是使用近似计数：

- 计数器不是无限增长。
- 访问次数越多，继续增加计数的概率越低。
- 长时间不访问会发生衰减。

相关配置：

```text
lfu-log-factor
lfu-decay-time
```

含义：

- `lfu-log-factor`：控制计数器增长难度。
- `lfu-decay-time`：控制访问频率随时间衰减。

## 9. maxmemory-samples

Redis 淘汰 Key 时不是扫描全量 Key，而是抽样。

配置：

```redis
CONFIG GET maxmemory-samples
```

默认通常是 5。样本数越大：

- 淘汰结果越接近真实 LRU 或 LFU。
- CPU 开销越高。

示例：

```redis
CONFIG SET maxmemory-samples 10
```

在数据量较大、热点差异明显的场景，可以适当提高样本数，但需要压测确认 CPU 开销。

## 10. 与牛券代码的关系

第 16 小节模板缓存：

```java
String couponTemplateCacheKey = String.format(EngineRedisConstant.COUPON_TEMPLATE_KEY, requestParam.getCouponTemplateId());
```

并通过 Lua 设置过期时间：

```java
args.add(String.valueOf(couponTemplateDO.getValidEndTime().getTime() / 1000));
```

第 17 小节空值缓存：

```java
stringRedisTemplate.opsForValue().set(couponTemplateIsNullCacheKey, "", 30, TimeUnit.MINUTES);
```

这些 Key 都有 TTL，因此在 `volatile-lfu` 策略下属于可淘汰对象。

布隆过滤器 Key 通常不建议随意淘汰，因为它承担防穿透职责。如果 Redis 内存紧张导致布隆过滤器相关 Key 被淘汰，可能需要重新初始化布隆过滤器。

## 11. Spring Boot 与 Redis 语法补充

### 11.1 Redis 配置文件写法

`redis.conf` 中可以配置：

```conf
maxmemory 2gb
maxmemory-policy volatile-lfu
maxmemory-samples 10
```

含义：

- 最大使用内存 2GB。
- 内存不足时从带 TTL 的 Key 中按 LFU 淘汰。
- 每次抽样 10 个 Key。

### 11.2 临时修改配置

在 Redis CLI 中：

```redis
CONFIG SET maxmemory 256mb
CONFIG SET maxmemory-policy volatile-lfu
CONFIG SET maxmemory-samples 10
```

注意：`CONFIG SET` 通常是临时生效。Redis 重启后是否保留，取决于配置文件和部署方式。

### 11.3 Spring 写入 TTL

字符串 Key：

```java
stringRedisTemplate.opsForValue().set("demo:key", "value", 10, TimeUnit.MINUTES);
```

Hash Key 设置 TTL：

```java
stringRedisTemplate.opsForHash().put("demo:hash", "field", "value");
stringRedisTemplate.expire("demo:hash", 10, TimeUnit.MINUTES);
```

本项目用 Lua 把写 Hash 和设置过期时间放在一起执行。

### 11.4 TTL 检查

```redis
TTL one-coupon_engine:template:1810966706881941507
```

返回含义：

- 正数：剩余过期秒数。
- `-1`：Key 存在但没有过期时间。
- `-2`：Key 不存在。

## 12. 如何运行与测试本节功能

### 12.1 使用上一节代码

本节没有独立代码分支，可以沿用：

```powershell
cd D:\study\java\oneCoupon\code\onecoupon
git worktree add D:\study\java\oneCoupon\worktrees\chapter-03-18 origin/20240827_dev_coupon-template-querypv-v2_cache_ding.ma
```

### 12.2 启动 Redis 和 engine

启动 Redis 后，启动 `engine`：

```powershell
cd D:\study\java\oneCoupon\worktrees\chapter-03-18
mvn -pl engine -am spring-boot:run
```

### 12.3 查看当前策略

```redis
CONFIG GET maxmemory
CONFIG GET maxmemory-policy
CONFIG GET maxmemory-samples
INFO memory
```

记录当前值，避免测试后不知道如何恢复。

### 12.4 临时设置策略

```redis
CONFIG SET maxmemory-policy volatile-lfu
CONFIG SET maxmemory-samples 10
```

如果只是学习，不建议把 `maxmemory` 设置得过小，因为可能影响本地其他 Redis 数据。

### 12.5 生成项目缓存并检查 TTL

调用第 16 或 17 小节的查询模板接口：

```text
GET /api/engine/coupon-template/query?shopNumber=1810714735922956666&couponTemplateId=1810966706881941507
```

检查：

```redis
HGETALL one-coupon_engine:template:1810966706881941507
TTL one-coupon_engine:template:1810966706881941507
```

再请求一个不存在 ID，检查空值缓存：

```redis
TTL one-coupon_engine:template_is_null:999999999999999999
```

### 12.6 恢复配置

如果你只是临时测试，可以改回原值：

```redis
CONFIG SET maxmemory-policy noeviction
CONFIG SET maxmemory-samples 5
```

生产环境不要随意在线修改 Redis 淘汰策略，应先评估缓存类型、内存水位、业务风险和回滚方案。

## 13. 常见问题

| 问题 | 解释 |
| --- | --- |
| `noeviction` 安全吗 | 数据不会被 Redis 主动淘汰，但内存满时写请求会失败 |
| `allkeys-lfu` 能不能用 | 可以，但所有 Key 都可能被淘汰，包括不该丢的 Key |
| 为什么推荐 `volatile-lfu` | 只淘汰带 TTL 的 Key，并优先保留高频访问热点 |
| TTL 为 -1 怎么办 | 说明 Key 没有过期时间，在 `volatile-*` 策略下不会被淘汰 |
| LFU 是否绝对准确 | 不是，Redis 使用近似统计和抽样 |

## 14. 阅读顺序建议

1. 回顾第 16 小节 Redis Hash 模板缓存。
2. 回顾第 17 小节空值缓存和布隆过滤器。
3. 阅读本节淘汰策略表。
4. 在本地 Redis 执行 `CONFIG GET` 和 `TTL`。
5. 理解为什么项目缓存 Key 必须设置合理 TTL。

## 15. 面试与复盘问题

- Redis 内存淘汰和 Key 过期删除有什么区别？
- `allkeys-lru` 和 `volatile-lru` 有什么区别？
- LRU 和 LFU 分别适合什么场景？
- 为什么牛券项目推荐 `volatile-lfu`？
- `maxmemory-samples` 调大有什么利弊？
- 如果模板缓存被淘汰，系统如何恢复？

## 16. 本节检查清单

- 能说出 Redis 至少 6 种淘汰策略。
- 能解释 `volatile` 和 `allkeys` 的区别。
- 能解释 LRU 和 LFU 的区别。
- 能用 Redis CLI 查看当前淘汰策略。
- 能检查牛券模板缓存是否有 TTL。
- 能说明本节没有独立 Java 代码 diff。
