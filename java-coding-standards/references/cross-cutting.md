# 跨切关注点：分层、持久化、测试、安全

> 横切多个模块的规则合集：分层职责 / 事务 / 异常 / 持久化 / 缓存 / 安全 / 外部调用 / 异步调度 / 测试 / 日志。

## 分层职责

| 层 | 职责 | 不做 |
| --- | --- | --- |
| Controller | 协议适配、参数校验、响应映射 | 业务规则、事务、Repository 直接调用 |
| Application Service | 用例编排、事务边界、领域规则协调 | 接收 Web 协议类型、返回可变内部集合、成为万能 Manager |
| Repository / Mapper | 持久化访问 | 复杂业务判断、业务事务边界 |
| Client / Gateway | 外部系统适配 | 向领域层泄漏 HTTP / SDK 类型 |
| DTO / Request / Response | 不可变数据契约 | 持久化行为和可变业务状态 |

Controller 路径用名词资源；DTO / Request / Response 优先 `record`。没有明确独立职责时不创建 `Manager`、`Util`、`Helper`、`CommonService`。

## 依赖注入与配置

- 依赖用构造器注入，字段 `final`。
- 业务配置用 `@ConfigurationProperties` 绑定，配置字段加 Bean Validation，启动时暴露缺失或非法配置。

## 事务

- 事务边界在 Application Service；Repository / Mapper 不定义业务事务。
- 只读查询可用 `@Transactional(readOnly = true)`，但不能把它当数据库绝对只读保证。
- 事务内只保留一致性所需的读取、校验、状态转换和写入；远程调用、长阻塞和无关计算移出事务。
- 内部方法调用不会触发 Spring AOP；异步线程不继承调用方事务；提交后动作使用事务同步、`@TransactionalEventListener` 或 outbox。

## 异常

- 领域错误用非受检异常；技术异常包装并补充上下文；创建领域专用异常（如 `MarketNotFoundException`）。
- 异常用专门类型或明确错误码向上传递。
- 全局异常处理用 `@RestControllerAdvice`，统一错误码、错误消息、日志策略。

## 空值与校验

- 默认非空约束；`@Nullable` 只在确实无法避免时标注。
- 输入参数优先用 Bean Validation：请求体使用 `@Valid`，方法级约束或 Service 校验使用 `@Validated`，字段按语义选择 `@NotNull`、`@NotBlank`、`@NotEmpty`、`@Size` 等。
- Controller / Service 入口完成校验；校验通过后不再为同一契约重复写 `request == null || ... ? null : ...`、`request != null ? ... : ...` 或手工 `hasText` 判空。
- 查询条件可选不等于请求对象可空：在 Request `record`、值对象或边界转换方法中集中完成 trim、默认值和跨字段校验；业务层只接收明确的不变量。
- 校验失败由全局异常处理统一返回；不可信输入仅在边界规范化，内部维持非空不变量。

## 外部调用

- 外部调用必须设置连接、响应和总耗时上限；失败转换为带业务上下文的领域异常并保留 cause。
- 优先批量接口或在调用方聚合请求；不在循环或 Stream 中逐条调用外部服务。
- 重试只用于幂等操作，设置最大次数、指数退避和抖动；写操作重试必须有幂等键。

## 缓存

- 缓存 key 稳定、可读、包含影响结果的所有业务维度。
- 显式 TTL、容量、失效策略。
- 写数据时同步清理或失效缓存。
- 缓存只用于性能优化。

## 持久化基础规则

- 数据库操作在循环和 Stream 外完成；链路内部只做内存关联、转换和聚合。
- 批量查询、保存和更新；按 id / code / slug 关联时先批量加载，再构建不可变 `Map` / `Set`。
- 分页显式使用稳定排序；非唯一字段后追加唯一键，深分页优先 keyset pagination。
- 大集合按数据库参数上限分片；同步短生命周期且原列表不再修改时可用 `subList`，异步或跨边界持有时复制。

### 性能信号

| 高代价写法 | 检测方式 | 推荐写法 |
| --- | --- | --- |
| 循环 `findById` | 循环体内单实体查询 | 收集 id 后 `findAllById`，再构建 `Map` |
| 循环外键查询 | 每个父对象查一次子对象 | `WHERE parent_id IN (...)` 后按父 id 分组 |
| 嵌套循环查询 | 查询次数随 N*M 增长 | JOIN FETCH、实体图或分阶段批量查询 |
| 循环 `save` / `insert` / `update` | 循环体内单条写 | 批量接口或一条 SQL 完成 |
| `saveAll` 未生效 batch | ORM 配置缺失 | JPA 配 JDBC batch；MyBatis 用 `ExecutorType.BATCH` |

### 验证

- 开启 SQL 日志或用集成测试统计查询次数。
- 批量读取应为 O(1) 或固定少量查询，随输入条数线性增长的实现需要重写。
- 批量写入确认驱动和 ORM 实际发送 batch。
- 验证空集合、重复 id、缺失关联数据、超过单条 SQL 参数上限的大集合。

### 一致性与领域值

- 唯一性、非空和并发不变量由数据库约束或条件更新兜底；应用层预检查只改善错误信息。
- 并发状态更新使用 `@Version`、版本字段或带旧状态条件的 SQL，并检查受影响行数。
- 金额用 `BigDecimal` 并明确币种、scale、`RoundingMode`；业务比较用 `compareTo`，不从 `double` 构造。
- 时刻优先 `Instant`，外部带偏移契约用 `OffsetDateTime`，日历日期用 `LocalDate`；不依赖 JVM 默认时区。

## 测试

| 目标 | 推荐测试 |
| --- | --- |
| 业务规则、状态转换 | 单元测试 + AssertJ |
| Controller 参数校验、路由、响应 | `@WebMvcTest` 或等价 slice |
| SQL、映射、分页、批量 | 数据层 / `@DataJpaTest` / Mapper 测试 |
| 多层协作、事务、配置 | `@SpringBootTest`（仅在确有需要时） |

### 规则

- 测试行为、边界和业务结果，不测试私有实现或源码形状。
- 每个测试一个清晰场景；失败时从测试名和断言直接看出哪条规则坏了。
- 覆盖正常路径、边界、异常、权限 / 状态分支、关键不变量和并发冲突。
- 单元测试 mock Repository / Client / Gateway / 消息 / Clock / ID 等外部边界；复杂 SQL 和数据库约束用真实数据库测试。
- `verify` 只用于有业务含义的副作用；值对象、DTO 和被测对象使用真实实例。
- 测试数据用紧凑 fixture / 静态工厂构造；只有字段众多时才用 builder，名称表达业务状态。
- 时间注入 `Clock`；UUID、雪花 ID 和随机数注入生成器；异步测试使用可控执行器或 Awaitility 并设置上限。
- 覆盖率只用于发现空白；关键业务分支必须有结果断言；修 bug 先补能复现问题的回归测试。

### 测试名

测试类：`XxxTest` / `XxxServiceTest` / `XxxControllerTest`。
方法名：场景 + 预期；不写 `testSave1` / `success` / `fail`。

### 信号更强的断言

- `assertThat(result).isNotNull()` 单独出现 → 至少断言一个业务关键字段或返回值含义。
- 断言被弱化以让测试通过 → 让失败暴露真实问题，必要时拆用例。
- 大量 `@SpringBootTest` → 默认优先单元测试和 slice test。
- 依赖当前日期、时区、网络、文件系统绝对路径或执行顺序 → 注入时钟、固定路径、配置替身。

## 日志

```java
private static final Logger log = LoggerFactory.getLogger(MarketService.class);
log.info("fetch_market slug={}", slug);
log.error("failed_fetch_market slug={}", slug, ex);
```

- 接口入参出参都需要加上字段日志
- 日志记录业务事件和失败上下文，不逐层重复打印入口、成功和同一个异常。
- 使用稳定事件名和参数化写法，不拼接字符串；异常对象作为最后一个参数。
- 同一异常原则上只在最终处理边界记录一次；领域拒绝不自动记为 `ERROR`。

| 级别 | 使用场景 |
| --- | --- |
| `INFO` | 有运营、审计或状态变化价值的正常事件 |
| `WARN` | 非预期但已恢复，值得观察趋势 |
| `ERROR` | 当前操作失败且需要处理或告警 |
