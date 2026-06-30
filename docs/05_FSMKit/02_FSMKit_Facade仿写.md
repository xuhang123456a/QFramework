# 05 · FSMKit Facade 仿写

> 约 120 行复刻：`IState` + `CustomState`（链式建造）+ `FSM<T>`（条件切换 + 帧/秒计量 + OnStateChanged）+ `AbstractState`（类式 + owner 注入）。几乎是原模块全貌（FSMKit 本就很小），仅砍掉 `OnGUI`、`FixedUpdate` 等可选生命周期与 Unity `Time` 依赖（用注入 dt 替代）。

## 设计映射表

| 原实现 | 精简版 | 取舍 |
|---|---|---|
| `IState` 六方法 | 精简为 `Condition/Enter/Update/Exit` | 简化（去 FixedUpdate/OnGUI） |
| `CustomState` 链式 | 保留 | ✅ 核心 |
| `FSM<T>` 字典 + 条件切换 | 保留 | ✅ 核心 |
| `FrameCount/Seconds` 计量 | 保留 Seconds（dt 注入） | ✅ 体现计量不变量 |
| `StartState` vs `ChangeState` 差异 | 保留 | ✅ 关键差异 |
| `AbstractState<TStateId,TOwner>` | 保留 | ✅ owner 注入母题 |
| `Time.deltaTime` 隐式驱动 | 改为 `Update(float dt)` 显式注入 | 简化平台依赖 |

## 最小复刻代码

```csharp
using System;
using System.Collections.Generic;

namespace MiniFSM
{
    public interface IState
    {
        bool Condition();
        void Enter();
        void Update();
        void Exit();
    }

    // ---------- 链式委托状态 ----------
    public class CustomState : IState
    {
        private Func<bool> mCond;
        private Action mEnter, mUpdate, mExit;
        public CustomState OnCondition(Func<bool> f) { mCond = f; return this; }
        public CustomState OnEnter(Action a) { mEnter = a; return this; }
        public CustomState OnUpdate(Action a) { mUpdate = a; return this; }
        public CustomState OnExit(Action a) { mExit = a; return this; }

        public bool Condition() { var r = mCond?.Invoke(); return r == null || r.Value; } // 默认 true
        public void Enter() => mEnter?.Invoke();
        public void Update() => mUpdate?.Invoke();
        public void Exit() => mExit?.Invoke();
    }

    // ---------- 状态机 ----------
    public class FSM<T>
    {
        private readonly Dictionary<T, IState> mStates = new();
        private IState mCurrentState;
        private T mCurrentStateId;

        public IState CurrentState => mCurrentState;
        public T CurrentStateId => mCurrentStateId;
        public T PreviousStateId { get; private set; }
        public long FrameCountOfCurrentState = 1;
        public float SecondsOfCurrentState = 0f;

        private Action<T, T> mOnStateChanged = (_, __) => { };
        public void OnStateChanged(Action<T, T> cb) => mOnStateChanged += cb;

        public void AddState(T id, IState state) => mStates.Add(id, state);   // 重复 id 抛异常

        public CustomState State(T id)
        {
            if (mStates.TryGetValue(id, out var s)) return s as CustomState;   // ⚠ 若是 AbstractState 得 null
            var cs = new CustomState();
            mStates.Add(id, cs);
            return cs;
        }

        public void StartState(T id)                                          // 无条件首次进入
        {
            if (!mStates.TryGetValue(id, out var s)) return;
            PreviousStateId = id;
            mCurrentState = s;
            mCurrentStateId = id;
            FrameCountOfCurrentState = 0;
            SecondsOfCurrentState = 0f;
            s.Enter();
        }

        public void ChangeState(T id)                                         // 条件切换
        {
            if (id.Equals(mCurrentStateId)) return;                           // 不重入同态
            if (!mStates.TryGetValue(id, out var s)) return;
            if (mCurrentState == null || !s.Condition()) return;              // 目标状态守门
            mCurrentState.Exit();                                            // 先退出旧
            PreviousStateId = mCurrentStateId;
            mCurrentState = s;
            mCurrentStateId = id;
            mOnStateChanged(PreviousStateId, mCurrentStateId);               // 广播迁移
            FrameCountOfCurrentState = 1;
            SecondsOfCurrentState = 0f;
            mCurrentState.Enter();                                           // 后进入新
        }

        public void Update(float dt)
        {
            mCurrentState?.Update();
            FrameCountOfCurrentState++;
            SecondsOfCurrentState += dt;
        }

        public void Clear()
        {
            mCurrentState = null;
            mCurrentStateId = default;
            mStates.Clear();
        }
    }

    // ---------- 类式状态 + owner 注入 ----------
    public abstract class AbstractState<TStateId, TOwner> : IState
    {
        protected FSM<TStateId> mFSM;
        protected TOwner mOwner;
        protected AbstractState(FSM<TStateId> fsm, TOwner owner) { mFSM = fsm; mOwner = owner; }

        bool IState.Condition() => OnCondition();
        void IState.Enter() => OnEnter();
        void IState.Update() => OnUpdate();
        void IState.Exit() => OnExit();

        protected virtual bool OnCondition() => true;
        protected virtual void OnEnter() { }
        protected virtual void OnUpdate() { }
        protected virtual void OnExit() { }
    }
}
```

## 使用示例

```csharp
using MiniFSM;
using System;

class Demo
{
    enum S { Idle, Run }

    static void Main()
    {
        var fsm = new FSM<S>();
        fsm.OnStateChanged((prev, next) => Console.WriteLine($"{prev} => {next}"));

        fsm.State(S.Idle)
           .OnCondition(() => fsm.CurrentStateId == S.Run)   // 只有从 Run 才能回 Idle
           .OnEnter(() => Console.WriteLine("Enter Idle"))
           .OnExit(() => Console.WriteLine("Exit Idle"));

        fsm.State(S.Run)
           .OnCondition(() => fsm.CurrentStateId == S.Idle)
           .OnEnter(() => Console.WriteLine("Enter Run"));

        fsm.StartState(S.Idle);     // Enter Idle（无条件，无广播）
        fsm.ChangeState(S.Run);     // Exit Idle / Idle => Run / Enter Run
        fsm.ChangeState(S.Run);     // 同态，直接 return（无输出）
        fsm.Update(0.016f);         // 驱动当前态 Run.Update + 计量
        fsm.Clear();
    }
}
```

## 取舍自检

- ✅ **保留的核心不变量**：目标状态 `Condition()` 守门；`ChangeState` 原子序列 Exit旧→换→广播→Enter新；不重入同状态；`StartState`（无条件/帧计数 0）与 `ChangeState`（条件/帧计数 1）的差异；帧/秒计量切换归零；`State()` 获取或创建 + `as CustomState` 语义；`AbstractState` 通过 `mOwner` 注入上下文。
- ❌ **砍掉的**：`FixedUpdate`/`OnGUI` 生命周期、`Time.deltaTime` 隐式依赖（改注入 dt）、API 文档特性。
- ⚠️ **最容易搞错的一处**：`State(id)` 对已用 `AddState` 注册成 `AbstractState` 的 id 调用，`as CustomState` 返回 **null**，链式 `.OnEnter()` 立刻 NRE。两种风格混用时，对同一 id 只能用一种注册方式。次坑：`ChangeState` 的 `Condition` 检查的是**目标**状态，不是当前状态——写守门条件时方向极易写反。
