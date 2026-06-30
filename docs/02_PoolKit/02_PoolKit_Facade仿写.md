# 02 · PoolKit Facade 仿写

> 约 140 行复刻：`Pool<T>` 基类 + 工厂注入 + `SafeObjectPool`（IsRecycled 不变量 + 容量上限）+ `SimpleObjectPool` + `ListPool`。砍掉单例集成、API 文档特性、NonPublic 反射工厂。

## 设计映射表

| 原实现 | 精简版 | 取舍 |
|---|---|---|
| `IPool<T>` / `Pool<T>` 基类 + Stack + 工厂 | 保留 | ✅ |
| `IObjectFactory<T>` + Default/Custom | 保留两种 | ✅ |
| `NonPublicObjectFactory`（反射私有构造） | 砍掉 | ❌ 边缘场景 |
| `SafeObjectPool<T>` IsRecycled + maxCount + OnRecycled | 保留全部不变量 | ✅ 核心 |
| `SafeObjectPool : ISingleton`（单例集成） | 砍掉（改为普通可实例化） | ❌ 平台/单例细节 |
| `MaxCacheCount` setter 裁剪逻辑 | 保留 | ✅ 不变量 |
| `SimpleObjectPool<T>` + resetMethod | 保留 | ✅ |
| `ListPool` 查重抛异常 | 保留 | ✅ 体现 Safe/Simple 差异 |
| `DictionaryPool` | 砍掉（与 List 同构） | ❌ |
| `[ClassAPI]` 等编辑器特性 | 砍掉 | ❌ |

## 最小复刻代码

```csharp
using System;
using System.Collections.Generic;

namespace MiniPool
{
    public interface IObjectFactory<T> { T Create(); }
    public class DefaultObjectFactory<T> : IObjectFactory<T> where T : new() { public T Create() => new T(); }
    public class CustomObjectFactory<T> : IObjectFactory<T>
    {
        private readonly Func<T> mMethod;
        public CustomObjectFactory(Func<T> method) => mMethod = method;
        public T Create() => mMethod();
    }

    public interface IPool<T> { T Allocate(); bool Recycle(T obj); }
    public interface IPoolable { void OnRecycled(); bool IsRecycled { get; set; } }

    public abstract class Pool<T> : IPool<T>
    {
        protected readonly Stack<T> mCacheStack = new();
        protected IObjectFactory<T> mFactory;
        protected int mMaxCount = 12;
        public int CurCount => mCacheStack.Count;

        public void SetObjectFactory(IObjectFactory<T> f) => mFactory = f;
        public void SetFactoryMethod(Func<T> m) => mFactory = new CustomObjectFactory<T>(m);

        public virtual T Allocate() => mCacheStack.Count == 0 ? mFactory.Create() : mCacheStack.Pop();
        public abstract bool Recycle(T obj);

        public void Clear(Action<T> onClearItem = null)
        {
            if (onClearItem != null) foreach (var o in mCacheStack) onClearItem(o);
            mCacheStack.Clear();
        }
    }

    // ---------- Safe：带 IsRecycled 不变量 + 容量上限 ----------
    public class SafeObjectPool<T> : Pool<T> where T : IPoolable, new()
    {
        public SafeObjectPool() => mFactory = new DefaultObjectFactory<T>();

        public void Init(int maxCount, int initCount)
        {
            MaxCacheCount = maxCount;
            if (maxCount > 0) initCount = Math.Min(maxCount, initCount);
            for (var i = CurCount; i < initCount; i++) Recycle(mFactory.Create());
        }

        public int MaxCacheCount
        {
            get => mMaxCount;
            set
            {
                mMaxCount = value;
                if (mMaxCount > 0 && mMaxCount < mCacheStack.Count)
                {
                    var remove = mCacheStack.Count - mMaxCount;
                    while (remove-- > 0) mCacheStack.Pop();   // 调小上限→裁剪现有缓存
                }
            }
        }

        public override T Allocate()
        {
            var r = base.Allocate();
            r.IsRecycled = false;                              // 取出即标记"在外面"
            return r;
        }

        public override bool Recycle(T t)
        {
            if (t == null || t.IsRecycled) return false;       // 防空/重复回收
            if (mMaxCount > 0 && mCacheStack.Count >= mMaxCount)
            {
                t.OnRecycled();                                // 满栈：仍回调，但不入栈→丢弃
                return false;
            }
            t.IsRecycled = true;
            t.OnRecycled();
            mCacheStack.Push(t);
            return true;
        }
    }

    // ---------- Simple：零校验，信任调用方 ----------
    public class SimpleObjectPool<T> : Pool<T>
    {
        private readonly Action<T> mReset;
        public SimpleObjectPool(Func<T> factory, Action<T> reset = null, int initCount = 0)
        {
            mFactory = new CustomObjectFactory<T>(factory);
            mReset = reset;
            for (var i = 0; i < initCount; i++) mCacheStack.Push(mFactory.Create());
        }
        public override bool Recycle(T obj) { mReset?.Invoke(obj); mCacheStack.Push(obj); return true; }
    }

    // ---------- 集合池：查重抛异常 ----------
    public static class ListPool<T>
    {
        private static readonly Stack<List<T>> mStack = new(8);
        public static List<T> Get() => mStack.Count == 0 ? new List<T>(8) : mStack.Pop();
        public static void Release(List<T> l)
        {
            if (mStack.Contains(l)) throw new InvalidOperationException("重复回收 List");
            l.Clear();
            mStack.Push(l);
        }
    }
    public static class ListPoolExt { public static void Release2Pool<T>(this List<T> self) => ListPool<T>.Release(self); }
}
```

## 使用示例

```csharp
using MiniPool;

class Bullet : IPoolable
{
    public bool IsRecycled { get; set; }
    public void OnRecycled() => System.Console.WriteLine("回收 Bullet");
}

class Demo
{
    static void Main()
    {
        var pool = new SafeObjectPool<Bullet>();
        pool.Init(maxCount: 50, initCount: 2);
        System.Console.WriteLine(pool.CurCount);     // 2

        var b = pool.Allocate();                     // CurCount=1, b.IsRecycled=false
        System.Console.WriteLine(pool.CurCount);     // 1
        pool.Recycle(b);                             // 回收 Bullet, CurCount=2
        pool.Recycle(b);                             // 直接 false（IsRecycled 已 true，不打印第二次? 实际仍 return false 前未回调）

        // 集合池
        var tmp = ListPool<int>.Get();
        tmp.Add(1);
        tmp.Release2Pool();                          // Clear 后入栈
    }
}
```

## 取舍自检

- ✅ **保留的核心不变量**：栈式 LIFO 复用；工厂注入创建策略；Safe 的 `IsRecycled` 单一真相（Allocate 置 false / Recycle 置 true + 入口查重）；满栈丢弃仍调 `OnRecycled`；`MaxCacheCount` 调小时裁剪现有栈；Simple 零校验；ListPool 查重抛异常。
- ❌ **砍掉的**：单例集成（`ISingleton`/`SingletonProperty`）、`NonPublicObjectFactory`、`DictionaryPool`、API 文档特性、`IPoolType.Recycle2Cache`。
- ⚠️ **最容易搞错的一处**：`Recycle` 中"满栈丢弃"分支**也要调用 `OnRecycled()`**。直觉上会以为"没入池就不用回调"，但原框架语义是 `OnRecycled` = "对象被归还"（无论是否真进池）。漏掉它会导致满栈被丢弃的对象未释放其持有资源。次要坑：`Allocate` 里若忘记 `IsRecycled=false`，对象取出后仍被判定在池中，后续无法被回收。
