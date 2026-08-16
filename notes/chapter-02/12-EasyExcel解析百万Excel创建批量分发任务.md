# 第 12 小节：EasyExcel 解析百万 Excel 创建批量分发任务

## 新手先读

本节开始进入批量发券。新手先把 Excel 分发理解成运营人员上传一份用户名单，系统根据这份名单给很多用户发券。

这里先创建“分发任务”，不是马上把所有券发完，是因为 Excel 可能很大，发券过程也可能很慢。先落任务可以让接口快速返回，也方便后续查看进度、失败原因和重试。

## 1. 本节目标

本节开始进入“优惠券批量分发任务”能力。学习目标：

- 理解为什么批量发券需要先创建任务，而不是直接同步发券。
- 理解 Excel 文件在批量发券中的作用。
- 看懂任务状态、发送类型等枚举建模。
- 看懂 Controller、DTO、Service、Mapper、DO 的分层协作。
- 掌握 EasyExcel 的基础读取方式。
- 理解 `AnalysisEventListener` 如何逐行处理 Excel。
- 理解 `IdUtil.getSnowflakeNextId()` 为什么适合生成批次号。
- 会生成测试 Excel，调用接口创建任务，并检查数据库任务记录。

教程路径：

```text
D:\study\java\oneCoupon\第②章节：后台管理服务\《牛券oneCoupon优惠券系统设计》第12小节：EasyExcel解析百万Excel创建批量分发任务.html
```

对应分支：

```text
目标分支：origin/20240822_dev_create-coupon-task_easyexcel_ding.ma
前置分支：origin/20240821_dev_coupon-template-close_rocketmq5_ding.ma
```

完整 diff 附录：

```text
D:\study\java\oneCoupon\notes\diffs\12-EasyExcel解析百万Excel创建批量分发任务.diff
```

## 2. 教程核心内容提炼

批量分发优惠券通常不会在接口请求中直接完成，因为 Excel 可能有几十万甚至上百万行。如果接口同步完成全部发券，会产生：

- HTTP 请求长时间不返回。
- 内存压力大。
- 中途失败难以恢复。
- 无法展示任务进度。
- 多个批次无法统一管理。

本节先实现“创建批量分发任务”：

1. 前端或调用方传入优惠券模板 ID、任务名称、发送类型、文件地址等信息。
2. 后端校验优惠券模板存在。
3. 后端创建一条任务记录。
4. 使用 EasyExcel 读取 Excel 行数。
5. 把行数写入任务的 `sendNum`。
6. 根据发送类型设置任务状态。

本节还没有真正执行批量发券，重点是任务建模和 Excel 解析。

## 3. 分支变更总览

```text
14 files changed, 965 insertions(+)
```

主要新增/修改：

```text
.gitignore
merchant-admin/pom.xml
merchant-admin/src/main/java/.../common/enums/CouponTaskSendTypeEnum.java
merchant-admin/src/main/java/.../common/enums/CouponTaskStatusEnum.java
merchant-admin/src/main/java/.../controller/CouponTaskController.java
merchant-admin/src/main/java/.../dao/entity/CouponTaskDO.java
merchant-admin/src/main/java/.../dao/mapper/CouponTaskMapper.java
merchant-admin/src/main/java/.../dto/req/CouponTaskCreateReqDTO.java
merchant-admin/src/main/java/.../service/CouponTaskService.java
merchant-admin/src/main/java/.../service/handler/excel/RowCountListener.java
merchant-admin/src/main/java/.../service/impl/CouponTaskServiceImpl.java
merchant-admin/src/test/java/.../task/ExcelGenerateTests.java
merchant-admin/src/test/java/.../task/FakerTests.java
merchant-admin/src/test/java/.../CouponTemplateConcurrentInCreaseNumberTests.java
```

核心变化：新增“优惠券推送任务”领域对象和创建接口，并引入 EasyExcel 读取文件行数。

## 4. 业务背景：为什么先创建任务

批量发券是异步任务场景。用户上传或指定一个 Excel 文件，文件中可能包含大量用户手机号、用户 ID 或其他分发目标。

更合理的系统行为是：

- 接口快速创建任务。
- 任务记录保存批次号、模板 ID、文件地址、发送数量、状态。
- 后续由后台线程、定时任务或消息消费者执行实际发券。
- 用户可以查询任务状态。

本节新增的任务表就是后续任务执行链路的基础。

## 5. Controller 入口

新增 `CouponTaskController`：

```java
@RestController
@RequiredArgsConstructor
@Tag(name = "优惠券推送任务管理")
public class CouponTaskController {

    private final CouponTaskService couponTaskService;

    @Operation(summary = "创建优惠券推送任务")
    @NoDuplicateSubmit(message = "请勿短时间内重复提交优惠券推送任务")
    @PostMapping("/api/merchant-admin/coupon-task/create")
    public Result<Void> createCouponTask(@RequestBody CouponTaskCreateReqDTO requestParam) {
        couponTaskService.createCouponTask(requestParam);
        return Results.success();
    }
}
```

这段代码体现了典型 Spring Boot 分层：

- Controller 只处理 HTTP 入参和响应。
- Service 处理业务逻辑。
- `Result<Void>` 统一返回格式。
- `@NoDuplicateSubmit` 复用第 09 小节的防重复提交组件。

## 6. 请求 DTO

`CouponTaskCreateReqDTO` 用来接收创建任务请求。

常见字段含义：

| 字段 | 含义 |
| --- | --- |
| `couponTemplateId` | 优惠券模板 ID |
| `taskName` | 任务名称 |
| `fileAddress` | Excel 文件地址 |
| `sendType` | 发送类型，比如立即发送或定时发送 |
| `sendTime` | 定时发送时间 |
| `notifyType` | 通知方式 |

DTO 的作用是隔离接口入参和数据库实体。调用方传什么，不代表数据库实体就必须长什么样。

示例：

```java
@Data
public class CouponTaskCreateReqDTO {
    private Long couponTemplateId;
    private String taskName;
    private String fileAddress;
    private Integer sendType;
    private Date sendTime;
    private Integer notifyType;
}
```

实际字段以分支代码为准，完整内容可查看 diff 附录。

## 7. 任务实体 DO

`CouponTaskDO` 对应数据库任务表。

任务实体通常需要保存：

- 主键 ID。
- 批次号 `batchId`。
- 商家编号 `shopNumber`。
- 操作人 ID `operatorId`。
- 优惠券模板 ID。
- 任务名称。
- 文件地址。
- 发送数量 `sendNum`。
- 发送类型。
- 任务状态。
- 创建时间、修改时间、删除标记。

其中 `batchId` 很重要。它不是数据库自增 ID，而是业务批次号，后续执行发券、记录失败明细、查询进度时都可以围绕批次号展开。

## 8. 任务枚举

本节新增两个枚举。

`CouponTaskSendTypeEnum` 表示发送类型：

```java
public enum CouponTaskSendTypeEnum {

    IMMEDIATE(0, "立即发送"),
    SCHEDULED(1, "定时发送");

    private final Integer type;
    private final String name;

    CouponTaskSendTypeEnum(Integer type, String name) {
        this.type = type;
        this.name = name;
    }
}
```

`CouponTaskStatusEnum` 表示任务状态：

```java
public enum CouponTaskStatusEnum {

    PENDING(0, "待执行"),
    IN_PROGRESS(1, "执行中"),
    COMPLETED(2, "已完成"),
    FAILED(3, "执行失败");

    private final Integer status;
    private final String name;

    CouponTaskStatusEnum(Integer status, String name) {
        this.status = status;
        this.name = name;
    }
}
```

实际项目代码中的枚举值名称和字段以分支为准。这里的重点是理解：枚举把魔法数字封装成有含义的业务概念。

不要在业务代码里散落这种写法：

```java
couponTaskDO.setStatus(1);
```

更推荐：

```java
couponTaskDO.setStatus(CouponTaskStatusEnum.IN_PROGRESS.getStatus());
```

## 9. Service 创建任务流程

`CouponTaskService` 定义业务接口：

```java
public interface CouponTaskService extends IService<CouponTaskDO> {

    void createCouponTask(CouponTaskCreateReqDTO requestParam);
}
```

`CouponTaskServiceImpl` 实现核心流程：

```java
@Service
@RequiredArgsConstructor
public class CouponTaskServiceImpl extends ServiceImpl<CouponTaskMapper, CouponTaskDO> implements CouponTaskService {

    private final CouponTemplateService couponTemplateService;
    private final CouponTaskMapper couponTaskMapper;

    @Override
    public void createCouponTask(CouponTaskCreateReqDTO requestParam) {
        CouponTemplateQueryRespDTO couponTemplate = couponTemplateService.findCouponTemplateById(
                String.valueOf(requestParam.getCouponTemplateId())
        );

        if (couponTemplate == null) {
            throw new ClientException("优惠券模板不存在");
        }

        CouponTaskDO couponTaskDO = BeanUtil.copyProperties(requestParam, CouponTaskDO.class);
        couponTaskDO.setBatchId(IdUtil.getSnowflakeNextId());
        couponTaskDO.setOperatorId(Long.parseLong(UserContext.getUserId()));
        couponTaskDO.setShopNumber(UserContext.getShopNumber());

        if (Objects.equals(requestParam.getSendType(), CouponTaskSendTypeEnum.IMMEDIATE.getType())) {
            couponTaskDO.setStatus(CouponTaskStatusEnum.IN_PROGRESS.getStatus());
        } else {
            couponTaskDO.setStatus(CouponTaskStatusEnum.PENDING.getStatus());
        }

        RowCountListener listener = new RowCountListener();
        EasyExcel.read(requestParam.getFileAddress(), listener).sheet().doRead();
        int totalRows = listener.getRowCount();
        couponTaskDO.setSendNum(totalRows);

        couponTaskMapper.insert(couponTaskDO);
    }
}
```

按业务顺序理解：

1. 查询优惠券模板，确认任务绑定的是有效模板。
2. 把请求 DTO 拷贝成数据库 DO。
3. 生成批次号。
4. 从登录上下文中获取操作人和商家编号。
5. 根据发送类型设置初始状态。
6. 读取 Excel 行数作为预计发送数量。
7. 插入任务记录。

## 10. EasyExcel 行数统计

新增 `RowCountListener`：

```java
public class RowCountListener extends AnalysisEventListener<Object> {

    @Getter
    private int rowCount = 0;

    @Override
    public void invoke(Object data, AnalysisContext context) {
        rowCount++;
    }

    @Override
    public void doAfterAllAnalysed(AnalysisContext context) {
    }
}
```

EasyExcel 读取文件时，不是一次性把所有行全部加载到内存，而是一行一行回调 `invoke`。本节只统计行数，所以每读到一行就执行 `rowCount++`。

调用方式：

```java
RowCountListener listener = new RowCountListener();
EasyExcel.read(fileAddress, listener).sheet().doRead();
int totalRows = listener.getRowCount();
```

流程解释：

- `EasyExcel.read(fileAddress, listener)`：指定文件路径和监听器。
- `.sheet()`：读取默认工作表。
- `.doRead()`：真正开始读取。
- `listener.getRowCount()`：读取完成后取统计结果。

## 11. Spring Boot 与框架语法补充

### 11.1 `@RestController`

`@RestController` 等价于 `@Controller` 加 `@ResponseBody`。它表示这个类中的接口方法返回 JSON，而不是跳转页面。

示例：

```java
@RestController
public class DemoController {

    @GetMapping("/demo")
    public String demo() {
        return "hello";
    }
}
```

浏览器或接口工具会收到响应体：

```text
hello
```

### 11.2 `@PostMapping`

`@PostMapping` 表示接收 HTTP POST 请求。

```java
@PostMapping("/user/create")
public Result<Void> createUser(@RequestBody UserCreateReqDTO requestParam) {
    return Results.success();
}
```

常见约定：

- 查询用 GET。
- 创建、修改、提交任务用 POST。
- JSON 请求体用 `@RequestBody` 接收。

### 11.3 `@RequestBody`

`@RequestBody` 告诉 Spring：把 HTTP 请求体中的 JSON 转成 Java 对象。

请求 JSON：

```json
{
  "name": "测试任务",
  "sendType": 0
}
```

Java DTO：

```java
@Data
public class TaskCreateReqDTO {
    private String name;
    private Integer sendType;
}
```

Controller：

```java
public Result<Void> create(@RequestBody TaskCreateReqDTO requestParam) {
    System.out.println(requestParam.getName());
    return Results.success();
}
```

### 11.4 `IService<T>` 与 `ServiceImpl<M, T>`

这是 MyBatis-Plus 提供的通用 Service 能力。

```java
public interface CouponTaskService extends IService<CouponTaskDO> {
}
```

表示 `CouponTaskService` 天然拥有保存、更新、查询等通用方法。

```java
public class CouponTaskServiceImpl
        extends ServiceImpl<CouponTaskMapper, CouponTaskDO>
        implements CouponTaskService {
}
```

泛型含义：

- `CouponTaskMapper`：当前 Service 使用哪个 Mapper。
- `CouponTaskDO`：当前 Service 操作哪个实体。

### 11.5 `BeanUtil.copyProperties`

`BeanUtil.copyProperties` 来自 Hutool，用于把一个对象中同名字段复制到另一个对象。

```java
UserCreateReqDTO requestParam = new UserCreateReqDTO();
requestParam.setName("张三");

UserDO userDO = BeanUtil.copyProperties(requestParam, UserDO.class);
```

如果两个类都有 `name` 字段，`userDO.getName()` 就会得到 `"张三"`。

注意：它只适合字段语义一致的场景。如果 DTO 和 DO 字段含义不同，不应该盲目复制。

### 11.6 `IdUtil.getSnowflakeNextId()`

雪花算法生成的是分布式唯一 ID。

```java
long batchId = IdUtil.getSnowflakeNextId();
```

它适合作为批次号，因为：

- 不依赖数据库自增。
- 多台机器生成时冲突概率极低。
- 大体按时间递增，便于排查。

### 11.7 `AnalysisEventListener`

`AnalysisEventListener` 是 EasyExcel 的事件监听器。

```java
public class DemoListener extends AnalysisEventListener<UserRow> {

    @Override
    public void invoke(UserRow data, AnalysisContext context) {
        System.out.println(data.getPhone());
    }

    @Override
    public void doAfterAllAnalysed(AnalysisContext context) {
        System.out.println("读取完成");
    }
}
```

适合大文件读取，因为它按行回调，不需要一次性把整个 Excel 放进内存。

## 12. 如何运行与测试本节功能

### 12.1 环境准备

建议使用独立 worktree：

```powershell
cd D:\study\java\oneCoupon\code\onecoupon
git worktree add D:\study\java\oneCoupon\worktrees\chapter-02-12 origin/20240822_dev_create-coupon-task_easyexcel_ding.ma
```

需要启动：

- MySQL。
- Redis。
- RocketMQ。
- `merchant-admin` 服务。

### 12.2 初始化数据库

确认任务表已经创建。表名通常是：

```text
t_coupon_task
```

如果接口插入时报表不存在，需要回到教程 SQL 或项目资源目录中执行对应建表语句。

### 12.3 生成测试 Excel

本节新增测试类：

```text
merchant-admin/src/test/java/.../task/ExcelGenerateTests.java
merchant-admin/src/test/java/.../task/FakerTests.java
```

可以先运行测试类生成 Excel 文件。生成后确认 `fileAddress` 指向真实存在的文件。

如果想手动准备一个最小 Excel，可以只放几行测试数据，用来验证行数统计。

### 12.4 启动服务

```powershell
cd D:\study\java\oneCoupon\worktrees\chapter-02-12
mvn -pl merchant-admin -am spring-boot:run
```

### 12.5 调用创建任务接口

接口：

```text
POST /api/merchant-admin/coupon-task/create
```

请求示例：

```json
{
  "couponTemplateId": 1812345678901234560,
  "taskName": "第12小节批量发券任务测试",
  "fileAddress": "D:\\study\\java\\oneCoupon\\data\\coupon-task-test.xlsx",
  "sendType": 0,
  "sendTime": null,
  "notifyType": 0
}
```

请求头需要带上项目中用户上下文需要的字段。具体字段名称以前面章节的 `UserContext` 过滤器为准。

### 12.6 验证数据库

查询任务表：

```sql
SELECT id, batch_id, shop_number, operator_id, coupon_template_id, task_name, send_num, status
FROM t_coupon_task
ORDER BY id DESC
LIMIT 5;
```

重点检查：

- `batch_id` 是否生成。
- `shop_number` 是否是当前登录商家。
- `operator_id` 是否是当前用户。
- `coupon_template_id` 是否是请求中的模板 ID。
- `send_num` 是否等于 Excel 数据行数。
- `status` 是否符合发送类型。

### 12.7 常见问题

| 现象 | 可能原因 | 处理方式 |
| --- | --- | --- |
| 文件读取失败 | `fileAddress` 路径不存在 | 使用绝对路径并确认文件存在 |
| 行数不对 | Excel 有表头或空行 | 明确 EasyExcel 是否跳过表头，检查文件内容 |
| 模板不存在 | `couponTemplateId` 不属于当前商家 | 先创建模板，再用正确 ID 创建任务 |
| 重复提交被拦截 | 触发第 09 小节防重复提交 | 等待锁过期或修改请求内容 |
| 数据库插入失败 | 任务表未初始化 | 执行建表 SQL |

## 13. 阅读代码顺序建议

建议按这个顺序读：

1. `CouponTaskCreateReqDTO`：先看接口需要哪些字段。
2. `CouponTaskController`：看 HTTP 入口。
3. `CouponTaskService`：看业务方法抽象。
4. `CouponTaskServiceImpl`：看创建任务完整流程。
5. `RowCountListener`：看 EasyExcel 如何统计行数。
6. `CouponTaskDO` 和 `CouponTaskMapper`：看数据如何落库。
7. 两个枚举：看任务类型和状态如何表达。

## 14. 面试与复盘问题

- 为什么批量发券不建议在 HTTP 接口中同步完成？
- EasyExcel 相比一次性读取整个 Excel，有什么优势？
- `RowCountListener` 只统计行数，有没有真正校验每一行数据？
- 为什么任务需要 `batchId`？
- 立即发送和定时发送的初始状态为什么不同？
- 如果 Excel 有 100 万行，本节代码的性能瓶颈可能在哪里？

## 15. 本节检查清单

- 能解释创建批量分发任务的业务价值。
- 能说清楚 Controller、DTO、Service、Mapper、DO 的职责。
- 能解释 EasyExcel 监听器读取模式。
- 能生成或准备一个 Excel 文件。
- 能调用创建任务接口并在数据库中查到任务记录。
- 能通过 diff 附录查看本节所有代码改动。
