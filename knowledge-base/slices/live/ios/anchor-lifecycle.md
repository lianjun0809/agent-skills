---
id: live/anchor-lifecycle
platform: ios
---

# 主播开播与结束生命周期 — iOS 实现

## 前置条件

**依赖安装（Podfile）**
```ruby
pod 'AtomicXCore', '~> 4.0'
```

**前置状态**：
- `LoginStore.shared` 登录成功
- `DeviceStore.shared.openLocalCamera` 已成功（摄像头就绪）
- `DeviceStore.shared.openLocalMicrophone` 已成功（麦克风就绪）
- `LiveInfo` 已配置完毕（通过 `init(seatTemplate:)` 初始化，liveID 必填）

## API 调用（真实签名）

```swift
// 开播：第一参数无标签；completion 返回 LiveInfo
LiveListStore.shared.createLive(_ liveInfo: LiveInfo,
                                completion: LiveInfoCompletionClosure?)
// LiveInfoCompletionClosure = (Result<LiveInfo, ErrorInfo>) -> Void

// 结束直播：无 liveID 参数
LiveListStore.shared.endLive(completion: StopLiveCompletionClosure?)
// StopLiveCompletionClosure = ...  // TODO: 待验证 — 确认实际签名

// 订阅直播列表事件（Combine）
LiveListStore.shared.liveListEventPublisher  // PassthroughSubject<LiveListEvent, Never>
```

**LiveListEvent 完整签名**
```swift
enum LiveListEvent {
    // 直播被动结束（服务端强制终止/超时）
    case onLiveEnded(liveID: String, reason: LiveEndedReason, message: String)

    // 被踢出直播间
    case onKickedOutOfLive(liveID: String, reason: LiveKickedOutReason, message: String)
}
```

**关键类型说明**
```swift
// LiveInfo 初始化（必须传 seatTemplate）
var liveInfo = LiveInfo(seatTemplate: .videoDynamicGrid9Seats)
liveInfo.liveID   = "your-live-id"
liveInfo.liveName = "直播间名称"

// 通用回调
typealias CompletionClosure         = (Result<Void, ErrorInfo>) -> Void
typealias LiveInfoCompletionClosure = (Result<LiveInfo, ErrorInfo>) -> Void

struct ErrorInfo {
    var code: Int
    var message: String
}
```

## 代码示例

```swift
import AtomicXCore
import Combine

var cancellables = Set<AnyCancellable>()
var isLiving = false

// MARK: - 开播

func startLive(liveID: String, liveName: String) {
    // ⚠️ 必须用 init(seatTemplate:)，不是 LiveInfo()
    var liveInfo = LiveInfo(seatTemplate: .videoDynamicGrid9Seats)
    liveInfo.liveID   = liveID
    liveInfo.liveName = liveName

    // ⚠️ 第一参数无标签（unnamed first param）
    LiveListStore.shared.createLive(liveInfo) { result in
        switch result {
        case .success(let createdLiveInfo):
            // 成功时回调携带服务端确认的 LiveInfo（含 createTime 等字段）
            isLiving = true
            print("[Lifecycle] 开播成功, liveID: \(createdLiveInfo.liveID)")

        case .failure(let errorInfo):
            print("[Lifecycle] 开播失败, code: \(errorInfo.code), msg: \(errorInfo.message)")
            handleCreateError(errorInfo)
        }
    }
}

// MARK: - 结束直播

func stopLive() {
    guard isLiving else { return }
    isLiving = false

    // Step 1: 关闭设备（先关设备再结束房间）
    DeviceStore.shared.closeLocalCamera()
    DeviceStore.shared.closeLocalMicrophone()

    // Step 2: 结束直播（无 liveID 参数，endLive 结束当前房间）
    LiveListStore.shared.endLive { result in
        switch result {
        case .success:
            print("[Lifecycle] 直播结束成功")
            cleanupSubscriptions()

        case .failure(let errorInfo):
            // 即使失败也应清理本地资源
            print("[Lifecycle] endLive 失败, code: \(errorInfo.code)")
            cleanupSubscriptions()
        }
    }
}

// MARK: - 订阅被动结束事件（先订阅再开播，防止事件丢失）

func subscribeLiveEvents(currentLiveID: String) {
    LiveListStore.shared.liveListEventPublisher
        .receive(on: DispatchQueue.main)
        .sink { event in
            switch event {
            // ⚠️ onLiveEnded 有三个关联值：liveID, reason, message
            case .onLiveEnded(let liveID, let reason, let message)
                where liveID == currentLiveID:
                print("[Lifecycle] 直播被动结束, reason: \(reason), msg: \(message)")
                handlePassiveEnd(reason: message)

            // ⚠️ onKickedOutOfLive 有三个关联值：liveID, reason, message
            case .onKickedOutOfLive(let liveID, let reason, let message)
                where liveID == currentLiveID:
                print("[Lifecycle] 被踢出直播间, reason: \(reason), msg: \(message)")
                handleKickedOut(message: message)

            default:
                break
            }
        }
        .store(in: &cancellables)
}

// MARK: - 被动结束处理

func handlePassiveEnd(reason: String) {
    isLiving = false
    DeviceStore.shared.closeLocalCamera()
    DeviceStore.shared.closeLocalMicrophone()
    cleanupSubscriptions()
    print("[Lifecycle] 被动结束处理完成：\(reason)")
}

func handleKickedOut(message: String) {
    isLiving = false
    DeviceStore.shared.closeLocalCamera()
    DeviceStore.shared.closeLocalMicrophone()
    cleanupSubscriptions()
    print("[Lifecycle] 被踢出处理完成：\(message)")
}

// MARK: - 错误处理

func handleCreateError(_ errorInfo: ErrorInfo) {
    switch errorInfo.code {
    case -2105: print("直播间 ID 格式非法")
    case -2107: print("直播间名称非法（超长或含特殊字符）")
    case -2108: print("您已在其他直播间，请先退出")
    default:    print("开播失败（code: \(errorInfo.code)）: \(errorInfo.message)")
    }
}

// MARK: - 清理

func cleanupSubscriptions() {
    cancellables.removeAll()
}
```

## 调用时序

```
设备就绪（openLocalCamera + openLocalMicrophone 成功）
        │
        ▼
subscribeLiveEvents(currentLiveID:)     ← ① 先订阅事件，防止 createLive 前丢失
        │
        ▼
构建 LiveInfo(seatTemplate: .videoDynamicGrid9Seats)
设置 liveID + liveName（+ 其他选填字段）
        │
        ▼
// 第一参数无标签
LiveListStore.shared.createLive(liveInfo) { result in ... }
        │
        ├─ .failure(errorInfo)
        │       ├─ code -2105/-2107/-2108 → 展示错误，退出
        │       └─ 其他 → 展示错误信息
        │
        └─ .success(createdLiveInfo)
                │
                ▼
        isLiving = true
        UI 更新为「🔴 直播中」
                │
        ┌───────────────────────────┐
        │       直播进行中          │
        │   onLiveEnded →被动结束   │  liveID + reason + message
        │   onKickedOutOfLive→踢出 │  liveID + reason + message
        └───────────────────────────┘
                │
        [用户主动结束 / 被动结束]
                │
                ▼
        步骤1: DeviceStore.closeLocalCamera()
               DeviceStore.closeLocalMicrophone()
        步骤2: LiveListStore.shared.endLive(completion:)
                │
                └─ .success / .failure
                        │
                        ▼
        步骤3: cancellables.removeAll()     ← 取消事件订阅
               返回上层页面
```

## 平台特有注意事项

### 1. createLive 第一参数无标签
```swift
// ✅ 正确
LiveListStore.shared.createLive(liveInfo) { result in ... }

// ❌ 错误（带标签）
LiveListStore.shared.createLive(liveInfo: liveInfo) { ... }
```

### 2. endLive 无 liveID 参数
`endLive(completion:)` 结束的是 SDK 当前所在的直播间，不需要传 liveID：
```swift
// ✅ 正确
LiveListStore.shared.endLive { result in ... }

// ❌ 错误（没有此参数）
LiveListStore.shared.endLive(liveID: liveID) { ... }
```

### 3. LiveListEvent 关联值有三个字段
`onLiveEnded` 和 `onKickedOutOfLive` 的关联值均为 `(liveID: String, reason: …, message: String)`，不是只有 `liveID`：
```swift
// ✅ 正确
case .onLiveEnded(let liveID, let reason, let message):
    // 使用 liveID 过滤当前直播间

// ❌ 错误（缺少 reason 和 message）
case .onLiveEnded(let liveID):
```

### 4. Combine 订阅生命周期管理
`liveListEventPublisher` 的 Combine 订阅存储在 `cancellables` 中。务必在退出直播间时调用 `cancellables.removeAll()`，否则：
- ViewController 被释放后订阅仍存活（内存泄漏）
- 后续事件可能触发已释放对象的回调（野指针）

### 5. endLive 失败时的兜底处理
网络异常可能导致 `endLive` 回调超时或失败。即使失败也应清理本地资源（关闭设备、取消订阅），并在下次启动时通过服务端状态检查直播间是否仍存在。

### 6. 导航返回的拦截
通过 `viewWillDisappear` + `isMovingFromParent` 检测用户通过导航返回键意外退出：
```swift
override func viewWillDisappear(_ animated: Bool) {
    super.viewWillDisappear(animated)
    if isLiving && isMovingFromParent {
        stopLive()
    }
}
```
