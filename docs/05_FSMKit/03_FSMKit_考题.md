# 05 · FSMKit 考题

## 🟢 概念题（5）

1. `IState` 定义了哪些生命周期方法？
2. 定义状态的两种风格分别是什么？各自适用场景？
3. `FSM<T>` 用什么数据结构存储状态？
4. `FrameCountOfCurrentState` 和 `SecondsOfCurrentState` 的作用是什么？
5. `OnStateChanged` 回调的两个参数分别是什么？

<details><summary>参考答案要点</summary>

1. `Condition()`、`Enter()`、`Update()`、`FixedUpdate()`、`OnGUI()`、`Exit()`。
2. 链式委托（`FSM.State(id).OnEnter(...)`，轻量、适合简单状态）；类式继承（`AbstractState`，可携带 `mOwner` 上下文、可复用、适合复杂状态）。
3. `Dictionary<T, IState> mStates`（T 通常是枚举）。
4. 记录当前状态已停留的帧数/秒数，每帧 Update 累加、切换时归零，用于"停留 N 秒/帧后转移"等逻辑。
5. `(PreviousStateId, CurrentStateId)`，即迁移前后的状态 id。
</details>

## 🟡 机制题（6）

1. `ChangeState` 的完整原子序列是什么？
2. `StartState` 与 `ChangeState` 有哪些关键差异？
3. `ChangeState` 中 `Condition()` 检查的是当前状态还是目标状态？
4. `State(id)` 的"获取或创建"行为是怎样的？
5. FSM 的状态对象是否使用对象池？它们何时被释放？
6. 为什么 `ChangeState` 把帧计数置 1，而 `StartState` 置 0？

<details><summary>参考答案要点</summary>

1. 检查 `t!=CurrentStateId` 且 `mStates` 含 t 且 `mCurrentState!=null` 且 `目标.Condition()` → `Exit()`旧 → 记 `PreviousStateId` → 换 `mCurrentState/Id` → `OnStateChanged(prev,next)` → 帧计数=1/秒数=0 → `Enter()`新。
2. StartState：无条件、不调 Exit、不检查 Condition、不触发 OnStateChanged、帧计数=0；ChangeState：检查 Condition、调 Exit、触发 OnStateChanged、帧计数=1，且同 id 直接 return。
3. 目标状态。`s.Condition()`，s 是目标 state。
4. 若 id 已存在返回 `mStates[id] as CustomState`；否则 `new CustomState()` 入字典并返回。
5. 不使用对象池。状态对象创建后常驻 `mStates`，直到 `Clear()` 清空字典释放引用。
6. StartState 是"第 0 帧进入"，当帧尚未经历 Update；ChangeState 后通常紧接着会有 Update，从 1 开始计更符合"已进入第 1 帧"的语义（细节约定，源码如此）。
</details>

## 🔴 架构陷阱题（7）

1. **同底座对比**：FSMKit 不自带驱动（要外部调 `FSM.Update()`），而 ActionKit 自带全局 Mono 驱动。这两种"驱动权归属"各有什么利弊？
2. 用 `AddState(S.A, new MyAbstractState(...))` 注册后，又调用 `FSM.State(S.A).OnUpdate(...)`。会发生什么？
3. 把 `OnCondition` 的条件方向写反了——`State(A).OnCondition(()=>CurrentStateId==A)`。切换会出什么问题？
4. 忘记在 `OnDestroy` 调 `FSM.Clear()`，且某 `AbstractState` 的构造捕获了 MonoBehaviour owner。后果？
5. 在状态的 `OnUpdate` 里调用 `FSM.ChangeState(B)`，B 的 `OnEnter` 又立即 `ChangeState(C)`。会有什么风险？
6. 连续 `ChangeState(同一个当前状态 id)`，会重新 Enter 吗？为什么这样设计？
7. 漏调 `FSM.Update()`，但状态切换仍正常。哪些功能会失效？

<details><summary>参考答案要点</summary>

1. 外部驱动（FSM）：灵活，可挂到任意 Update/FixedUpdate/手动 tick，便于暂停/变速/嵌套；但漏调就静默不工作。全局驱动（ActionKit）：开箱即用、调用方零负担；但驱动时机固定（全局 Update），不易精细控制。
2. `State(S.A)` 返回 `mStates[A] as CustomState`，而 A 是 `MyAbstractState`（非 CustomState），`as` 得 null，`.OnUpdate(...)` 抛 NullReferenceException。
3. `OnCondition(()=>CurrentStateId==A)` 意为"只有当前已在 A 时才允许进入 A"。但 `ChangeState(A)` 在 `id==CurrentStateId` 时已直接 return，所以从其他状态永远进不了 A（Condition 恒 false），状态机卡死无法切到 A。
4. FSM 字段（连同 `mStates` 里的 AbstractState、其捕获的 owner）随持有者存活无法 GC；若 owner 是已销毁的 GameObject，状态逻辑访问 owner 会触发 Unity 的"已销毁对象"假 null 异常。
5. 重入切换：A.Update→ChangeState(B)→B.Enter→ChangeState(C)。本身能工作（递归同步执行），但会在一次 Update 内连跳多个状态，可能跳过 B 的 Update，且若条件成环会栈溢出。需谨慎设计 Enter 内切换。
6. 不会。`ChangeState` 开头 `if(t.Equals(CurrentStateId)) return`。设计目的：防止"切到自己"导致无谓的 Exit→Enter 重置（清空帧计数、重跑进入逻辑）。
7. `mCurrentState.Update()` 不执行（状态的 OnUpdate 逻辑全失效）、`FrameCount/Seconds` 不累加（基于计时的转移失效）。但 `ChangeState`（含 Exit/Enter/广播）是独立调用，仍正常。
</details>

## ✍️ 实操题（3）

1. 用链式风格实现一个敌人 AI 的 `Idle/Chase/Attack` 三态机：Idle 满足"发现玩家"切 Chase，Chase 满足"进入攻击范围"切 Attack。写出 `State().OnCondition().OnEnter().OnUpdate()` 骨架，并说明谁负责调 `Update`。
2. 用类式 `AbstractState<States, EnemyController>` 实现 `ChaseState`，通过 `mOwner` 访问敌人的移动方法。说明相比链式风格它多了什么能力。
3. 实现"在某状态停留 3 秒后自动转移"，利用 `SecondsOfCurrentState`，并指出若忘记调 `FSM.Update()` 会怎样。

<details><summary>参考答案要点</summary>

1. `fsm.State(Idle).OnCondition(()=>fsm.CurrentStateId==X).OnUpdate(()=>{ if(发现玩家) fsm.ChangeState(Chase); })` 同理 Chase→Attack。由持有 fsm 的 MonoBehaviour 在自身 `Update()` 里调 `fsm.Update()` 驱动。
2. `class ChaseState:AbstractState<States,EnemyController>{ public ChaseState(fsm,owner):base(...){} protected override void OnUpdate(){ mOwner.MoveTowardPlayer(); } }`。比链式多了：可携带强类型 owner 上下文、可被复用/继承、可拆分到独立文件、可有自己的字段维护状态内数据。
3. 在 `OnUpdate` 里 `if(fsm.SecondsOfCurrentState>=3f) fsm.ChangeState(Next);`。忘记 `FSM.Update()`：`SecondsOfCurrentState` 永不累加，恒为 0，自动转移永不触发。
</details>
