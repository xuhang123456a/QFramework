# 02 · PoolKit 考题

## 🟢 概念题（5）

1. `Pool<T>` 底层用 `Stack<T>` 还是 `Queue<T>`？这对 `Allocate`/`Recycle` 意味着什么？
2. 四种 `IObjectFactory<T>` 各在什么场景使用？
3. `SafeObjectPool` 与 `SimpleObjectPool` 最本质的区别是什么？
4. `IPoolable.IsRecycled` 表达的是什么状态？
5. `ListPool<T>.Get()` 在池空时返回什么？

<details><summary>参考答案要点</summary>

1. `Stack<T>`（LIFO）。`Allocate=Pop`、`Recycle=Push`，最近回收的最先被复用（cache 友好）。
2. Default=`new T()`（需 `new()` 约束）；Custom=自定义 `Func<T>`（如造 GameObject）；NonPublic=反射调私有无参构造；可运行时 `SetObjectFactory` 替换。
3. Safe 有重复回收防护（`IsRecycled`）+ 容量上限 + 单例；Simple 零校验、可处理任意类型、性能优先。
4. 对象当前是否"在池中"。true=在池/已被回收，false=在外被使用。
5. `new List<T>(8)`（容量 8 的新列表）。
</details>

## 🟡 机制题（6）

1. `SafeObjectPool.Init(maxCount, initCount)` 内部如何预填充？为什么用 `Recycle` 而非直接 `Push`？
2. 当 `MaxCacheCount` 被调小到小于当前栈量时会发生什么？
3. `SafeObjectPool.Recycle` 在"池已满"时返回 false，但它还做了一件容易被忽略的事，是什么？为什么重要？
4. `SimpleObjectPool` 的 `resetMethod` 在何时被调用？
5. `Pool.Clear(Action<T> onClearItem)` 与直接 `mCacheStack.Clear()` 有何区别？
6. `ListPool.Release` 和 `DictionaryPool.Release` 在防重复回收上有何不对称？

<details><summary>参考答案要点</summary>

1. 循环 `for i=CurCount..initCount: Recycle(mFactory.Create())`。用 `Recycle` 是为了让对象走完整的入池路径（置 `IsRecycled=true` + `OnRecycled`），保证状态一致。
2. setter 里 `while(栈量>上限) Pop()` 丢弃超额对象，交给 GC。
3. 仍调用 `t.OnRecycled()`。重要性：`OnRecycled` 是"对象被归还"的资源清理钩子，满栈丢弃的对象也需要清理它持有的资源，否则泄漏。
4. `Recycle(obj)` 时，入栈前 `mResetMethod?.Invoke(obj)`。
5. 带 `onClearItem` 会先对栈中每个对象回调（用于 `Object.Destroy` 等显式释放），再 Clear；不带则只丢引用靠 GC。
6. `ListPool.Release` 会 `Contains` 查重并抛 `InvalidOperationException`；`DictionaryPool.Release` 无此检查——重复回收同一字典不会报错但会破坏池语义。
</details>

## 🔴 架构陷阱题（7）

1. **同底座对比**：ActionKit 内部全用 `SimpleObjectPool` 而非 `SafeObjectPool`，为什么它"敢"用不安全的池？
2. 某对象实现了 `IPoolable`，但 `OnRecycled` 里只重置了字段，没释放它持有的 `Texture`。在"满栈丢弃"场景下会怎样？
3. 调用方对同一个 `Bullet` 连续 `Recycle` 两次，Safe 池和 Simple 池分别会怎样？后果是什么？
4. 用 `Allocate` 取出对象后忘了在某条分支 `Recycle`，会"泄漏"吗？和忘记 `Recycle` 一个 `SafeObjectPool` 对象相比，对池本身有何影响？
5. `SafeObjectPool<T>` 是单例（`SingletonProperty<SafeObjectPool<T>>`）。这意味着 `SafeObjectPool<Bullet>` 全局只有一份。这在多场景/多功能复用同类型对象时可能带来什么问题？
6. 把 `mCacheStack` 的初始 `mMaxCount` 默认值（12）误解为"池最多装 12 个"，但 `SimpleObjectPool` 从不检查 `mMaxCount`。会有什么后果？
7. 两个线程同时 `Allocate`/`Recycle` 同一个 `SafeObjectPool`，安全吗？

<details><summary>参考答案要点</summary>

1. ActionKit 内部严格控制每个 Action 的 `Deinit` 流程（且通过 `Deinited` 标志 + `ActionQueue` 延迟回收防重入），调用方不直接碰 Recycle，重复回收的风险被框架内部消除，于是用零开销的 Simple 池即可。
2. 满栈时 `OnRecycled` 仍被调用，但因为 `OnRecycled` 没释放 Texture，且对象被丢弃（不入栈），Texture 引用随对象等待 GC——托管内存最终回收，但若是 Unity 非托管资源（未 `Destroy`）则真正泄漏。
3. Safe：第二次 `Recycle` 因 `IsRecycled==true` 直接 return false（安全拒绝）。Simple：第二次直接 `Push`，导致**同一对象在栈里出现两次**，后续两次 `Allocate` 会拿到同一个对象的两个"别名"，引发诡异的状态共享 bug。
4. 忘记 Recycle：对象不会回池，下次 Allocate 会 new 新的——池的复用率下降（"逻辑泄漏"），但对象本身能被 GC（无强引用时）。对池本身：池只是少了一个可复用对象，不会崩，但失去池化收益。
5. 单例意味着所有使用方共享同一缓存栈与同一 `MaxCacheCount`。若场景 A 设置 max=10、场景 B 期望 max=100，会互相干扰；且对象可能携带上一使用方的残留状态（依赖 `OnRecycled` 彻底重置）。
6. `mMaxCount` 默认 12 只对 `SafeObjectPool` 生效；`SimpleObjectPool.Recycle` 无视它，会无限增长——若回收远多于分配，栈会持续膨胀（内存只增不减），需要手动 `Clear`。
7. 不安全。`Stack<T>` 非线程安全，`IsRecycled` 读写也无锁。并发 Allocate/Recycle 会破坏栈结构或导致同一对象被两个线程同时使用。PoolKit 设计面向 Unity 主线程单线程模型。
</details>

## ✍️ 实操题（3）

1. 实现一个 `SafeObjectPool<Particle>`（`Particle : IPoolable`），`OnRecycled` 重置位置与速度。写代码验证 Allocate→Recycle→Allocate 拿到的是同一实例。
2. 用 `SimpleObjectPool<GameObject>` 实现一个子弹对象池，工厂方法 `Instantiate(prefab)`，reset 方法 `SetActive(false)`。说明为什么这里**不能**用 `SafeObjectPool`。
3. 用 `ListPool<int>` 优化一段每帧 `new List<int>()` 的临时计算代码，并指出若忘记 `Release` 会怎样、若 `Release` 后继续使用该 List 会怎样。

<details><summary>参考答案要点</summary>

1. `Particle:IPoolable{IsRecycled;OnRecycled(){pos=0;vel=0;}}`；`var p1=pool.Allocate(); pool.Recycle(p1); var p2=pool.Allocate(); Assert(ReferenceEquals(p1,p2))`（栈 LIFO 复用同一对象）。
2. `new SimpleObjectPool<GameObject>(()=>Object.Instantiate(prefab), go=>go.SetActive(false), 10)`。不能用 Safe：`GameObject` 不实现 `IPoolable` 且非 `new()`（Unity 对象必须 Instantiate）。
3. `var list=ListPool<int>.Get(); ...; list.Release2Pool();`。忘记 Release：每帧仍 new（失去优化，且池永远空）。Release 后继续用：该 List 已被 Clear 且可能被其他代码 Get 出去复用，造成数据错乱/并发修改——属于"释放后使用"经典错误。
</details>
