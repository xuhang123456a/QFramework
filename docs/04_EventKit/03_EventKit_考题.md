# 04 · EventKit 考题

## 🟢 概念题（5）

1. EventKit 自己实现了事件触发逻辑吗？它复用了 Core 的什么？
2. `StringEventSystem`、`EnumEventSystem`、`TypeEventSystem` 三者的 key 维度分别是什么？
3. `EnumEventSystem` 的回调签名为什么是 `Action<int, object[]>`？
4. 两个事件系统的 `Send` 在 key 不存在时如何处理？
5. `StringEventSystem.Global` 与 `new StringEventSystem()` 的区别是什么？

<details><summary>参考答案要点</summary>

1. 没有。它只是 `Dictionary<key, IEasyEvent>` 路由表，注册/触发逻辑全在 Core 的 `EasyEvent`/`EasyEvent<T>`。
2. String=字符串 key；Enum=枚举转 int；Type=事件类型本身。
3. Enum 系统统一用 `EasyEvent<int, object[]>`，回调收到"消息 ID（int）+ 变长参数（object[]）"，是老式消息派发风格。
4. 静默失败：`TryGetValue` 未命中或 `?.` 跳过，不报错、不缓存。
5. `Global` 是进程级全局总线（静态只读单例）；`new` 出的是局部作用域总线，互不影响。
</details>

## 🟡 机制题（6）

1. 描述 `StringEventSystem.Register(key, cb)` 的懒创建流程。
2. `StringEventSystem.UnRegister(key, cb)` 之后，字典里那个 key 还在吗？
3. `EnumEventSystem` 提供了 `StringEventSystem` 没有的哪两个注销 API？
4. `EnumEventSystem` 如何把不同枚举类型统一成字典 key？
5. 为什么 EventKit 的注册返回 `IUnRegister` 而不是 void？
6. 原框架用 `.As<EasyEvent<T>>()` 而非直接强转，二者行为上有何细微差异？

<details><summary>参考答案要点</summary>

1. `TryGetValue(key)`：命中则 `As<EasyEvent>()` 后 `Register(cb)`；未命中则 `new EasyEvent()` → `Add(key, ev)` → `ev.Register(cb)`。返回 `IUnRegister`。
2. 还在。`UnRegister` 只做委托 `-=`，从不移除字典项（残留空 EasyEvent）。
3. `UnRegister<T>(key)`（移除整个 key）和 `UnRegisterAll()`（清空字典）。
4. 所有 API `where T:IConvertible`，内部 `key.ToInt32(null)` 转 int 做 key。
5. 把"如何注销"封装成句柄交给注册方，便于绑定到对象生命周期/统一管理，避免手工记住 key+委托去 UnRegister。
6. `.As<T>()` 是 FluentAPI 的安全/语义化转型（通常等价 `as T`，失败得 null）；直接 `(T)e` 强转失败会抛 `InvalidCastException`。前者更"静默"，后者更"暴露问题"。
</details>

## 🔴 架构陷阱题（7）

1. **同底座对比**：同样基于 `EasyEvent`，`TypeEventSystem`（type key）相比 `StringEventSystem`（string key），在重构改名、查找引用、payload 安全上各有什么优劣？项目应如何选择？
2. 先 `Register("evt", () => {})`（无参），后 `Register<int>("evt", i => {})`（带参）。会发生什么？
3. 某模块 `Send("playerDied", deadInfo)` 但订阅方写的是 `Register("playerDie", ...)`（拼写差一个字母）。编译、运行各是什么表现？
4. 一个长生命周期的 UI 在 `StringEventSystem.Global` 注册了回调（捕获 UI 引用），UI 销毁时忘了 UnRegister。后果？String 版能像 Core 那样 `UnRegisterWhenGameObjectDestroyed` 吗？
5. `EnumEventSystem.Send(key, "a", 123)` 的接收方 `args[1]` 取出后当 string 用。会怎样？
6. 为什么 Architecture 内部用 `TypeEventSystem` 而不用 EventKit 的 String/Enum 系统？
7. 两个无关模块恰好都用了字符串 key `"update"`，但 payload 形态不同。会互相干扰吗？

<details><summary>参考答案要点</summary>

1. Type 版：改名/查引用靠 IDE 重构安全、payload 编译期类型安全、事件自带文档性；缺点是每个事件要定义类型。String 版：零定义、灵活，缺点是魔法字符串、无类型检查、易错且静默失败。选择：核心业务/需可维护性用 Type 版；临时/跨模块松耦合/不愿定义类型的轻量广播可用 String 版。
2. 第二次注册时 `As<EasyEvent<int>>()` 把字典里的 `EasyEvent`（无参）当 `EasyEvent<int>` 转型——`.As<>()` 失败返回 null 后 `.Register` 抛空引用，或精简版直接 `InvalidCastException`。无论哪种都是运行时崩/失效。
3. 编译通过（字符串无检查）。运行时：发送方 Send 到 "playerDied" 的槽，订阅方挂在 "playerDie" 的槽——两个不同 key，订阅方永远收不到，且无任何报错（静默）。
4. 回调（连同 UI 引用）随全局总线存活无法 GC，且 UI 销毁后事件仍触发其回调导致空引用/逻辑错误。String 版本身只返回 `IUnRegister`——可以手动 `.UnRegisterWhenGameObjectDestroyed(go)`（该扩展作用于任意 `IUnRegister`，来自 Core），所以能绑定，只是没有自动机制，需显式调用。
5. `args[1]` 实际是 `int 123` 装箱成 object，强转 string 会抛 `InvalidCastException`。Enum 版 payload 弱类型，参数顺序/类型全靠约定，错位即崩。
6. Architecture 追求类型安全的 MVC 数据流，事件作为强类型 DTO（`SendEvent<T>(e)`），便于 Command/System/View 间结构化通信；String/Enum 的弱类型不利于大型架构的可维护性。
7. 会。它们共享同一个 `StringEventSystem.Global` 字典，key `"update"` 指向同一个 EasyEvent 槽。payload 形态不同时还会触发第 2 题的转型崩溃。应使用各自 `new` 的局部总线或带命名空间前缀的 key 避免碰撞。
</details>

## ✍️ 实操题（3）

1. 用 `StringEventSystem.Global` 实现"关卡完成"广播：发送方携带 `LevelResult` payload，两个订阅方分别更新 UI 和存档，并在场景卸载时正确注销（用 `IUnRegister`）。
2. 用 `EnumEventSystem` 实现一个 `enum NetMsg { Login, Logout }` 的消息派发，`Login` 携带用户名和 token 两个参数，接收方安全取出。指出弱类型参数的两个风险点。
3. 设计一个 key 命名规范，避免不同模块在 `StringEventSystem.Global` 上的 key 碰撞与 payload 形态冲突，并说明为什么 `TypeEventSystem` 天然没有这个问题。

<details><summary>参考答案要点</summary>

1. `class LevelResult{...}`；发送 `Global.Send("level.complete", result)`；两处 `var u=Global.Register<LevelResult>("level.complete", r=>...)` 保存 `IUnRegister`；卸载时 `u.UnRegister()` 或 `u.UnRegisterWhenCurrentSceneUnloaded()`。
2. `Global.Register(NetMsg.Login, (id,args)=>{ var name=(string)args[0]; var token=(string)args[1]; })`；`Global.Send(NetMsg.Login, "alice", "tok123")`。风险：①参数顺序/类型全靠约定，错位即 `InvalidCastException`；②`args.Length` 不保证，越界访问需自行防御。
3. 规范：key 用 `模块名.事件名` 前缀（如 `"battle.enemyDead"`），并约定每个 key 的 payload 类型写在常量/文档里。`TypeEventSystem` 没问题是因为 key 就是类型本身，类型名全局唯一（命名空间隔离）、payload 即类型字段，编译期强约束，不存在拼写/形态冲突。
</details>
