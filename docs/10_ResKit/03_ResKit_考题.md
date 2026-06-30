# 10 · ResKit 考题

## 🟢 概念题（5）

1. ResKit 的"双层引用计数"指哪两层？
2. `ResState` 有哪几个状态？
3. `SimpleRC`、`SafeARC`、`UnsafeARC` 三种引用计数实现的区别？
4. 同一个资源被两个 ResLoader 加载，全局表里有几份 Res 对象？
5. `ResLoader` 为什么实现 `IPoolable`？回收时会发生什么？

<details><summary>参考答案要点</summary>

1. 资源级（`Res : SimpleRC` 的 `RefCount`，归零卸载 asset）和加载器级（`ResLoader.mResList` 对持有资源的 Retain/Release）。
2. `Waiting`、`Loading`、`Ready`。
3. `SimpleRC` 用整数计数，Release 到 0 触发 `OnZeroRef`；`SafeARC` 用 `HashSet<object> owners`，按 owner 去重（重复 Retain/Release 同一 owner 会报错，便于 debug）；`UnsafeARC` 继承 SimpleRC（无额外检查）。
4. 一份。`ResMgr.GetRes` 先查全局表，命中即返回同一对象；资源全局唯一，靠引用计数管理共享。
5. 因为加载器本身高频创建。回收（`OnRecycled`/`Recycle2Cache`）时调 `ReleaseAllRes()`——释放它持有的所有资源引用 + 销毁登记的实例化对象，再还池。
</details>

## 🟡 机制题（6）

1. `ResLoader.Add2Load` 做了哪些事？引用计数如何变化？
2. 资源 `RefCount` 归零后，asset 立即从全局表删除吗？描述完整卸载流程。
3. `RemoveUnusedRes` 的卸载判定条件是什么？为什么要判断 `State != Loading`？
4. `Res.RegisteOnResLoadDoneEvent` 在资源已 Ready 时的行为？事件触发后为什么置 null？
5. `ReleaseAllRes` 为什么要 `mResList.Reverse()`？
6. `ResMgr` 如何控制异步加载的并发数？

<details><summary>参考答案要点</summary>

1. 先在 mResList 查是否已有该资源（有则只加监听返回）；否则 `ResMgr.GetRes(keys, true)` 取/建资源 → 连带 `Add2Load` 其依赖 → `AddRes2Array`：`res.Retain()`（RefCount+1）+ 加入 mResList + 若未 Ready 加入 mWaitLoadList。
2. 不立即。`Release()` 归零触发 `OnZeroRef→ReleaseRes`（卸载 asset，State 回 Waiting），同时 `ResLoader` 调 `ResMgr.ClearOnUpdate()` 标脏；`ResMgr.Update` 检测脏标记后 `RemoveUnusedRes` 遍历表，对 RefCount<=0 且非 Loading 的资源 `ReleaseRes + Table.Remove + Recycle2Cache`（资源对象回池）。
3. `res.RefCount <= 0 && res.State != ResState.Loading`。判断非 Loading 是因为正在异步加载的资源即使引用为 0 也不能卸——加载协程还在用它，卸了会崩或加载完成写入已释放对象。
4. 立即同步回调 `listener(true, this)` 并 return（不挂事件）。触发后置 null 是一次性语义——加载完成事件只通知一次，避免重复回调和悬挂引用。
5. 确保先释放 AssetBundle 后释放 Asset（注释明示），这样能对 Asset 的卸载做优化（AB 卸载时机正确）。
6. `mIEnumeratorTaskStack`（LinkedList 排队）+ `mMaxCoroutineCount=8` + `mCurrentCoroutineCount` 计数。`TryStartNextIEnumeratorTask` 在当前协程数 < 上限时才启动下一个，完成时 `--count` 再尝试启动。
</details>

## 🔴 架构陷阱题（8）

1. **同母题对比**：ResKit 的"延迟卸载（脏标记+Update扫描）"和 ActionKit 的"延迟回收（ActionQueue）"都属于"迭代安全"母题。二者的触发与执行机制有何异同？
2. 某 ResLoader `Add2Load("tex")` 了两次同一资源。RefCount 会 +2 吗？
3. 一个 ResLoader 加载了资源但 `Dispose` 时忘了调用（既不 Dispose 也不 Recycle2Cache）。后果？
4. 资源 A 正在异步加载（State=Loading），此时唯一持有它的 Loader 被 Dispose（RefCount→0）。资源会被卸载吗？加载完成后呢？
5. 两个 Loader 共享资源 X。LoaderA 调 `Release()` 两次（多释放一次）。会发生什么？
6. 为什么 `RemoveUnusedRes` 遍历的是 `Table.ToArray()` 而不是直接 `foreach Table`？
7. `ResSearchKeys.Allocate` 里把 `assetName.ToLower()`。这个细节会导致什么隐蔽问题？
8. 加载一个有依赖的 AB 资源，只 Release 了主资源没管依赖。依赖会泄漏吗？框架如何处理依赖引用？

<details><summary>参考答案要点</summary>

1. 相同点：都把"修改共享集合"的操作推迟到安全点统一执行，避免遍历中改集合。不同点：ActionKit 是 `Deinit` 时主动入回收队列、`ActionQueue.Update` 末尾 Flush（每帧都查队列）；ResKit 是 `Release` 时置脏标记 `mIsResMapDirty`、`ResMgr.Update` 仅在脏时才扫描全表卸载（按需触发，且是全表扫描而非精确队列）。ResKit 更"批量惰性"，ActionKit 更"精确即时（下一帧）"。
2. 不会 +2。`Add2Load` 先 `FindResInArray(mResList)` 查重，已存在则只加监听直接 return，不重复 Retain。同一 Loader 对同一资源只持有一份引用。
3. 该 Loader 持有的所有资源引用永不释放（RefCount 不减），这些资源永远 RefCount>0 不会被卸载——资源内存泄漏。这是为什么 ResLoader 通常配合 `ResLoaderOnDestroyRecycler` 或在 MonoBehaviour.OnDestroy 里 Recycle2Cache。
4. 不会立即卸载。`OnZeroRef` 检查 `State==Loading` 时 return（不卸）；`RemoveUnusedRes` 也跳过 Loading 的资源。加载完成（State→Ready）后，因 RefCount 仍为 0，下次 `ClearOnUpdate` 触发的扫描会把它卸载。（注意：需有后续的脏标记触发扫描）
5. RefCount 会变成意外的值（比共享方少 1）。若 LoaderB 仍认为自己持有 X，但 X 的 RefCount 已被多减导致提前归零卸载，LoaderB 后续访问 `X.Asset` 得到已卸载的 null/无效资源——悬挂引用。`SafeARC` 的 owner HashSet 正是为在 debug 期捕获这类"未持有却 Release"。
6. 因为 `RemoveUnusedRes` 在遍历中会 `Table.Remove(res)` 修改表。直接 foreach 会抛"集合已修改"异常。`ToArray()` 先做快照，在快照上遍历、对原表删除，迭代安全。
7. assetName 被强制小写，但 `OriginalAssetName` 保留原名。隐患：若两个资源名仅大小写不同（`Hero.png` vs `hero.png`）会被当成同一资源（key 冲突）；且区分大小写的平台（如某些 AB/文件系统）上按小写名查找可能找不到文件——需保证命名规范统一小写。
8. 不会泄漏（框架处理了）。`Add2Load` 时对 `GetDependResList()` 的每个依赖也 `Add2Load`（连带 Retain）；`Res.HoldDependRes/UnHoldDependRes` 维护依赖引用。主资源 Release 时其依赖也通过加载器持有的引用链一并 Release。但前提是依赖也加入了 mResList——若手动绕过 Add2Load 直接操作就可能泄漏。
</details>

## ✍️ 实操题（3）

1. 用两个 ResLoader 加载同一资源，验证引用计数共享：A 释放后资源不卸载，B 也释放后才卸载。写出关键断言。
2. 解释为什么 ResLoader 通常作为 MonoBehaviour 的字段并在 OnDestroy 里 Recycle2Cache，以及不这样做的后果。
3. 设计一个"加载完成回调"的用法：异步加载一组资源，全部 Ready 后执行初始化。说明 `Add2Load(keys, listener)` 与 `RegisteOnResLoadDoneEvent` 在"资源已 Ready"时的行为差异如何影响你的代码。

<details><summary>参考答案要点</summary>

1. `a.LoadResSync("x"); b.LoadResSync("x");` → `Assert(ResMgr.Count==1)`；`a.Dispose(); ResMgr.Tick(); Assert(ResMgr.Count==1)`（B 还持有）；`b.Dispose(); ResMgr.Tick(); Assert(ResMgr.Count==0)`（引用归零卸载）。
2. ResLoader 持有资源引用，生命周期应绑定到使用它的对象。作为字段 + OnDestroy 里 Recycle2Cache，保证对象销毁时自动释放所有引用，避免泄漏。不这样做：对象销毁但 Loader 仍持有引用，资源 RefCount 不归零，永不卸载（内存泄漏累积）。
3. `loader.Add2Load("a", onDone); loader.Add2Load("b", onDone); loader.LoadAsync(()=>Init())`——`LoadAsync` 的 listener 在全部加载完成后触发。`RegisteOnResLoadDoneEvent`/`Add2Load` 的 listener 若资源已 Ready 会**立即同步回调**（不等下一帧），这意味着对已缓存资源注册回调时代码会同步执行——若回调里假设"异步稍后执行"可能导致时序错误（如在注册前没准备好的状态被立即访问）。需注意这种"已就绪立即回调"的同步性。
</details>
