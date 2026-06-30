# 10 · ResKit Facade 仿写

> 约 170 行复刻：双层引用计数（资源级 `SimpleRC` + 加载器级 `ResLoader`）+ 全局资源表去重 + 延迟卸载（脏标记 + tick 扫描）+ 池化查询键。砍掉 AssetBundle/依赖链/异步协程节流/热更/具体资源类型（用一个可注入的同步加载模拟）。

## 设计映射表

| 原实现 | 精简版 | 取舍 |
|---|---|---|
| `SimpleRC` 资源级引用计数 + OnZeroRef | 保留 | ✅ 核心 |
| `Res` 状态机 + 加载完成事件 | 保留精简（Waiting/Loading/Ready） | ✅ |
| `ResMgr` 全局表 + 延迟卸载 | 保留（手动 Tick 代替 Update） | ✅ 核心 |
| `ResLoader` 加载器级引用 + 整批释放 | 保留 | ✅ 核心 |
| `ResSearchKeys` 池化 + Match | 简化为 string key | 简化 |
| 依赖链连带引用 | 砍掉 | ❌ AB 专属 |
| 异步加载 + 协程节流 | 砍掉（仅同步） | ❌ |
| AssetBundle/热更/具体 Res 子类 | 砍为可注入 `Func<string,object>` 加载器 | ❌ 平台细节 |
| `SafeARC`(HashSet owner) | 砍掉（仅 SimpleRC） | ❌ debug 辅助 |

## 最小复刻代码

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace MiniRes
{
    public enum ResState { Waiting, Loading, Ready }

    // ---------- 资源级引用计数 ----------
    public interface IRefCounter { int RefCount { get; } void Retain(); void Release(); }
    public abstract class SimpleRC : IRefCounter
    {
        public int RefCount { get; private set; }
        public void Retain() => ++RefCount;
        public void Release() { if (--RefCount == 0) OnZeroRef(); }
        protected virtual void OnZeroRef() { }
    }

    // ---------- 单个资源 ----------
    public class Res : SimpleRC
    {
        public string AssetName { get; }
        public ResState State { get; private set; } = ResState.Waiting;
        public object Asset { get; private set; }
        private event Action<bool, Res> mOnDone;

        // 注入的实际加载逻辑（真实项目对接 AB/Resources）
        public static Func<string, object> Loader = name => name; // 默认假加载

        public Res(string name) => AssetName = name;

        public void RegisterOnDone(Action<bool, Res> cb)
        {
            if (State == ResState.Ready) { cb(true, this); return; } // 已就绪立即回调
            mOnDone += cb;
        }

        public bool LoadSync()
        {
            if (State != ResState.Waiting) return State == ResState.Ready;
            State = ResState.Loading;
            Asset = Loader(AssetName);
            State = ResState.Ready;
            mOnDone?.Invoke(true, this);   // 一次性触发后清空
            mOnDone = null;
            return true;
        }

        public bool ReleaseRes()
        {
            if (State == ResState.Loading) return false;     // 加载中不可卸
            if (State != ResState.Ready) return true;
            Asset = null;                                    // 真实项目: Resources.UnloadAsset
            State = ResState.Waiting;
            mOnDone = null;
            return true;
        }

        protected override void OnZeroRef() { if (State != ResState.Loading) ReleaseRes(); }
    }

    // ---------- 全局资源管理器：表去重 + 延迟卸载 ----------
    public class ResMgr
    {
        public static readonly ResMgr Instance = new();
        private readonly Dictionary<string, Res> mTable = new();
        private bool mDirty;

        public Res GetRes(string name, bool createNew)
        {
            if (mTable.TryGetValue(name, out var res)) return res;  // 全局唯一
            if (!createNew) return null;
            res = new Res(name);
            mTable.Add(name, res);
            return res;
        }
        public void ClearOnUpdate() => mDirty = true;               // 标脏，延迟卸载

        public void Tick()                                          // 真实项目是 MonoBehaviour.Update
        {
            if (!mDirty) return;
            mDirty = false;
            foreach (var res in mTable.Values.ToArray())            // ToArray 快照避免迭代中改表
            {
                if (res.RefCount <= 0 && res.State != ResState.Loading && res.ReleaseRes())
                    mTable.Remove(res.AssetName);
            }
        }
        public int Count => mTable.Count;
    }

    // ---------- 用户侧加载器：管理一组引用，整批释放 ----------
    public class ResLoader : IDisposable
    {
        private readonly List<Res> mResList = new();

        public Res LoadResSync(string name)
        {
            Add2Load(name);
            var res = ResMgr.Instance.GetRes(name, false);
            res?.LoadSync();
            return res;
        }
        public void Add2Load(string name, Action<bool, Res> listener = null)
        {
            var existed = mResList.FirstOrDefault(r => r.AssetName == name);
            if (existed != null) { if (listener != null) existed.RegisterOnDone(listener); return; }
            var res = ResMgr.Instance.GetRes(name, true);
            if (res == null) return;
            res.Retain();                       // 加载器级 +1
            mResList.Add(res);
            if (listener != null) res.RegisterOnDone(listener);
        }
        public void ReleaseRes(string name)
        {
            var res = mResList.FirstOrDefault(r => r.AssetName == name);
            if (res == null) return;
            mResList.Remove(res);
            res.Release();                      // 加载器级 -1（可能触发 OnZeroRef）
            ResMgr.Instance.ClearOnUpdate();    // 标脏待表清理
        }
        public void ReleaseAllRes()
        {
            for (var i = mResList.Count - 1; i >= 0; i--) mResList[i].Release();
            mResList.Clear();
            ResMgr.Instance.ClearOnUpdate();
        }
        public void Dispose() => ReleaseAllRes();
    }
}
```

## 使用示例

```csharp
using MiniRes;
using System;

class Demo
{
    static void Main()
    {
        var loaderA = new ResLoader();
        var loaderB = new ResLoader();

        var texA = loaderA.LoadResSync("hero.png");  // 创建 Res, RefCount=1
        var texB = loaderB.LoadResSync("hero.png");  // 复用同一 Res, RefCount=2
        Console.WriteLine(ReferenceEquals(texA, texB)); // True（全局唯一）
        Console.WriteLine(ResMgr.Instance.Count);       // 1

        loaderA.Dispose();                  // RefCount=1，资源仍存活（B 还在用）
        ResMgr.Instance.Tick();
        Console.WriteLine(ResMgr.Instance.Count);       // 1（未卸载）

        loaderB.Dispose();                  // RefCount=0 → OnZeroRef 卸载 asset
        ResMgr.Instance.Tick();             // 延迟卸载: 从表移除
        Console.WriteLine(ResMgr.Instance.Count);       // 0
    }
}
```

## 取舍自检

- ✅ **保留的核心不变量**：双层引用计数（加载器 Retain/Release ↔ 资源 RefCount 的 OnZeroRef）；资源全局表去重（同名唯一、多加载器共享）；卸载需 `RefCount<=0 && State!=Loading` 双条件；延迟卸载（Release 只标脏，Tick 帧末 `ToArray` 快照遍历后删表）；加载完成事件一次性触发后清空 + 已 Ready 立即回调；加载器 Dispose/回收 = 整批释放持有引用。
- ❌ **砍掉的**：AssetBundle/依赖链连带引用/卸载顺序（Reverse）、异步加载 + 协程节流（mMaxCoroutineCount）、热更下载、具体资源子类（AssetBundleRes/ResourcesRes/NetImageRes）、`ResSearchKeys` 池化（简化为 string）、`SafeARC` 的 owner 去重、`ResLoader` 自身池化。
- ⚠️ **最容易搞错的一处**：`OnZeroRef`/`RemoveUnusedRes` 必须检查 `State != Loading`。正在异步加载的资源即使引用归零也绝不能卸载（加载协程还持有它，卸了会崩或加载完写入已销毁对象）。其次：每个 `Add2Load`（Retain）必须有且仅有一次对应 `Release`——加载器持有引用与释放引用不配对是引用计数系统的头号 bug（泄漏或悬挂）。第三：延迟卸载用 `ToArray()` 快照遍历，绝不能直接 `foreach mTable` 中 `Remove`（迭代中改集合崩溃）。
