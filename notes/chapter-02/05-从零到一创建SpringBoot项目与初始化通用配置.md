# 第 05 小节：从零到一创建 SpringBoot 项目与初始化通用配置

## 新手先读

本节是后续所有代码的工程地基。新手不要急着理解每个 starter 和自动装配细节，先搞清楚 Maven 多模块为什么要拆、`framework` 为什么放公共能力、`gateway` 为什么放入口过滤器。

如果你对 Spring Boot 不熟，先记住：启动类负责启动应用，配置文件负责告诉应用连接哪些外部服务，Controller/Service/Mapper 负责把请求、业务、数据库访问分层。

## 1. 本节目标

本节是代码正式开始的第一节，目标不是实现某个业务接口，而是搭建项目骨架和公共基础设施。

你需要掌握：

- 多模块 Maven 项目如何组织。
- 为什么项目使用 Java 17、Spring Boot 3 和 Spring Cloud。
- `framework` 模块为什么存在。
- 统一返回、统一异常、错误码和自动装配如何作为基础能力提供给业务模块。
- 网关日志过滤器在整体链路中的位置。

教程路径：

```text
D:\study\java\oneCoupon\第②章节：后台管理服务\《牛券oneCoupon优惠券系统设计》第05小节：从零到一创建SpringBoot项目&初始化通用配置.html
```

对应分支：

```text
origin/20240708_init-code_ding.ma
```

## 2. 教程核心内容提炼

教程主要包含：

- IDEA、Maven、JDK17 环境说明。
- 第 05-09 小节的学习分支说明。
- 多 Module 项目创建方式。
- `framework` 中初始化的公共能力：
  - 异常码。
  - 全局统一返回类。
  - 全局异常拦截器。
  - Spring Boot Starter 自动装配。

这节的重点是打地基。后续所有业务服务都会复用这些基础设施。

## 3. 分支代码结构

在 `origin/20240708_init-code_ding.ma` 分支中，项目已经具备完整的初始模块：

```text
framework
distribution
settlement
search
engine
merchant-admin
gateway
resources
```

关键文件：

```text
pom.xml
framework/pom.xml
framework/src/main/java/com/nageoffer/onecoupon/framework/result/Result.java
framework/src/main/java/com/nageoffer/onecoupon/framework/web/Results.java
framework/src/main/java/com/nageoffer/onecoupon/framework/web/GlobalExceptionHandler.java
framework/src/main/java/com/nageoffer/onecoupon/framework/config/WebAutoConfiguration.java
framework/src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
gateway/src/main/java/com/nageoffer/onecoupon/gateway/filter/RequestLoggingFilter.java
```

## 4. 多模块 Maven 设计

根 `pom.xml` 使用 `pom` 打包类型：

```xml
<packaging>pom</packaging>
```

它本身不产出业务 jar，而是作为父工程统一管理模块、依赖版本和构建插件。

模块列表：

```xml
<modules>
    <module>framework</module>
    <module>distribution</module>
    <module>settlement</module>
    <module>search</module>
    <module>engine</module>
    <module>merchant-admin</module>
    <module>gateway</module>
</modules>
```

父工程统一管理版本：

```xml
<java.version>17</java.version>
<spring-boot.version>3.0.7</spring-boot.version>
<spring-cloud.version>2022.0.3</spring-cloud.version>
<spring-cloud-alibaba.version>2022.0.0.0-RC2</spring-cloud-alibaba.version>
```

这样做的好处是：每个子模块不需要重复声明技术栈版本，避免不同服务依赖版本不一致。

## 5. `framework` 模块的职责

`framework` 是公共基础设施模块。业务模块引入它后，可以复用统一返回、异常处理、Web 配置、Redis/幂等等能力。

初始分支中重点包括：

```text
framework/src/main/java/com/nageoffer/onecoupon/framework
  ├─ config
  ├─ errorcode
  ├─ exception
  ├─ result
  └─ web
```

### 5.1 统一返回对象

`Result<T>` 是所有接口返回给前端的统一结构。

核心字段：

```java
public class Result<T> implements Serializable {

    public static final String SUCCESS_CODE = "0";

    private String code;
    private String message;
    private T data;
    private String requestId;

    public boolean isSuccess() {
        return SUCCESS_CODE.equals(code);
    }
}
```

解释：

- `code`：业务响应码，`0` 表示成功。
- `message`：错误或提示信息。
- `data`：真正的响应数据。
- `requestId`：请求链路追踪字段，后续排查问题会用到。

如果没有统一返回对象，不同接口可能返回不同 JSON 结构，前端和调用方处理成本会很高。

### 5.2 统一返回构造器

`Results` 用来快速构造成功或失败响应。

常用形式：

```java
public final class Results {

    public static Result<Void> success() {
        return new Result<Void>()
                .setCode(Result.SUCCESS_CODE);
    }

    public static <T> Result<T> success(T data) {
        return new Result<T>()
                .setCode(Result.SUCCESS_CODE)
                .setData(data);
    }
}
```

业务 Controller 只需要写：

```java
return Results.success();
```

或者：

```java
return Results.success(responseData);
```

### 5.3 错误码体系

错误码用于统一表达错误类型。项目初始分支中有：

```text
framework/errorcode/IErrorCode.java
framework/errorcode/BaseErrorCode.java
```

常见分类：

- 客户端错误：参数错误、非法请求。
- 服务端错误：系统异常、数据库异常、远程调用异常。
- 业务错误：库存不足、优惠券不存在、状态不允许变更等。

后续业务中抛出的 `ClientException`、`ServiceException` 都会进入统一异常处理。

### 5.4 全局异常处理器

`GlobalExceptionHandler` 使用 `@RestControllerAdvice` 统一拦截异常。

核心处理对象：

```java
@ExceptionHandler(value = MethodArgumentNotValidException.class)
public Result validExceptionHandler(HttpServletRequest request, MethodArgumentNotValidException ex) {
    ...
}

@ExceptionHandler(value = {AbstractException.class})
public Result abstractException(HttpServletRequest request, AbstractException ex) {
    ...
}

@ExceptionHandler(value = Throwable.class)
public Result defaultErrorHandler(HttpServletRequest request, Throwable throwable) {
    ...
}
```

解释：

- `MethodArgumentNotValidException`：处理参数校验失败。
- `AbstractException`：处理项目内主动抛出的业务异常。
- `Throwable`：兜底处理未预期异常。

这样 Controller 不需要每个接口都写 `try-catch`。

## 6. Spring Boot 3 自动装配

本节教程强调了 Starter 机制。项目中 `framework` 通过 Spring Boot 自动装配把公共配置交给业务模块。

关键文件：

```text
framework/src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

内容：

```text
com.nageoffer.onecoupon.framework.config.WebAutoConfiguration
```

含义：

- Spring Boot 3 不再使用旧版 `spring.factories` 作为推荐自动装配入口。
- 业务模块引入 `onecoupon-framework` 后，Spring Boot 会读取 `AutoConfiguration.imports`。
- 里面声明的配置类会被自动加载。

这是后续公共能力可复用的基础。

## 7. 网关日志过滤器

初始分支中 `gateway` 已经有 `RequestLoggingFilter`。

它实现：

```java
public class RequestLoggingFilter implements GlobalFilter, Ordered
```

核心逻辑：

```java
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    ServerHttpRequest request = exchange.getRequest();
    HttpMethod method = request.getMethod();

    String traceId = UUID.randomUUID().toString();
    long startTime = System.currentTimeMillis();
    MDC.put("traceId", traceId);

    LOG.info("请求URI: {}", request.getURI());
    LOG.info("请求类型: {}", method);
    LOG.info("请求头: {}", request.getHeaders());

    return chain.filter(exchange).then(Mono.fromRunnable(() -> {
        long duration = System.currentTimeMillis() - startTime;
        LOG.info("响应时间：{} ms", duration);
    }));
}
```

学习重点：

- `GlobalFilter` 是 Spring Cloud Gateway 的全局过滤器。
- `Ordered` 控制过滤器执行顺序。
- `MDC` 用于在线程日志上下文中放入 `traceId`。
- 网关层记录请求信息，有利于统一排查入口流量。

## 8. 新概念解释

### 8.1 Maven 父工程

父工程负责管理公共配置，不直接写业务。

本项目中父工程管理：

- 子模块列表。
- Java/Spring/MyBatis/RocketMQ 等版本。
- 编译插件。
- Spotless 代码格式插件。

### 8.2 Spring Boot Starter

Starter 的本质是“依赖 + 自动配置”的组合。

业务模块引入一个 starter，就能得到一组默认能力。例如项目中的 `framework` 虽然不是标准命名的 starter，但思路类似：把公共 Web 配置、统一异常等能力封装起来，让业务模块直接复用。

### 8.3 全局异常处理

全局异常处理是把异常响应集中到一个地方。它的价值是：

- Controller 更干净。
- 错误响应格式统一。
- 日志记录一致。
- 后续接入告警和链路追踪更容易。

### 8.4 自动装配入口变化

Spring Boot 2 常见方式：

```text
META-INF/spring.factories
```

Spring Boot 3 推荐方式：

```text
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

本项目使用的是 Spring Boot 3，所以采用后者。

## 9. Spring Boot / 框架语法补充

本节虽然没有复杂业务，但已经出现了很多 Spring Boot、Spring Cloud Gateway、Lombok 和 Maven 语法。下面按初学者视角解释。

### 9.1 `pom.xml`、父工程和子模块

`pom.xml` 是 Maven 项目的配置文件。Maven 会根据它下载依赖、编译代码、运行测试、打包。

父工程的关键写法：

```xml
<packaging>pom</packaging>

<modules>
    <module>framework</module>
    <module>gateway</module>
    <module>merchant-admin</module>
</modules>
```

解释：

- `<packaging>pom</packaging>` 表示这个工程主要用来管理子模块，不直接打业务 jar。
- `<modules>` 表示当前父工程包含哪些子模块。
- 子模块可以继承父工程中的依赖版本、插件版本和公共配置。

最小示例：

```xml
<!-- 父工程 pom.xml -->
<project>
    <packaging>pom</packaging>
    <modules>
        <module>common</module>
        <module>web-app</module>
    </modules>
</project>
```

这代表 `common` 和 `web-app` 是同一个大项目里的两个模块。

### 9.2 `dependencyManagement`

父工程中有很多依赖写在 `dependencyManagement` 下。

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>${spring-boot.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

它的作用是“管理版本”，不是“真正引入依赖”。

子模块真正使用依赖时，还要在自己的 `pom.xml` 中声明：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

因为版本已经由父工程管理，子模块通常不用重复写 `<version>`。

### 9.3 `@RestControllerAdvice`

`GlobalExceptionHandler` 使用：

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
}
```

解释：

- `@RestControllerAdvice` 是 Spring MVC 提供的全局增强注解。
- 它可以拦截 Controller 抛出的异常，并统一返回 JSON。
- 你可以把它理解成“所有 Controller 外面包了一层统一异常处理器”。

最小示例：

```java
@RestControllerAdvice
public class DemoExceptionHandler {

    @ExceptionHandler(IllegalArgumentException.class)
    public Result<Void> handleIllegalArgument(IllegalArgumentException ex) {
        return new Result<Void>()
                .setCode("400")
                .setMessage(ex.getMessage());
    }
}
```

如果 Controller 中抛出：

```java
throw new IllegalArgumentException("参数错误");
```

就会被 `handleIllegalArgument` 统一处理。

### 9.4 `@ExceptionHandler`

`@ExceptionHandler` 用来声明“这个方法处理哪类异常”。

项目代码：

```java
@ExceptionHandler(value = Throwable.class)
public Result defaultErrorHandler(HttpServletRequest request, Throwable throwable) {
    log.error("[{}] {} ", request.getMethod(), getUrl(request), throwable);
    return Results.failure();
}
```

解释：

- `Throwable.class` 是兜底异常类型，几乎所有异常都会被它覆盖。
- 更具体的异常处理方法优先匹配，例如 `MethodArgumentNotValidException`。
- 返回 `Result` 后，前端拿到的是统一 JSON，而不是一段 HTML 错误页。

### 9.5 Lombok 的 `@Data` 和 `@Accessors(chain = true)`

`Result` 类使用：

```java
@Data
@Accessors(chain = true)
public class Result<T> {
    private String code;
    private String message;
}
```

`@Data` 会自动生成：

- `getCode()`
- `setCode(String code)`
- `getMessage()`
- `setMessage(String message)`
- `toString()`
- `equals()`
- `hashCode()`

`@Accessors(chain = true)` 让 setter 返回当前对象，因此可以链式调用。

没有链式调用时：

```java
Result<Void> result = new Result<>();
result.setCode("0");
result.setMessage("success");
```

有链式调用时：

```java
Result<Void> result = new Result<Void>()
        .setCode("0")
        .setMessage("success");
```

项目中的 `Results.success()` 就用了这种写法。

### 9.6 Java 泛型 `Result<T>`

`Result<T>` 中的 `T` 是泛型，表示响应数据的类型不固定。

示例：

```java
Result<String> stringResult = Results.success("ok");
Result<Integer> intResult = Results.success(100);
Result<UserDTO> userResult = Results.success(userDTO);
```

这样统一返回结构可以复用，但 `data` 的具体类型仍然是明确的。

### 9.7 `GlobalFilter`

`RequestLoggingFilter` 实现：

```java
public class RequestLoggingFilter implements GlobalFilter, Ordered
```

`GlobalFilter` 是 Spring Cloud Gateway 的全局过滤器接口。每个经过网关的请求都会进入它。

最小示例：

```java
@Component
public class DemoGatewayFilter implements GlobalFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        System.out.println("请求进入网关");
        return chain.filter(exchange);
    }
}
```

解释：

- `exchange`：当前请求和响应上下文。
- `chain.filter(exchange)`：继续执行后续过滤器和路由逻辑。
- 如果不调用 `chain.filter(exchange)`，请求就不会继续往后走。

### 9.8 `Ordered`

`Ordered` 用来控制执行顺序。

```java
@Override
public int getOrder() {
    return -1;
}
```

数字越小，优先级越高。`-1` 表示这个过滤器会比较早执行。

### 9.9 Reactor 的 `Mono<Void>`

Gateway 基于 WebFlux，返回值经常是 `Mono<Void>`。

简单理解：

- `Mono` 表示一个异步结果。
- `Mono<Void>` 表示异步流程完成，但没有具体返回数据。
- `chain.filter(exchange).then(...)` 表示请求处理完成后，再执行一段逻辑。

项目里用它记录响应耗时：

```java
return chain.filter(exchange).then(Mono.fromRunnable(() -> {
    long duration = System.currentTimeMillis() - startTime;
    LOG.info("响应时间：{} ms", duration);
}));
```

### 9.10 `MDC`

`MDC` 是日志上下文。它可以把 `traceId` 放到当前线程的日志上下文中。

```java
String traceId = UUID.randomUUID().toString();
MDC.put("traceId", traceId);
```

如果日志格式配置了 `%X{traceId}`，后续日志就能自动带上这个请求 ID，方便排查同一次请求的完整日志。

## 10. 如何运行与测试本节

本节对应初始工程，主要验证项目能编译、基础服务能启动、自动装配和网关过滤器能工作。

### 10.1 建议使用独立 worktree

当前主工作区有未提交改动，不建议直接切分支。建议新建只用于学习的 worktree：

```powershell
cd D:\study\java\oneCoupon\code\onecoupon
git worktree add D:\study\java\oneCoupon\worktrees\onecoupon-05 origin/20240708_init-code_ding.ma
cd D:\study\java\oneCoupon\worktrees\onecoupon-05
```

后续命令都在 `onecoupon-05` 目录执行。

### 10.2 检查 JDK 和 Maven

```powershell
java -version
.\mvnw.cmd -version
```

期望：

- Java 版本是 17。
- Maven 能正常启动。

如果 `java -version` 不是 17，需要在 IDEA 的 Project SDK 或命令行环境中切换 JDK。

### 10.3 编译整个项目

```powershell
.\mvnw.cmd clean package -DskipTests
```

如果只是想快速验证 `framework` 和 `gateway`：

```powershell
.\mvnw.cmd -pl framework,gateway -am package -DskipTests
```

解释：

- `-pl framework,gateway`：只构建指定模块。
- `-am`：同时构建这些模块依赖的上游模块。
- `-DskipTests`：跳过测试执行，加快构建。

### 10.4 启动 `merchant-admin`

第 05 小节还没有真实业务接口，但可以启动商家后台应用验证 Spring Boot 基础工程。

IDEA 中启动：

```text
merchant-admin/src/main/java/com/nageoffer/onecoupon/merchant/admin/MerchantAdminApplication.java
```

命令行启动：

```powershell
.\mvnw.cmd -pl merchant-admin spring-boot:run
```

观察点：

- 控制台没有启动失败异常。
- 能看到 Spring Boot 启动日志。
- 端口使用 `merchant-admin/src/main/resources/application.yaml` 中配置的端口。

### 10.5 启动 `gateway` 并观察过滤器

IDEA 中启动：

```text
gateway/src/main/java/com/nageoffer/onecoupon/gateway/GatewayApplication.java
```

命令行启动：

```powershell
.\mvnw.cmd -pl gateway spring-boot:run
```

启动后访问一个任意网关地址，例如：

```powershell
curl http://127.0.0.1:10000/not-exist
```

即使路由不存在，也可以观察控制台中是否输出：

```text
请求URI
请求类型
请求头
响应时间
```

这说明 `RequestLoggingFilter` 被 Spring 扫描并执行了。

### 10.6 验证自动装配

本节可以通过启动任意引入 `framework` 的 Web 服务来间接验证自动装配。如果启动时 `GlobalExceptionHandler`、`WebAutoConfiguration` 没有报错，说明 `AutoConfiguration.imports` 已经被 Spring Boot 识别。

更直接的验证方式是在 IDE 中搜索：

```text
org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

确认文件中包含：

```text
com.nageoffer.onecoupon.framework.config.WebAutoConfiguration
```

### 10.7 常见启动问题

- `JAVA_HOME` 不是 JDK17：切换 JDK。
- 依赖下载慢：检查 Maven 镜像源。
- 端口占用：修改对应模块 `application.yaml` 的 `server.port`。
- Lombok 报错：IDEA 安装 Lombok 插件，并开启 annotation processing。
- 中文乱码：确认文件按 UTF-8 打开。

## 11. 本节代码阅读顺序

推荐按这个顺序读：

```text
pom.xml
  -> framework/pom.xml
  -> Result.java
  -> Results.java
  -> IErrorCode.java / BaseErrorCode.java
  -> AbstractException.java / ClientException.java / ServiceException.java / RemoteException.java
  -> GlobalExceptionHandler.java
  -> WebAutoConfiguration.java
  -> AutoConfiguration.imports
  -> RequestLoggingFilter.java
```

## 12. 常见问题

### 12.1 为什么统一返回不直接返回 HTTP 状态码？

HTTP 状态码表达的是协议层结果，业务错误码表达的是业务层结果。比如优惠券库存不足，HTTP 请求本身成功到达服务，但业务执行失败，更适合通过业务 `code` 和 `message` 表达。

### 12.2 为什么要有 `ClientException`、`ServiceException`、`RemoteException`？

它们表达错误来源：

- `ClientException`：调用方参数或请求不合法。
- `ServiceException`：当前服务内部错误。
- `RemoteException`：远程服务调用错误。

分类越清晰，排查越快。

### 12.3 为什么 `framework` 要做自动装配？

如果没有自动装配，每个业务模块都要手动扫描或声明公共配置。自动装配让业务模块只需要引入依赖，就能使用公共能力。

## 13. 面试点

- Maven 多模块项目的父 `pom` 有什么作用？
- 为什么微服务项目要有公共 `framework` 模块？
- 统一返回对象解决了什么问题？
- `@RestControllerAdvice` 和 `@ExceptionHandler` 如何配合？
- Spring Boot 3 的自动装配文件和 Spring Boot 2 有什么区别？
- Gateway 全局过滤器适合做哪些事情？

## 14. 学习检查清单

- 能说清楚每个模块的初始职责。
- 能解释 `Result`、`Results`、`GlobalExceptionHandler` 的关系。
- 能找到 Spring Boot 3 自动装配入口文件。
- 能说明业务异常为什么不建议散落在 Controller 里处理。
- 能理解网关日志过滤器在请求链路中的位置。
- 能独立编译初始分支并启动 `merchant-admin` 或 `gateway`。
