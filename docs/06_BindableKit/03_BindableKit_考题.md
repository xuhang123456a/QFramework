# 06 · BindableKit 考题

## 🟢 概念题（5）

1. `BindableList<T>` 继承自哪个 BCL 类型？为什么不直接继承 `List<T>`？
2. `BindableDictionary` 为什么不能用和 `BindableList` 一样的"继承重写"手法？
3. BindableKit 的事件字段为什么都是惰性创建的？
4. `BindableList` 提供了哪些细粒度变更事件？
5. `BindableProperty<T>` 在 BindableKit 还是 Core 里定义？

<details><summary>参考答案要点</summary>

1. `Collection<T>`。因为 `Collection<T>` 把所有写操作收敛到 `InsertItem/RemoveItem/SetItem/ClearItem` 等 protected 虚方法，子类一处重写即可拦截全部写入；`List<T>` 的方法非虚，无法拦截。
2. `Dictionary<TKey,TValue>` 没有可重写的虚写入方法，所以只能"包装内部 Dictionary + 自己实现完整 IDictionary 接口"，在写方法里手动 Trigger。
3. 集合操作高频，若每个实例一创建就 new 多个 EasyEvent 开销大；惰性（`?? new`）让无人订阅的事件零成本。
4. `OnAdd`、`OnRemove`、`OnReplace`、`OnMove`、`OnClear`、`OnCountChanged`。
5. Core（`QFramework.cs`）。BindableKit 提供的是响应式集合（List/Dictionary）和持久化变体。
</details>

## 🟡 机制题（6）

1. 调用 `list.Add(x)` 时，事件是如何被触发的？经过了哪些方法？
2. `list[i] = newItem` 触发哪些事件？哪个**不**触发？为什么？
3. `Clear()` 一个本来就空的列表，会触发 `OnCountChanged` 吗？
4. `BindableDictionary` 的 `this[key]=value` 在 key 已存在和不存在时分别触发什么？
5. 为什么事件触发处用 `mOnAdd?.Trigger(...)` 而不是 `OnAdd.Trigger(...)`？
6. `BindableProperty.RegisterWithInitValue` 比 `Register` 多做了什么？

<details><summary>参考答案要点</summary>

1. `list.Add(x)` → `Collection<T>` 内部调 `InsertItem(Count, x)` → 子类 override 的 `InsertItem` → `base.InsertItem` 真插入 → `mCollectionAdd?.Trigger(index, item)` → `mOnCountChanged?.Trigger(Count)`。
2. 触发 `OnReplace(i, old, new)`；**不触发** `OnCountChanged`，因为替换不改变元素数量。
3. 不会。`ClearItems` 里 `if(beforeCount>0)` 才触发 `OnCountChanged`；空集合 beforeCount==0，跳过。但 `OnClear` 仍会触发。
4. key 存在：`OnReplace(key, old, new)`；key 不存在：`OnAdd(key, value)` + `OnCountChanged`。
5. 用字段 `mOnAdd?.` 在未订阅（字段为 null）时直接跳过，保持惰性；用属性 `OnAdd` 会触发 `?? new` 强制创建事件对象，破坏惰性优化。
6. 在注册的瞬间先用当前值回调一次 `on(mValue)`，再注册——保证订阅者初始即拿到当前值（UI 初始化与数据同步）。
</details>

## 🔴 架构陷阱题（7）

1. **同手法对比**：`BindableList`（继承 `Collection<T>`）与 `BindableDictionary`（包装 `Dictionary`）两种"加事件"手法，各自的代价与局限是什么？
2. 有人为了省事，在 `BindableList` 的触发处写成 `OnAdd.Trigger(i,item)`（用属性）。功能正常吗？有什么隐藏代价？
3. UI 订阅了 `inventory.OnAdd` 做增量刷新。某次代码用 `inventory.Clear()` 后重新 `Add` 了 50 个元素。UI 会发生什么？这样性能好吗？
4. 一个 `BindableProperty<List<int>>`，每次往 List 里 `Add` 但不重新赋值 `Value`。订阅者会收到通知吗？为什么？这揭示了 BindableProperty 的什么局限？
5. `BindableList` 标了 `[Serializable]` 但事件字段标 `[NonSerialized]`。如果事件字段没标 `[NonSerialized]` 会有什么风险？
6. 订阅 `OnReplace` 期望"数量变化时刷新总数 UI"。这个订阅放错了事件，会怎样？正确应订阅哪个？
7. 在遍历 `BindableList` 的过程中，某个 `OnAdd` 回调又往这个 list 里 Add 元素。会出问题吗？

<details><summary>参考答案要点</summary>

1. 继承 `Collection<T>`：优雅、一处重写全拦截，但只适用于"BCL 提供了虚写方法收敛点"的类型；包装 `Dictionary`：必须手动实现完整 `IDictionary`/`ICollection`/`ISerializable` 等一大堆显式接口成员（代码冗长），且每个写入口都要记得手动 Trigger，易漏。
2. 功能正常，但每次 Add 都通过属性 `?? new` 确保事件存在——即便没有任何订阅者，也会分配并持有 EasyEvent 对象，惰性优化失效，高频场景产生持续小额分配。
3. UI 会收到 1 次 `OnClear` + 50 次 `OnAdd`（外加 Count 变更）。若 UI 在 OnAdd 里逐项创建条目，等于 50 次增量刷新——对"批量重置"场景不如先 `OnClear` 清空再批量重建。增量事件擅长零散增删，批量重置时反而产生大量细粒度回调，需在 UI 侧合并/分帧处理。
4. 不会收到通知。`BindableProperty` 只在 `Value` 的 set 被赋值且经 `Comparer` 判定变化时 Trigger；往引用类型内部 Add 不触发 set。局限：BindableProperty 监听的是"引用/值的替换"，不是"对象内部状态变化"。要监听集合内容变化应改用 `BindableList`。
5. 序列化时会尝试序列化事件委托链（指向各订阅者对象），可能拉入大量无关对象、序列化失败或反序列化后产生悬挂回调。`[NonSerialized]` 把事件排除在序列化之外，只持久化数据本身。
6. `OnReplace` 在替换（数量不变）时触发，订阅它刷新"总数"会在数量没变时白刷，且真正数量变化（Add/Remove）时收不到。正确应订阅 `OnCountChanged`。
7. 取决于遍历方式。`Collection<T>` 底层是 `List`，用 `foreach` 遍历时若回调修改集合会抛 `InvalidOperationException`（集合已修改）。这是迭代中增删的经典问题——需用快照遍历或延迟修改（参见 ActionKit 的 ActionQueue 延迟回收母题）。
</details>

## ✍️ 实操题（3）

1. 用 `BindableList<Item>` 建模背包，View 订阅 `OnAdd`/`OnRemove` 做增量 UI（加一个格子/删一个格子），并在 GameObject 销毁时注销。说明为什么这比"每次变化都重建整个背包 UI"高效。
2. 用 `BindableProperty<int>` 建模金币，演示 `RegisterWithInitValue` 让金币 UI 一注册就显示当前值；再演示设置 `Comparer` 没必要（int 已由 `ComparerAutoRegister` 处理）的原因。
3. 解释为什么"监听一个 List 内部元素增删"必须用 `BindableList` 而不能用 `BindableProperty<List<T>>`，并写出错误用法与正确用法对比。

<details><summary>参考答案要点</summary>

1. `OnAdd.Register((i,item)=>CreateSlot(i,item)).UnRegisterWhenGameObjectDestroyed(go)`；`OnRemove` 同理删格子。高效原因：只更新发生变化的单个条目，避免每次增删都 Destroy 全部子物体再重建（O(n) GC + 布局重算）。
2. `coin.RegisterWithInitValue(v=>label.text=v.ToString())` 注册即显示当前金币；无需手动 `Comparer`，因为 `ComparerAutoRegister`（Core）已在 BeforeSceneLoad 为 `BindableProperty<int>` 设了 `==` 比较器，避免装箱并正确去重。
3. 错误：`var bp=new BindableProperty<List<int>>(new()); bp.Register(...); bp.Value.Add(1);`——内部 Add 不触发 set，订阅者收不到。正确：`var list=new BindableList<int>(); list.OnAdd.Register(...); list.Add(1);`。原因：BindableProperty 监听引用替换，BindableList 监听内容变更。
</details>
