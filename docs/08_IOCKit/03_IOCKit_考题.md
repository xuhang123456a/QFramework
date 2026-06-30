# 08 · IOCKit 考题

## 🟢 概念题（5）

1. IOCKit 的 `QFrameworkContainer` 和 CoreArchitecture 的 `IOCContainer` 是同一个东西吗？关键差异？
2. 容器的三张表分别存什么？
3. `[Inject]` 特性可以标记在哪些成员上？
4. `RegisterInstance` 与 `Register<TSrc,TTgt>` 注册出来的对象生命周期有何不同？
5. 为什么 IOCKit 自己实现了一个 `Tuple<T1,T2>` class？

<details><summary>参考答案要点</summary>

1. 不是。Core 的 `IOCContainer` 极简（Type→object，仅 Register/Get，被 Architecture 使用）；IOCKit 的 `QFrameworkContainer` 是完整 DI 容器（命名实例、构造注入、特性注入、关系映射），Architecture 不用它。
2. `Instances`（(Type,name)→单例实例）、`Mappings`（(Type,name)→实现类型，瞬态）、`RelationshipMappings`（(for,base)→concrete，上下文相关）。
3. 字段（Field）和属性（Property）。
4. `RegisterInstance` 是单例（Resolve 总返回同一实例）；`Register<TSrc,TTgt>` 是映射，每次 Resolve 都 `CreateInstance` 新建（瞬态）。
5. 老版 Unity Mono 不支持 `System.Tuple`/`ValueTuple`（源码注释吐槽），需要一个能正确 `Equals/GetHashCode` 的复合类型作字典 key，故手写。
</details>

## 🟡 机制题（6）

1. `Resolve(type, name, requireInstance)` 的解析优先级链是怎样的？
2. `requireInstance=true` 时 Resolve 的行为有什么不同？
3. `CreateInstance` 如何选择构造函数？每个参数如何解析？
4. 构造注入完成后，IOCKit 还会做什么？
5. `ResolveAll(type)` 会返回哪些对象？
6. 命名注入是怎样把"命名注册的依赖"匹配到构造参数的？

<details><summary>参考答案要点</summary>

1. 先查 `Instances[type,name]`，命中即返回（单例）；否则若 `requireInstance` 返回 null；否则查 `Mappings[type,name]`，命中则 `CreateInstance` 新建；都无则返回 null。
2. 只从已注册的实例中找，不会通过映射新建——找不到实例就返回 null（强制要求"必须是已存在的实例"）。
3. 有显式 args 直接用；否则取所有 public 实例构造，选**参数最多**的那个。每参数：数组类型→`ResolveAll`；否则 `Resolve(参数类型) ?? Resolve(参数类型, 参数名)`（先按类型，回退按参数名当 name）。
4. 再调一次 `Inject(obj)` 填充对象上带 `[Inject]` 的字段/属性——构造注入 + 特性注入双管齐下。
5. `Instances` 中类型匹配且命名非空的实例；以及 `Mappings` 中"目标类型可赋值给 type 且命名非空"的映射，每个 `Activator.CreateInstance`+Inject 新建。
6. 构造参数解析时第二级回退 `Resolve(p.ParameterType, p.Name)` —— 用参数名当 name 去查命名注册，从而把命名实例/映射喂给对应参数。
</details>

## 🔴 架构陷阱题（7）

1. **同框架对比**：QFramework 同时有 Core 的 `IOCContainer` 和 IOCKit 的 `QFrameworkContainer`。为什么 Architecture 不直接用功能更强的 IOCKit？这反映了什么设计取舍？
2. A 的构造函数需要 B，B 的构造函数需要 A，两者都 `Register` 了映射。`Resolve<A>()` 会发生什么？
3. 用 `Register<IService, ServiceImpl>()` 注册后连续 `Resolve<IService>()` 三次，得到几个对象？若想要单例应该怎么做？
4. 某个 `[Inject]` 字段的类型从未被注册。`Inject` 时会怎样？字段会变成什么？
5. 构造函数注入选"参数最多的构造"，但其中一个参数类型没注册。`CreateInstance` 会怎样？
6. `Tuple<Type,string>` 的 `GetHashCode` 若实现错误（比如只用 Item1），命名解析会出什么问题？
7. 把循环依赖从"构造注入"改为"特性注入"，为什么就能打破循环？

<details><summary>参考答案要点</summary>

1. Architecture 追求的是"按 Type 注册/获取 System/Model/Utility"的简单确定语义，极简 `IOCContainer`（Type→object）足够且高效、无反射开销、无歧义；IOCKit 的命名/构造/关系等高级特性对 Architecture 是冗余复杂度。取舍：核心用最简方案保证可预测，把可选的高级 DI 独立成 Kit 按需引入——"核心极简、能力外挂"。
2. 无限递归栈溢出：`Resolve<A>`→`CreateInstance(A)`→解析参数 B→`Resolve<B>`→`CreateInstance(B)`→解析参数 A→… IOCKit 无循环依赖检测。
3. 三个不同对象（映射是瞬态，每次 `CreateInstance` 新建）。要单例：用 `RegisterInstance<IService>(new ServiceImpl())` 注册一个实例，Resolve 总返回同一个。
4. `Resolve(字段类型, name)` 返回 null，`SetValue(obj, null)` 把字段设为 null。不报错，但运行时使用该字段会 NRE——静默失败的依赖缺失。
5. 该参数 `Resolve` 返回 null，被当作 null 传给构造函数。构造函数若不容忍 null 参数会抛异常或后续 NRE。容器不会报"缺少依赖"。
6. 命名解析失灵：不同 name 但相同 Type 的 key 会有相同 hash，虽然 `Equals` 仍能区分（落到同一桶后线性比较），但若 `Equals` 也错则不同命名会互相覆盖/命中错误。正确的 `Equals+GetHashCode`（含 null 处理）是命名 DI 的地基。
7. 构造注入要求"创建时依赖已就绪"，环上没有起点；特性注入是"对象先 new 出来（无参或部分构造），之后再 SetValue 填充"——A 和 B 可以都先被创建，再互相注入引用，环被"创建"与"填充"两阶段打破。
</details>

## ✍️ 实操题（3）

1. 用容器注册一个单例 `ILogger` 和瞬态的 `OrderService`，`OrderService` 通过构造函数依赖 `ILogger`。验证多次 Resolve `OrderService` 得到不同实例但共享同一个 logger。
2. 用命名注册实现"同一个 `IPaymentChannel` 接口、`alipay`/`wechat` 两个实现"，并演示按 name 解析。说明命名 DI 相比"每个实现定义一个接口"的优势。
3. 演示一个能被特性注入但会因构造注入循环依赖而栈溢出的例子，并改写成特性注入版本使其工作。

<details><summary>参考答案要点</summary>

1. `c.RegisterInstance<ILogger>(new ConsoleLogger()); c.Register<OrderService,OrderService>();`，`OrderService(ILogger l)` 构造注入。`var a=c.Resolve<OrderService>(); var b=c.Resolve<OrderService>(); Assert(a!=b)`（瞬态）但 `a.Logger==b.Logger`（单例 logger 被注入两者）。
2. `c.RegisterInstance<IPaymentChannel>(new Alipay(),"alipay"); c.RegisterInstance<IPaymentChannel>(new Wechat(),"wechat"); c.Resolve<IPaymentChannel>("alipay")`。优势：无需为每个实现造一个子接口，运行时按名选择实现，符合开闭原则，配置驱动。
3. 循环版：`class A{ public A(B b){} } class B{ public B(A a){} }` + `Register` 两者，`Resolve<A>()` 栈溢出。特性版：`class A{ [Inject] public B B; } class B{ [Inject] public A A; }`，`RegisterInstance(new A()); RegisterInstance(new B()); InjectAll();`——两对象先创建再互注，工作正常。
</details>
