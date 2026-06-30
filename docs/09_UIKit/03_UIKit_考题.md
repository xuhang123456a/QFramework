# 09 · UIKit 考题

## 🟢 概念题（5）

1. UIKit 的三层结构是什么？各自职责？
2. `PanelSearchKeys` 和 `PanelInfo` 分别是什么？为什么都池化？
3. `UIPanelTable` 用了几个索引？分别按什么 key？
4. `PanelOpenType.Single` 和非 Single 在打开时有何区别？
5. `UIPanel` 的生命周期钩子有哪些？

<details><summary>参考答案要点</summary>

1. 门面 `UIKit`（静态、无状态、打包参数 + 重载糖）→ `UIManager`（Mono 单例、真正调度）→ `UIPanelTable`（双索引注册表、状态）。
2. SearchKeys 是"一次操作的查询参数快照"（Type/名/Level/Data/OpenType），调用级临时对象；PanelInfo 是"面板的重建元数据快照"，面板级对象。都池化是因为 UI 操作高频，避免频繁 new/GC，都用 `SafeObjectPool`。
3. 两个：`TypeIndex`（按 `panel.GetType()`）、`GameObjectNameIndex`（按 `Transform.name`），每个 key 对应 `List<IPanel>`。
4. Single：先查 Table 有无已存在面板，有则复用（仅重新 Open+Show），无才 CreateUI；非 Single：每次都 CreateUI 新建（允许同类型多实例）。
5. `OnInit`、`OnOpen`、`OnShow`、`OnHide`、`OnClose`（子类重写），由基类的 Init/Open/Show/Hide/Close 模板调用。
</details>

## 🟡 机制题（6）

1. 描述 `UIKit.OpenPanel<T>` 一次调用中 `PanelSearchKeys` 的生命周期。
2. `CreateUI` 做了哪些步骤？
3. `Close(destroy=true)` 时面板和资源分别如何处理？
4. `UIPanelStack.Push` 和 `Pop` 各做了什么？栈里存的是什么？
5. 为什么 `CloseUI` 用 `LastOrDefault` 而 `GetUI`/`ShowUI` 用 `FirstOrDefault`？
6. 为什么 `UIPanel` 的 `Close` 是显式接口实现 `void IPanel.Close()`？

<details><summary>参考答案要点</summary>

1. `PanelSearchKeys.Allocate()`（从 SafeObjectPool 取）→ 填 Type/Level/UIData/OpenType → 传给 `UIManager.OpenUI` → 方法末尾 `Recycle2Cache()` 还池。整个生命周期闭环在一次 API 调用内。
2. `Config.LoadPanel(keys)` 加载面板 → `Root.SetLevelOfPanel` 设层级 → `SetDefaultSizeOfPanel` 设尺寸 → 设 GameObject 名 → `PanelInfo.Allocate` 分配元数据 → `Table.Add`（双索引）→ `panel.Init(uiData)`。
3. 面板：回写 UIData 到 Info → 触发 mOnClosed → Hide → State=Closed → OnClose → `Destroy(gameObject)`；资源：`Loader.Unload()` 卸载 + `PanelLoaderPool.RecycleLoader` 回收 Loader + Loader 置 null。
4. Push：把 `panel.Info` 压栈 → `panel.Close()` → `RemoveUI`（仅从 Table 移除，不再持有面板）；Pop：弹出 `PanelInfo` → 用其字段重建 `PanelSearchKeys` → `OpenUI` 重新打开。栈里存的是 `PanelInfo`（元数据快照），不是面板实例。
5. 多开场景下：关闭时关"最新打开的那个"（Last），查询/显示时取"最早的那个"（First）——符合栈式打开、就近关闭的直觉。
6. 防止子类直接调用 Close 绕过管理器流程。子类只能用 `CloseSelf()`→`UIKit.ClosePanel(this)`，强制走 UIManager 的 Table 移除 + Info 回收 + Loader 卸载完整流程。
</details>

## 🔴 架构陷阱题（7）

1. **同母题对比**：UIKit 的 `PanelSearchKeys` 和 ActionKit 的动作对象都池化。但 SearchKeys 是"调用级"、动作是"任务级"生命周期，二者回收时机的设计差异是什么？
2. 某 API 实现里 `PanelSearchKeys.Allocate()` 后在某条 return 分支忘了 `Recycle2Cache()`。会发生什么？
3. 维护 `UIPanelTable` 时只在 `OnAdd` 更新了 `TypeIndex`，漏了 `GameObjectNameIndex`。哪些 API 会失灵？
4. 用 `UIPanelStack.Push` 压栈后，又手动 `UIKit.ClosePanel<T>()` 关闭了同名面板。之后 `Pop` 会怎样？
5. Pop 出来的面板和 Push 之前的面板是同一个对象吗？这对面板内部持有的运行时状态（如动画进度、临时字段）意味着什么？
6. `Close` 时若忘记把 `mUIData` 回写到 `Info.UIData`。对导航栈 Pop 有什么影响？
7. `OpenType.Multiple` 连续打开 3 个同类型面板，再 `ClosePanel<T>()` 一次。关掉的是哪个？剩下几个？它们的 GameObject 名冲突吗？

<details><summary>参考答案要点</summary>

1. SearchKeys 在单次 API 内 Allocate→用→Recycle 闭环（同步、立即回收）；动作对象生命周期跨多帧，回收必须延迟到安全点（ActionQueue）。差异根源：SearchKeys 用完即知可还（无并发遍历），动作可能正被 Executor 遍历故需延迟。体现"回收时机取决于对象是否可能正被他处引用/遍历"。
2. 该 SearchKeys 不回池——SafeObjectPool 里少一个可复用对象，下次 Allocate 走 new（轻微逻辑泄漏）。且因为 `IsRecycled` 仍是 false，对象本身能 GC，但池的复用率下降。
3. 按名字查的 API 失灵：`OpenPanel("name")`、`ClosePanel("name")`、`GetPanel("name")`、`Back("name")`——`GameObjectNameIndex.Get` 返回空，找不到面板。按 Type 的 API 仍正常。
4. `ClosePanel<T>` 已经把面板 Close + 从 Table 移除 + Info 回收（`Info.Recycle2Cache` + 置 null）。但栈里压的 PanelInfo 可能正是那个被回收的对象（已被 OnRecycled 清空字段并可能被复用）——Pop 时用到的是脏/复用的 Info，会重建出错误面板或 NRE。这是"栈持有已回收对象"的危险。
5. 不是同一对象。Push 时面板被 Close（默认 Destroy）、从 Table 移除；Pop 用 PanelInfo 重新 LoadPanel 创建**新实例**。意味着面板内部的运行时状态（动画进度、未持久化的临时字段）全部丢失，只有 `PanelInfo` 里记录的（含 UIData）能还原。
6. Pop 时用 `PanelInfo.UIData` 重建面板。若 Close 没回写，Info.UIData 可能是旧值或 null，重建的面板拿不到关闭前的最新数据状态，导致"返回后数据不一致"。
7. 关掉的是 `LastOrDefault`——最后打开的第 3 个。剩 2 个。它们的 GameObject 名默认都是 `typeof(T).Name`（CreateUI 里 `name = GameObjName ?? PanelType.Name`），**会重名**——这正是为什么 `GameObjectNameIndex` 的每个 key 对应 `List<IPanel>` 而非单个，以及为什么多开场景要靠 Type/Panel 引用而非名字精确定位。
</details>

## ✍️ 实操题（3）

1. 实现一个 `UISettingsPanel`，重写 `OnInit/OnOpen/OnClose`，用 `IUIData` 传入初始音量值，关闭时打印。说明 `OnInit` 与 `OnOpen` 调用时机的区别。
2. 用 `UIKit.Stack` 实现"主菜单→设置→返回主菜单"的导航，解释为什么返回后主菜单是新实例，以及如何用 `IUIData` 保留主菜单的滚动位置。
3. 解释若要把 UIKit 的面板加载对接到 ResKit（异步加载 AssetBundle），应该在哪个注入点接入，以及为什么 `OpenPanelAsync` 返回 `IEnumerator` 而非直接返回面板。

<details><summary>参考答案要点</summary>

1. `class SettingsData:IUIData{public float Volume;}`；`OnInit(d)` 在 CreateUI 时调用一次（面板首次创建/初始化，绑定组件、读初值）；`OnOpen(d)` 每次打开都调用（Single 复用时 Init 不再调但 Open 会调）。所以"只需一次的初始化"放 OnInit，"每次打开都要刷新的"放 OnOpen。
2. `UIKit.OpenPanel<MainMenu>(); UIKit.Stack.Push(mainMenu); UIKit.OpenPanel<SettingsPanel>();` 返回时 `UIKit.Back(settingsPanel)`（关设置 + Stack.Pop 重开主菜单）。新实例原因：Push 时主菜单被 Close/Destroy，Pop 用 PanelInfo 重建。保留滚动位置：在主菜单 Close 前把滚动值写入其 `IUIData`，该 UIData 随 Info 入栈，Pop 重建时经 OnOpen(uiData) 还原滚动位置。
3. 接入点：`UIKit.Config.LoadPanelAsync`（或自定义 `IPanelLoader`），在其中调用 ResKit 异步加载 AB + 实例化面板后回调。`OpenPanelAsync` 返回 `IEnumerator` 是因为加载是异步的（跨帧），调用方需 `yield return` 等待加载完成；不能同步返回面板因为面板此刻还没加载好。可 `.ToAction().Start(this)` 接入 ActionKit 驱动。
</details>
