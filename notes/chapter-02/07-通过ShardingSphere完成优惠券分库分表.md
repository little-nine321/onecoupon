# 第 07 小节：通过 ShardingSphere 完成优惠券分库分表

## 新手先读

本节容易卡在“分库分表配置很多，看不懂”。先用一个简单问题理解：如果优惠券模板表未来有几千万行，单表查询和维护会变慢，所以要把一张逻辑表拆到多张真实表。

新手先抓住三个词：逻辑表是代码里看到的表名，真实表是数据库里实际存在的分片表，分片键是决定数据落到哪个分片的字段。本节后面的算法和配置都围绕这三个词展开。

## 1. 本节目标

本节把第 06 小节的单表 `t_coupon_template` 改造成分库分表模型。学习目标：

- 理解为什么优惠券模板需要分库分表。
- 理解分片键为什么选择 `shop_number`。
- 掌握 ShardingSphere-JDBC 在项目里的接入方式。
- 看懂 `shardingsphere-config.yaml` 的数据源、真实表、分库策略、分表策略。
- 看懂自定义分库算法和分表算法。
- 会运行接口并验证数据落到哪个库、哪张表。

教程路径：

```text
D:\study\java\oneCoupon\第②章节：后台管理服务\《牛券oneCoupon优惠券系统设计》第07小节：通过ShardingSphere完成优惠券分库分表.html
```

对应分支：

```text
目标分支：origin/20240815_dev_coupon-tablue_shardingsphere_ding.ma
前置分支：origin/20240814_dev_create-template_chain_ding.ma
```

完整 diff 附录：

```text
D:\study\java\oneCoupon\notes\diffs\07-通过ShardingSphere完成优惠券分库分表.diff
```

## 2. 教程核心内容提炼

教程主要讲：

- 优惠券模板数据量预估。
- 分库、分表、分库分表的区别。
- 什么场景需要分表、分库、分库分表。
- 分片键选择原则。
- HashMod 和时间范围两类常见分片算法。
- 为什么选择 ShardingSphere，而不是 MyCat 或分布式数据库。
- 优惠券模板如何按店铺编号分库分表。
- ShardingSphere-JDBC 配置和自定义分片算法。

## 3. 分支变更总览

执行：

```bash
git diff --stat origin/20240814_dev_create-template_chain_ding.ma..origin/20240815_dev_coupon-tablue_shardingsphere_ding.ma
```

结果概要：

```text
7 files changed, 448 insertions(+), 3 deletions(-)
```

主要变化：

```text
merchant-admin/pom.xml
merchant-admin/src/main/java/.../dao/sharding/DBHashModShardingAlgorithm.java
merchant-admin/src/main/java/.../dao/sharding/TableHashModShardingAlgorithm.java
merchant-admin/src/main/resources/application.yaml
merchant-admin/src/main/resources/shardingsphere-config.yaml
merchant-admin/src/test/java/.../CouponTemplateTest.java
merchant-admin/src/test/java/.../MockCouponTemplateDataTests.java
```

本节没有大改业务 Service，核心是把数据访问层从“直连 MySQL 单库单表”切换为“通过 ShardingSphere 路由到真实库表”。

## 4. 业务背景

教程中用商家数量和优惠券创建量做估算：

```text
商家数量：约 3000 万
每个商家创建优惠券模板：约 100 张
总模板量：约 30 亿
```

单表承载几十亿数据会带来问题：

- 索引高度增加，查询性能下降。
- DDL、备份、归档成本升高。
- 单表写入和维护压力过大。
- 后续按店铺分页查询会越来越慢。

因此需要分片，把逻辑表 `t_coupon_template` 拆成多个真实物理表。

## 5. 分库分表概念

### 5.1 分库

分库是把数据拆到多个数据库。

示例：

```text
one_coupon_rebuild_0
one_coupon_rebuild_1
```

每个库可以在同一 MySQL 实例，也可以在不同 MySQL 实例。真实生产环境通常会放到不同实例，才能分散连接和写入压力。

### 5.2 分表

分表是把一张逻辑表拆成多张物理表。

示例：

```text
t_coupon_template_0
t_coupon_template_1
...
t_coupon_template_15
```

业务代码仍然操作逻辑表 `t_coupon_template`，ShardingSphere 负责路由到真实表。

### 5.3 逻辑表和真实表

逻辑表是业务代码看到的表：

```text
t_coupon_template
```

真实表是数据库实际存在的表：

```text
one_coupon_rebuild_0.t_coupon_template_0
one_coupon_rebuild_0.t_coupon_template_1
...
one_coupon_rebuild_1.t_coupon_template_15
```

ShardingSphere 的核心价值就是把“逻辑 SQL”改写并路由到“真实库表”。

## 6. 分片键选择

本节选择 `shop_number` 作为分片键。

原因：

- 优惠券模板由商家创建，天然属于某个店铺。
- 商家后台高频查询通常是“当前店铺的优惠券模板列表”。
- `shop_number` 在模板创建后不会变化，适合作为稳定分片键。
- 用店铺编号做分片，可以让同一店铺的数据落到固定分片，避免跨分片查询。

不适合用时间作为分片键的原因：

- 早期商家少，后期商家多，按时间容易造成新分片热点。
- 当前业务主要按店铺维度查询，而不是按时间范围查询。

## 7. ShardingSphere 接入方式

### 7.1 Maven 依赖

第 07 小节在 `merchant-admin/pom.xml` 中新增：

```xml
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-jdbc-core</artifactId>
    <version>${shardingsphere.version}</version>
</dependency>
```

`shardingsphere-jdbc-core` 是 ShardingSphere-JDBC 的核心依赖。它运行在应用进程内，表现得像一个增强版 JDBC Driver。

### 7.2 应用数据源改造

第 06 小节中 `application.yaml` 直连 MySQL：

```yaml
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/one_coupon_rebuild?...
    username: root
    password: root
```

第 07 小节改成：

```yaml
spring:
  datasource:
    driver-class-name: org.apache.shardingsphere.driver.ShardingSphereDriver
    url: jdbc:shardingsphere:classpath:shardingsphere-config.yaml
```

含义：

- 应用不再直接连接单个 MySQL。
- 应用连接 ShardingSphere Driver。
- ShardingSphere Driver 读取 `shardingsphere-config.yaml`。
- 真正的 MySQL 数据源、分片规则都在 ShardingSphere 配置里。

## 8. ShardingSphere 配置文件

核心文件：

```text
merchant-admin/src/main/resources/shardingsphere-config.yaml
```

### 8.1 数据源配置

```yaml
dataSources:
  ds_0:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://127.0.0.1:3306/one_coupon_rebuild_0?...
    username: root
    password: root
  ds_1:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://127.0.0.1:3306/one_coupon_rebuild_1?...
    username: root
    password: root
```

解释：

- `ds_0`、`ds_1` 是 ShardingSphere 内部数据源名称。
- 每个数据源对应一个真实 MySQL 库。
- `HikariDataSource` 是连接池实现。

### 8.2 真实节点

```yaml
t_coupon_template:
  actualDataNodes: ds_${0..1}.t_coupon_template_${0..15}
```

这表示逻辑表 `t_coupon_template` 的真实表范围是：

```text
ds_0.t_coupon_template_0 ~ ds_0.t_coupon_template_15
ds_1.t_coupon_template_0 ~ ds_1.t_coupon_template_15
```

总共：

```text
2 个库 * 16 张表 = 32 个真实表
```

### 8.3 分库策略

```yaml
databaseStrategy:
  standard:
    shardingColumn: shop_number
    shardingAlgorithmName: coupon_template_database_mod
```

解释：

- `standard` 表示单分片键标准分片。
- `shardingColumn` 是分片键。
- `shardingAlgorithmName` 指向下面定义的算法。

### 8.4 分表策略

```yaml
tableStrategy:
  standard:
    shardingColumn: shop_number
    shardingAlgorithmName: coupon_template_table_mod
```

分表也使用 `shop_number`，但执行的是表分片算法。

### 8.5 自定义算法绑定

```yaml
shardingAlgorithms:
  coupon_template_database_mod:
    type: CLASS_BASED
    props:
      algorithmClassName: com.nageoffer.onecoupon.merchant.admin.dao.sharding.DBHashModShardingAlgorithm
      sharding-count: 16
      strategy: standard
  coupon_template_table_mod:
    type: CLASS_BASED
    props:
      algorithmClassName: com.nageoffer.onecoupon.merchant.admin.dao.sharding.TableHashModShardingAlgorithm
      strategy: standard
```

解释：

- `CLASS_BASED` 表示算法由 Java 类实现。
- `algorithmClassName` 指向自定义算法类。
- `props` 会传给算法类的 `init(Properties props)` 方法。

## 9. 自定义分库算法

文件：

```text
DBHashModShardingAlgorithm.java
```

核心代码：

```java
public final class DBHashModShardingAlgorithm implements StandardShardingAlgorithm<Long> {

    private int shardingCount;
    private static final String SHARDING_COUNT_KEY = "sharding-count";

    @Override
    public String doSharding(Collection<String> availableTargetNames, PreciseShardingValue<Long> shardingValue) {
        long id = shardingValue.getValue();
        int dbSize = availableTargetNames.size();
        int mod = (int) hashShardingValue(id) % shardingCount / (shardingCount / dbSize);
        int index = 0;
        for (String targetName : availableTargetNames) {
            if (index == mod) {
                return targetName;
            }
            index++;
        }
        throw new IllegalArgumentException("No target found for value: " + id);
    }

    @Override
    public void init(Properties props) {
        this.props = props;
        shardingCount = getShardingCount(props);
    }
}
```

假设：

```text
availableTargetNames = [ds_0, ds_1]
sharding-count = 16
shop_number hash 后对 16 取模 = 9
```

计算：

```text
dbSize = 2
shardingCount / dbSize = 8
mod = 9 / 8 = 1
```

结果落到第 2 个库，也就是 `ds_1`。

这里的 `sharding-count` 表示总分片数量，不是库数量。算法先算全局 16 个分片中的位置，再映射到 2 个库。

## 10. 自定义分表算法

文件：

```text
TableHashModShardingAlgorithm.java
```

核心代码：

```java
public final class TableHashModShardingAlgorithm implements StandardShardingAlgorithm<Long> {

    @Override
    public String doSharding(Collection<String> availableTargetNames, PreciseShardingValue<Long> shardingValue) {
        long id = shardingValue.getValue();
        int shardingCount = availableTargetNames.size();
        int mod = (int) hashShardingValue(id) % shardingCount;
        int index = 0;
        for (String targetName : availableTargetNames) {
            if (index == mod) {
                return targetName;
            }
            index++;
        }
        throw new IllegalArgumentException("No target found for value: " + id);
    }
}
```

假设：

```text
availableTargetNames = [t_coupon_template_0 ... t_coupon_template_15]
shop_number hash 后对 16 取模 = 9
```

结果落到：

```text
t_coupon_template_9
```

## 11. SQL 执行链路

业务代码仍然执行：

```java
couponTemplateMapper.insert(couponTemplateDO);
```

MyBatis-Plus 生成逻辑 SQL：

```sql
INSERT INTO t_coupon_template (...) VALUES (...)
```

ShardingSphere 拦截并改写为真实 SQL：

```sql
INSERT INTO one_coupon_rebuild_?.t_coupon_template_? (...) VALUES (...)
```

问号由 `shop_number` 经过分库、分表算法计算得出。

## 12. Spring Boot / 框架语法补充

### 12.1 JDBC Driver

JDBC Driver 是 Java 连接数据库的驱动。普通 MySQL 连接使用：

```yaml
driver-class-name: com.mysql.cj.jdbc.Driver
```

本节改成：

```yaml
driver-class-name: org.apache.shardingsphere.driver.ShardingSphereDriver
```

这表示应用先连接 ShardingSphere Driver，再由 ShardingSphere 管理真实 MySQL 数据源。

### 12.2 YAML 配置

YAML 通过缩进表达层级。

```yaml
spring:
  datasource:
    driver-class-name: org.apache.shardingsphere.driver.ShardingSphereDriver
```

等价理解：

```properties
spring.datasource.driver-class-name=org.apache.shardingsphere.driver.ShardingSphereDriver
```

注意 YAML 缩进不能乱，通常使用两个空格，不使用 Tab。

### 12.3 `StandardShardingAlgorithm<T>`

这是 ShardingSphere 标准分片算法接口。

```java
public final class TableHashModShardingAlgorithm implements StandardShardingAlgorithm<Long>
```

泛型 `Long` 表示分片键类型是 `Long`，本项目的 `shop_number` 是 `Long`。

必须实现的关键方法：

```java
String doSharding(Collection<String> availableTargetNames, PreciseShardingValue<Long> shardingValue)
```

含义：

- `availableTargetNames`：候选库名或候选表名。
- `shardingValue`：当前 SQL 中的分片键值。
- 返回值：最终选中的库名或表名。

### 12.4 `PreciseShardingValue` 和 `RangeShardingValue`

精确分片：

```sql
WHERE shop_number = 1810714735922956666
```

对应：

```java
PreciseShardingValue<Long>
```

范围分片：

```sql
WHERE shop_number BETWEEN 1 AND 100
```

对应：

```java
RangeShardingValue<Long>
```

本节暂无范围查询场景，所以范围分片方法直接返回空集合：

```java
public Collection<String> doSharding(Collection<String> availableTargetNames, RangeShardingValue<Long> shardingValue) {
    return List.of();
}
```

### 12.5 `Properties`

`Properties` 是 Java 的键值配置对象。ShardingSphere 会把 YAML 中的 `props` 传入算法。

YAML：

```yaml
props:
  sharding-count: 16
```

Java：

```java
public void init(Properties props) {
    shardingCount = Integer.parseInt(props.getProperty("sharding-count"));
}
```

## 13. 如何运行与测试本节功能

### 13.1 建议使用独立 worktree

```powershell
cd D:\study\java\oneCoupon\code\onecoupon
git worktree add D:\study\java\oneCoupon\worktrees\onecoupon-07 origin/20240815_dev_coupon-tablue_shardingsphere_ding.ma
cd D:\study\java\oneCoupon\worktrees\onecoupon-07
```

### 13.2 准备 MySQL 和 Redis

本节需要：

- MySQL：两个逻辑库 `one_coupon_rebuild_0`、`one_coupon_rebuild_1`。
- Redis：用于创建模板后的缓存预热。

创建库：

```sql
CREATE DATABASE IF NOT EXISTS one_coupon_rebuild_0;
CREATE DATABASE IF NOT EXISTS one_coupon_rebuild_1;
```

然后分别在两个库里创建 `t_coupon_template_0` 到 `t_coupon_template_15`。建表 SQL 以教程第 07 小节为准，所有物理表字段结构应保持一致。

### 13.3 检查 ShardingSphere 配置

确认配置文件：

```text
merchant-admin/src/main/resources/shardingsphere-config.yaml
```

检查 MySQL 账号密码是否与你本地一致：

```yaml
username: root
password: root
```

检查 `application.yaml` 是否使用 ShardingSphere Driver：

```yaml
spring:
  datasource:
    driver-class-name: org.apache.shardingsphere.driver.ShardingSphereDriver
    url: jdbc:shardingsphere:classpath:shardingsphere-config.yaml
```

### 13.4 编译和启动

```powershell
.\mvnw.cmd -pl merchant-admin -am package -DskipTests
.\mvnw.cmd -pl merchant-admin spring-boot:run
```

启动成功后访问：

```text
http://127.0.0.1:10010/doc.html
```

### 13.5 调用创建模板接口

使用第 06 小节的请求体即可。关键是当前模拟登录用户的 `shop_number`：

```java
new UserInfoDTO("1810518709471555585", "pdd45305558318", 1810714735922956666L)
```

这个值会参与分库分表。

### 13.6 验证路由结果

因为 `shardingsphere-config.yaml` 中开启了：

```yaml
props:
  sql-show: true
```

控制台会打印 ShardingSphere 实际执行 SQL。重点看：

```text
Actual SQL
```

它会显示最终落到哪个库、哪张表。

也可以直接在两个库中查询：

```sql
SELECT id, shop_number, name FROM one_coupon_rebuild_0.t_coupon_template_0 LIMIT 5;
SELECT id, shop_number, name FROM one_coupon_rebuild_0.t_coupon_template_1 LIMIT 5;
...
SELECT id, shop_number, name FROM one_coupon_rebuild_1.t_coupon_template_15 LIMIT 5;
```

找到新增记录后，对照控制台 Actual SQL。

### 13.7 运行测试类

本节新增：

```text
merchant-admin/src/test/java/com/nageoffer/onecoupon/merchant/admin/template/CouponTemplateTest.java
```

可在 IDEA 中右键运行 `testInsertCouponTemplate`，或命令行：

```powershell
.\mvnw.cmd -pl merchant-admin -Dtest=CouponTemplateTest test
```

测试通过说明 ShardingSphere 路由和 MyBatis-Plus 插入链路基本可用。

### 13.8 常见问题排查

- 报 `Table doesn't exist`：真实表没有按 `t_coupon_template_0` 到 `t_coupon_template_15` 创建完整。
- 报数据库连接失败：检查 `shardingsphere-config.yaml` 中 `jdbcUrl`、用户名、密码。
- 看不到 Actual SQL：确认 `sql-show: true`。
- 插入失败但第 06 小节能跑：通常是 ShardingSphere 配置或物理表初始化问题。
- 查询逻辑表失败：SQL 中缺少分片键时，可能触发广播路由或全路由，学习阶段应优先带上 `shop_number`。

## 14. 本节代码阅读顺序

```text
merchant-admin/pom.xml
  -> application.yaml
  -> shardingsphere-config.yaml
  -> DBHashModShardingAlgorithm.java
  -> TableHashModShardingAlgorithm.java
  -> CouponTemplateTest.java
```

## 15. 常见问题与面试点

- 分库和分表有什么区别？
- 为什么优惠券模板选择 `shop_number` 作为分片键？
- 逻辑表和真实表有什么区别？
- ShardingSphere-JDBC 和 MyCat 的接入方式有什么差异？
- ShardingSphere 如何根据逻辑 SQL 路由到真实库表？
- 为什么分片键应该尽量稳定、不频繁修改？
- 如果查询条件没有分片键，会有什么问题？

## 16. 学习检查清单

- 能解释 `actualDataNodes` 的含义。
- 能看懂 `databaseStrategy` 和 `tableStrategy`。
- 能说明 `DBHashModShardingAlgorithm` 和 `TableHashModShardingAlgorithm` 的区别。
- 能启动商家后台并创建优惠券模板。
- 能通过 Actual SQL 或数据库查询验证数据落到哪个物理表。
