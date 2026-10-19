# 设计与简洁美学

> 先寻找能删除状态、分支、样板和间接层的表达方式。最好的抽象不是代码更“高级”，而是更少代码就能看见完整业务语义。

## 信条

- **类型优于字符串**、**表达式优于语句**、**穷尽优于默认值**、**数据优于过程**、**组合优于继承**。
- **纯函数优于共享状态**、**不可变结果优于原地修改**、**方法引用优于重复 lambda**。
- 当下只有一个实现就写具体类；第二个真实实现出现时再抽象，不为“以后可能”创建接口、工厂和注册表。
- 简短必须同时清楚：一行表达式能完整说明意图就不拆；出现隐藏副作用、重复计算或难读嵌套时立即命名。

## 类型选择决策表

| 场景 | 选这个 | 不用这个 |
| --- | --- | --- |
| 纯数据载体（DTO / 值对象 / 事件 / 命令 / 查询参数） | `record` | 手写 class + getter/setter；Lombok `@Data` |
| 有限业务结果 / 状态 / 命令 / 事件，各分支携带不同数据 | `sealed interface` + `record` 子类型；| 字符串分发；`Map<String, Handler>`；开放的 `instanceof` 链；Visitor |
| 有限常量或纯计算策略，不依赖 Spring Bean | `enum` + 行为 | 多个无状态实现类 + 工厂 + 注册表 |
| 有限策略，各策略携带不同字段 | `sealed interface` + `record` | 用 nullable 字段拼成一个万能类型 |
| 子类型由外部 jar / 插件提供，运行时注册 | 普通 interface + 注册 | `sealed`（`permits` 列表限制同 jar） |
| 跨团队 / 跨服务的公共契约 | 普通 interface + 文档枚举 | `sealed`（分布式场景不现实） |
| 业务上有"可能没有"的返回值 | `Optional<T>` | `null` 字段；`Optional` 字段 / 参数 |
| 数据集合的不可变快照 | `List.copyOf` / `Stream.toList()` / `Map.copyOf` | `new ArrayList<>(source)` + `Collections.unmodifiableList(...)` |
| 多行 SQL / JSON / HTML | 文本块 `"""..."""` | `+ "\n"` 拼接 |

## 何时用更朴素的写法

1. 一行 `if` / `return` 比抽象更清楚 → 直接写表达式或 guard clause。
2. 策略需要 Repository / Client / 动态配置 → 使用注入的普通策略对象，不把 Spring 依赖塞进 `enum`。
3. 子类型由插件、外部模块或跨团队契约扩展 → 普通 interface + 注册，不使用 `sealed`。
4. 历史代码只重构本次触及范围；删除触及范围内的重复和样板，不发动无关改造。
5. 热点路径先测量；Stream 产生明显装箱、分配或重复遍历时使用清晰循环。

## 与其他规范的关系

- `record` / `sealed` / `switch` / `Optional` / 不可变集合的语法补充：见 `references/language.md`。
- 分层、事务、持久化、测试、安全：见 `references/cross-cutting.md`。
