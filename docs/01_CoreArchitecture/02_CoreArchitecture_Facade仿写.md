# 01 · CoreArchitecture Facade 仿写

> 目标：用约 160 行独立可编译的 C# 复刻"架构母体"的核心不变量：CRTP 单例架构 + IOC 容器 + 能力接口 + EasyEvent + BindableProperty。砍掉 Query、OrEvent、Mono 生命周期绑定、多参事件等。

## 设计映射表

| 原实现 | 精简版 | 取舍 |
|---|---|---|
| `Architecture<T>` CRTP 单例 | 保留 | ✅ 核心不变量 |
| `IOCContainer`（Type→object） | 保留 | ✅ |
| Model→System 双批次初始化 + `mInited` 双路径 | 保留 | ✅ 核心不变量 |
| `ICanGetModel` 等 8 个能力接口 + 扩展 | 精简为 3 个（GetModel/SendEvent/SendCommand） | 简化 |
| `ICommand` / `ICommand<TResult>` | 仅保留无返回值 `ICommand` | 简化 |
| `IQuery<T>` | 砍掉 | ❌ Query 与 Command 同构，省略 |
| `TypeEventSystem` + `EasyEvents` 字典 | 保留（事件按类型 key） | ✅ |
| `EasyEvent` / `EasyEvent<T,K,S>` | 仅 `EasyEvent<T>` | 简化 |
| `IUnRegister` + `CustomUnRegister` | 保留 | ✅ 注销不变量 |
| `UnRegisterTrigger` (Mono 绑定) | 砍掉 | ❌ 平台细节 |
| `BindableProperty<T>` + `Comparer` | 保留 | ✅ 可观测快照不变量 |
| `OnRegisterPatch` | 砍掉 | ❌ 测试注入点 |

## 最小复刻代码

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace MiniQF
{
    // ---------- 能力接口（标记接口）+ 扩展方法 ----------
    public interface IBelongToArchitecture { IArchitecture GetArchitecture(); }
    public interface ICanSetArchitecture { void SetArchitecture(IArchitecture a); }
    public interface ICanGetModel : IBelongToArchitecture { }
    public interface ICanSendEvent : IBelongToArchitecture { }
    public interface ICanSendCommand : IBelongToArchitecture { }

    public static class CanExtensions
    {
        public static T GetModel<T>(this ICanGetModel self) where T : class, IModel
            => self.GetArchitecture().GetModel<T>();
        public static void SendEvent<T>(this ICanSendEvent self, T e)
            => self.GetArchitecture().SendEvent(e);
        public static void SendCommand<T>(this ICanSendCommand self, T cmd) where T : ICommand
            => self.GetArchitecture().SendCommand(cmd);
    }

    // ---------- 生命周期 ----------
    public interface ICanInit { bool Initialized { get; set; } void Init(); }
    public interface IModel : IBelongToArchitecture, ICanSetArchitecture, ICanSendEvent, ICanInit { }
    public interface ISystem : IBelongToArchitecture, ICanSetArchitecture, ICanGetModel, ICanSendEvent, ICanInit { }
    public interface IUtility { }

    public abstract class AbstractModel : IModel
    {
        private IArchitecture mArch;
        public IArchitecture GetArchitecture() => mArch;
        public void SetArchitecture(IArchitecture a) => mArch = a;
        public bool Initialized { get; set; }
        public void Init() => OnInit();
        protected abstract void OnInit();
    }
    public abstract class AbstractSystem : ISystem
    {
        private IArchitecture mArch;
        public IArchitecture GetArchitecture() => mArch;
        public void SetArchitecture(IArchitecture a) => mArch = a;
        public bool Initialized { get; set; }
        public void Init() => OnInit();
        protected abstract void OnInit();
    }

    // ---------- Command ----------
    public interface ICommand : IBelongToArchitecture, ICanSetArchitecture, ICanGetModel, ICanSendEvent { void Execute(); }
    public abstract class AbstractCommand : ICommand
    {
        private IArchitecture mArch;
        public IArchitecture GetArchitecture() => mArch;
        public void SetArchitecture(IArchitecture a) => mArch = a;
        public void Execute() => OnExecute();
        protected abstract void OnExecute();
    }

    // ---------- IOC ----------
    public class IOCContainer
    {
        private readonly Dictionary<Type, object> mInstances = new();
        public void Register<T>(T inst) => mInstances[typeof(T)] = inst;          // 重复 key 覆盖
        public T Get<T>() where T : class => mInstances.TryGetValue(typeof(T), out var o) ? o as T : null;
        public IEnumerable<T> ByType<T>() => mInstances.Values.OfType<T>();
        public void Clear() => mInstances.Clear();
    }

    // ---------- 架构母体（CRTP 单例）----------
    public interface IArchitecture
    {
        void RegisterModel<T>(T m) where T : IModel;
        void RegisterSystem<T>(T s) where T : ISystem;
        void RegisterUtility<T>(T u) where T : IUtility;
        T GetModel<T>() where T : class, IModel;
        T GetSystem<T>() where T : class, ISystem;
        void SendCommand<T>(T cmd) where T : ICommand;
        void SendEvent<T>(T e);
        IUnRegister RegisterEvent<T>(Action<T> on);
    }

    public abstract class Architecture<T> : IArchitecture where T : Architecture<T>, new()
    {
        private bool mInited;
        private static T mArchitecture;                       // 按封闭类型各一份
        private readonly IOCContainer mContainer = new();
        private readonly TypeEventSystem mEvents = new();

        public static IArchitecture Interface { get { if (mArchitecture == null) Init(); return mArchitecture; } }

        private static void Init()
        {
            mArchitecture = new T();
            mArchitecture.OnInit();
            foreach (var m in mArchitecture.mContainer.ByType<IModel>().Where(x => !x.Initialized))
                { m.Init(); m.Initialized = true; }            // Model 先
            foreach (var s in mArchitecture.mContainer.ByType<ISystem>().Where(x => !x.Initialized))
                { s.Init(); s.Initialized = true; }            // System 后
            mArchitecture.mInited = true;
        }
        protected abstract void OnInit();

        public void RegisterModel<TM>(TM m) where TM : IModel
        {
            m.SetArchitecture(this); mContainer.Register(m);
            if (mInited) { m.Init(); m.Initialized = true; }   // 双路径
        }
        public void RegisterSystem<TS>(TS s) where TS : ISystem
        {
            s.SetArchitecture(this); mContainer.Register(s);
            if (mInited) { s.Init(); s.Initialized = true; }
        }
        public void RegisterUtility<TU>(TU u) where TU : IUtility => mContainer.Register(u);
        public TM GetModel<TM>() where TM : class, IModel => mContainer.Get<TM>();
        public TS GetSystem<TS>() where TS : class, ISystem => mContainer.Get<TS>();
        public void SendCommand<TC>(TC cmd) where TC : ICommand { cmd.SetArchitecture(this); cmd.Execute(); }
        public void SendEvent<TE>(TE e) => mEvents.Send(e);
        public IUnRegister RegisterEvent<TE>(Action<TE> on) => mEvents.Register(on);
    }

    // ---------- EasyEvent + 注销句柄 ----------
    public interface IUnRegister { void UnRegister(); }
    public struct CustomUnRegister : IUnRegister
    {
        private Action mOn;
        public CustomUnRegister(Action on) => mOn = on;
        public void UnRegister() { mOn?.Invoke(); mOn = null; } // 防重复注销
    }
    public class EasyEvent<T>
    {
        private Action<T> mOn = _ => { };
        public IUnRegister Register(Action<T> on) { mOn += on; return new CustomUnRegister(() => mOn -= on); }
        public void Trigger(T t) => mOn?.Invoke(t);
    }
    public class TypeEventSystem
    {
        private readonly Dictionary<Type, object> mEvents = new();
        public IUnRegister Register<T>(Action<T> on)
        {
            if (!mEvents.TryGetValue(typeof(T), out var e)) { e = new EasyEvent<T>(); mEvents[typeof(T)] = e; }
            return ((EasyEvent<T>)e).Register(on);
        }
        public void Send<T>(T t) { if (mEvents.TryGetValue(typeof(T), out var e)) ((EasyEvent<T>)e).Trigger(t); }
    }

    // ---------- BindableProperty（可观测快照）----------
    public class BindableProperty<T>
    {
        private T mValue;
        public static Func<T, T, bool> Comparer { get; set; } = (a, b) => Equals(a, b);
        public BindableProperty(T def = default) => mValue = def;
        private readonly EasyEvent<T> mOnChanged = new();
        public T Value
        {
            get => mValue;
            set
            {
                if (value == null && mValue == null) return;
                if (value != null && Comparer(value, mValue)) return; // 去重
                mValue = value; mOnChanged.Trigger(value);
            }
        }
        public IUnRegister Register(Action<T> on) => mOnChanged.Register(on);
        public IUnRegister RegisterWithInitValue(Action<T> on) { on(mValue); return Register(on); }
    }
}
```

## 使用示例

```csharp
using MiniQF;
using System;

class CountModel : AbstractModel
{
    public BindableProperty<int> Count = new(0);
    protected override void OnInit() { }
}

class AddCommand : AbstractCommand
{
    protected override void OnExecute() => this.GetModel<CountModel>().Count.Value++;
}

class GameArch : Architecture<GameArch>
{
    protected override void OnInit() => RegisterModel(new CountModel());
}

class Demo
{
    static void Main()
    {
        GameArch.Interface.GetModel<CountModel>().Count
            .RegisterWithInitValue(v => Console.WriteLine($"Count={v}")); // 立刻打印 Count=0
        GameArch.Interface.SendCommand(new AddCommand());                 // Count=1
        GameArch.Interface.SendCommand(new AddCommand());                 // Count=2
    }
}
```

## 取舍自检

- ✅ **保留的核心不变量**：CRTP 每类型独立单例容器；Model 先于 System 初始化；注册双路径（Init 前批量 / Init 后即时）幂等；事件按 `Type` key 化；`CustomUnRegister` 用后置 null 防重复注销；`BindableProperty` 经 `Comparer` 去重再 Trigger + `RegisterWithInitValue` 立即回灌。
- ❌ **砍掉的**：`IQuery`（与 Command 同构）、多参 `EasyEvent<T,K,S>`、`OrEvent`、Mono 生命周期自动注销、`OnRegisterPatch`、`SetValueWithoutEvent`、`ICommand<TResult>`。
- ⚠️ **最容易搞错的一处**：`static T mArchitecture` 必须声明在**泛型类** `Architecture<T>` 内，依赖 CLR"封闭泛型类型各持一份静态字段"的语义。若把它提到非泛型基类或写成 `static Architecture`，所有派生架构会共享同一容器，多架构场景彻底失效。其次是 `RegisterEvent` 返回的 `IUnRegister` 必须被调用方持有并在合适时机 `UnRegister`，否则回调随对象存活而泄漏。
