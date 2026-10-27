# 小米电视 KTV 模式：完整链路与可行性

调研日期：2026-10-28。本文针对本仓库源码及用户电视上的 Apple Music TV V32，只做设计与可行性调研；未修改代码、构建 APK、连接或改动电视。目标是边看同步歌词边唱，并可调低原唱、把自己的声音从电视或音响播放。**这不是苹果官方 Apple Music Sing 的移植**。

## 先说结论

| 环节 | 现有基础 | 结论 |
| --- | --- | --- |
| Apple Music 选歌、播放 | 本仓库已有独立播放链路；V32 也已在电视上播放成功 | 可沿用，先核对两版功能差异 |
| 逐字歌词、大屏唱歌界面 | 已有逐字高亮和全屏歌词 | 可复用，补遥控器 KTV 控件即可 |
| 实时调低原唱 | 已有 PCM 音频处理链，但没有原唱控制 | 可做轻量“中置削弱”，效果因歌曲而异 |
| USB 麦克风采集 | 源码和 V32 均未申请录音权限 | 软件可做；电视是否识别具体麦克风待实测 |
| 人声与伴奏混音 | 现有代码无麦克风监听输出 | 软件可做；这台电视的延迟、路由与啸叫是主要门槛 |
| 混响、音量、静音 | 无 KTV 麦克风控制 | 可选，先完成干声监听与限幅 |
| 官方 Sing 人声分轨及 iPhone 麦克风协议 | 未见公开的 Android Sing 控制接口 | 不以此为目标或依赖 |

## 当前代码与电视安装包的边界

- [处理链](../android/app/src/main/java/com/applemusicktv/media/BeatAwareRenderersFactory.kt)把 `GapConcealProcessor → GainProcessor → BeatProcessor` 接到 Media3 `DefaultAudioSink`。普通歌曲和淡入淡出伙伴在 [PlayerViewModel](../android/app/src/main/java/com/applemusicktv/ui/viewmodel/PlayerViewModel.kt) 中各建一个播放器；这提供了歌曲**解码后 PCM**的插入点。电台另走安全解码路径，不能预先保证会执行相同效果。[Media3 官方文档](https://developer.android.com/reference/androidx/media3/exoplayer/audio/DefaultAudioSink.Builder)也限定音频处理链用于 PCM，不用于直通或卸载播放。
- [NowPlayingScreen](../android/app/src/main/java/com/applemusicktv/ui/screens/NowPlayingScreen.kt)已有全屏 `FullScreenLyrics` 和逐字高亮的 `LyricsPanel`。歌词有现成的播放进度与偏移设置；KTV 界面应复用同一时钟和歌词模型，不另开独立计时器。无同步歌词的歌曲需要明确提示与普通播放退路。
- 本仓库 [AndroidManifest.xml](../android/app/src/main/AndroidManifest.xml)没有 `RECORD_AUDIO`。本地留存的 V32 APK 也没有该权限，所以两者目前都没有完整麦克风输入链路。
- 电视上的 V32 是 `com.applemusic.lavande`、`versionCode 32`；本仓库 [构建配置](../android/app/build.gradle.kts)是 `com.applemusicktv`、`versionCode 3`。V32 APK 内可见部分同源类名和 ExoPlayer/FFmpeg 依赖，但不能证明本仓库就是 V32 的完整源码。用户接受最终卸载 V32、安装新包并重新登录；无损覆盖安装不是前提。试听阶段可保留不同包名并存。[Android 更新规则](https://developer.android.com/google/play/app-updates)

## 推荐的音频链路

```text
Apple Music 歌曲 → 现有播放器解码 → 歌曲 PCM → 原唱削弱（可关闭）
                                          → 现有音量均衡/节拍分析 → 伴奏输出 ┐
                                                                        ├→ Android 音频混音 → 电视/有线音响
USB 麦克风 → AudioRecord → 输入增益/高通/限幅 → 低延迟 AudioTrack ──────────┘
                         （后续可选轻混响）
同一首歌的播放进度 → 现有全屏逐字歌词
```

**先让麦克风走独立输出**，由 Android 音频系统混合两条音轨。Android 允许多条音轨同时输出并混合。[音频焦点说明](https://developer.android.com/media/optimize/audio-focus) 这样麦克风不依附于 ExoPlayer 的歌曲缓冲、暂停和双播放器淡入淡出。若把麦克风直接混进每个歌曲 `AudioProcessor`，淡入淡出时可能从两个播放器重复放大麦克风；切歌或缓冲时也可能中断监听。独立输出仍须实测路由与延迟，不能据此保证 KTV 体验。进阶方案是统一混音输出，但需要重构现有播放器输出，暂不作为第一版。

## 各环节的具体方案

### 1. 选歌、歌词与电视交互

保留原有选歌和同步歌词。KTV 模式可从 Now Playing 或全屏歌词进入，遥控器能操作：原唱削弱强度、麦克风开关/音量、歌词偏移、退出 KTV。歌词继续跟随歌曲进度；麦克风回放延迟另测，不能通过改歌词偏移来掩盖自己声音的延迟。歌曲缓冲、暂停、切歌和退出时，麦克风状态必须明确，避免画面已退出仍在监听。苹果官方 Sing 提供的可调原唱和 iPhone 麦克风是额外能力，并非现有全屏歌词自动具备。[苹果 Sing 功能说明](https://support.apple.com/en-us/127940)

### 2. 压低原唱或得到伴奏

在现有歌曲 PCM 链增加可关闭的 `VocalReductionProcessor`，对双声道做中置衰减。设 `M=(L+R)/2`、`S=(L-R)/2`，输出 `L'=kM+S`、`R'=kM-S`；`k=1` 原样，`k` 越小中央越弱。处理器只改变采样值，不改变样本数或播放时间；处理 16 位及浮点 PCM，单声道和不支持的编码旁路，防止削波。普通与淡入淡出播放器各有处理器实例、共享控制值。FFmpeg 的 [`stereotools` K 歌示例](https://www.ffmpeg.org/ffmpeg-filters.html)可用于离线听感对照。

这是**立体声中置削弱，不是真正人声分离**：居中的鼓和贝斯也会变轻，偏置人声、和声和混响会残留，单声道几乎无效。若试听损伤明显，第二阶段可试只衰减中置的部分频段；仍不能承诺每首歌有纯净伴奏。实时 AI 分离不是这台 Android 9、32 位、约 1.5 GB 内存电视的首选，需单独做资源和实时性验证。公开 Android MusicKit 播放接口没有提供可供本项目调用的 Sing 原唱滑块或伴奏 stem；本项目也并非直接使用官方 MusicKit 播放。[Android MusicKit 播放接口](https://developer.apple.com/musickit/android/com/apple/android/music/playback/controller/MediaPlayerController.html)

### 3. 麦克风接入与权限

第一版优先试**有线 USB 音频麦克风或 USB 音频接口**。Android 支持部分 USB Audio Class 设备，但厂商固件和具体外设仍可能不兼容；USB 接入还可能把**输出**自动切到 USB 设备，必须核对电视扬声器/音响是否仍在出声。[AOSP USB 音频说明](https://source.android.com/docs/core/audio/usb) 启动时枚举输入设备；实际开始采集后，用 `AudioRecord.getRoutedDevice()`核实真实路由，而不只看用户选择或 `setPreferredDevice()` 的返回值。[AudioRecord 路由接口](https://developer.android.com/reference/android/media/AudioRecord)

新增 `RECORD_AUDIO` 清单权限，并在用户首次明确开启麦克风时请求运行时授权；拒绝时歌曲和歌词仍可正常使用。仅在前台 KTV 模式采集，不默认录音、上传或保存人声；退出、切到后台或权限撤销时停止并释放输入。Android 9 起后台麦克风使用受限制。[录音权限与后台限制](https://developer.android.com/media/platform/mediarecorder) 这台 Android 9 **不能用**专为实时演唱设计的 `VOICE_PERFORMANCE` 输入源，它从 Android 10/API 29 才有。可检测 `UNPROCESSED` 支持情况并试听；不支持时比较 `VOICE_RECOGNITION` 与普通 `MIC`，避免默认语音通话处理把歌声压坏。[音源 API 级别](https://developer.android.com/reference/android/media/MediaRecorder.AudioSource)

蓝牙麦克风或手机经局域网当麦克风可作为后续探索，但涉及额外编解码、无线延迟及设备路由，不能套用苹果的 iPhone Continuity Mic 协议。第一版不依赖它们。

### 4. 麦克风处理、输出与防啸叫

建议先做 `AudioRecord → 输入增益 → 低频噪声过滤 → 限幅 → AudioTrack`，把干声稳定放出来，再考虑少量混响。麦克风音量、伴奏音量、原唱削弱是三个独立控制量；不能只用系统总音量。麦克风回放要避免削波、突发大音量和无输入时的底噪。自动音量、噪声门、回声消除都应经过真实听感验证；Android 的 `AcousticEchoCanceler` 原本针对通话回声，不保证适合唱歌，尤其可能改变歌声尾音。[AEC 接口说明](https://developer.android.com/reference/android/media/audiofx/AcousticEchoCanceler)

电视扬声器会被麦克风再次拾取，形成回声或啸叫。第一轮用低麦克风增益、麦克风远离扬声器测试；若必须外放且软件回声控制效果差，外置 KTV 混音器/音频接口是更稳妥的硬件路径。此路径可让电视只负责伴奏和歌词，麦克风在电视音频链路之后混入，但需确认电视可用的音频输出接口和现有音响接法。

### 5. 延迟和同步

最关键的指标是**从张嘴唱到自己听到回放的人声延迟**，另测歌词相对伴奏的偏移。输入、缓冲、输出、电视画面/音效处理和蓝牙设备都可能增加延迟；不同采集/播放设备的时钟也可能漂移。Android 9 可使用 `AudioTrack` 的低延迟模式或 Oboe/AAudio，但它们只是请求低延迟，实际效果由电视硬件和路由决定。[Android 低延迟指南](https://developer.android.com/games/sdk/oboe/low-latency-audio) · [AudioTrack 低延迟模式](https://developer.android.com/reference/android/media/AudioTrack)

先用有线输入/输出测实际往返延迟与卡顿，不预设毫秒数。若电视提供游戏模式、音效处理开关或 AV Sync Bypass，可分别试听和测量；苹果官方对自己的 Sing 麦克风模式也专门提示这些电视处理会造成延迟，说明这是整条链路问题。[Apple 延迟排查](https://support.apple.com/en-ca/123880) 调整歌词偏移只解决文字与伴奏的观感，**不能**修复麦克风监听过慢。录音测试材料只留本地，测试结束删除。

## 分阶段实施与验收

1. **只读设备检查**：列出电视的音频输入/输出设备、USB 麦克风是否被识别、真实录音路由及采样格式。确认 USB 插入后是否夺走电视音频输出。没有可用输入时，不启动软件麦克风实现。
2. **歌词与伴奏原型**：复用全屏歌词；仅加入原唱削弱开关/滑块。用自制双声道测试素材验证 `k=1` 原样、`k<1` 中置降低、单声道旁路；再试听多首不同混音风格的歌。测试切歌、拖动、淡入淡出、歌词同步和 CPU/内存。
3. **麦克风单独回放原型**：按需请求权限，用 USB 麦克风完成干声监听；测真实输入/输出设备、延迟、底噪、啸叫及长时间运行是否卡顿。没有伴奏时先让人声链路稳定。
4. **与歌曲同播**：系统混合伴奏和人声；测伴奏/麦克风各自音量、缓冲、暂停、切歌、淡入淡出、输出设备拔插、返回桌面和重新进入。麦克风不能因两个 ExoPlayer 同时存在而重复播放。
5. **可选体验增强**：小幅混响、设备选择、延迟提示、歌词大字布局。只有前四阶段稳定后再考虑手机麦克风或更重的人声分离。
6. **最终替换**：核对新版本与 V32 的常用功能，备好重新登录方式后卸载 V32、安装新包。用户已接受完整重装；不复制旧应用的登录令牌。本文发布的是调研，不代表上述测试已完成。

## 尚未实测、会改变方案的问题

- 这台电视和用户现有麦克风/音响的具体接口、USB 音频兼容性、输入及输出路由。
- 电视是否提供足够低延迟的监听路径，外放时是否会啸叫；蓝牙输出会怎样影响同步。
- V32 与本仓库当前版本在登录、音乐库、歌词、设置上的实际差异。
- 歌曲是否稳定以双声道 PCM 经过新增处理器，电台和视频是否需要旁路。

## 隐私与测试范围

本轮仅查阅源码、已保存 APK 的静态元数据及官方文档；没有读取 Music-User-Token、捕获麦克风音频、连接电视、构建 APK 或运行真人演唱测试。后续麦克风权限须由用户在试验版中主动授予，原型默认不保存或上传声音。
