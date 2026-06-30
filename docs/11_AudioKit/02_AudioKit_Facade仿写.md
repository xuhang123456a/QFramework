# 11 · AudioKit Facade 仿写

> 约 160 行复刻：模块私有 `Architecture<Architecture>` + 自动初始化 + 门面转发 Command + `PlayingSoundPoolModel`（在播记录）+ 池化 `AudioPlayer`（自管进出记录）+ `BindableProperty` 音量联动。砍掉 Voice/去重通道/clip 异步准备/Fluent/Unity AudioSource（用桩）。
> 复用前面 Facade 的 Mini 架构与 BindableProperty（此处内联最小版以独立编译）。

## 设计映射表

| 原实现 | 精简版 | 取舍 |
|---|---|---|
| `Architecture<Architecture>` 私有架构 + AutoInit | 保留 | ✅ 核心母题落地 |
| 门面转发 Command | 保留 | ✅ MVC 不变量 |
| `PlayingSoundPoolModel` 在播记录 | 保留 | ✅ |
| `AudioPlayer` 池化 + 自管进出记录 | 保留 | ✅ 核心 |
| `BindableProperty` 音量联动 | 保留 | ✅ |
| `PlaySoundChannelSystem` 去重 | 砍掉（CanPlay 恒 true + IsSoundOn） | ❌ 复杂度 |
| Music/Voice 播放器 | 砍掉（仅 Sound） | ❌ |
| Unity AudioSource / clip 加载 | 桩（Console 输出 + 立即完成） | ❌ 平台细节 |
| Fluent API / Action 包装 | 砍掉 | ❌ |

## 最小复刻代码

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace MiniAudioKit
{
    // ===== 复用的极简 Core（见 01 CoreArchitecture Facade）=====
    public interface IBelongToArchitecture { IArch GetArchitecture(); }
    public interface ICanSetArchitecture { void SetArchitecture(IArch a); }
    public interface ICanInit { bool Initialized { get; set; } void Init(); }
    public interface IModel : ICanSetArchitecture, ICanInit { }
    public interface ISystem : ICanSetArchitecture, ICanInit { }
    public interface ICommand { void SetArchitecture(IArch a); void Execute(); }
    public interface IArch
    {
        void RegisterModel<T>(T m) where T : IModel;
        void RegisterSystem<T>(T s) where T : ISystem;
        void SendCommand<T>(T c) where T : ICommand;
    }
    public abstract class AbstractModel : IModel
    {
        private IArch mArch; public IArch GetArchitecture() => mArch; public void SetArchitecture(IArch a) => mArch = a;
        public bool Initialized { get; set; } public void Init() => OnInit(); protected abstract void OnInit();
    }
    public abstract class ArchBase<T> where T : ArchBase<T>, new()
    {
        private static T mArch; private readonly List<object> mContainer = new();
        public static T Interface { get { if (mArch == null) Init(); return mArch; } }
        private static void Init() { mArch = new T(); mArch.OnInit(); }
        protected abstract void OnInit();
        protected void Reg(object o) => mContainer.Add(o);
    }

    public class BindableProperty<T>
    {
        private T mValue; private Action<T> mOn = _ => { };
        public BindableProperty(T def = default) => mValue = def;
        public T Value { get => mValue; set { if (!Equals(value, mValue)) { mValue = value; mOn(value); } } }
        public void RegisterWithInitValue(Action<T> cb) { cb(mValue); mOn += cb; }
        public void UnRegister(Action<T> cb) => mOn -= cb;
    }

    // ===== AudioKit 私有架构 =====
    internal class Architecture : ArchBase<Architecture>
    {
        internal static PlayingSoundPoolModel PlayingSoundPool { get; } = new();
        internal static SettingsModel Settings { get; } = new();

        // 真实项目用 [RuntimeInitializeOnLoadMethod]；这里手动触发
        public static void AutoInit() { var _ = Interface; }
        protected override void OnInit() { Reg(PlayingSoundPool); Reg(Settings); }
    }

    internal class SettingsModel { public BindableProperty<bool> IsSoundOn = new(true); public BindableProperty<float> SoundVolume = new(1f); }

    internal class PlayingSoundPoolModel
    {
        internal readonly Dictionary<string, List<AudioPlayer>> Playing = new();
        internal void Add(string name, AudioPlayer p)
        {
            if (Playing.TryGetValue(name, out var l)) l.Add(p);
            else Playing[name] = new List<AudioPlayer> { p };
        }
        internal void Remove(string name, AudioPlayer p) { if (Playing.TryGetValue(name, out var l)) l.Remove(p); }
        public void ForEachAll(Action<AudioPlayer> op)
        {
            var snapshot = Playing.SelectMany(kv => kv.Value).ToList(); // 快照遍历，防遍历中增删
            foreach (var p in snapshot) op(p);
        }
    }

    // ===== 池化播放器，自管进出在播记录 =====
    public class AudioPlayer
    {
        public string AudioName; private bool mPlayed;
        public BindableProperty<float> Volume;
        private static readonly Stack<AudioPlayer> mPool = new();

        public static AudioPlayer Allocate(BindableProperty<float> volume)
        {
            var p = mPool.Count > 0 ? mPool.Pop() : new AudioPlayer();
            p.Volume = volume;
            p.Volume.RegisterWithInitValue(p.OnVolume);  // 音量联动
            return p;
        }
        private void OnVolume(float v) => Console.WriteLine($"  [AudioSource] volume={v}");

        public void Play(string name)
        {
            AudioName = name;
            if (!CanPlay()) { mPlayed = false; return; }
            mPlayed = true;
            Architecture.PlayingSoundPool.Add(name, this);   // 登记
            Console.WriteLine($"▶ play {name}");
            // 模拟立即播放完成
            OnFinished();
        }
        private bool CanPlay() => Architecture.Settings.IsSoundOn.Value;

        private void OnFinished() => Stop();

        public void Stop()
        {
            if (mPlayed) { Architecture.PlayingSoundPool.Remove(AudioName, this); mPlayed = false; } // 注销
            Recycle();
        }
        private void Recycle()
        {
            Volume?.UnRegister(OnVolume); Volume = null; AudioName = null;
            mPool.Push(this);
        }
    }

    // ===== Command =====
    internal class PlaySoundCommand : ICommand
    {
        private readonly string mName;
        public PlaySoundCommand(string name) => mName = name;
        public void SetArchitecture(IArch a) { }
        public void Execute()
        {
            var player = AudioPlayer.Allocate(Architecture.Settings.SoundVolume);
            player.Play(mName);
        }
        public static void Execute(string name) => new PlaySoundCommand(name).Execute();
    }
    internal class StopAllSoundCommand
    {
        public static void Execute() => Architecture.PlayingSoundPool.ForEachAll(p => p.Stop());
    }

    // ===== 门面：无逻辑，全转发 Command =====
    public static class AudioKit
    {
        static AudioKit() => Architecture.AutoInit();
        public static SettingsModel Settings => Architecture.Settings;
        public static void PlaySound(string name) => PlaySoundCommand.Execute(name);
        public static void StopAllSound() => StopAllSoundCommand.Execute();
    }
}
```

## 使用示例

```csharp
using MiniAudioKit;

class Demo
{
    static void Main()
    {
        AudioKit.PlaySound("EnemyDie");
        // [AudioSource] volume=1 / ▶ play EnemyDie

        AudioKit.Settings.SoundVolume.Value = 0.5f; // 后续播放音量联动
        AudioKit.Settings.IsSoundOn.Value = false;
        AudioKit.PlaySound("Click");                // 被 CanPlay 拦截，不播、不登记

        AudioKit.Settings.IsSoundOn.Value = true;
        AudioKit.PlaySound("Click");                // 正常播放
        AudioKit.StopAllSound();                    // 快照遍历停止全部
    }
}
```

## 取舍自检

- ✅ **保留的核心不变量**：模块私有 `Architecture<Architecture>` + 自动初始化使音频子系统自包含；门面无逻辑、全转发 Command；逻辑在 Command、状态在 Model；`AudioPlayer` 池化 + **只在确实播放（mPlayed）时配对登记/注销** `PlayingSoundPool`；音量用 `BindableProperty` 联动 AudioSource；`StopAllSound` 用快照（ToList）遍历防遍历中增删。
- ❌ **砍掉的**：`PlaySoundChannelSystem` 去重通道、Music/Voice 播放器、Unity AudioSource/clip 异步加载（桩）、Fluent API、Action 包装、`AudioLoaderPoolModel`、能力接口（模块内直接静态访问 Model）。
- ⚠️ **最容易搞错的一处**：登记/注销 `PlayingSoundPool` 必须以 `mPlayed` 为条件且严格配对。若音效被 `CanPlay` 拦截（没真正播放）却仍注销，或播放了却忘注销，`Playing` 字典会与现实不符——`StopAllSound` 会漏停或对已回收对象重复操作。其次：`ForEachAll` 必须先 `ToList()` 快照，因为遍历中 `p.Stop()` 会 `Remove` 改字典（与 ResKit/ActionKit 同源的"迭代安全"母题）。
