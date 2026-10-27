# 在小米电视上加入“压低原唱”功能：可行性调研

调研日期：2026-10-28。本文只分析方案，没有修改播放器、构建 APK 或安装到电视。

## 结论

基于**本仓库源码**加入可调的“中央声道衰减”是可行的，适合先做小范围实验。它可以实时削弱通常位于立体声中央的主唱，但也会削弱同样位于中央的鼓、贝斯等乐器，不能等同于 Apple Music Sing 或真正的人声分离。电视上的既有安装包与本仓库源码**不是可直接互相更新的同一构建**；功能开发可行，不代表能原位覆盖安装并保留登录状态。

## 已核实的事实

1. 本仓库的 `BeatAwareRenderersFactory` 已通过 `DefaultAudioSink` 注入 PCM 处理链，顺序为 `GapConcealProcessor → GainProcessor → BeatProcessor`。`PlayerViewModel` 的普通歌曲、淡入淡出伙伴及电台播放器都使用此工厂；其中电台另走安全解码路径，不能预先保证同样的处理器会运行。现有 `GainProcessor` 已展示了 16 位及浮点 PCM 的处理方式。因此，不需要另造播放器，也不应把此功能放在加密流或网络请求层。[处理链](../android/app/src/main/java/com/applemusicktv/media/BeatAwareRenderersFactory.kt) · [播放器构造](../android/app/src/main/java/com/applemusicktv/ui/viewmodel/PlayerViewModel.kt) · [现有 PCM 处理器](../android/app/src/main/java/com/applemusicktv/media/GainProcessor.kt)
2. Android Media3 的音频处理链只处理 PCM，不处理 passthrough/offload 输出。实际歌曲是否经过新增处理器，需要在设备上验证；不能仅凭 UI 开关状态判断。[Media3 文档](https://developer.android.com/reference/androidx/media3/exoplayer/audio/DefaultAudioSink.Builder)
3. 本地留存的电视安装包是 `com.applemusic.lavande`、`3.0-V32`、`versionCode 32`，由 Android Debug 证书签名；本仓库当前 `applicationId` 是 `com.applemusicktv`、`versionCode 3`。[本仓库构建配置](../android/app/build.gradle.kts) 安装包里可见部分 `com.applemusicktv` 类名及 ExoPlayer/FFmpeg 依赖，说明代码有共同来源或继承关系，但这**不能证明**本仓库 `main` 就是 V32 的完整源码。
4. 安卓原位更新要求应用 ID、签名证书及版本号满足更新条件；用自己的密钥构建，即使把应用 ID 改成 `com.applemusic.lavande`，通常也无法覆盖现有 V32。卸载旧包会清掉其应用数据。因此试验版宜先保持不同应用 ID，与现有 V32 并存；用户令牌由用户在试验版中重新输入，不复制、导出或提交旧包里的令牌。[Android 更新规则](https://developer.android.com/google/play/app-updates)

## 建议的最小实现

在本仓库添加独立 `VocalReductionProcessor`，放在 `GainProcessor` 之后、`BeatProcessor` 之前。只对双声道 PCM 生效；单声道、未知编码和开关关闭时原样通过。用一个 0–100% 的“原唱削弱强度”调节中央分量：

```text
M = (L + R) / 2        // 中央分量
S = (L - R) / 2        // 两侧分量
L' = k*M + S
R' = k*M - S          // k=1 原样；k 越小，中央越弱
```

处理器应保持样本数、采样率和播放时间不变，对 16 位及浮点 PCM 做限幅，避免爆音。普通歌曲和淡入淡出播放器各需独立实例，参数共享；切歌、拖动、暂停、淡入淡出时不能重置用户设置。第一版只需一个关闭/开启及强度滑块，默认关闭。FFmpeg 的 `stereotools` 已提供同类 `mlev` K 歌效果，能作为试听对照，但**安装包内含 FFmpeg 解码库**并不表示可以直接把 FFmpeg 命令插入现有 ExoPlayer 播放链。[FFmpeg `stereotools`](https://www.ffmpeg.org/ffmpeg-filters.html)

## 音质与性能边界

- 这是中置削弱，不是分离。居中的乐器会一起变轻；混响、和声、偏置人声可能残留；单声道几乎无效。
- 若试听发现伴奏损失明显，第二阶段可只衰减中央分量的部分频段，再与低频和高频原信号混合。这样能改善部分歌曲，但仍不是独立的人声轨。
- 独立的人声/伴奏分离需要更复杂的模型及算力。对这台 Android 9、32 位、约 1.5 GB 内存的电视，应先验证轻量处理器的 CPU、内存和音画同步，再决定是否探索电脑侧处理。不能在未经实测时承诺电视端实时 AI 分离。
- 苹果公开的 Android MusicKit 播放接口没有提供 Apple Music Sing 的原唱音量滑块或伴奏 stem；本方案只处理播放器最终拿到的立体声音频。[Android MusicKit 播放接口](https://developer.apple.com/musickit/android/com/apple/android/music/playback/controller/MediaPlayerController.html)

## 后续验证门槛

1. 用自制或明确可使用的双声道样本测试：`k=1` 位级或听感与原播放一致，`k<1` 中置声降低，单声道旁路；检查 16 位、浮点 PCM、削波及左右声道。
2. 在**并存的试验版**上测试几首风格不同的歌曲，观察主唱压低程度、鼓/贝斯损失、卡顿与 CPU/内存；检查切歌、拖动、淡入淡出、歌词同步。
3. 若以上通过，再讨论界面、持久化设置以及是否与 V32 的发布方合作提供同签名更新。未经签名和数据迁移方案验证，不替换用户当前已登录的安装包。

## 范围说明

此文档依据本仓库当前源码和本地保存的 V32 APK 静态元数据。没有读取用户令牌、连接电视、修改应用、构建 APK 或运行真实歌曲的音质测试。关于 V32 与本仓库的准确版本关系、电视的具体解码输出和听感效果，仍需后续验证。
