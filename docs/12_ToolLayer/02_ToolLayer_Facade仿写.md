# 12 · 工具层 Facade 仿写

> 约 140 行复刻三个代表数据结构/工具：`EasyGrid<T>`（越界保护 + 函数式 API）、`Table<T>`+`TableIndex`（多索引联合查询，**框架内 UIPanelTable/ResTable 的母版**）、`LogKit`（等级过滤）。这三者最能体现工具层"轻量数据结构 + Helper 注入 + 池化复用"的共性。

## 设计映射表

| 原实现 | 精简版 | 取舍 |
|---|---|---|
| `EasyGrid<T>` 二维 + 越界保护 + Fill/ForEach/Resize | 保留（简化 Resize 为同向扩展） | ✅ |
| `Table<T>` + `TableIndex<K,V>` | 保留 | ✅ 核心母版 |
| TableIndex 用 ListPool | 简化为普通 List（标注原版池化） | 简化 |
| `LogKit` 等级过滤 | 保留 | ✅ |
| FluentAPI | 给 Table 加一个示例扩展 | 简化 |
| DynaGrid/JsonKit/其余 | 砍掉 | ❌ |

## 最小复刻代码

```csharp
using System;
using System.Collections;
using System.Collections.Generic;
using System.Linq;

namespace MiniTool
{
    // ===== EasyGrid：二维网格 + 越界保护 + 函数式 API =====
    public class EasyGrid<T>
    {
        private T[,] mGrid;
        public int Width { get; private set; }
        public int Height { get; private set; }

        public EasyGrid(int w, int h) { Width = w; Height = h; mGrid = new T[w, h]; }

        public void Fill(T value) { ForEachIndex((x, y) => mGrid[x, y] = value); }
        public void Fill(Func<int, int, T> onFill) { ForEachIndex((x, y) => mGrid[x, y] = onFill(x, y)); }

        public T this[int x, int y]
        {
            get
            {
                if (InBounds(x, y)) return mGrid[x, y];
                Console.WriteLine($"warn: out of bounds [{x},{y}]");   // 越界告警而非抛异常
                return default;
            }
            set { if (InBounds(x, y)) mGrid[x, y] = value; else Console.WriteLine($"warn: out of bounds [{x},{y}]"); }
        }

        public void ForEach(Action<int, int, T> each) => ForEachIndex((x, y) => each(x, y, mGrid[x, y]));

        public void Resize(int w, int h, Func<int, int, T> onAdd)
        {
            var ng = new T[w, h];
            for (var x = 0; x < w; x++)
                for (var y = 0; y < h; y++)
                    ng[x, y] = (x < Width && y < Height) ? mGrid[x, y] : onAdd(x, y); // 拷旧 / 填新
            Width = w; Height = h; mGrid = ng;
        }

        public void Clear(Action<T> cleanup = null)
        {
            if (cleanup != null) ForEach((x, y, c) => cleanup(c));
            mGrid = null;
        }

        private bool InBounds(int x, int y) => x >= 0 && x < Width && y >= 0 && y < Height;
        private void ForEachIndex(Action<int, int> a)
        { for (var x = 0; x < Width; x++) for (var y = 0; y < Height; y++) a(x, y); }
    }

    // ===== Table + TableIndex：多索引联合查询（UIPanelTable/ResTable 母版）=====
    public abstract class Table<TItem> : IEnumerable<TItem>, IDisposable
    {
        public void Add(TItem item) => OnAdd(item);
        public void Remove(TItem item) => OnRemove(item);
        public void Clear() => OnClear();
        protected abstract void OnAdd(TItem item);
        protected abstract void OnRemove(TItem item);
        protected abstract void OnClear();
        public abstract IEnumerator<TItem> GetEnumerator();
        IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
        public void Dispose() => OnDispose();
        protected abstract void OnDispose();
    }

    public class TableIndex<TKey, TItem> : IDisposable
    {
        private readonly Dictionary<TKey, List<TItem>> mIndex = new();
        private readonly Func<TItem, TKey> mGetKey;   // 注入: 如何从 item 取 key
        public TableIndex(Func<TItem, TKey> keyGetter) => mGetKey = keyGetter;
        public IReadOnlyDictionary<TKey, List<TItem>> Dictionary => mIndex;

        public void Add(TItem item)
        {
            var key = mGetKey(item);
            if (mIndex.TryGetValue(key, out var list)) list.Add(item);
            else mIndex[key] = new List<TItem> { item };   // 原版: ListPool.Get()
        }
        public void Remove(TItem item) { if (mIndex.TryGetValue(mGetKey(item), out var l)) l.Remove(item); }
        public IEnumerable<TItem> Get(TKey key) => mIndex.TryGetValue(key, out var l) ? l : Enumerable.Empty<TItem>();
        public void Clear() { foreach (var v in mIndex.Values) v.Clear(); mIndex.Clear(); }
        public void Dispose() => mIndex.Clear();
    }

    // ===== LogKit：等级过滤 =====
    public enum LogLevel { None = 0, Error = 1, Warning = 2, Normal = 3 }
    public static class LogKit
    {
        public static LogLevel Level = LogLevel.Normal;
        public static void I(object msg) { if (Level >= LogLevel.Normal) Console.WriteLine("[I] " + msg); }
        public static void W(object msg) { if (Level >= LogLevel.Warning) Console.WriteLine("[W] " + msg); }
        public static void E(object msg) { if (Level >= LogLevel.Error) Console.WriteLine("[E] " + msg); }
    }
    public static class LogFluent { public static void LogInfo(this object self) => LogKit.I(self); } // FluentAPI 样例
}
```

## 使用示例

```csharp
using MiniTool;
using System.Linq;

class Student { public string Name; public int Age; public int Level; }

class School : Table<Student>
{
    public TableIndex<int, Student> AgeIndex = new(s => s.Age);
    public TableIndex<int, Student> LevelIndex = new(s => s.Level);
    protected override void OnAdd(Student s) { AgeIndex.Add(s); LevelIndex.Add(s); }     // 维护所有索引
    protected override void OnRemove(Student s) { AgeIndex.Remove(s); LevelIndex.Remove(s); }
    protected override void OnClear() { AgeIndex.Clear(); LevelIndex.Clear(); }
    public override System.Collections.Generic.IEnumerator<Student> GetEnumerator()
        => AgeIndex.Dictionary.Values.SelectMany(s => s).GetEnumerator();
    protected override void OnDispose() { AgeIndex.Dispose(); LevelIndex.Dispose(); }
}

class Demo
{
    static void Main()
    {
        // EasyGrid
        var grid = new EasyGrid<string>(3, 3);
        grid.Fill("Empty");
        grid[1, 1] = "Hero";
        grid[9, 9] = "X";                       // 越界告警，不崩
        grid.ForEach((x, y, c) => { if (c == "Hero") System.Console.WriteLine($"({x},{y})={c}"); });

        // Table 联合查询：Level==2 且 Age<3
        var school = new School();
        school.Add(new Student { Name = "liang", Age = 1, Level = 2 });
        school.Add(new Student { Name = "ava", Age = 2, Level = 2 });
        school.Add(new Student { Name = "abc", Age = 3, Level = 2 });
        foreach (var s in school.LevelIndex.Get(2).Where(s => s.Age < 3))
            System.Console.WriteLine($"{s.Age}:{s.Level}:{s.Name}"); // 1:2:liang / 2:2:ava

        // LogKit
        LogKit.Level = LogLevel.Warning;
        LogKit.I("hidden");                     // 被等级过滤
        LogKit.E("shown");                      // [E] shown
    }
}
```

## 取舍自检

- ✅ **保留的核心不变量**：EasyGrid 越界保护（告警 + default，不抛）+ 函数式 Fill/ForEach/Resize；Table 抽象基类 + 多 TableIndex 组合 + 子类在 OnAdd/OnRemove 同步维护所有索引；TableIndex 用 keyGetter 注入分桶 + `Dictionary<K,List<V>>` + 联合查询（索引取桶 + LINQ 二次过滤）；LogKit 等级阈值提前 return 过滤。
- ❌ **砍掉的**：DynaGrid（动态/负坐标网格）、TableIndex 的 ListPool（简化为 List）、JsonKit 序列化、LocaleKit/GraphKit/CodeGenKit/PackageKit、FluentAPI 的海量扩展（仅留一例）。
- ⚠️ **最容易搞错的一处**：`Table` 的子类**必须在 `OnAdd`/`OnRemove`/`OnClear` 里同步维护它声明的每一个 `TableIndex`**。漏掉一个索引（比如 OnAdd 加了 AgeIndex 忘了 LevelIndex），按该索引查询会得到不完整结果，且这种 bug 不报错、只表现为"查不全"——与 UIKit 双索引同源的陷阱。其次：EasyGrid.Resize 拷贝旧数据时务必正确判断"哪些是拷贝区、哪些是新增区"，原框架分 x/y 两段循环处理，简化版用统一 `x<Width && y<Height` 判定。
