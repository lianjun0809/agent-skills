# AtomicXCore iOS API 真实签名参考

> 来源：https://tencent-rtc.github.io/TUIKit_iOS/documentation/atomicxcore
> 抓取日期：2026-03-27
> 用途：作为 slice 内容校验的唯一权威来源

## 通用类型

```swift
typealias CompletionClosure = (Result<Void, ErrorInfo>) -> Void
typealias LiveInfoCompletionClosure = (Result<LiveInfo, ErrorInfo>) -> Void
typealias StopLiveCompletionClosure = ...  // 结束直播回调
typealias MetaDataCompletionClosure = ...  // 元数据回调
typealias BattleRequestClosure = ...       // PK 请求回调

struct ErrorInfo {
    var code: Int
    var message: String
}
```

## LoginStore（单例 .shared）

```swift
class LoginStore {
    static let shared: LoginStore

    var state: StatePublisher<LoginState>
    let loginEventPublisher: PassthroughSubject<LoginEvent, Never>
    var sdkAppID: Int32

    func login(sdkAppID: Int32, userID: String, userSig: String, completion: CompletionClosure?)
    func logout(completion: CompletionClosure?)
    func setSelfInfo(userProfile: UserProfile, completion: CompletionClosure?)
}

struct LoginState {
    var loginStatus: LoginStatus
    var loginUserInfo: UserProfile?
}

enum LoginEvent {
    case kickedOffline
    case loginExpired
}
```

## DeviceStore（单例 .shared）

```swift
class DeviceStore {
    static let shared: DeviceStore

    var state: StatePublisher<DeviceState>

    func openLocalCamera(isFront: Bool, completion: CompletionClosure?)
    func closeLocalCamera()
    func openLocalMicrophone(completion: CompletionClosure?)
    func closeLocalMicrophone()
    func switchCamera(isFront: Bool)
    func switchMirror(mirrorType: MirrorType)
    func setCaptureVolume(volume: Int)        // range [0, 100]
    func setOutputVolume(_ volume: Int)       // range [0, 100]
    func setAudioRoute(_ route: AudioRoute)
    func updateVideoQuality(_ quality: VideoQuality)
    func startScreenShare(appGroup: String)
    func stopScreenShare()
    func startCameraTest(cameraView: UIView, completion: CompletionClosure?)
    func stopCameraTest()
    func reset()
}

struct DeviceState {
    var microphoneStatus: DeviceStatus
    var microphoneLastError: DeviceError
    var captureVolume: Int                    // range [0, 100]
    var currentMicVolume: Int
    var outputVolume: Int                     // range [0, 100]
    var cameraStatus: DeviceStatus
    var cameraLastError: DeviceError
    var isFrontCamera: Bool
    var localMirrorType: MirrorType
    var localVideoQuality: VideoQuality
    var currentAudioRoute: AudioRoute
    var screenStatus: DeviceStatus
    var networkInfo: NetworkInfo
}
```

## LiveListStore（单例 .shared）

```swift
class LiveListStore {
    static let shared: LiveListStore

    let state: StatePublisher<LiveListState>
    let liveListEventPublisher: PassthroughSubject<LiveListEvent, Never>

    func createLive(_ liveInfo: LiveInfo, completion: LiveInfoCompletionClosure?)
    func endLive(completion: StopLiveCompletionClosure?)
    func fetchLiveList(cursor: String, count: Int, completion: CompletionClosure?)
    func fetchLiveInfo(liveID: String, completion: LiveInfoCompletionClosure?)
    func joinLive(liveID: String, completion: LiveInfoCompletionClosure?)
    func leaveLive(completion: CompletionClosure?)
    func updateLiveInfo(_ liveInfo: LiveInfo, modifyFlag: LiveInfo.ModifyFlag, completion: CompletionClosure?)
    func updateLiveMetaData(_ metaData: [String: String], completion: CompletionClosure?)
    func queryMetaData(keys: [String], completion: MetaDataCompletionClosure?)
    func reset()
}

struct LiveListState {
    var liveList: [LiveInfo]
    var liveListCursor: String
    var currentLive: LiveInfo
}

enum LiveListEvent {
    case onLiveEnded(liveID: String, reason: LiveEndedReason, message: String)
    case onKickedOutOfLive(liveID: String, reason: LiveKickedOutReason, message: String)
}

struct LiveInfo {
    // 初始化
    init(seatTemplate: SeatLayoutTemplate)

    // 属性
    var liveID: String
    var liveName: String
    var coverURL: String
    var backgroundURL: String
    var liveOwner: LiveUserInfo
    var categoryList: [NSNumber]
    var metaData: [String: String]
    var notice: String
    var seatLayoutTemplateID: UInt            // Deprecated
    var seatTemplate: SeatLayoutTemplate
    var seatMode: TakeSeatMode
    var maxSeatCount: Int                     // Deprecated
    var keepOwnerOnSeat: Bool
    var isSeatEnabled: Bool                   // Deprecated
    var isGiftEnabled: Bool
    var isMessageDisable: Bool
    var isPublicVisible: Bool
    var activityStatus: Int
    var createTime: Int
    var totalViewerCount: Int
    var isEmpty: Bool
}

struct LiveUserInfo {
    var userID: String
    var userName: String
    var avatarURL: String
}
```

## CoGuestStore（工厂 create(liveID:)）

```swift
class CoGuestStore {
    static func create(liveID: String) -> CoGuestStore

    let state: StatePublisher<CoGuestState>
    let hostEventPublisher: PassthroughSubject<HostEvent, Never>
    let guestEventPublisher: PassthroughSubject<GuestEvent, Never>

    func applyForSeat(seatIndex: Int = -1, timeout: TimeInterval, extraInfo: String?, completion: CompletionClosure?)
    func cancelApplication(completion: CompletionClosure?)
    func acceptApplication(userID: String, completion: CompletionClosure?)
    func rejectApplication(userID: String, completion: CompletionClosure?)
    func inviteToSeat(userID: String, seatIndex: Int, timeout: TimeInterval, extraInfo: String?, completion: CompletionClosure?)
    func cancelInvitation(inviteeID: String, completion: CompletionClosure?)
    func acceptInvitation(inviterID: String, completion: CompletionClosure?)
    func rejectInvitation(inviterID: String, completion: CompletionClosure?)
    func disConnect(completion: CompletionClosure?)
}

struct CoGuestState {
    var connected: [SeatUserInfo]     // 已上麦用户
    var invitees: [LiveUserInfo]      // 主播已邀请（等待响应）
    var applicants: [LiveUserInfo]    // 观众已申请（等待主播处理）
    var candidates: [LiveUserInfo]    // 可邀请候选用户
}

enum HostEvent {
    case onGuestApplicationReceived(guestUser: LiveUserInfo)
    case onGuestApplicationCancelled(guestUser: LiveUserInfo)
    case onGuestApplicationProcessedByOtherHost(guestUser: LiveUserInfo, hostUser: LiveUserInfo)
    case onHostInvitationResponded(isAccept: Bool, guestUser: LiveUserInfo)
    case onHostInvitationNoResponse(guestUser: LiveUserInfo, reason: NoResponseReason)
}

enum GuestEvent {
    case onHostInvitationReceived(hostUser: LiveUserInfo)
    case onHostInvitationCancelled(hostUser: LiveUserInfo)
    case onGuestApplicationResponded(isAccept: Bool, hostUser: LiveUserInfo)
    case onGuestApplicationNoResponse(reason: NoResponseReason)
    case onKickedOffSeat(seatIndex: Int, hostUser: LiveUserInfo)
}
```

## CoHostStore（工厂 create(liveID:)）

```swift
class CoHostStore {
    static func create(liveID: String) -> CoHostStore

    let state: StatePublisher<CoHostState>
    let coHostEventPublisher: PassthroughSubject<CoHostEvent, Never>

    func requestHostConnection(targetHost: String, layoutTemplate: CoHostLayoutTemplate, timeout: TimeInterval, extraInfo: String, completion: CompletionClosure?)
    func cancelHostConnection(toHostLiveID: String, completion: CompletionClosure?)
    func acceptHostConnection(fromHostLiveID: String, completion: CompletionClosure?)
    func rejectHostConnection(fromHostLiveID: String, completion: CompletionClosure?)
    func exitHostConnection(completion: CompletionClosure?)
    func getCoHostCandidates(cursor: String, completion: CompletionClosure?)
}

struct CoHostState {
    var coHostStatus: CoHostStatus
    var connected: [SeatUserInfo]       // 注意：不是 LiveUserInfo
    var invitees: [LiveUserInfo]
    var applicant: LiveUserInfo?        // 注意：单数
    var candidates: [LiveUserInfo]
    var candidatesCursor: String
}

enum CoHostEvent {
    case onCoHostRequestReceived(inviter: LiveUserInfo, extensionInfo: String)
    case onCoHostRequestAccepted(invitee: LiveUserInfo)
    case onCoHostRequestRejected(invitee: LiveUserInfo)
    case onCoHostRequestCancelled(inviter: LiveUserInfo, invitee: LiveUserInfo)
    case onCoHostRequestTimeout(inviter: LiveUserInfo, invitee: LiveUserInfo)
    case onCoHostUserJoined(userInfo: LiveUserInfo)
    case onCoHostUserLeft(userInfo: LiveUserInfo)
}
```

## BattleStore（工厂 create(liveID:)）

```swift
class BattleStore {
    static func create(liveID: String) -> BattleStore

    var state: StatePublisher<BattleState>
    var battleEventPublisher: PassthroughSubject<BattleEvent, Never>

    func requestBattle(config: BattleConfig, userIDList: [String], timeout: TimeInterval, completion: BattleRequestClosure?)
    func cancelBattleRequest(battleId: String, userIdList: [String], completion: CompletionClosure?)
    func acceptBattle(battleID: String, completion: CompletionClosure?)
    func rejectBattle(battleID: String, completion: CompletionClosure?)
    func exitBattle(battleID: String, completion: CompletionClosure?)
}

struct BattleState {
    var currentBattleInfo: BattleInfo?
    var battleUsers: [BattleUser]       // 注意文档里实际类型待确认
    var battleScore: ...                // 分数相关
}

struct BattleInfo {
    var battleID: String
    var config: BattleConfig
    var startTime: Int
    var endTime: Int
}

struct BattleConfig {
    init(duration: Int, needResponse: Bool, extensionInfo: String?)
    var duration: Int
    var needResponse: Bool
    var extensionInfo: String?
}

enum BattleEvent {
    case onBattleRequestReceived(battleID: String, inviter: LiveUserInfo, invitee: LiveUserInfo)
    case onBattleRequestAccept(battleID: String, inviter: LiveUserInfo, invitee: LiveUserInfo)
    case onBattleRequestReject(battleID: String, inviter: LiveUserInfo, invitee: LiveUserInfo)
    case onBattleRequestCancelled(battleID: String, inviter: LiveUserInfo, invitee: LiveUserInfo)
    case onBattleRequestTimeout(battleID: String, inviter: LiveUserInfo, invitee: LiveUserInfo)
    case onBattleStarted(battleInfo: BattleInfo, inviter: LiveUserInfo, invitees: [LiveUserInfo])
    case onBattleEnded(battleInfo: BattleInfo, reason: BattleEndedReason)
    case onUserJoinBattle(battleID: String, battleUser: ...)
    case onUserExitBattle(battleID: String, battleUser: ...)
}
```

## LiveAudienceStore（工厂 create(liveID:)）

```swift
class LiveAudienceStore {
    static func create(liveID: String) -> LiveAudienceStore

    var state: StatePublisher<LiveAudienceState>
    var liveAudienceEventPublisher: PassthroughSubject<LiveAudienceEvent, Never>

    func fetchAudienceList(completion: CompletionClosure?)
    func kickUserOutOfRoom(userID: String, completion: CompletionClosure?)
    func setAdministrator(userID: String, completion: CompletionClosure?)
    func revokeAdministrator(userID: String, completion: CompletionClosure?)
    func disableSendMessage(userID: String, isDisable: Bool, completion: CompletionClosure?)
}

struct LiveAudienceState {
    var audienceList: [LiveUserInfo]
    var audienceCount: UInt                  // 注意：UInt 不是 Int
    var messageBannedUserList: [LiveUserInfo]
}

enum LiveAudienceEvent {
    case onAudienceJoined(audience: LiveUserInfo)
    case onAudienceLeft(audience: LiveUserInfo)
    case onAudienceMessageDisabled(audience: LiveUserInfo, isDisable: Bool)
}
```

## BarrageStore（工厂 create(liveID:)）

```swift
class BarrageStore {
    static func create(liveID: String) -> BarrageStore

    var state: StatePublisher<BarrageState>

    func sendTextMessage(text: String, extensionInfo: [String: String]?, completion: CompletionClosure?)
    func sendCustomMessage(businessID: String, data: String, completion: CompletionClosure?)
    func appendLocalTip(message: Barrage)
}

struct BarrageState {
    var messageList: [Barrage]
}

enum BarrageType {
    case text    // 0
    case custom  // 1
}
```

## GiftStore（工厂 create(liveID:)）

```swift
class GiftStore {
    static func create(liveID: String) -> GiftStore

    var state: StatePublisher<GiftState>
    var giftEventPublisher: PassthroughSubject<GiftEvent, Never>

    func refreshUsableGifts(completion: CompletionClosure?)
    func sendGift(giftID: String, count: UInt, completion: CompletionClosure?)
    func setLanguage(_ language: String)
}

struct GiftState {
    var usableGifts: [GiftCategory]
}

struct GiftCategory {
    let categoryID: String
    let name: String
    let desc: String
    let extensionInfo: [String: String]
    let giftList: [Gift]
}

enum GiftEvent {
    case onReceiveGift(liveID: String, gift: Gift, count: UInt, sender: LiveUserInfo)
}
```

## AudioEffectStore（单例 .shared）

```swift
class AudioEffectStore {
    static let shared: AudioEffectStore

    var state: StatePublisher<AudioEffectState>

    func setAudioChangerType(type: AudioChangerType)
    func setAudioReverbType(type: AudioReverbType)
    func setVoiceEarMonitorEnable(enable: Bool)
    func setVoiceEarMonitorVolume(volume: Int)   // range 0-100
    func reset()
}

struct AudioEffectState {
    var audioChangerType: AudioChangerType
    var audioReverbType: AudioReverbType
    var isEarMonitorOpened: Bool
    var earMonitorVolume: Int                     // range 0-100
}

// 重要：离房后效果自动失效，无需手动 reset
// 警告：蓝牙耳机不支持耳返，需要用有线耳机

enum AudioChangerType {
    case none, child, littleGirl, man, ethereal, cold
    case foreignerr, heavyMachinery, heavyMetal, strongCurrent, fatso, trappedBeast
}

enum AudioReverbType {
    case none, ktv, smallRoom, auditorium, loud, deep, magnetic, metallic
}
```

## BaseBeautyStore（单例 .shared）

```swift
class BaseBeautyStore {
    static let shared: BaseBeautyStore

    var state: StatePublisher<BaseBeautyState>

    func setSmoothLevel(smoothLevel: Float)       // 注意：Float 不是 Int
    func setWhitenessLevel(whitenessLevel: Float)  // 注意：Float 不是 Int
    func setRuddyLevel(ruddyLevel: Float)         // 注意：Float 不是 Int
    func reset()
}

struct BaseBeautyState {
    var smoothLevel: Float      // range [0-9]
    var whitenessLevel: Float   // range [0-9]
    var ruddyLevel: Float       // range [0-9]
}
```

## SeatLayoutTemplate

```swift
enum SeatLayoutTemplate {
    case audioSalon(seatCount: Int)
    case karaoke(seatCount: Int)
    case videoDynamicFloat7Seats
    case videoDynamicGrid9Seats
    case videoFixedFloat7Seats
    case videoFixedGrid9Seats
    case videoLandscape4Seats
}
```

## TakeSeatMode

```swift
enum TakeSeatMode {
    case apply
    case free
}
```
