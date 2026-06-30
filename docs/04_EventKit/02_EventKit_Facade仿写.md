# 04 · EventKit Facade 仿写

> 约 110 行复刻：底层 `EasyEvent`/`EasyEvent<T>` + `StringEventSystem`（懒创建路由 + 类型化 payload）+ `EnumEventSystem`（key 归一化 + 变长参数）。保留"key 化路由 + 静默失败 + 懒创建"不变量。

## 设计映射表

| 原实现 | 精简版 | 取舍 |
|---|---|---|
| Core 的 `IEasyEvent`/`EasyEvent`/`EasyEvent<T>` | 内联精简版 | ✅（自包含可编译） |
| `CustomUnRegister`（struct 注销句柄） | 保留 | ✅ 注销不变量 |
| `StringEventSystem` 懒创建 + `As<>` 转型 | 保留 | ✅ 核心 |
| `Register`(无参) + `Register<T>`(带参) | 保留两者 | ✅ 体现混用风险 |
| `EnumEventSystem` int 归一化 + `object[]` | 保留 | ✅ |
| `Global` 单例 | 保留 | ✅ |
| `[Obsolete]` 兼容壳 | 砍掉 | ❌ |
| EventTrigger/Component（Unity 包装） | 砍掉 | ❌ 平台细节 |

## 最小复刻代码

```csharp
using System;
using System.Collections.Generic;

namespace MiniEvent
{
    // ---------- 底层事件单元（来自 Core，精简内联）----------
    public interface IUnRegister { void UnRegister(); }
    public struct CustomUnRegister : IUnRegister
    {
        private Action mOn;
        public CustomUnRegister(Action on) => mOn = on;
        public void UnRegister() { mOn?.Invoke(); mOn = null; }
    }
    public interface IEasyEvent { }
    public class EasyEvent : IEasyEvent
    {
        private Action mOn = () => { };
        public IUnRegister Register(Action on) { mOn += on; return new CustomUnRegister(() => mOn -= on); }
        public void UnRegister(Action on) => mOn -= on;
        public void Trigger() => mOn?.Invoke();
    }
    public class EasyEvent<T> : IEasyEvent
    {
        private Action<T> mOn = _ => { };
        public IUnRegister Register(Action<T> on) { mOn += on; return new CustomUnRegister(() => mOn -= on); }
        public void UnRegister(Action<T> on) => mOn -= on;
        public void Trigger(T t) => mOn?.Invoke(t);
    }

    // ---------- String key 总线：懒创建路由 ----------
    public class StringEventSystem
    {
        public static readonly StringEventSystem Global = new();
        private readonly Dictionary<string, IEasyEvent> mEvents = new();

        public IUnRegister Register(string key, Action on)
        {
            if (mEvents.TryGetValue(key, out var e)) return ((EasyEvent)e).Register(on);
            var ev = new EasyEvent(); mEvents.Add(key, ev); return ev.Register(on);
        }
        public IUnRegister Register<T>(string key, Action<T> on)
        {
            if (mEvents.TryGetValue(key, out var e)) return ((EasyEvent<T>)e).Register(on);
            var ev = new EasyEvent<T>(); mEvents.Add(key, ev); return ev.Register(on);
        }
        public void Send(string key) { if (mEvents.TryGetValue(key, out var e)) (e as EasyEvent)?.Trigger(); }      // 静默失败
        public void Send<T>(string key, T data) { if (mEvents.TryGetValue(key, out var e)) (e as EasyEvent<T>)?.Trigger(data); }
        public void UnRegister(string key, Action on) { if (mEvents.TryGetValue(key, out var e)) (e as EasyEvent)?.UnRegister(on); }
        public void UnRegister<T>(string key, Action<T> on) { if (mEvents.TryGetValue(key, out var e)) (e as EasyEvent<T>)?.UnRegister(on); }
    }

    // ---------- Enum key 总线：int 归一化 + 变长参数 ----------
    public class EnumEventSystem
    {
        public static readonly EnumEventSystem Global = new();
        private readonly Dictionary<int, IEasyEvent> mEvents = new(50);

        public IUnRegister Register<T>(T key, Action<int, object[]> on) where T : IConvertible
        {
            var k = key.ToInt32(null);
            if (mEvents.TryGetValue(k, out var e)) return ((EasyEvent<int, object[]>)e).Register(on);
            var ev = new EasyEvent<int, object[]>(); mEvents.Add(k, ev); return ev.Register(on);
        }
        public void Send<T>(T key, params object[] args) where T : IConvertible
        {
            var k = key.ToInt32(null);
            if (mEvents.TryGetValue(k, out var e)) ((EasyEvent<int, object[]>)e).Trigger(k, args);
        }
        public void UnRegister<T>(T key, Action<int, object[]> on) where T : IConvertible
        {
            var k = key.ToInt32(null);
            if (mEvents.TryGetValue(k, out var e)) (e as EasyEvent<int, object[]>)?.UnRegister(on);
        }
        public void UnRegister<T>(T key) where T : IConvertible { var k = key.ToInt32(null); mEvents.Remove(k); } // 整 key 移除
        public void UnRegisterAll() => mEvents.Clear();
    }
}
```

## 使用示例

```csharp
using MiniEvent;
using System;

enum GameEvent { ScoreChanged, PlayerDied }

class Demo
{
    static void Main()
    {
        // String 版：类型化 payload
        var u1 = StringEventSystem.Global.Register<int>("score", s => Console.WriteLine($"分数={s}"));
        StringEventSystem.Global.Send("score", 100);      // 分数=100
        u1.UnRegister();
        StringEventSystem.Global.Send("score", 200);      // 已注销，无输出（静默）

        // Enum 版：消息 ID + 变长参数
        EnumEventSystem.Global.Register(GameEvent.PlayerDied, (id, args) =>
            Console.WriteLine($"玩家死亡, 凶手={args[0]}, 位置={args[1]}"));
        EnumEventSystem.Global.Send(GameEvent.PlayerDied, "Boss", "(3,4)");
        EnumEventSystem.Global.Send(GameEvent.ScoreChanged);  // 无订阅者，静默
    }
}
```

## 取舍自检

- ✅ **保留的核心不变量**：EventKit 只做 "key→EasyEvent" 路由，事件单元复用 Core；懒创建（首注册才建 EasyEvent）；`Send` 找不到 key 静默失败；String 版类型化 payload、Enum 版 int 归一化 + `object[]` 变长参数；Enum 版额外支持整 key 移除/全清。
- ❌ **砍掉的**：`[Obsolete]` 兼容壳（MsgDispatcher/QEventSystem）、EventTrigger/Component 的 Unity 包装、原版的 `.As<>()` 改用直接强转。
- ⚠️ **最容易搞错的一处**：同一 string key 不能在"无参 `EasyEvent`"和"带参 `EasyEvent<T>`"之间混用。原框架用 `As<>()` 转型，混用会转型失败拿 null；精简版用 `(EasyEvent<T>)e` 强转，混用会直接抛 `InvalidCastException`。这是弱类型 key 总线最隐蔽的坑——key 相同但 payload 形态不同时没有任何编译期保护。
