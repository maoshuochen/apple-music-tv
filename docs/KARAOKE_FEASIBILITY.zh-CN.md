# 小米电视 KTV 模式：完整链路与可行性

调研日期：2026-10-28。本文针对本仓库源码及用户电视上的 Apple Music TV V32，只做设计与可行性调研；未修改代码、构建 APK、连接或改动电视。目标是边看同步歌词边唱，并可调低原唱、把自己的声音从电视或音响播放。**这不是苹果官方 Apple Music Sing 的移植**。

## 先说结论

| 环节 | 现有基础 | 结论 |
| --- | --- | --- |
| Apple Music 选歌、播放 | 本仓库已有独立播放链路；V32 也已在电视上播放成功 | 可沿用，先核对两版功能差异 |
| 逐字歌词、大屏唱歌界面 | 已有逐字高亮和全屏歌词 | 可复用，补遥控器 KTV 控件即可 |
| 实时调低原唱 | 已有 PCM 音频处理链，但没有原唱控制 | 可做轻量“中置削弱”，效果因歌曲而异 |
| USB 麦克风采集 | 源码和 V32 均未申请录音权限 | 软件可做；电视是否识别具体麦克风待实测 |
| 安卓/iPhone 扫码当麦克风 | 现有手机网页只有 HTTP 设置服务，没有实时音频接收端 | 可设计独立的 HTTPS 配对页 + WebRTC；需在电视上实测延迟和 32 位兼容性 |
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

**先让麦克风走独立输出**，由 Android 音频系统混合两条音轨。Android 允许多条音轨同时输出并混合。[音频焦点说明](https://developer.android.com/media/optimize/audio-focus) 这样麦克风不依附于 ExoPlayer 的歌曲缓冲、暂停和双播放器淡入淡出。若把麦克风直接混进每个歌曲 `AudioProcessor`，淡入淡出时可能从两个播放器重复放大麦克风；切歌或缓冲时也可能中断监听。但此方案必须使用**可与歌曲共享的输出路径**：AAudio/Oboe 的独占模式占用输出设备，不能同时假定 ExoPlayer 还能在同一设备混音。[AAudio 共享模式说明](https://developer.android.com/ndk/guides/audio/aaudio/aaudio) 独立输出仍须实测路由与延迟，不能据此保证 KTV 体验。若共享混音达不到延迟要求，优先评估电视音频输出之后的外置混音器；统一软件混音则需重构播放器输出。

## 各环节的具体方案

### 1. 选歌、歌词与电视交互

保留原有选歌和同步歌词。KTV 模式可从 Now Playing 或全屏歌词进入，遥控器能操作：原唱削弱强度、麦克风开关/音量、歌词偏移、退出 KTV。歌词继续跟随歌曲进度；麦克风回放延迟另测，不能通过改歌词偏移来掩盖自己声音的延迟。歌曲缓冲、暂停、切歌和退出时，麦克风状态必须明确，避免画面已退出仍在监听。苹果官方 Sing 提供的可调原唱和 iPhone 麦克风是额外能力，并非现有全屏歌词自动具备。[苹果 Sing 功能说明](https://support.apple.com/en-us/127940)

**现有歌词时钟与切歌有两个具体缺口。** [PlayerViewModel](../android/app/src/main/java/com/applemusicktv/ui/viewmodel/PlayerViewModel.kt) 的淡入淡出默认开启，[默认时长](../android/app/src/main/java/com/applemusicktv/data/CrossfadePreferences.kt)为 7 秒；淡入淡出一开始就把 `currentSong` 和歌词切到下一首，而旧歌还会继续播放。KTV 第一版应在进入模式时暂停淡入淡出，并在退出时恢复用户原设置，否则唱到结尾会提前跳词。`PlayerViewModel` 还把 Fire TV 实测的歌词基线写死为 200 ms，把蓝牙补偿估为 350 ms；蓝牙判断依据是“存在已连接输出设备”，不等于当前歌曲实际路由。小米电视的有线/扬声器/蓝牙输出须分别测量，按**真实路由**校准并允许用户微调。歌词、伴奏和麦克风返听是三个不同时间轴，不能共用一个偏移值。

### 2. 压低原唱或得到伴奏

在现有歌曲 PCM 链增加可关闭的 `VocalReductionProcessor`，对双声道做中置衰减。设 `M=(L+R)/2`、`S=(L-R)/2`，输出 `L'=kM+S`、`R'=kM-S`；`k=1` 原样，`k` 越小中央越弱。处理器只改变采样值，不改变样本数或播放时间；处理 16 位及浮点 PCM，单声道和不支持的编码旁路，防止削波。普通与淡入淡出播放器各有处理器实例、共享控制值。FFmpeg 的 [`stereotools` K 歌示例](https://www.ffmpeg.org/ffmpeg-filters.html)可用于离线听感对照。

这是**立体声中置削弱，不是真正人声分离**：居中的鼓和贝斯也会变轻，偏置人声、和声和混响会残留，单声道几乎无效。若试听损伤明显，第二阶段可试只衰减中置的部分频段；仍不能承诺每首歌有纯净伴奏。实时 AI 分离不是这台 Android 9、32 位、约 1.5 GB 内存电视的首选，需单独做资源和实时性验证。公开 Android MusicKit 播放接口没有提供可供本项目调用的 Sing 原唱滑块或伴奏 stem；本项目也并非直接使用官方 MusicKit 播放。[Android MusicKit 播放接口](https://developer.apple.com/musickit/android/com/apple/android/music/playback/controller/MediaPlayerController.html)

### 3. 麦克风接入与权限

第一版优先试**有线 USB 音频麦克风或 USB 音频接口**。Android 支持部分 USB Audio Class 设备，但厂商固件和具体外设仍可能不兼容；USB 接入还可能把**输出**自动切到 USB 设备，必须核对电视扬声器/音响是否仍在出声。[AOSP USB 音频说明](https://source.android.com/docs/core/audio/usb) 启动时枚举输入设备；实际开始采集后，用 `AudioRecord.getRoutedDevice()`核实真实输入路由，并核对歌曲及返听的实际输出路由，而不只看用户选择或 `setPreferredDevice()` 的返回值。[AudioRecord 路由接口](https://developer.android.com/reference/android/media/AudioRecord) 同时确认电视 USB 口能供电、麦克风插拔后可恢复；这些仍未在用户设备上实测。

新增 `RECORD_AUDIO` 清单权限，并在用户首次明确开启麦克风时请求运行时授权；拒绝时歌曲和歌词仍可正常使用。仅在前台 KTV 模式采集，不默认录音、上传或保存人声；退出、切到后台或权限撤销时停止并释放输入。Android 9 起后台麦克风使用受限制。[录音权限与后台限制](https://developer.android.com/media/platform/mediarecorder) 这台 Android 9 **不能用**专为实时演唱设计的 `VOICE_PERFORMANCE` 输入源，它从 Android 10/API 29 才有。可检测 `UNPROCESSED` 支持情况并试听；不支持时比较 `VOICE_RECOGNITION` 与普通 `MIC`，避免默认语音通话处理把歌声压坏。[音源 API 级别](https://developer.android.com/reference/android/media/MediaRecorder.AudioSource)

蓝牙麦克风涉及额外设备路由。安卓/iPhone 手机经局域网当麦克风可以作为 **USB 之外的独立输入路线**，详见下文扫码方案；不能套用苹果的 iPhone Continuity Mic 协议。

### 4. 麦克风处理、输出与防啸叫

建议先做 `AudioRecord → 输入增益 → 低频噪声过滤 → 限幅 → AudioTrack`，把干声稳定放出来，再考虑少量混响。麦克风音量、伴奏音量、原唱削弱是三个独立控制量；不能只用系统总音量。麦克风回放要避免削波、突发大音量和无输入时的底噪。自动音量、噪声门、回声消除都应经过真实听感验证；Android 的 `AcousticEchoCanceler` 原本针对通话回声，不保证适合唱歌，尤其可能改变歌声尾音。[AEC 接口说明](https://developer.android.com/reference/android/media/audiofx/AcousticEchoCanceler)

电视扬声器会被麦克风再次拾取，形成回声或啸叫。第一轮用低麦克风增益、麦克风远离扬声器测试；若必须外放且软件回声控制效果差，外置 KTV 混音器/音频接口是更稳妥的硬件路径。此路径可让电视只负责伴奏和歌词，麦克风在电视音频链路之后混入，但需确认电视可用的音频输出接口和现有音响接法。

### 5. 延迟和同步

最关键的指标是**从张嘴唱到自己听到回放的人声延迟**，另测歌词相对伴奏的偏移。输入、缓冲、输出、电视画面/音效处理和蓝牙设备都可能增加延迟；不同采集/播放设备的时钟也可能漂移。Android 9 可使用 `AudioTrack` 的低延迟模式或 Oboe/AAudio，但它们只是请求低延迟，实际效果由电视硬件和路由决定。[Android 低延迟指南](https://developer.android.com/games/sdk/oboe/low-latency-audio) · [AudioTrack 低延迟模式](https://developer.android.com/reference/android/media/AudioTrack)

官方最低延迟建议包含独占输出；**本方案需要歌曲和麦克风同时输出，不能直接照搬独占设置**。使用共享输出、设备原生采样率和小缓冲分别测试；缓冲太小出现断音时要以稳定为先。输入与输出若使用不同设备时钟，长时间监听还须检查缓冲积压/耗尽及漂移，必要时做重采样或缓冲调节。[AAudio 共享模式](https://developer.android.com/ndk/guides/audio/aaudio/aaudio) · [Android 音频格式建议](https://developer.android.com/reference/android/media/AudioFormat)

先用有线输入/输出测实际往返延迟与卡顿，不预设毫秒数。若电视提供游戏模式、音效处理开关或 AV Sync Bypass，可分别试听和测量；苹果官方对自己的 Sing 麦克风模式也专门提示这些电视处理会造成延迟，说明这是整条链路问题。[Apple 延迟排查](https://support.apple.com/en-ca/123880) 调整歌词偏移只解决文字与伴奏的观感，**不能**修复麦克风监听过慢。录音测试材料只留本地，测试结束删除。

## 完整链路复查：目前的失效点与退出条件

| 失效点 | 现有代码/方案暴露的风险 | 原型阶段的退出条件 |
| --- | --- | --- |
| 共享混音与返听延迟 | 两个独立输出可以混音，但低延迟独占输出与此冲突；电视还可能额外处理扬声器音频 | 用同一首歌同时测伴奏、返听及断音；若共享输出唱起来仍明显拖拍，转外置混音器方案，不继续堆软件效果 |
| USB 接入与真实路由 | USB 麦克风可能同时成为输出；“已连接”不等于正在被使用 | 记录采集、歌曲和返听各自的实际设备；拔插后自动静音并提示，不能误把人声送入错误设备 |
| 音频焦点 | 主播放器和淡入淡出播放器都在 `setAudioAttributes(..., false)` 中关闭了 ExoPlayer 自动焦点处理；新增返听流若单独抢焦点，可能压低伴奏 | 设计一个歌曲与返听共用的焦点策略；电话/语音助手/HDMI 切换后验证双方一起暂停或恢复，绝不留下单独持续监听的麦克风。[Media3 焦点参数](https://developer.android.com/reference/androidx/media3/exoplayer/ExoPlayer.Builder) |
| 状态与隐私 | [MainActivity.onStop](../android/app/src/main/java/com/applemusicktv/MainActivity.kt) 会暂停普通音乐，但 PiP 可继续播放；仅跟随歌曲 `isPlaying` 无法覆盖 Home、权限撤销、USB 拔出等事件 | 将麦克风采集设成独立状态机：用户主动开启且 KTV 前台可见时才采集；Home、HDMI、PiP、权限丢失、拔出设备、退出 KTV 时立即停止并释放，返回后不自动重新打开 |
| 歌词和切歌 | 默认跨曲淡入淡出提前切歌词；固定 Fire TV/蓝牙偏移在小米不可靠 | KTV 模式先关闭淡入淡出，实测小米每条输出路径的文字偏移；切歌、拖动、暂停后不跳错行 |
| 长时间稳定性 | USB 输入与电视输出可能用不同采样时钟；小缓冲会断音，电视内存也紧张 | 至少连续唱数首歌并切换输出，记录延迟是否增长、缓冲错误、CPU/内存和界面卡顿；这些指标尚无实测结果 |
| 内容范围与替换 | PCM 处理链不保证覆盖安全解码、电台或视频；V32 与 fork 并非已验证的同一版本 | 初版只承诺普通双声道歌曲；无歌词、单声道、电台/视频要明确旁路；另做 V32 常用功能对照后才考虑替换 |

防啸叫还有一个硬件边界：电视扬声器外放时，麦克风可能再次收进伴奏与自己的返听。降低增益、调整摆位、试听 AEC 只能作为实验；若无法在用户常用音量下稳定工作，应要求有线耳机/音响或外置混音器，而不是默认开启麦克风返听。软件原型必须先提供一键静音与音量上限。

## 扫码让安卓或 iPhone 充当麦克风

**结论：可以做通用扫码配对，但不是复用苹果 Sing 的 iPhone 协议；手机麦克风经无线网络、电视解码和混音后的延迟仍是验收门槛。** 优先做手机浏览器原型，让 Android Chrome 与 iPhone Safari 共用一套界面；若 iPhone 锁屏/切后台、音质或延迟不能满足唱歌，再评估原生手机 App。[Safari 的 WebRTC 麦克风支持](https://webkit.org/blog/7763/a-closer-look-into-webrtc/) · [WebRTC 连接与信令](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Signaling_and_video_calling)

```text
电视 KTV 页：生成一次性会话和二维码 ──→ 手机 HTTPS 配对页
电视与手机：通过 HTTPS/WSS 信令服务交换配对确认、SDP/ICE；服务不接收音频
手机：用户点击“开始麦克风” → 浏览器申请权限 → 音频 WebRTC 上行
电视：WebRTC 接收/解码 → 独立返听音轨（共享输出）→ 与歌曲在系统输出混合
```

1. **配对与安全**：电视生成短时、随机、一次性会话码；二维码指向 HTTPS 配对页及会话标识，电视屏幕再显示确认信息。配对页不能读写 Apple Music 登录令牌；手机可主动静音/退出，电视可踢除并撤销会话。手机与电视通过信令交换 WebRTC 参数，**音频优先只在同一局域网内点对点传输**；若路由器开启设备隔离或 ICE 不能直连，首版报错，不悄悄改走互联网 TURN 中继。[WebRTC ICE 说明](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/RTCPeerConnection)
2. **HTTPS 是浏览器方案的前提**：手机浏览器的 `getUserMedia()` 要求安全上下文和用户授权。现有电视 [InAppWebServer](../android/app/src/main/java/com/applemusicktv/media/InAppWebServer.kt) 只提供 `http://电视IP:8080`，手机访问这一局域网 HTTP 地址不能当成 HTTPS/`localhost` 安全上下文。其页面还提供令牌设置/清除与日志等接口，不能直接改作扫码麦克风入口。最易验证的原型是**独立 HTTPS 手机页面 + 只传配对消息的 WSS 服务**，电视主动连该服务；网络断开时无法新配对，但已建立的局域网音频是否继续需实测。完全离线方案需要电视提供手机信任的本地 HTTPS 服务，或分别安装原生手机 App，复杂度明显更高。[浏览器麦克风安全上下文](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia) 当前电视 IP 是否为 `.104` 不应写进二维码的长期配置，优先用会话码寻址。
3. **电视端与 32 位兼容性**：现有 App 没有 WebRTC 接收器。需要引入或构建支持 `armeabi-v7a`、Android 9 的原生 WebRTC 音频接收依赖，检查 APK 大小、ABI、内存及 CPU；[WebRTC Android 文档](https://webrtc.github.io/webrtc-org/native-code/android/)说明 32 位 ARM 构建路径，但**不等于**本项目已找到合用的现成依赖或已在这台电视运行。收到的麦克风音频要走与歌曲可共享的输出路径，不能为追求低延迟独占音频设备。电视端负责返听音量、限幅和连接中断时立即静音。
4. **手机端录音与使用方式**：扫码后仍须在页面主动点“开始麦克风”并同意权限；只上行麦克风，不把伴奏回传手机，避免手机扬声器额外回声。先比较 `echoCancellation`、`noiseSuppression`、`autoGainControl` 开/关的真实唱歌音色，并用 `getSettings()`确认浏览器实际采用的设置；这些语音通话处理不保证适合唱歌。由于伴奏从电视外放、没有在手机本地播放，手机浏览器的回声消除也未必有可用的伴奏参考信号，不能指望它消除电视漏音。手机要保持页面前台、屏幕亮着；iPhone 浏览器切后台或锁屏后的采集可靠性不能预先保证，须实测。[音频约束与实际设置](https://developer.mozilla.org/en-US/docs/Web/API/MediaStreamTrack/getSettings) · [WebKit 后台采集问题记录](https://bugs.webkit.org/show_bug.cgi?id=204681)
5. **延迟与验收**：测“手机前唱/拍手 → 电视扬声器听到”的端到端延迟和抖动，而不只看网络 ping；WebRTC 抖动缓冲、Opus 编解码、电视共享输出及音效处理都会叠加。记录所选局域网候选、丢包与接收端抖动缓冲统计；在 Android 与 iPhone 上分别测几分钟、切歌、断网、来电、锁屏和重新加入。网络波动造成的动态延迟不能靠固定歌词偏移修复。[WebRTC 接收统计](https://developer.mozilla.org/en-US/docs/Web/API/RTCInboundRtpStreamStats) 首版只接一部手机；多人合唱需要每路独立音量/静音、混音与更严格的防啸叫测试。

**路线选择**：扫码浏览器方案的安装门槛低，适合先验证跨平台；USB 麦克风路径最少一段无线传输，仍值得作为低延迟对照。若扫码方案无法在用户可接受的延迟内稳定返听，可以保留手机作为遥控器/歌词控制端，让实际麦克风走 USB 或外置混音器。两条路线共用电视 KTV 状态与麦克风静音控制，但输入采集方式不同。

## 分阶段实施与验收

1. **只读设备检查**：列出电视的音频输入/输出设备、USB 麦克风是否被识别、真实录音路由及采样格式。确认 USB 插入后是否夺走电视音频输出。没有可用输入或无法保持伴奏输出时，不启动软件麦克风实现。
2. **歌词与伴奏原型**：复用全屏歌词，KTV 模式先关闭淡入淡出；仅加入原唱削弱开关/滑块。用自制双声道测试素材验证 `k=1` 原样、`k<1` 中置降低、单声道旁路；再试听多首不同混音风格的歌。测试切歌、拖动、歌词同步和 CPU/内存。原有淡入淡出在普通模式保留，其 KTV 支持留待单独设计。
3. **麦克风单独回放原型**：按需请求权限，用 USB 麦克风完成干声监听；同时可独立验证扫码网页能否在两类手机上获得麦克风、完成配对，以及电视是否能运行 32 位 WebRTC 接收端。两条输入路线分别测实际输入/输出设备、延迟、底噪、啸叫及长时间运行是否卡顿。没有伴奏时先让人声链路稳定。
4. **与歌曲同播**：分别用 USB 与扫码手机输入测试系统混合伴奏和人声；测伴奏/麦克风各自音量、缓冲、暂停、切歌、输出设备拔插、焦点变化、返回桌面和重新进入。麦克风不能因两个 ExoPlayer 同时存在而重复播放；KTV 模式也不能抢焦点压低歌曲。若共享输出的延迟、啸叫或长时间漂移无法接受，即停止该软件返听路线，转评估外置混音器。
5. **可选体验增强**：小幅混响、设备选择、延迟提示、歌词大字布局。只有前四阶段稳定后再考虑多人手机麦克风或更重的人声分离。
6. **最终替换**：核对新版本与 V32 的常用功能，备好重新登录方式后卸载 V32、安装新包。用户已接受完整重装；不复制旧应用的登录令牌。本文发布的是调研，不代表上述测试已完成。

## 尚未实测、会改变方案的问题

- 这台电视和用户现有麦克风/音响的具体接口、USB 音频兼容性、输入及输出路由。
- 电视是否提供足够低延迟的监听路径，外放时是否会啸叫；蓝牙输出会怎样影响同步。
- V32 与本仓库当前版本在登录、音乐库、歌词、设置上的实际差异。
- 歌曲是否稳定以双声道 PCM 经过新增处理器，电台和视频是否需要旁路。
- 手机与电视是否能在当前路由器上直接建立 WebRTC 音频连接；浏览器麦克风权限、iPhone 前后台行为，以及电视 32 位接收依赖的包体和性能。

## 隐私与测试范围

本轮仅查阅源码、已保存 APK 的静态元数据及官方文档；没有读取 Music-User-Token、捕获麦克风音频、连接电视、构建 APK 或运行真人演唱测试。后续麦克风权限须由用户在电视试验版或手机配对页中主动授予；设计中的信令服务不接收音频，原型默认不保存或上传声音。
