# 03 · SingletonKit 考题

## 🟢 概念题（5）

1. `ISingleton` 强制实现的唯一方法是什么？它的调用时机？
2. 继承式单例与属性式单例的核心差异是什么？
3. `SingletonCreator` 如何决定一个类型走"纯 C#"还是"Mono"创建路径？
4. 纯 C# 单例为什么要求私有构造函数？
5. `MonoSingletonPathAttribute` 的作用是什么？

<details><summary>参考答案要点</summary>

1. `OnSingletonInit()`。在 `SingletonCreator` 把实例造好之后、返回给调用方之前调用一次。
2. 继承式（`Singleton<T>`/`MonoSingleton<T>`）通过基类继承注入 `Instance`，占用唯一继承位；属性式（`*Property<T>`）通过静态泛型类提供 `Instance`，类只需 `implements ISingleton`，保留继承位。
3. `typeof(MonoBehaviour).IsAssignableFrom(typeof(T))`：是则 `CreateMonoSingleton`，否则反射私有构造造纯 C# 对象。
4. 防止外部 `new` 破坏唯一性；Creator 用反射 `BindingFlags.NonPublic` 调私有无参构造。
5. 指定 Mono 单例 GameObject 在 Hierarchy 的路径与命名（如 `[MyGame]/GameManager`），创建时按路径查找/构建节点树。
</details>

## 🟡 机制题（6）

1. 描述 `CreateMonoSingleton<T>` 的完整查找/创建回退链。
2. 为什么 `CreateMonoSingleton` 开头要判断 `!IsUnitTestMode && !Application.isPlaying`？
3. 纯 C# 单例的 `Instance` getter 为什么要 `lock`，而 Mono 版没有？
4. `MonoSingleton.Dispose()` 在测试态和运行态分别做什么？
5. `OnApplicationQuit` 在 `MonoSingleton` 里为什么要清 `mInstance`？
6. `SafeObjectPool<T>` 是怎样借助 SingletonKit 成为单例的？

<details><summary>参考答案要点</summary>

1. `FindObjectOfType(type)` 命中则用；否则读 `MonoSingletonPath` 特性按路径 `FindGameObject`(可创建) + `AddComponent`；仍为 null 则 `new GameObject(类型名)` + `DontDestroyOnLoad` + `AddComponent`。最后 `OnSingletonInit`。
2. 编辑器非运行态访问 Mono 单例会创建脏 GameObject 污染场景，故直接返回 null；测试态例外（允许）。
3. 纯 C# 单例可能在非主线程访问，用 lock 双检防并发重复创建；Mono 单例依赖 `FindObjectOfType`/GameObject API，本就只能主线程调用，无需锁。
4. 测试态：向上遍历销毁整条父链 GameObject（`DestroyImmediate`）+ `mInstance=null`；运行态：`Destroy(gameObject)`。
5. 应用退出时销毁单例 GO 并置 null，避免退出过程中残留引用触发空对象访问/重复创建。
6. `SafeObjectPool<T> : Pool<T>, ISingleton`，`Instance => SingletonProperty<SafeObjectPool<T>>.Instance`，构造私有（`protected SafeObjectPool()`），由 `SingletonCreator` 反射创建。
</details>

## 🔴 架构陷阱题（7）

1. **同底座对比**：`Singleton<T>`（继承）与 `SingletonProperty<T>`（属性）都能造纯 C# 单例，分别在什么情况下只能用其中一个？
2. 一个类写了 `public XxxManager() {}`（公有构造）又继承 `Singleton<T>`。唯一性还成立吗？
3. 场景里手动拖了一个挂着 `GameManager : MonoSingleton<GameManager>` 的物体，代码又在别处首次访问 `GameManager.Instance`。会出现两个吗？
4. 在编辑器非运行态（如自定义 EditorWindow）访问某 `MonoSingleton.Instance`，拿到什么？为什么这样设计？
5. `MonoSingleton` 跨场景切换时不会被销毁的原因是什么？如果忘记这一点，会出现什么 bug？
6. 两个不同类型 `SafeObjectPool<A>` 和 `SafeObjectPool<B>` 是同一个单例吗？依据是什么？
7. 调用 `GameData.Instance.Dispose()` 之后再访问 `GameData.Instance`，是同一个对象吗？此前修改的字段还在吗？

<details><summary>参考答案要点</summary>

1. 若单例类已经需要继承别的基类（如继承某 `BaseManager`），继承位被占，只能用属性式 `SingletonProperty<T>`；若类是全新、无其他基类需求，继承式更简洁。Mono 同理。
2. 名义上 `Instance` 仍唯一，但公有构造让任何代码能 `new XxxManager()` 绕过单例，破坏"全局唯一"保证。框架靠"私有构造 + 反射"来堵死这个口子，公有构造等于自废武功。
3. 不会。`CreateMonoSingleton` 先 `FindObjectOfType` 命中场景里手动摆的那个并复用（还会调一次 `OnSingletonInit`）。这正是回退链第一步的意义。
4. 拿到 null（除非 `IsUnitTestMode`）。设计目的：编辑器非运行态不应创建运行时 GameObject，否则会在场景里生成无法预期的幽灵对象。
5. `CreateMonoSingleton` 新建 GO 时调了 `DontDestroyOnLoad`。忘记的话（仿写时漏掉），场景切换后单例 GO 被销毁，`OnDestroy` 置 `mInstance=null`，下次访问又新建——单例在每次场景切换"重置"，丢失跨场景状态。
6. 不是。`SingletonProperty<SafeObjectPool<A>>` 与 `SingletonProperty<SafeObjectPool<B>>` 是两个不同的封闭泛型静态类型，各持一份 `mInstance`。泛型类型参数不同 = 不同单例。
7. 不是同一个。`Dispose` 把 `mInstance=null`，下次访问重新反射构造新实例并重跑 `OnSingletonInit`；旧对象的字段修改全部丢失（旧对象交给 GC）。
</details>

## ✍️ 实操题（3）

1. 写一个纯 C# 的 `ConfigManager : Singleton<ConfigManager>`，私有构造，`OnSingletonInit` 里加载默认配置，提供 `Get(key)`。并解释为什么不能用 `new ConfigManager()`。
2. 写一个属性式 Mono 单例 `PoolRoot`（它已经必须继承某个业务基类 `GameComponent`），用 `MonoSingletonProperty<PoolRoot>` 暴露 `Instance`，说明为什么这里不能用 `MonoSingleton<T>`。
3. 演示 `SafeObjectPool<Bullet>.Instance.Init(50,25)` 背后经过了哪些单例创建步骤（结合 SingletonCreator 流程）。

<details><summary>参考答案要点</summary>

1. `class ConfigManager:Singleton<ConfigManager>{ private ConfigManager(){} public override void OnSingletonInit(){/*load*/} public string Get(string k)=>...; }`。不能 `new`：构造私有，编译期就报错；只能 `ConfigManager.Instance`。
2. `PoolRoot : GameComponent, ISingleton`（继承位已被 `GameComponent` 占），`public static PoolRoot Instance => MonoSingletonProperty<PoolRoot>.Instance;` + 实现 `OnSingletonInit`/`Dispose` 转发。不能用 `MonoSingleton<PoolRoot>`：C# 单继承，`PoolRoot` 已继承 `GameComponent`，无法再继承 `MonoSingleton`。
3. `Instance` getter → `SingletonProperty<SafeObjectPool<Bullet>>.Instance` → lock 双检 mInstance==null → `SingletonCreator.CreateSingleton<SafeObjectPool<Bullet>>()` → 判定非 MonoBehaviour → 反射调 `protected SafeObjectPool()`（设 `mFactory=DefaultObjectFactory`）→ `OnSingletonInit()`（空）→ 返回实例 → 再调 `.Init(50,25)` 预填充 25 个 Bullet。
</details>
