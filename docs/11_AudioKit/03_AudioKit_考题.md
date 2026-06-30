# 11 · AudioKit 考题

## 🟢 概念题（5）

1. AudioKit 与其他 Kit 最本质的结构差异是什么？
2. `AudioKit` 门面里的 `PlayMusic`/`PlaySound` 自己实现播放逻辑吗？
3. `PlayingSoundPoolModel` 记录什么？数据结构是什么？
4. `AudioPlayer` 为什么实现 `IPoolable`？
5. AudioKit 的音量是怎么和 AudioSource、UI 联动的？

<details><summary>参考答案要点</summary>

1. AudioKit 完整构建在 CoreArchitecture 之上——有自己的 `Architecture<Architecture>` + Model/System/Command，是 QFramework 架构范式的范例；其他 Kit 多是独立工具或仅用底座工具类。
2. 不。门面方法全部转发给对应 Command（如 `PlayMusicCommand.Execute(...)`），逻辑在 Command 里。
3. 当前正在播放的音效，按音效名分组：`Dictionary<string, List<AudioPlayer>>`。
4. 音效高频播放，AudioPlayer 池化复用避免频繁 new；`OnRecycled`→`OnDeinit` 清理状态。
5. `Settings` 的音量是 `BindableProperty<float>`；播放器 `OnInit` 里 `Volume.RegisterWithInitValue(AudioSourceProxy.SetVolume)` 把音量变更同步到 AudioSource；UI 滑条 `onValueChanged` 写 `Volume.Value`、`RegisterWithInitValue` 读初值——双向绑定。
</details>

## 🟡 机制题（6）

1. AudioKit 的架构是怎么初始化的？什么时候？
2. 模块内部访问 Model 为什么直接 `Architecture.PlayingSoundPoolModel` 而不走 `this.GetModel<>()`？
3. `AudioPlayer` 在什么时机把自己加入/移出 `PlayingSoundPoolModel`？条件是什么？
4. `AbstractAudioPlayer` 组合了哪三个内部对象？体现什么设计？
5. `PlayingSoundPoolModel.ForEachAllSound` 为什么要用 `ListPool` 拷贝快照？
6. `AudioPlayer.OnPlayFinished` 对 loop 和非 loop 的处理差异？

<details><summary>参考答案要点</summary>

1. `Architecture` 的 `[RuntimeInitializeOnLoadMethod(BeforeSceneLoad)]` 标记的 `AutoInit` 在场景加载前自动调 `InitArchitecture()`，注册并初始化 3 Model + 2 System。
2. 因为 AudioKit 的架构是单一确定的（`Architecture<Architecture>` 单例），Model/System 都是静态属性，直接访问省去能力接口的间接层；能力接口（`ICanGetModel` 等）主要用于"上层对象不知道具体架构"的场景，模块内部不需要。
3. `OnPlayStarted` 里加入（`AddSoundPlayer2Pool`）、`OnBeforeStop` 里移出（`RemoveSoundPlayerFromPool`），且都以 `mPlayedAudio==true`（确实播放了）为条件。
4. `AudioSourceProxy`（封装 Unity AudioSource）、`AudioPlayerLifeCycle`（OnStart/OnFinish 一次性回调）、`ClipPrepareModeController`（clip 准备策略）。体现"组合派生"——功能拆成可组合 feature 而非塞进大类。
5. 因为遍历过程中对每个 player 的操作（如 Stop）会 `RemoveSoundPlayerFromPool` 修改字典；先用 ListPool 拷贝快照再遍历，避免"遍历中修改集合"崩溃。
6. 非 loop：`LifeCycle.CallOnFinishOnce()` + 通知 System `SoundFinish` + `Stop()`（回收）；loop：不触发结束、不回收，继续循环播放。
</details>

## 🔴 架构陷阱题（7）

1. **母题集大成对比**：AudioKit 同时用到了 CoreArchitecture（MVC）、PoolKit（AudioPlayer 池化）、BindableKit（音量）、"快照遍历"（ForEachAllSound）。请说明这是如何印证"每个 Kit 是同一套设计直觉的投影"的。
2. AudioKit 的 `Architecture` 实例和游戏主架构 `GameArchitecture` 是同一个吗？为什么这样设计？
3. 某音效被去重通道拦截没真正播放（`mPlayedAudio=false`），但代码错误地仍调用了 `AddSoundPlayer2Pool`。后果？
4. `StopAllSound` 遍历 `SoundPlayerInPlaying` 时直接 `foreach` 并在回调里 `Stop`（不拷快照）。会发生什么？
5. AudioPlayer 从池中复用，但 `OnRecycled`/`OnDeinit` 忘了 `Volume.UnRegister`。会有什么累积问题？
6. `[RuntimeInitializeOnLoadMethod]` 的 AutoInit 如果改成"首次 PlaySound 时才初始化"，会有什么时序风险？
7. 为什么 AudioKit 把"是否能播放"（`CanPlayAudio`）拆成 System（去重通道）+ Model（IsSoundOn）两处判断，而不是写死在 Command 里？

<details><summary>参考答案要点</summary>

1. 同一个真实功能（播放音效）里：MVC 提供"门面→命令→模型"的解耦骨架、PoolKit 解决播放器高频创建的 GC、BindableKit 让音量变更自动联动、快照遍历解决批量停止时的迭代安全。每个底座解决一个正交问题，组合成完整功能——印证"底座是直觉、业务是组装"。
2. 不是。AudioKit 有自己独立的 `Architecture<Architecture>`（CRTP 每类型一实例）。这样设计让音频子系统自包含、可独立初始化（RuntimeInitializeOnLoad）、不与游戏主架构耦合——库可被任意项目直接使用。
3. `Playing` 字典记录了一个实际没在播放的 player。`StopAllSound` 会对它 Stop（可能重复 Stop 或操作已回收对象），且去重通道/计数会把它算作"在播"，导致后续同名音效误判被忽略。登记必须与实际播放配对。
4. 遍历中 `Stop`→`RemoveSoundPlayerFromPool` 修改正在遍历的字典/列表，抛"集合已修改"异常。必须先 `ListPool` 拷贝快照（框架的做法）。
5. 每次 Allocate 都 `RegisterWithInitValue` 给 Volume 加一个回调，回收不 UnRegister 的话，Volume 的事件链会累积越来越多指向已回收 player 的回调——内存泄漏 + 音量变更时触发一堆失效回调（操作已回收对象的 AudioSource）。
6. 风险：若某段早期代码（如另一个 RuntimeInitializeOnLoad 或场景 Awake）在首次 PlaySound 之前就访问 `AudioKit.Settings`，此时架构未初始化，Model 为空/未注册，拿到 null 或未初始化状态。BeforeSceneLoad 自动初始化保证"任何业务代码运行前架构已就绪"。
7. 拆分让"去重策略"（System，可配置 EveryOne/同帧忽略等）与"全局开关"（Model 的 BindableProperty，可被 UI 绑定）各自独立演化、可被外部配置/观察；写死在 Command 里则无法复用去重逻辑、无法让 UI 绑定开关、违反单一职责。
</details>

## ✍️ 实操题（3）

1. 用 AudioKit 实现"设置面板的音乐开关 + 音量滑条"，要求开关和滑条与 `AudioKit.Settings` 双向绑定，并解释 `RegisterWithInitValue` 在这里的作用。
2. 解释一次 `AudioKit.PlaySound("Hit")` 从门面到播放器回收的完整调用链（点出 Command/Model/Pool 各自的角色）。
3. 设计一个"同一帧内最多播放一次同名音效"的需求，说明应该用 AudioKit 的哪个机制实现，以及为什么这个判断放在 System 而非 Command。

<details><summary>参考答案要点</summary>

1. `musicToggle.onValueChanged.AddListener(v=>AudioKit.Settings.IsMusicOn.Value=v); AudioKit.Settings.MusicVolume.RegisterWithInitValue(v=>volumeSlider.value=v); volumeSlider.onValueChanged.AddListener(v=>AudioKit.Settings.MusicVolume.Value=v);`。`RegisterWithInitValue` 让 UI 一注册就拿到当前音量初值（滑条立即显示正确位置），而非等下次变更。
2. `AudioKit.PlaySound("Hit")`（门面）→ `PlaySoundCommand.Execute`（命令，编排流程）→ `AudioPlayer.Allocate`（PoolKit 复用播放器）→ 加载 clip → `CanPlayAudio`（System 去重 + Model 开关判断）→ `OnPlayStarted` 把自己登记进 `PlayingSoundPoolModel`（Model 记录状态）→ 播放 → 完成 `OnPlayFinished` → `Stop`→`OnBeforeStop` 从 Model 注销 → `Recycle2Cache` 还池。
3. 用 `PlaySoundMode = IgnoreSameSoundInGlobalFrames`（或 SoundFrames）+ 对应的 `PlaySoundChannelSystem` 去重通道。放在 System 是因为：去重是跨多次播放请求的全局策略（需维护"上次播放帧"状态），是可复用、可配置、可被多个 Command 共享的逻辑；放 Command 会重复实现且无法共享去重状态。
</details>
