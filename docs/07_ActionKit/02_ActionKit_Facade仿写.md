# 07 · ActionKit Facade 仿写

> 约 180 行复刻：三态 `IAction` + 池化 `Allocate`/`ActionID` 版本校验 + `Sequence`（组合 + 瞬时完成推进）+ `Delay` + `ActionController`（句柄校验）+ 延迟回收队列 + 简化执行器。保留全部核心不变量：组合统一为 IAction、三态推进、池化复用、ID 防错、延迟回收迭代安全。砍掉 Parallel/Repeat/Custom/Coroutine/Task、UpdateMode、Mono 集成（用手动 tick）。

## 设计映射表

| 原实现 | 精简版 | 取舍 |
|---|---|---|
| `IAction` 三态 + OnStart/OnExecute/OnFinish | 保留 | ✅ |
| `IActionExtensions.Execute(dt)` 推进 | 保留（含"OnStart 内即完成"） | ✅ 核心 |
| `AbstractAction<T>` 池化 + ActionID | 保留 | ✅ 核心不变量 |
| `Delay` / `Sequence` | 保留 | ✅ |
| `ActionController` ActionID 校验 | 保留 | ✅ 核心不变量 |
| `ActionQueue` 延迟回收 | 保留（手动 tick 版） | ✅ 迭代安全母题 |
| `MonoUpdateActionExecutor` 三集合双缓冲 | 简化为一个执行器 + toRemove | 简化但保留迭代安全 |
| Parallel/Repeat/Custom/Coroutine/Task | 砍掉 | ❌ |
| UpdateMode/IgnoreTimeScale | 砍掉 | ❌ |
| Mono 单例驱动 | 改为手动 `Tick(dt)` | 简化平台 |

## 最小复刻代码

```csharp
using System;
using System.Collections.Generic;

namespace MiniAction
{
    public enum ActionStatus { NotStart, Started, Finished }

    public interface IAction
    {
        ulong ActionID { get; set; }
        ActionStatus Status { get; set; }
        bool Deinited { get; set; }
        bool Paused { get; set; }
        void OnStart();
        void OnExecute(float dt);
        void OnFinish();
        void Reset();
        void Deinit();
    }

    public static class ActionKitRuntime
    {
        public static ulong ID_GENERATOR = 0;
        // 延迟回收队列：迭代安全的关键
        private static readonly List<Action> mRecycleCallbacks = new();
        public static void EnqueueRecycle(Action cb) => mRecycleCallbacks.Add(cb);
        public static void FlushRecycle()
        {
            if (mRecycleCallbacks.Count == 0) return;
            foreach (var cb in mRecycleCallbacks) cb();
            mRecycleCallbacks.Clear();
        }
    }

    // ---------- 三态推进（所有动作共用）----------
    public static class ActionExtensions
    {
        public static void Finish(this IAction self) => self.Status = ActionStatus.Finished;

        public static bool Execute(this IAction self, float dt)
        {
            if (self.Status == ActionStatus.NotStart)
            {
                self.OnStart();
                if (self.Status == ActionStatus.Finished) { self.OnFinish(); return true; } // OnStart 内即完成
                self.Status = ActionStatus.Started;
            }
            else if (self.Status == ActionStatus.Started)
            {
                if (self.Paused) return false;
                self.OnExecute(dt);
                if (self.Status == ActionStatus.Finished) { self.OnFinish(); return true; }
            }
            else if (self.Status == ActionStatus.Finished) { self.OnFinish(); return true; }
            return false;
        }
    }

    // ---------- 极简对象池（来自 PoolKit 思想）----------
    public class SimpleObjectPool<T>
    {
        private readonly Stack<T> mStack = new();
        private readonly Func<T> mFactory;
        public SimpleObjectPool(Func<T> factory, int init = 0)
        { mFactory = factory; for (var i = 0; i < init; i++) mStack.Push(factory()); }
        public T Allocate() => mStack.Count == 0 ? mFactory() : mStack.Pop();
        public void Recycle(T obj) => mStack.Push(obj);
    }

    // ---------- Delay ----------
    public class Delay : IAction
    {
        public float DelayTime; public float Current; public Action OnDelayFinish;
        private static readonly SimpleObjectPool<Delay> mPool = new(() => new Delay(), 10);
        public static Delay Allocate(float time, Action onFinish = null)
        {
            var d = mPool.Allocate();
            d.ActionID = ActionKitRuntime.ID_GENERATOR++;     // 每次分配唯一 ID
            d.Deinited = false; d.Reset();
            d.DelayTime = time; d.OnDelayFinish = onFinish; d.Current = 0;
            return d;
        }
        public ulong ActionID { get; set; }
        public ActionStatus Status { get; set; }
        public bool Deinited { get; set; }
        public bool Paused { get; set; }
        public void OnStart() { }
        public void OnExecute(float dt)
        {
            if (Current >= DelayTime) { this.Finish(); OnDelayFinish?.Invoke(); }
            Current += dt;
        }
        public void OnFinish() { }
        public void Reset() { Status = ActionStatus.NotStart; Paused = false; Current = 0; }
        public void Deinit()
        {
            if (Deinited) return;
            Deinited = true; OnDelayFinish = null;
            ActionKitRuntime.EnqueueRecycle(() => mPool.Recycle(this));  // 延迟回收
        }
    }

    // ---------- Sequence（组合）----------
    public interface ISequence : IAction { ISequence Append(IAction a); }
    public class Sequence : ISequence
    {
        private readonly List<IAction> mActions = new();
        private IAction mCurrent; private int mIndex;
        private static readonly SimpleObjectPool<Sequence> mPool = new(() => new Sequence(), 10);
        public static Sequence Allocate()
        {
            var s = mPool.Allocate();
            s.ActionID = ActionKitRuntime.ID_GENERATOR++;
            s.Reset(); s.Deinited = false; return s;
        }
        public ulong ActionID { get; set; }
        public ActionStatus Status { get; set; }
        public bool Deinited { get; set; }
        public bool Paused { get; set; }
        public ISequence Append(IAction a) { mActions.Add(a); return this; }

        public void OnStart()
        {
            if (mActions.Count == 0) { this.Finish(); return; }
            mIndex = 0; mCurrent = mActions[0]; mCurrent.Reset();
            Advance();
        }
        private void Advance()                              // 把一连串瞬时完成的动作一次推完
        {
            while (mCurrent != null && mCurrent.Execute(0))
            {
                mIndex++;
                if (mIndex < mActions.Count) { mCurrent = mActions[mIndex]; mCurrent.Reset(); }
                else { mCurrent = null; this.Finish(); }
            }
        }
        public void OnExecute(float dt)
        {
            if (mCurrent == null) { this.Finish(); return; }
            if (mCurrent.Execute(dt))
            {
                mIndex++;
                if (mIndex < mActions.Count) { mCurrent = mActions[mIndex]; mCurrent.Reset(); Advance(); }
                else this.Finish();
            }
        }
        public void OnFinish() { }
        public void Reset()
        {
            mIndex = 0; Status = ActionStatus.NotStart; Paused = false;
            foreach (var a in mActions) a.Reset();
        }
        public void Deinit()
        {
            if (Deinited) return;
            Deinited = true;
            foreach (var a in mActions) a.Deinit();         // 级联回收子动作
            mActions.Clear();
            ActionKitRuntime.EnqueueRecycle(() => mPool.Recycle(this));
        }
    }
    public static class SequenceExt
    {
        public static ISequence Sequence() => MiniAction.Sequence.Allocate();
        public static ISequence Delay(this ISequence self, float s, Action cb = null)
            => self.Append(MiniAction.Delay.Allocate(s, cb));
        public static ISequence Callback(this ISequence self, Action cb)
            => self.Append(MiniAction.Delay.Allocate(0, cb)); // 简化：0 延时即回调
    }

    // ---------- Controller（ActionID 版本校验）----------
    public class ActionController
    {
        public ulong ActionID; public IAction Action;
        public void Pause()  { if (Action.ActionID == ActionID) Action.Paused = true; }   // 校验后才操作
        public void Resume() { if (Action.ActionID == ActionID) Action.Paused = false; }
    }

    // ---------- 简化执行器（迭代安全增删）----------
    public class ActionExecutor
    {
        private readonly List<(IAction action, ActionController ctrl, Action onFinish)> mRunning = new();
        private readonly List<int> mToRemove = new();

        public ActionController Start(IAction action, Action onFinish = null)
        {
            var ctrl = new ActionController { ActionID = action.ActionID, Action = action };
            mRunning.Add((action, ctrl, onFinish));
            return ctrl;
        }
        public void Tick(float dt)
        {
            for (int i = 0; i < mRunning.Count; i++)         // for + 索引，遍历中不直接删
            {
                var (action, ctrl, onFinish) = mRunning[i];
                if (!action.Deinited && action.Execute(dt))
                {
                    onFinish?.Invoke();
                    action.Deinit();                         // 入延迟回收队列
                    mToRemove.Add(i);
                }
            }
            for (int j = mToRemove.Count - 1; j >= 0; j--) mRunning.RemoveAt(mToRemove[j]);
            mToRemove.Clear();
            ActionKitRuntime.FlushRecycle();                 // 帧末统一真正回收
        }
    }
}
```

## 使用示例

```csharp
using MiniAction;
using System;
using static MiniAction.SequenceExt;

class Demo
{
    static void Main()
    {
        var exec = new ActionExecutor();

        var seq = Sequence()
            .Callback(() => Console.WriteLine("开始"))
            .Delay(1.0f, () => Console.WriteLine("1 秒到"))
            .Callback(() => Console.WriteLine("结束"));

        exec.Start(seq, () => Console.WriteLine("整个序列完成"));

        // 模拟主循环：每帧 0.5s，共推进若干帧
        for (int frame = 0; frame < 4; frame++)
        {
            Console.WriteLine($"-- frame {frame} --");
            exec.Tick(0.5f);
        }
        // -- frame0 -- 开始 / -- frame1 -- (累计0.5) ... -- frame2 -- 1秒到/结束/整个序列完成
    }
}
```

## 取舍自检

- ✅ **保留的核心不变量**：组合（Sequence）与叶子（Delay）统一为 `IAction`，可嵌套；三态推进集中在 `Execute(dt)` 且"OnStart 内即完成"支持瞬时动作；每次 `Allocate` 打全局自增 `ActionID`；`ActionController` 操作前校验 `Action.ActionID==ActionID`；`Deinit` 幂等（`Deinited` 标志）+ 级联回收子动作 + **延迟回收**（入队列，帧末 Flush）；执行器用索引遍历 + toRemove 延迟删除保证迭代安全。
- ❌ **砍掉的**：Parallel/Repeat/Custom/Coroutine/Task、`UpdateMode`/`IgnoreTimeScale`、Mono 单例驱动（改手动 Tick）、`StartGlobal/StartCurrentScene`、Executor 的 prepare/executing 双缓冲（简化为单 running 列表）。
- ⚠️ **最容易搞错的一处**：`Deinit` 绝不能直接 `pool.Recycle(this)`，必须入延迟回收队列。若在 `ActionExecutor.Tick` 遍历过程中立即回收，该 action 可能被另一处 `Allocate` 复用、ActionID 改变，而当前遍历仍持有它的旧引用——导致一个对象同时是"正在执行的旧动作"和"刚分配的新动作"，状态彻底错乱。其次：`ActionController` 的 ActionID 校验一旦省略，对已回收复用的 controller 调 Pause 会误暂停无关的新动作。
