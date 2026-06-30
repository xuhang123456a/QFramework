# 09 · UIKit Facade 仿写

> 约 170 行复刻：门面 `UIKit` → 单例 `UIManager` → 双索引 `UIPanelTable` 三层；池化 `PanelSearchKeys`/`PanelInfo`；`UIPanel` 生命周期模板；`UIPanelStack` 元数据栈导航。砍掉异步加载、AssetBundle、Level/Root、组件层、重载爆炸（只留泛型版）。用极简池 + 接口注入加载器代替 ResKit。

## 设计映射表

| 原实现 | 精简版 | 取舍 |
|---|---|---|
| 门面/管理器/表 三层 | 保留 | ✅ 核心结构 |
| `PanelSearchKeys` 池化 + 用后还 | 保留 | ✅ 母题 |
| `PanelInfo` 元数据快照 | 保留 | ✅ 导航不变量 |
| `UIPanelTable` 双索引（Type+Name） | 保留 | ✅ 核心 |
| `OpenType` Single/多开 | 保留 | ✅ |
| `UIPanel` 生命周期模板 + 钩子 | 保留 | ✅ |
| `UIPanelStack` 存 PanelInfo | 保留 | ✅ 导航母题 |
| `IPanelLoader`/`UIKitConfig` 加载策略 | 简化为 `Func<Type,IPanel>` 注入 | 简化（去 ResKit/AB） |
| UILevel/UIRoot/Canvas | 砍掉 | ❌ Unity 层级细节 |
| 异步 OpenPanelAsync | 砍掉 | ❌ |
| 字符串/重载版 API | 只留泛型版 | 简化 |

## 最小复刻代码

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace MiniUIKit
{
    public enum PanelState { Opening, Hide, Closed }
    public enum PanelOpenType { Single, Multiple }
    public interface IUIData { }

    public interface IPanel
    {
        string Name { get; set; }
        PanelState State { get; set; }
        PanelInfo Info { get; set; }
        void Init(IUIData data);
        void Open(IUIData data);
        void Show();
        void Hide();
        void Close();
    }

    // ---------- 池化的元数据快照 ----------
    public class PanelInfo
    {
        public string Name; public Type PanelType; public IUIData UIData;
        private static readonly Stack<PanelInfo> mPool = new();
        public static PanelInfo Allocate(string name, Type type, IUIData data)
        {
            var i = mPool.Count > 0 ? mPool.Pop() : new PanelInfo();
            i.Name = name; i.PanelType = type; i.UIData = data; return i;
        }
        public void Recycle() { Name = null; PanelType = null; UIData = null; mPool.Push(this); }
    }

    // ---------- 池化的查询参数 ----------
    public class PanelSearchKeys
    {
        public Type PanelType; public string Name; public IUIData UIData; public PanelOpenType OpenType = PanelOpenType.Single;
        private static readonly Stack<PanelSearchKeys> mPool = new();
        public static PanelSearchKeys Allocate() => mPool.Count > 0 ? mPool.Pop() : new PanelSearchKeys();
        public void Recycle() { PanelType = null; Name = null; UIData = null; OpenType = PanelOpenType.Single; mPool.Push(this); }
    }

    // ---------- 基类：生命周期模板 ----------
    public abstract class UIPanel : IPanel
    {
        public string Name { get; set; }
        public PanelState State { get; set; }
        public PanelInfo Info { get; set; }
        protected IUIData mUIData;
        private Action mOnClosed;

        public void Init(IUIData data) { mUIData = data; OnInit(data); }
        public void Open(IUIData data) { State = PanelState.Opening; OnOpen(data); }
        public void Show() { State = PanelState.Opening; OnShow(); }
        public void Hide() { State = PanelState.Hide; OnHide(); }
        public void Close()
        {
            Info.UIData = mUIData;        // 回写数据→保证 Pop 重建能还原
            mOnClosed?.Invoke(); mOnClosed = null;
            Hide(); State = PanelState.Closed; OnClose();
            mUIData = null;
        }
        public void OnClosed(Action cb) => mOnClosed = cb;
        protected virtual void OnInit(IUIData d) { }
        protected virtual void OnOpen(IUIData d) { }
        protected virtual void OnShow() { }
        protected virtual void OnHide() { }
        protected abstract void OnClose();
    }

    // ---------- 双索引注册表 ----------
    public class UIPanelTable : IEnumerable<IPanel>
    {
        private readonly Dictionary<Type, List<IPanel>> mTypeIndex = new();
        private readonly Dictionary<string, List<IPanel>> mNameIndex = new();

        public void Add(IPanel p)
        {
            (mTypeIndex.TryGetValue(p.GetType(), out var tl) ? tl : mTypeIndex[p.GetType()] = new()).Add(p);
            (mNameIndex.TryGetValue(p.Name, out var nl) ? nl : mNameIndex[p.Name] = new()).Add(p);
        }
        public void Remove(IPanel p)
        {
            if (mTypeIndex.TryGetValue(p.GetType(), out var tl)) tl.Remove(p);
            if (mNameIndex.TryGetValue(p.Name, out var nl)) nl.Remove(p);
        }
        public IEnumerable<IPanel> Query(PanelSearchKeys k)
        {
            if (k.PanelType != null && mTypeIndex.TryGetValue(k.PanelType, out var tl)) return tl;
            if (k.Name != null && mNameIndex.TryGetValue(k.Name, out var nl)) return nl;
            return Enumerable.Empty<IPanel>();
        }
        public void Clear() { mTypeIndex.Clear(); mNameIndex.Clear(); }
        public IEnumerator<IPanel> GetEnumerator() => mTypeIndex.Values.SelectMany(v => v).ToList().GetEnumerator();
        System.Collections.IEnumerator System.Collections.IEnumerable.GetEnumerator() => GetEnumerator();
    }

    // ---------- 导航栈：存元数据 ----------
    public class UIPanelStack
    {
        private readonly Stack<PanelInfo> mStack = new();
        public void Push(IPanel p) { mStack.Push(p.Info); p.Close(); UIManager.Instance.Remove(p); }
        public void Pop()
        {
            var info = mStack.Pop();
            var k = PanelSearchKeys.Allocate();
            k.PanelType = info.PanelType; k.Name = info.Name; k.UIData = info.UIData;
            UIManager.Instance.OpenUI(k);
            k.Recycle();
        }
    }

    // ---------- 单例管理器 ----------
    public class UIManager
    {
        public static readonly UIManager Instance = new();
        public UIPanelTable Table { get; } = new();
        public Func<Type, IPanel> PanelLoader;   // 加载策略注入点（替代 ResKit/Config）

        public IPanel OpenUI(PanelSearchKeys k)
        {
            IPanel panel = null;
            if (k.OpenType == PanelOpenType.Single) panel = Table.Query(k).FirstOrDefault();
            if (panel == null) panel = CreateUI(k);
            panel.Open(k.UIData);
            panel.Show();
            return panel;
        }
        private IPanel CreateUI(PanelSearchKeys k)
        {
            var panel = PanelLoader(k.PanelType);
            panel.Name = k.Name ?? k.PanelType.Name;
            panel.Info = PanelInfo.Allocate(panel.Name, k.PanelType, k.UIData);
            Table.Add(panel);
            panel.Init(k.UIData);
            return panel;
        }
        public void CloseUI(PanelSearchKeys k)
        {
            var panel = Table.Query(k).LastOrDefault();  // 关最新的那个
            if (panel == null) return;
            panel.Close(); Table.Remove(panel); panel.Info.Recycle(); panel.Info = null;
        }
        public void Remove(IPanel p) => Table.Remove(p);
    }

    // ---------- 门面 ----------
    public static class UIKit
    {
        public static UIPanelStack Stack { get; } = new();
        public static T OpenPanel<T>(IUIData data = null, PanelOpenType open = PanelOpenType.Single) where T : UIPanel
        {
            var k = PanelSearchKeys.Allocate();
            k.PanelType = typeof(T); k.UIData = data; k.OpenType = open;
            var p = UIManager.Instance.OpenUI(k) as T;
            k.Recycle();
            return p;
        }
        public static void ClosePanel<T>() where T : UIPanel
        {
            var k = PanelSearchKeys.Allocate();
            k.PanelType = typeof(T);
            UIManager.Instance.CloseUI(k);
            k.Recycle();
        }
    }
}
```

## 使用示例

```csharp
using MiniUIKit;
using System;

class HomePanelData : IUIData { public string From; }
class HomePanel : UIPanel
{
    protected override void OnOpen(IUIData d) => Console.WriteLine($"Home Open, from={(d as HomePanelData)?.From}");
    protected override void OnShow() => Console.WriteLine("Home Show");
    protected override void OnClose() => Console.WriteLine("Home Close");
}

class Demo
{
    static void Main()
    {
        // 注入加载策略（真实项目对接 ResKit；这里直接 new）
        UIManager.Instance.PanelLoader = type => (IPanel)Activator.CreateInstance(type);

        var home = UIKit.OpenPanel<HomePanel>(new HomePanelData { From = "Boot" });
        // Home Open, from=Boot / Home Show

        UIKit.Stack.Push(home);            // 关闭 Home，压入其 PanelInfo
        // Home Close
        UIKit.Stack.Pop();                 // 用 PanelInfo 重建并重新打开 Home
        // Home Open, from=Boot / Home Show  （UIData 经 Info 还原）

        UIKit.ClosePanel<HomePanel>();     // Home Close
    }
}
```

## 取舍自检

- ✅ **保留的核心不变量**：门面→单例管理器→双索引表三层；`PanelSearchKeys`/`PanelInfo` 池化 + 用后即还；双索引（Type+Name）在 Add/Remove 同步维护；`OpenType.Single` 复用、多开新建；`UIPanel` 生命周期模板 + 子类钩子 + `Close` 回写 UIData 到 Info；导航栈存 `PanelInfo` 元数据快照、Pop 重建；CloseUI 取 `LastOrDefault`、查询取 `FirstOrDefault`。
- ❌ **砍掉的**：异步加载、AssetBundle、UILevel/UIRoot/Canvas、`IPanelLoader`/`PanelLoaderPool`（简化为 `Func<Type,IPanel>`）、组件层（UIComponent/UIElement）、字符串与多重载 API、`SafeObjectPool`（简化为内置 Stack 池）。
- ⚠️ **最容易搞错的一处**：双索引必须在 Add/Remove **同时**维护两张表。只更新 `TypeIndex` 忘了 `NameIndex`，会导致 `OpenPanel<T>` 能复用但 `ClosePanel("name")` 找不到面板（或反之）。其次：`UIPanelStack.Push` 存的是 `panel.Info`（元数据）而非 panel 本身，且 Push 时面板被 `Close`（销毁）+ 从 Table `Remove`——Pop 是"按快照重建新面板"，不是"恢复旧对象"。若误以为是恢复对象、在 Close 时没把 `mUIData` 回写 `Info.UIData`，Pop 出来的面板会丢数据。
