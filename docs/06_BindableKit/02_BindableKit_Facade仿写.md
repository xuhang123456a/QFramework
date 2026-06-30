# 06 · BindableKit Facade 仿写

> 约 130 行复刻：`BindableProperty<T>`（单值，来自 Core）+ `BindableList<T>`（继承 `Collection<T>` 五虚方法拦截 + 惰性细粒度事件）。砍掉 `BindableDictionary` 的完整 `IDictionary` 实现（保留其语义骨架）、Move、序列化。

## 设计映射表

| 原实现 | 精简版 | 取舍 |
|---|---|---|
| `EasyEvent`/`EasyEvent<T..>`（Core） | 内联精简版 | ✅ 自包含 |
| `BindableProperty<T>` 去重 + RegisterWithInitValue | 保留 | ✅ |
| `BindableList<T> : Collection<T>` 五虚方法 | 保留 Insert/Remove/Set/Clear | ✅ 核心手法 |
| 惰性事件 `?? new` + 触发 `?.Trigger` | 保留 | ✅ 内存不变量 |
| `OnAdd/OnRemove/OnReplace/OnClear/OnCountChanged` | 保留 | ✅ |
| `OnMove`/`MoveItem` | 砍掉 | ❌ |
| `BindableDictionary` 完整 IDictionary | 砍为最小包装示意 | 简化 |
| `[Serializable]`/`[NonSerialized]` | 砍掉 | ❌ Unity 序列化细节 |

## 最小复刻代码

```csharp
using System;
using System.Collections.ObjectModel;

namespace MiniBindable
{
    // ---------- 事件单元（Core，精简）----------
    public interface IUnRegister { void UnRegister(); }
    public struct CustomUnRegister : IUnRegister
    {
        private Action mOn; public CustomUnRegister(Action on) => mOn = on;
        public void UnRegister() { mOn?.Invoke(); mOn = null; }
    }
    public class EasyEvent
    {
        private Action mOn = () => { };
        public IUnRegister Register(Action on) { mOn += on; return new CustomUnRegister(() => mOn -= on); }
        public void Trigger() => mOn?.Invoke();
    }
    public class EasyEvent<T>
    {
        private Action<T> mOn = _ => { };
        public IUnRegister Register(Action<T> on) { mOn += on; return new CustomUnRegister(() => mOn -= on); }
        public void Trigger(T t) => mOn?.Invoke(t);
    }
    public class EasyEvent<T, K>
    {
        private Action<T, K> mOn = (_, __) => { };
        public IUnRegister Register(Action<T, K> on) { mOn += on; return new CustomUnRegister(() => mOn -= on); }
        public void Trigger(T t, K k) => mOn?.Invoke(t, k);
    }
    public class EasyEvent<T, K, S>
    {
        private Action<T, K, S> mOn = (_, __, ___) => { };
        public IUnRegister Register(Action<T, K, S> on) { mOn += on; return new CustomUnRegister(() => mOn -= on); }
        public void Trigger(T t, K k, S s) => mOn?.Invoke(t, k, s);
    }

    // ---------- 单值可观测属性 ----------
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
                if (value != null && Comparer(value, mValue)) return;   // 去重
                mValue = value; mOnChanged.Trigger(value);
            }
        }
        public IUnRegister Register(Action<T> on) => mOnChanged.Register(on);
        public IUnRegister RegisterWithInitValue(Action<T> on) { on(mValue); return Register(on); } // 立即回灌
    }

    // ---------- 响应式 List：继承 Collection<T>，重写五虚方法 ----------
    public class BindableList<T> : Collection<T>
    {
        // 惰性事件：未订阅不分配
        private EasyEvent<int> mOnCountChanged;
        public EasyEvent<int> OnCountChanged => mOnCountChanged ??= new();
        private EasyEvent mOnClear;
        public EasyEvent OnClear => mOnClear ??= new();
        private EasyEvent<int, T> mOnAdd;
        public EasyEvent<int, T> OnAdd => mOnAdd ??= new();
        private EasyEvent<int, T> mOnRemove;
        public EasyEvent<int, T> OnRemove => mOnRemove ??= new();
        private EasyEvent<int, T, T> mOnReplace;
        public EasyEvent<int, T, T> OnReplace => mOnReplace ??= new();

        protected override void InsertItem(int index, T item)
        {
            base.InsertItem(index, item);
            mOnAdd?.Trigger(index, item);                    // ?. 跳过未订阅
            mOnCountChanged?.Trigger(Count);
        }
        protected override void RemoveItem(int index)
        {
            var item = this[index];
            base.RemoveItem(index);
            mOnRemove?.Trigger(index, item);
            mOnCountChanged?.Trigger(Count);
        }
        protected override void SetItem(int index, T item)
        {
            var old = this[index];
            base.SetItem(index, item);
            mOnReplace?.Trigger(index, old, item);           // 替换：不触发 Count
        }
        protected override void ClearItems()
        {
            var before = Count;
            base.ClearItems();
            mOnClear?.Trigger();
            if (before > 0) mOnCountChanged?.Trigger(Count); // 空集合 Clear 不通知
        }
    }
}
```

## 使用示例

```csharp
using MiniBindable;
using System;

class Demo
{
    static void Main()
    {
        // 单值
        var hp = new BindableProperty<int>(100);
        hp.RegisterWithInitValue(v => Console.WriteLine($"HP={v}")); // HP=100
        hp.Value = 100;     // 去重，无输出
        hp.Value = 80;      // HP=80

        // 集合（增量刷新）
        var inv = new BindableList<string>();
        inv.OnAdd.Register((i, item) => Console.WriteLine($"+ [{i}]={item}"));
        inv.OnReplace.Register((i, o, n) => Console.WriteLine($"~ [{i}] {o}->{n}"));
        inv.OnCountChanged.Register(c => Console.WriteLine($"count={c}"));

        inv.Add("sword");   // + [0]=sword / count=1
        inv.Add("shield");  // + [1]=shield / count=2
        inv[0] = "axe";     // ~ [0] sword->axe  （注意：无 count 变化）
        inv.Clear();        // OnClear 触发 / count=0
    }
}
```

## 取舍自检

- ✅ **保留的核心不变量**：`BindableList` 继承 `Collection<T>`、重写五虚方法一处拦截所有写入；事件惰性创建（`??=`）+ 触发处 `?.` 跳过；细粒度事件携带位置与新旧值；替换不触发 `OnCountChanged`、空集合 Clear 不触发 Count；`BindableProperty` 去重 + `RegisterWithInitValue` 立即回灌。
- ❌ **砍掉的**：`OnMove`/`MoveItem`、`BindableDictionary` 完整 `IDictionary`/序列化实现、`[Serializable]`/`[NonSerialized]`、持久化型 BindableProperty 变体。
- ⚠️ **最容易搞错的一处**：触发处必须用 `mOnAdd?.Trigger(...)` 而非 `OnAdd.Trigger(...)`。后者会通过属性 `??=` **强制创建**事件对象，惰性优化彻底失效（每次操作都分配）。其次：`SetItem`（替换）绝不能触发 `OnCountChanged`——数量没变，否则 UI 会做无意义的"列表数量"刷新。
