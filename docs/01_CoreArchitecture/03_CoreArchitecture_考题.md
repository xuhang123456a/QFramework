# 01 · CoreArchitecture 考题

> 陷阱题优先考"同底座的两个机制有何不同"与"漏掉某一步的后果"。

## 🟢 概念题（5）

1. `IArchitecture` 把哪三类对象注册进容器？它们职责上的根本区别是什么？
2. `Command` 和 `Query` 的能力接口差异体现在哪里？为什么 Query 不能改数据？
3. `EasyEvent`、`EasyEvent<T>`、`TypeEventSystem` 三者是什么关系？
4. `IUnRegister` 的作用是什么？`CustomUnRegister` 为什么是 struct？
5. `BindableProperty<T>.RegisterWithInitValue` 与 `Register` 的区别是什么？

<details><summary>参考答案要点</summary>

1. System（逻辑）、Model（数据）、Utility（无状态工具）。Utility 注册时不调用 Init（无生命周期），Model/System 有 `Init/Deinit`。
2. Command 继承 `ICanSendCommand/ICanSendEvent/ICanGetSystem/ICanGetModel/...`（能读能写能发事件）；Query 只继承 `ICanGetModel/ICanGetSystem/ICanSendQuery`（只读）。靠接口组合在编译期约束，无运行时检查。
3. `EasyEvent`/`EasyEvent<T>` 是委托封装的最小事件单元；`EasyEvents` 是 `Type→IEasyEvent` 字典；`TypeEventSystem` 是对 `EasyEvents` 的门面，把"事件类型"当 key。
4. 注销句柄，把"如何注销"封装成对象返回给注册方。struct 避免每次注册都堆分配（注册是高频操作）。
5. `RegisterWithInitValue` 在注册瞬间先用当前值回调一次，再注册——保证 UI 初始值与数据同步；`Register` 只在之后的变更时回调。
</details>

## 🟡 机制题（6）

1. 描述首次访问 `XxxArchitecture.Interface` 时发生的完整初始化序列。
2. 为什么 `InitArchitecture` 先初始化所有 Model 再初始化所有 System？
3. 在架构已 `Init` 之后再 `RegisterSystem`，会发生什么？源码靠什么字段判断？
4. `SendEvent<T>()`（无参，`where T:new()`）和 `SendEvent<T>(T e)` 在内存上有何不同？
5. `BindableProperty<T>.Value` 的 set 在什么条件下**不会**触发事件？
6. `ComparerAutoRegister` 解决了什么问题？它在何时运行？

<details><summary>参考答案要点</summary>

1. `mArchitecture==null` → `new T()` → `mArchitecture.Init()`（派生类注册 Model/System/Utility）→ `OnRegisterPatch?.Invoke` → 遍历容器中未初始化的 Model 依次 `Init()`+置 `Initialized` → 同样处理 System → `mInited=true`。
2. 数据层先就绪，逻辑层启动时才能安全读到 Model；反序会让 System.OnInit 读到未初始化的 Model。
3. 立即 `system.Init(); system.Initialized=true`。靠 `mInited` 字段（true 表示已过批量初始化阶段）。
4. 无参版 `mEvents.GetEvent<EasyEvent<T>>()?.Trigger(new T())` 每次 new 一个事件对象（有分配）；带参版直接 Trigger 传入对象（调用方负责分配）。
5. ①新旧都为 null；②新值非 null 且 `Comparer(value, mValue)` 返回 true（视为相等）。
6. 默认 `Comparer = (a,b)=>a.Equals(b)` 对值类型会装箱且语义可能不准；`ComparerAutoRegister` 用 `[RuntimeInitializeOnLoadMethod(BeforeSceneLoad)]` 在场景加载前为 int/float/Vector3 等常用类型替换为 `==` 比较，避免装箱、提升正确性与性能。
</details>

## 🔴 架构陷阱题（7）

1. **同底座对比**：`TypeEventSystem`（按 Type key）与 `StringEventSystem`（EventKit，按 string key）都基于 `EasyEvents`/`EasyEvent`，二者各自的优缺点与误用风险是什么？
2. 有人把 `Architecture<T>` 的 `static T mArchitecture` 重构进一个非泛型基类 `ArchitectureBase`。会引发什么后果？
3. 注册了 100 个 `RegisterEvent<T>` 但从不 `UnRegister`，且这些回调捕获了 View 引用。会发生什么？框架提供了什么补救机制？
4. `UnRegisterEvent<T>(cb)` 之后，`mEvents` 字典里那个 `EasyEvent<T>` 对象还在吗？有何隐患？
5. 在 `Model.OnInit` 里调用 `this.GetSystem<XxxSystem>()` —— 能编译吗？为什么这是设计上故意的？
6. 调用 `Deinit()` 后立刻再访问 `Interface`，拿到的是同一个架构实例吗？容器里的 Model 数据还在吗？
7. 两个不同的 Command 对象在 `OnExecute` 中互相 `SendCommand`，会有什么风险？框架对 Command 的生命周期是怎么管理的？

<details><summary>参考答案要点</summary>

1. TypeEventSystem：编译期类型安全、事件即 DTO、IDE 可追溯，但每个事件要定义一个类型。StringEventSystem：灵活、无需定义类型，但 key 是魔法字符串、无类型检查、易拼写错且 payload 是 `object`/`object[]` 需强转。误用风险：String 版的 key 散落各处难维护，且 `Send` 找不到 key 时静默失败。
2. CLR 中静态字段属于"声明它的类型"。放进非泛型基类后所有派生架构共享同一个 `mArchitecture` 和容器——后初始化的架构会覆盖前者，多架构互相污染，`GetModel` 拿到错误架构的数据。
3. 这些回调（连同捕获的 View）随事件对象存活而无法 GC，造成内存泄漏 + View 销毁后仍被触发导致空引用。补救：`UnRegisterWhenGameObjectDestroyed/WhenDisabled/WhenCurrentSceneUnloaded`，把注销挂到 Unity 生命周期。
4. 还在。`UnRegister` 只是 `mOnEvent -= cb`，不移除字典项。隐患：长期运行会累积大量空 `EasyEvent<T>`（轻微泄漏），但更重要的是它不会引发功能错误，只是不回收空壳。
5. **不能编译**。`IModel` 只继承 `ICanGetUtility/ICanSendEvent`，不含 `ICanGetModel/ICanGetSystem`。这是故意的：防止 Model 依赖其他 Model/System，保持 Model 为纯数据层、避免循环耦合。
6. 不是同一个。`Deinit` 把 `mArchitecture=null`，下次 `Interface` 重新 `new T()` 并重新初始化；容器已 `Clear()`，旧 Model 数据全部丢失（除非 Model 自己持久化）。
7. 风险是递归/重入导致的栈溢出或状态不一致。框架不持有 Command（执行完即弃，靠 GC），不做重入保护——开发者需自行避免 Command 链路成环。
</details>

## ✍️ 实操题（3）

1. 写一个 `ScoreModel`（含 `BindableProperty<int> Score`）+ `AddScoreCommand` + `GameArchitecture`，并在控制台订阅分数变化。要求用 `RegisterWithInitValue`。
2. 用 `TypeEventSystem.Global` 实现一个跨模块的 `PlayerDiedEvent` 广播：一处发送，两处订阅，并演示其中一个订阅者在"满足条件后"注销自己。
3. 给 `BindableProperty<Vector3>` 设置自定义 `Comparer`，使得仅当两个向量距离超过 0.01 才视为"变化"并触发事件。解释这样做对 UI 刷新频率的影响。

<details><summary>参考答案要点</summary>

1. `ScoreModel : AbstractModel`，`OnInit` 空；`AddScoreCommand : AbstractCommand` 里 `this.GetModel<ScoreModel>().Score.Value += n`；`GameArchitecture : Architecture<GameArchitecture>` 在 `Init` 注册 Model；订阅 `Score.RegisterWithInitValue(v=>...)` 立即打印初值再追踪变更。
2. `struct PlayerDiedEvent {}`；`TypeEventSystem.Global.Send(new PlayerDiedEvent())`；两处 `Global.Register<PlayerDiedEvent>(...)`；其中一个回调内保存 `IUnRegister`，条件满足时调用 `.UnRegister()`。
3. `BindableProperty<Vector3>.Comparer = (a,b)=>Vector3.Distance(a,b)<0.01f;`。效果：小幅抖动不触发事件，降低 UI 重绘频率（节流），但牺牲了对微小变化的实时性。注意 `Comparer` 是 static，会影响所有 `BindableProperty<Vector3>` 实例。
</details>
