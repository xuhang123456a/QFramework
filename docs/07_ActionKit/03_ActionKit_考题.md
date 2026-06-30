# 07 · ActionKit 考题

## 🟢 概念题（5）

1. ActionKit 综合了哪三种设计模式？分别对应什么结构？
2. `ActionStatus` 的三个状态是什么？
3. 一个动作的核心三个钩子方法是什么？
4. `IActionController` 的作用是什么？
5. `Sequence` 与 `Parallel` 在结构上有什么共性？

<details><summary>参考答案要点</summary>

1. 命令模式（`IAction` 是命令对象）、组合模式（Sequence/Parallel 持子动作列表且自身也是 IAction，可嵌套）、建造者模式（链式 `.Delay().Callback()` 拼装）。
2. `NotStart`、`Started`、`Finished`。
3. `OnStart()`、`OnExecute(float dt)`、`OnFinish()`。
4. 动作的"外部遥控器/句柄"：持 `ActionID` 做版本校验，可 Pause/Resume/Reset/Deinit，外部通过它操作动作而不直接碰 action 对象。
5. 都是组合节点：持有 `List<IAction>`、自身实现 `IAction`，因此可以互相嵌套（Sequence 里放 Parallel，反之亦然）。
</details>

## 🟡 机制题（6）

1. `IActionExtensions.Execute(dt)` 在 `NotStart` 状态下做了什么？为什么 `OnStart` 后立即检查 `Finished`？
2. `Sequence.OnStart` 里的 `TryExecuteUntilNextNotFinished` 解决了什么问题？
3. 每个动作 `Allocate` 时为什么要 `ActionID = ID_GENERATOR++`？
4. `Deinit` 为什么不直接 `pool.Recycle`，而是 `ActionQueue.AddCallback`？
5. `MonoUpdateActionExecutor` 用了哪三个集合？分别承担什么？
6. `Sequence.Deinit` 与单个 `Delay.Deinit` 的差异是什么？

<details><summary>参考答案要点</summary>

1. NotStart 时调 `OnStart()`，然后立即查 `Status==Finished`：若 OnStart 内就 `this.Finish()`（瞬时完成动作），直接 `OnFinish` 返回 true，不浪费一帧进入 Started。否则置 Started。
2. 让一连串"瞬时完成"的子动作（如 Callback、已满足的 Condition）在一次推进里用 `Execute(0)` 连续推完，直到遇到第一个未完成的动作，避免每个瞬时动作各占一帧。
3. 因为对象池复用：同一个 IAction 实例会被回收后重新 Allocate 成"另一个动作"。全局自增 ID 给每次分配唯一标识，配合 `ActionController.ActionID` 校验，防止对已复用对象误操作。
4. 避免"迭代中销毁"：动作可能正在被 Executor 遍历或被 Sequence 级联 Deinit，立即 Recycle 会让对象在遍历途中被归还池中复用，破坏正在进行的迭代。延迟到 `ActionQueue.Update` 末尾统一回收，迭代安全。
5. `mPrepareExecutionActions`（本帧新加入、下帧合并）、`mExecutingActions`（执行中字典）、`mToActionRemove`（完成待移除，遍历后统一删 + Recycle controller）。
6. Sequence.Deinit 会先 `foreach action.Deinit()` 级联回收所有子动作 + `mActions.Clear()`，再把自己入回收队列；Delay.Deinit 只清自己的回调引用 + 入回收队列。组合节点负责回收其持有的子节点。
</details>

## 🔴 架构陷阱题（8）

1. **同底座对比**：ActionKit 和 BindableList 都面临"迭代中修改集合"的问题。ActionKit 用 ActionQueue 延迟回收 + Executor 三集合，BindableList 会在 foreach 修改时抛异常。这两种应对策略反映了什么不同的设计取舍？
2. 你保存了一个 `IActionController`，等动作完成（已被回收复用）很久后才调 `controller.Pause()`。会发生什么？框架靠什么避免灾难？
3. ActionKit 内部用 `SimpleObjectPool` 而非 `SafeObjectPool`，为什么这是安全的？（结合 PoolKit 的 IsRecycled 不变量）
4. 某自定义动作的 `OnExecute` 永远不调 `this.Finish()`。会怎样？它占用的对象会被回收吗？
5. 在 `Sequence` 执行到第 3 个子动作时，外部对整个 Sequence 的 controller 调 `Deinit`。第 4、5 个还没执行的子动作会怎样？
6. `action.Start(this)` 中的 `this`（一个 MonoBehaviour）在动作执行途中被 `Destroy`。动作还会继续吗？换成 `StartGlobal()` 呢？
7. 把 `Deinited` 标志去掉，对同一动作连续 `Deinit` 两次。会发生什么？
8. `ActionController` 的 `Reset()` 内为什么也要 `if(Action.ActionID == ActionID)` 校验？

<details><summary>参考答案要点</summary>

1. ActionKit 主动设计了"延迟到安全点统一处理"的机制（因为动作增删极频繁且涉及池复用，必须支持运行中安全增删）；BindableList 直接复用 BCL 的"修改即抛异常"快速失败策略（集合修改属于使用者错误，应暴露而非吞掉）。前者是"框架内部高频场景的工程化容错"，后者是"对外 API 的防御性快速失败"。
2. 若不校验，会误暂停那个被复用成新动作的对象。框架靠 `controller.ActionID` 与 `Action.ActionID` 比对：复用后 action 的 ActionID 已变（新的自增值），不等于 controller 记录的旧 ID，`Pause` 内的 `if` 不通过，操作被安全忽略。
3. 因为 ActionKit 内部严格控制每个动作的 Deinit 流程（`Deinited` 标志防重复 + ActionQueue 延迟回收防迭代期复用），调用方不直接碰 Recycle，重复回收风险被消除，于是用零开销的 Simple 池即可（无需 SafeObjectPool 的 IsRecycled 防护）。
4. 动作永远停在 Started 状态，每帧 `OnExecute` 被调用但永不完成（"永动"）。它不会被回收（Deinit 不会被触发），且持续占用 Executor 的执行槽——逻辑泄漏。需外部 controller 主动结束或动作内部设超时。
5. `Sequence.Deinit` 级联 `foreach action.Deinit()`，包括尚未执行的第 4、5 个子动作也会被 Deinit（入回收队列）+ `mActions.Clear()`。整个序列被整体回收，未执行的动作不会再执行。
6. `Start(this)` 把动作挂在 this 的 `MonoUpdateActionExecutor` 组件上，this 被 Destroy 后该组件随之销毁，Update 停止，动作不再被驱动（静默停止）。`StartGlobal()` 挂在全局单例 `ActionKitMonoBehaviourEvents` 上，不随业务对象销毁，动作会继续执行至完成。
7. 第二次 Deinit 会再次把回收回调入 ActionQueue，导致同一对象被 `pool.Recycle` 两次——栈里出现两份相同引用，后续两次 Allocate 拿到同一对象的别名，状态共享错乱（与 PoolKit 重复回收陷阱同源）。`Deinited` 标志正是防这个。
8. 因为 controller 可能持有的是已被复用的 action。Reset 一个版本不匹配的 action 会重置一个无关的新动作的状态。校验确保只 Reset 自己当初启动的那个动作。
</details>

## ✍️ 实操题（3）

1. 用 `ActionKit.Sequence()` 实现："淡入(1s) → 等待点击 → 播放音效 → 淡出(1s)"，并在完成后打印日志。指出哪些子动作是"瞬时完成"的。
2. 用 `ActionKit.Custom` 实现一个"每帧累加，满 5 次后完成"的自定义动作，并解释 `a.Finish()` 在其中的作用。
3. 解释为什么把一个 `Delay` 动作 `Start` 两次（同一 action 实例）是危险的，应该如何正确地"重复执行"一个延时逻辑。

<details><summary>参考答案要点</summary>

1. `ActionKit.Sequence().Callback(FadeIn 起步).Delay(1).Condition(()=>clicked).Callback(PlaySfx).Delay(1).Start(this, ()=>Debug.Log("done"))`（Lerp/Fade 用对应 API）。瞬时完成的：`Callback`（OnStart 内即 Finish）、`Condition` 在条件已满足那一帧即完成。
2. `ActionKit.Custom(a=>{ a.OnStart(()=>count=0).OnExecute(dt=>{ count++; if(count>=5) a.Finish(); }).OnFinish(()=>Debug.Log("done")); }).Start(this)`。`a.Finish()` 把 Status 置 Finished，下次推进时触发 OnFinish 并让执行器回收该动作——它是动作"告知自己已完成"的唯一方式。
3. 同一 action 实例被 Start 两次会让两个 controller 指向同一对象，且该对象可能已完成并准备回收，ActionID/状态会冲突。正确做法：每次需要执行时重新 `ActionKit.Delay(...)` 分配一个新动作（从池复用、新 ActionID），或用 `ActionKit.Repeat(n)` 组合来表达重复语义。
</details>
