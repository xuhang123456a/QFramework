# 03 · SingletonKit Facade 仿写

> 约 130 行复刻：`ISingleton` + `SingletonCreator`（反射造私有构造 + Mono 分流）+ 继承式 `Singleton<T>`/`MonoSingleton<T>` + 属性式 `MonoSingletonProperty<T>`。保留"统一创建工厂 + OnSingletonInit 时序 + 私有构造唯一性"不变量。砍掉路径查找、ScriptableObject/Prefab 变体。

## 设计映射表

| 原实现 | 精简版 | 取舍 |
|---|---|---|
| `ISingleton.OnSingletonInit` | 保留 | ✅ 初始化时序不变量 |
| `SingletonCreator` 反射 + Mono 分流 | 保留 | ✅ 统一创建真相 |
| 纯 C# 反射私有构造 | 保留 | ✅ 唯一性不变量 |
| `Singleton<T>` 继承式 + lock 双检 | 保留 | ✅ |
| `MonoSingleton<T>` 继承式 + 生命周期清理 | 保留（简化回退链） | ✅ |
| `MonoSingletonProperty<T>` 属性式 | 保留 | ✅ 展示两套权衡 |
| `MonoSingletonPathAttribute` 路径查找 | 砍掉（直接新建 GO） | ❌ Hierarchy 细节 |
| `IsUnitTestMode` 全套测试分支 | 砍掉 | ❌ |
| Prefab/Scriptable/Persistent/Replaceable 变体 | 砍掉 | ❌ |

## 最小复刻代码

```csharp
using System;
using System.Reflection;
using UnityEngine;

namespace MiniSingleton
{
    public interface ISingleton { void OnSingletonInit(); }

    // ---------- 统一创建工厂：唯一的"如何造单例"真相 ----------
    internal static class SingletonCreator
    {
        public static T CreateSingleton<T>() where T : class, ISingleton
        {
            // Mono 子类走 GameObject 路径，否则反射私有构造
            if (typeof(MonoBehaviour).IsAssignableFrom(typeof(T)))
                return CreateMonoSingleton<T>();

            var instance = CreateNonPublicCtorObject<T>();
            instance.OnSingletonInit();
            return instance;
        }

        private static T CreateNonPublicCtorObject<T>() where T : class
        {
            var ctors = typeof(T).GetConstructors(BindingFlags.Instance | BindingFlags.NonPublic);
            var ctor = Array.Find(ctors, c => c.GetParameters().Length == 0)
                       ?? throw new Exception("需要私有无参构造: " + typeof(T));
            return ctor.Invoke(null) as T;                 // 强制构造私有→外部 new 不了
        }

        public static T CreateMonoSingleton<T>() where T : class, ISingleton
        {
            if (!Application.isPlaying) return null;         // 编辑器非运行态不建脏对象
            var instance = UnityEngine.Object.FindObjectOfType(typeof(T)) as T; // 先复用场景已有
            if (instance == null)
            {
                var go = new GameObject(typeof(T).Name);
                UnityEngine.Object.DontDestroyOnLoad(go);    // 跨场景存活
                instance = go.AddComponent(typeof(T)) as T;
            }
            instance.OnSingletonInit();
            return instance;
        }
    }

    // ---------- 继承式纯 C# 单例（带 lock 双检）----------
    public abstract class Singleton<T> : ISingleton where T : Singleton<T>
    {
        protected static T mInstance;
        private static readonly object mLock = new();
        public static T Instance
        {
            get { lock (mLock) { if (mInstance == null) mInstance = SingletonCreator.CreateSingleton<T>(); } return mInstance; }
        }
        public virtual void Dispose() => mInstance = null;   // 纯 C#：丢引用靠 GC
        public virtual void OnSingletonInit() { }
    }

    // ---------- 继承式 Mono 单例 ----------
    public abstract class MonoSingleton<T> : MonoBehaviour, ISingleton where T : MonoSingleton<T>
    {
        protected static T mInstance;
        public static T Instance
        {
            get { if (mInstance == null) mInstance = SingletonCreator.CreateMonoSingleton<T>(); return mInstance; }
        }
        public virtual void OnSingletonInit() { }
        public virtual void Dispose() => Destroy(gameObject);
        protected virtual void OnApplicationQuit() { if (mInstance != null) { Destroy(mInstance.gameObject); mInstance = null; } }
        protected virtual void OnDestroy() => mInstance = null;
    }

    // ---------- 属性式 Mono 单例（保留继承位）----------
    public static class MonoSingletonProperty<T> where T : MonoBehaviour, ISingleton
    {
        private static T mInstance;
        public static T Instance
        {
            get { if (mInstance == null) mInstance = SingletonCreator.CreateMonoSingleton<T>(); return mInstance; }
        }
        public static void Dispose() { if (mInstance) UnityEngine.Object.Destroy(mInstance.gameObject); mInstance = null; }
    }
}
```

## 使用示例

```csharp
using MiniSingleton;
using UnityEngine;

// 纯 C# 单例：私有构造 + 继承
public class GameData : Singleton<GameData>
{
    private GameData() { }                     // 必须私有，否则反射也行但失去唯一性保护
    public int Score;
    public override void OnSingletonInit() => Debug.Log("GameData 初始化");
}

// Mono 单例：属性式，保留继承位给别的基类
public class AudioManager : MonoBehaviour, ISingleton
{
    public static AudioManager Instance => MonoSingletonProperty<AudioManager>.Instance;
    public void OnSingletonInit() => Debug.Log("AudioManager 初始化");
    public void Play(string clip) { /* ... */ }
}

public class Demo : MonoBehaviour
{
    void Start()
    {
        GameData.Instance.Score = 100;         // 首访触发反射构造 + OnSingletonInit
        AudioManager.Instance.Play("bgm");     // 首访新建 GO + DontDestroyOnLoad
    }
}
```

## 取舍自检

- ✅ **保留的核心不变量**：`SingletonCreator` 是唯一创建真相并按 `typeof(T)` 分流；纯 C# 单例靠反射私有无参构造保证外部不可 new；`OnSingletonInit` 在实例就绪后统一调用一次；Mono 版编辑器非运行态返回 null + 复用场景已有 + DontDestroyOnLoad；纯 C# 版 lock 双检线程安全。
- ❌ **砍掉的**：`MonoSingletonPath` 路径查找回退链、`IsUnitTestMode` 测试分支、Prefab/Scriptable/Persistent/Replaceable 变体、Dispose 向上销毁父链。
- ⚠️ **最容易搞错的一处**：Mono 单例**绝不能**在 `Instance` getter 里无条件 new GameObject——必须先 `Application.isPlaying` 守卫 + `FindObjectOfType` 复用。否则①编辑器下访问会污染场景生成幽灵对象；②场景里手动摆放的单例会被忽略、又新建一个，导致"两个单例"。次坑：纯 C# 单例若写了 public 构造，唯一性保护就形同虚设（任何人能 `new`）。
