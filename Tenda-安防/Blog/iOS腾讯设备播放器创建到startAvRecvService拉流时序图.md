# iOS 腾讯设备播放器创建到 `startAvRecvService` 拉流时序图

> 范围：腾讯设备直播的冷启动路径，从 `LiveViewController` 创建 `CloudLivePlayer` 到 `TencentManager` 调用 `startAvRecvService`。已处于 P2P 就绪状态时，`TencentLivePlayer` 会跳过建连，直接进入直播状态查询。

```mermaid
sequenceDiagram
    autonumber
    participant VC as LiveViewController
    participant CP as CloudLivePlayer
    participant TP as TencentLivePlayer
    participant TM as TencentManager
    participant Cloud as 设备状态服务
    participant P2P as XP2P
    participant Pipeline as TDSMediaPipeline

    VC->>CP: initWithIotId:p2pInfo:channel:provider:
    CP->>TP: initialPlayer()，provider 选择腾讯实现
    VC->>CP: start()
    CP->>TP: start()
    TP->>TM: setDelegateWithIotId:delegate:
    TP->>TM: checkDeviceConnectingIsReadyWithIotId:

    alt P2P 未就绪
        TP->>TM: establishDeviceP2PWithIotId:
        TM->>Cloud: queryDeviceP2PToken(...)
        Cloud-->>TM: p2pInfo
        TM->>P2P: startP2PServiceWithIotId:p2pInfo:
        P2P-->>TM: XP2PTypeDetectReady (1004)
        TM-->>TP: establishBlock(YES)
    else P2P 已就绪
        TM-->>TP: YES
    end

    TP->>TM: queryDeviceVideoStatus(..., type = live)
    TM->>Cloud: 查询直播状态
    Cloud-->>TM: status == 0
    TM-->>TP: complete(resp, nil)
    TP->>Pipeline: start() 并注册解码回调
    TP->>TM: startLiveWithIotId:excuteStr:livePlayer:
    TM->>P2P: startAvRecvService(iotId, liveCmd, false)
```

## 源码依据

- `TendaSecurity/Classes/Main/Home/SubFunc/Live/ViewController/LiveViewController.mm`：创建 `CloudLivePlayer`，并在直播启动处调用 `start()`。
- `TestAliSDK/Container/CloudLivePlayer.mm`：根据 `provider` 创建腾讯播放器，并把 `start()` 转发给 `TencentLivePlayer`。
- `TestAliSDK/Internal/ImplManager/Tencent/TencentLivePlayer.mm`：P2P 就绪检查、`startToPlay` 的直播许可校验、媒体管线启动和 `startLiveWithIotId` 调用。
- `TestAliSDK/Internal/ImplManager/Tencent/TencentManager.mm`：P2P Token 查询、`1004` 处理、`startLiveWithIotId` 和最终的 `startAvRecvService` 调用。

## Android 腾讯设备播放器创建到 `startAvRecvService` 拉流

> 范围：腾讯设备直播的冷启动路径，从 `BaseLivePlayerActivity` 创建 `TDLivePlayer` 到 `VCSDKInteral` 调用 `XP2P.startAvRecvService`。`prepare()` 负责 P2P 信息查询、建连与设备直播状态校验；收到 `TD_STATE_READY` 后，业务层再调用 `start()` 启动收流。

```mermaid
sequenceDiagram
    autonumber
    participant App as BaseLivePlayerActivity
    participant Player as TDLivePlayer
    participant Internal as TDLivePlayerInteral
    participant VCI as VCSDKInteral
    participant Cloud as TDCloudSDK
    participant P2P as XP2P
    participant JNI as native-lib.cpp

    App->>Player: new TDLivePlayer(context)
    App->>Player: setPlayerListener(listener)
    App->>Player: setLiveDataSource(iotId, streamType)
    App->>Player: setTextureView(textureView)
    Player->>Player: 创建并缓存 Surface
    App->>Player: setAudioTrack(null)
    Player->>Player: 异步创建并 play() AudioTrack
    Player-->>VCI: 异步 setLiveAudioTrack(iotId, audioTrack)
    VCI-->>JNI: processAudioTrackCreated(...)
    App->>Player: 后台线程：prepare(deviceName, productKey)
    Player->>Internal: prepare()
    Internal->>VCI: setStreamType(streamType)<br/>reConnectP2p(iotId, dn, pk)

    alt P2P 未就绪
        VCI->>Cloud: getDeviceXp2pInfo(iotId, dn, pk)
        Cloud-->>VCI: Xp2pInfo
        VCI->>P2P: startService(appContext, xp2pInfo, config)
        P2P-->>VCI: 1004：P2P 连接就绪
    else P2P 已就绪
        VCI->>VCI: 复用现有连接
    end

    VCI->>Cloud: checkDevicePlayState(iotId)
    Cloud-->>VCI: status == 0（允许直播）
    VCI-->>Internal: notifyOnStateChange(TD_STATE_READY)
    Internal-->>App: onPlayerStateChange(READY)

    App->>Player: start()
    Player->>Internal: start()
    Internal->>VCI: setLiveSurface(iotId, surface, true)
    VCI->>JNI: processSurfaceCreated(...)
    Internal->>VCI: onLive(iotId)
    VCI->>P2P: startAvRecvService(iotId, liveCmd)
```

### Android 关键点

- `setTextureView()` 只创建并缓存 `Surface`；`setAudioTrack(null)` 在异步任务中创建、`play()` 并调用 `setLiveAudioTrack(...)`，因此这条音频绑定链可与 `prepare()` 并行。
- P2P 建连入口是 `prepare()` 阶段的 `reConnectP2p(...)`，不是 `start()`。
- `1004` 仅表示 P2P 连接就绪；随后必须确认 `status == 0`，才向页面发出 `TD_STATE_READY`。
- 页面在 `onPlayerStateChange(READY)` 中通过 `executorService` 调用 `start()`；它绑定缓存的 `Surface`，再通过 `onLive(iotId)` 触发最终的 `startAvRecvService(...)`。

### Android 源码依据

- `tendasecurity-android/app/src/main/java/com/tenda/security/activity/live/BaseLivePlayerActivity.java`：`createPlayer()` 创建 `TDLivePlayer`；初始化阶段设置数据源；`onSurfaceTextureAvailable()` 调用 `setTextureView()`、`setAudioTrack(null)`，并在线程中执行 `prepare(deviceName, productKey)`；READY 回调中提交 `mPlayer.start()`。
- `TDLivePlayer` / `TDLivePlayerInteral`：播放器配置、音频异步初始化、`prepare()` 与 `start()` 的实现入口。
- `VCSDKInteral`：`reConnectP2p(...)`、P2P `1004` 处理、播放状态检查、JNI 转发和 `startAvRecvService(...)` 的调用入口。
