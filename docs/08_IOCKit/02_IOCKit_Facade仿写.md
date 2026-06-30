# 08 · IOCKit Facade 仿写

> 约 150 行复刻：复合 key（Type+name）+ 三张表（Instances/Mappings/Relations）+ 实例优先解析 + 特性注入 + 贪婪构造注入。保留命名 DI、单例/瞬态语义、双注入。砍掉 ResolveAll 的部分重载、Relation 的泛型糖、MemberSearchModes 多档（固定 All）。

## 设计映射表

| 原实现 | 精简版 | 取舍 |
|---|---|---|
| `InjectAttribute(name)` | 保留 | ✅ |
| `Tuple<Type,string>` 复合 key | 用 C# `ValueTuple`（现代环境支持） | 简化（原因 Mono 不支持已过时） |
| `Instances`/`Mappings`/`Relations` 三表 | 保留 | ✅ 核心 |
| `Resolve` 实例优先→映射新建 | 保留 | ✅ 核心不变量 |
| `Inject` 反射扫 `[Inject]` | 保留 | ✅ |
| `CreateInstance` 贪婪构造 + 命名回退 | 保留 | ✅ 核心难点 |
| `RegisterRelation`/`ResolveRelation` | 保留精简 | ✅ |
| `MemberSearchModes` 三档 | 固定 All | 简化 |
| `ResolveAll` 全部重载 | 保留一个 | 简化 |

## 最小复刻代码

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Reflection;

namespace MiniIOC
{
    [AttributeUsage(AttributeTargets.Field | AttributeTargets.Property)]
    public class InjectAttribute : Attribute
    {
        public string Name { get; }
        public InjectAttribute() { }
        public InjectAttribute(string name) => Name = name;
    }

    public class Container
    {
        // 复合 key：(类型, 命名)
        private readonly Dictionary<(Type, string), object> mInstances = new();   // 单例
        private readonly Dictionary<(Type, string), Type> mMappings = new();      // 映射(瞬态)
        private readonly Dictionary<(Type, Type), Type> mRelations = new();       // (for,base)->concrete

        // ---------- 注册 ----------
        public void Register<TSrc, TTgt>(string name = null) => mMappings[(typeof(TSrc), name)] = typeof(TTgt);

        public void RegisterInstance<T>(T instance, string name = null, bool injectNow = true) where T : class
        {
            mInstances[(typeof(T), name)] = instance;
            if (injectNow) Inject(instance);
        }
        public void RegisterRelation<TFor, TBase, TConcrete>() => mRelations[(typeof(TFor), typeof(TBase))] = typeof(TConcrete);

        // ---------- 解析：实例优先 → 映射新建 ----------
        public T Resolve<T>(string name = null, bool requireInstance = false, params object[] args) where T : class
            => (T)Resolve(typeof(T), name, requireInstance, args);

        public object Resolve(Type type, string name = null, bool requireInstance = false, params object[] args)
        {
            if (mInstances.TryGetValue((type, name), out var inst) && inst != null) return inst; // 单例优先
            if (requireInstance) return null;
            if (mMappings.TryGetValue((type, name), out var impl)) return CreateInstance(impl, args); // 映射新建
            return null;
        }

        public IEnumerable<object> ResolveAll(Type type)
        {
            foreach (var kv in mInstances)
                if (kv.Key.Item1 == type && !string.IsNullOrEmpty(kv.Key.Item2)) yield return kv.Value;
            foreach (var kv in mMappings)
                if (!string.IsNullOrEmpty(kv.Key.Item2) && type.IsAssignableFrom(kv.Key.Item1))
                {
                    var item = Activator.CreateInstance(kv.Value);
                    Inject(item);
                    yield return item;
                }
        }

        public object ResolveRelation(Type tfor, Type tbase, params object[] args)
            => mRelations.TryGetValue((tfor, tbase), out var c) ? CreateInstance(c, args) : null;

        // ---------- 特性注入：事后填充 ----------
        public void Inject(object obj)
        {
            if (obj == null) return;
            var members = obj.GetType().GetMembers(BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
            foreach (var m in members)
            {
                var attr = m.GetCustomAttributes(typeof(InjectAttribute), true).FirstOrDefault() as InjectAttribute;
                if (attr == null) continue;
                if (m is PropertyInfo p) p.SetValue(obj, Resolve(p.PropertyType, attr.Name), null);
                else if (m is FieldInfo f) f.SetValue(obj, Resolve(f.FieldType, attr.Name));
            }
        }

        // ---------- 贪婪构造注入 ----------
        public object CreateInstance(Type type, params object[] args)
        {
            object obj;
            if (args != null && args.Length > 0) { obj = Activator.CreateInstance(type, args); Inject(obj); return obj; }

            var ctors = type.GetConstructors(BindingFlags.Public | BindingFlags.Instance);
            if (ctors.Length < 1) { obj = Activator.CreateInstance(type); Inject(obj); return obj; }

            var max = ctors.First().GetParameters();          // 选参数最多的构造
            foreach (var c in ctors)
                if (c.GetParameters().Length > max.Length) max = c.GetParameters();

            var resolved = max.Select(p => p.ParameterType.IsArray
                ? (object)ResolveAll(p.ParameterType.GetElementType()).ToArray()
                : Resolve(p.ParameterType) ?? Resolve(p.ParameterType, p.Name)).ToArray(); // 类型→参数名 两级回退
            obj = Activator.CreateInstance(type, resolved);
            Inject(obj);                                      // 构造注入后再补特性注入
            return obj;
        }

        public void Clear() { mInstances.Clear(); mMappings.Clear(); mRelations.Clear(); }
    }
}
```

## 使用示例

```csharp
using MiniIOC;
using System;

interface ILogger { void Log(string m); }
class ConsoleLogger : ILogger { public void Log(string m) => Console.WriteLine("[LOG] " + m); }

class UserService
{
    [Inject] public ILogger Logger { get; set; }            // 特性注入
    public void Do() => Logger.Log("working");
}

class OrderService
{
    private readonly ILogger mLogger;
    public OrderService(ILogger logger) => mLogger = logger; // 构造注入
    public void Pay() => mLogger.Log("paid");
}

class Demo
{
    static void Main()
    {
        var c = new Container();
        c.RegisterInstance<ILogger>(new ConsoleLogger());     // 单例
        c.Register<UserService, UserService>();               // 映射(瞬态)
        c.Register<OrderService, OrderService>();

        var us = c.Resolve<UserService>();  us.Do();          // 特性注入 Logger → [LOG] working
        var os = c.Resolve<OrderService>(); os.Pay();         // 构造注入 Logger → [LOG] paid

        // 命名多实现
        c.RegisterInstance<ILogger>(new ConsoleLogger(), "audit");
        var audit = c.Resolve<ILogger>("audit");
    }
}
```

## 取舍自检

- ✅ **保留的核心不变量**：复合 key `(Type, name)` 支持同类型多命名注册；解析"实例优先→映射新建"（单例 vs 瞬态语义）；特性注入事后填充 `[Inject]` 成员；贪婪构造选参数最多的构造 + 每参数"类型→参数名"两级回退 + 数组走 ResolveAll；构造注入后再补一次特性注入；三表可独立 Clear。
- ❌ **砍掉的**：自实现 `Tuple`（现代用 ValueTuple）、`MemberSearchModes` 三档（固定 All）、`ResolveRelation` 的泛型糖重载、`InjectAll`、部分 `RegisterInstance` 重载。
- ⚠️ **最容易搞错的一处**：**没有循环依赖检测**。若 A 的构造需要 B、B 的构造需要 A，`CreateInstance` 会 A→Resolve(B)→CreateInstance(B)→Resolve(A)→… 无限递归栈溢出。原框架同样无此保护。仿写/使用时要么避免构造层循环依赖（改用特性注入打破环——因为特性注入是对象创建后才填充，能容忍环），要么自行加"正在解析中"的栈检测。次坑：`RegisterInstance` 与 `Register` 语义相反（前者单例后者瞬态），混用会拿到错误生命周期。
