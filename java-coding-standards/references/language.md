# 语言与风格信条

> 用最少的代码承载完整语义：类型约束状态，表达式传递结果，纯函数组合行为，副作用停在边界。

## 命名与格式

| 类别 | 规则 |
| --- | --- |
| 类 / record / 接口 | PascalCase；每个文件一个 public 顶级类型 |
| 方法 / 字段 | camelCase；依赖用 `final` 字段 |
| 常量 | UPPER_SNAKE_CASE |
| 成员顺序 | 常量 → 字段 → 构造器 → public → protected → private |
| 方法长度 | 短小聚焦；只提取有业务命名价值或可复用的 helper，禁止为缩短行数制造跳转 |
类名引用统一使用简单类名并通过 `import` 导入；禁止在 Java 源码中直接书写类全限定名（包括包名 + 类名）。

## `var` 使用

| 右侧类型明显时 | 用 `var` |
| 右侧是字面量、构造器、显式泛型调用、静态工厂 | 用 `var` |
| 右侧是接口返回但实现类型重要，或影响可读性 | 写具体类型 |

## 数据载体

- **DTO / 值对象 / 事件 / 命令 / 查询参数 → `record`**。紧凑构造器只做不变量校验或规范化，不调用 Repository / Client。
- **纯领域实体 → 普通 class**，状态变化收口为有业务语义的方法；**JPA Entity → 遵守 ORM 构造器、代理和持久化生命周期约束，不使用 `record`，不强制字段 `final`**。

## 有限类型：`sealed` + `record` + 穷尽 `switch`

有限业务结果、状态、命令或事件在各分支携带不同数据时，优先用 `sealed interface` + `record` 子类型建模；只有常量差异时用 `enum`；两个简单分支直接用表达式。子类型由外部模块提供或频繁新增时用普通 interface。模式匹配 `switch` 按 Java 21+ 使用，Java 17 不默认启用 preview。

```java
public sealed interface ProductSaveResult
        permits ProductSaveResult.Created, ProductSaveResult.Updated, ProductSaveResult.Rejected {

    record Created(String spuCode) implements ProductSaveResult {}
    record Updated(String spuCode) implements ProductSaveResult {}
    record Rejected(String reason) implements ProductSaveResult {}
}

ProductSaveResponse toResponse(ProductSaveResult result) {
    return switch (result) {
        case ProductSaveResult.Created c  -> ProductSaveResponse.created(c.spuCode());
        case ProductSaveResult.Updated u  -> ProductSaveResponse.updated(u.spuCode());
        case ProductSaveResult.Rejected r -> ProductSaveResponse.rejected(r.reason());
    };
}
```

Java 21+ 的封闭类型和枚举用穷尽 `switch` 表达式并省略 `default`；非封闭类型保留 `default` 且抛异常。Java 17 使用正式支持的 `switch` 表达式、`instanceof` 模式或多态，不为语法偏好开启 preview。

## 静态工厂

`record` 规范构造器是公开的；额外提供命名静态工厂提升业务可读性：

- `of(...)`：聚合参数，校验后构造。
- `from(...)`：从另一种类型转换。
- `parse(...)`：从字符串解析，失败抛异常。
- `empty()` / `defaultOf()`：空对象或默认值。

## 不可变集合

```java
List.copyOf(source)              // 防御性拷贝
skuInfos.stream().toList()      // Java 16+ 直接得到不可变 List
Map.copyOf(source)               // 不可变 Map
```

多行字符串用文本块（`"""..."""`）。

## `Optional`

- 只在返回值表达"可能没有"。
- 字段、DTO 属性、方法参数都用具体类型或 `null`。
- 链式调用用 `map` / `flatMap` / `filter` + `.orElseThrow(supplier)`，不 `.get()`。

## 空值与判空

- **先建契约，再写实现**：Controller / Service 的信任边界优先使用 `@Valid`、`@Validated` 以及 `@NotNull`、`@NotBlank`、`@NotEmpty`、`@Size` 等 Bean Validation 约束；DTO / Request 优先使用 `record` 表达不可变输入契约。
- **校验通过后直接使用**：已由 `@Valid` / `@Validated` 校验的请求对象及其必填字段，方法体内禁止再次写 `request == null || ... ? null : ...`、`request != null ? request.getXxx() : null` 或同类防御式三元表达式。此类写法会掩盖接口契约缺失、扩散可空状态并制造无意义分支。
**推荐的边界写法：**

```java
public PageResult<?> search(@Valid SearchRequest request) {
    return service.search(request.keyword(), request.status());
}
```

不要把请求对象当作可选值，在 Service 中重复拆解：

```java
// 反模式：用防御式判空掩盖请求契约缺失
String keyword = request == null || !StringUtils.hasText(request.getTypeName())
        ? null : request.getTypeName().trim();
Integer status = request == null ? null : request.getStatus();
```

如果 `keyword` 业务上可选，在边界 Request / 值对象中完成规范化；`request` 本身仍应由 `@Valid` / `@NotNull` 保证存在。

- **查询参数也要有边界**：搜索条件确实可选时，在 Request `record` 的紧凑构造器、值对象或边界转换方法中统一完成 trim、默认值和规范化；Service 内部接收清晰的不变量，不散落清洗逻辑。
- **只检查不可信来源**：外部回调、MQ、缓存、反序列化结果和第三方响应需要规范化或校验，之后内部代码保持非空不变量。
- **沿用项目已有工具**：字符串空白优先项目已有 `StringUtils`；集合允许为 null 时用已有 `CollectionUtils`；方法引用用 `Objects::nonNull`。
- 普通控制流允许更直接的 `value == null` / `value != null`；禁止冗长组合 `value != null && !value.isEmpty()`，应在边界规范化或使用已有工具表达。
- 基本类型不做 null 检查；需要缺失语义时使用包装类型并在边界校验。
- 业务规则需要“有 / 无”分派时用返回值 `Optional` 或封闭类型，不让 `null` 穿透业务链路。

## Stream

- 集合转换、过滤、分组、索引、聚合默认用 `stream()`；能一条表达式返回结果就不创建可变临时集合。
- 单条链路保持可扫读；复杂条件提取为有业务名字的 `Predicate` / `Function`，不要机械按操作数量拆链。
- `map()` 只转换，`filter()` 只过滤，`flatMap()` 只展开；Stream 内不访问数据库、网络，不修改外部状态。
- `peek()` 仅临时调试，提交前删除；副作用在链路末端或明确的边界方法执行。
- 默认顺序流；`parallelStream()` 只在数据量、线程安全、上下文传播和收益均被测量后使用。
- 收集 `Map` 必须声明 key 冲突策略；优先返回不可变集合。
- 数据先批量取回，再用 Stream 完成内存关联、转换和聚合。

## 函数式组合

- 条件抽成 `Predicate<T>`，转换抽成 `Function<T, R>`，延迟创建用 `Supplier<T>`；仅在副作用确实是目标时使用 `Consumer<T>`。
- 优先方法引用：`SkuInfo::getSkuCode`、`Objects::nonNull`、`this::buildResponse`。
- 重复分支、字段映射和校验通过函数参数化；只调用一次且没有命名价值的 lambda / helper 直接内联。
- 用函数组合消除重复，不用函数接口包装本来更清楚的一行 `if`。

## 泛型

泛型参数显式声明；可复用的工具方法优先有界泛型（`<T extends Identifiable>`）。

## 信号更强的写法

- 参数列表过长 → 先检查职责是否过多；参数天然内聚时收拢为 `record`，大量可选参数才用 builder。
- 嵌套过深 → guard clause、`Optional`、穷尽 `switch` 或命名函数拉平。
- 魔法值 → 提取有领域含义的常量或值对象。
- 静态可变状态 → 删除或通过依赖注入管理生命周期。
- 异常被吞掉 → 转换为领域异常并保留 cause，或交给统一边界处理。

## 注释

注释默认写“为什么”而不是复述代码；但数据契约必须显式、完整地记录字段语义：

- **入参模型、出参模型和实体类的每个字段都必须有注释**，不得因命名看似清晰而省略。
- 注释必须说明字段的业务含义；存在单位、格式、取值范围、枚举值、默认值或可空规则时一并写明，禁止“名称”“状态”等无信息量描述。
- 普通字段使用字段 Javadoc；`record` 组件使用类型 Javadoc 的 `@param` 逐项说明。项目已有 OpenAPI 等文档注解时可同步填写，但不得产生两套不一致的描述。
- 公开 API 的方法按需用 Javadoc 说明方法意图、参数语义、返回约定和异常；简单 getter、setter、转发方法不写重复注释。
- 业务规则、算法选择、外部接口怪癖和反直觉行为记录约束、权衡与来源；不注释显而易见的控制流或框架注解。
