---
id: conference/room-moderation
platform: web
api_docs:
  - title: 房间管理
    url: https://cloud.tencent.com/document/product/647/126919
---

# 会议会控与秩序管理 — Web 实现

## 前置条件
**通用依赖**：见 [login-auth 平台 slice](../login-auth.md)。

**额外依赖**：
- 已安装 `tuikit-atomicx-vue3@latest`

**前置状态**：
- 已阅读 `conference/room-moderation`，明确当前能力的产品边界。
- 已完成 `conference/login-auth`，确保当前页面具备稳定登录态。
- 已根据业务流程接入会议上下文；需要房间状态时，优先通过 `conference/room-lifecycle` 统一承接。

## 最佳实践
### 1. 明确 `disableAllDevices` 的作用范围和成员侧表现
- 房主和管理员都可以调用 `disableAllDevices()`。
- `disableAllDevices()` 可作用于麦克风、摄像头和屏幕分享，通常只限制普通成员；房主和管理员仍应保留对应会控与协作能力。
- 开启全体设备禁止后，普通成员的设备或共享入口应表现为 disabled，同时通过 toast 或等价提示明确告知当前受房间规则限制。
- 用户点击被禁用的入口时，可按客户需求决定是否继续提供“申请开启”动作；如果要做该能力，建议通过 `requestToOpenDevice()` 向房主或管理员发起申请。
- 如果只是被房主或管理员单独关闭摄像头、麦克风或屏幕分享，普通成员也应收到 toast 提示；但这类限制通常不是持续禁用，只要没有叠加房间级禁用，成员仍可再次主动开启对应能力。

### 2. 明确 `disableAllMessages` 的作用范围
房主和管理员都可以调用 `disableAllMessages()`；但该规则更适合作用于普通成员。房主和管理员通常仍应保留必要的管理沟通能力，避免在处理会控时把自己也一并锁死。

## 代码示例
### 会中会控：全员规则控制与单成员设备管理

```ts
import { onMounted, onUnmounted } from 'vue';
import {
  DeviceType,
  RoomParticipantEvent,
  useRoomParticipantState,
} from 'tuikit-atomicx-vue3/room';

const {
  disableAllDevices,
  disableAllMessages,
  closeParticipantDevice,
  subscribeEvent,
  unsubscribeEvent,
} = useRoomParticipantState();

await disableAllDevices({
  deviceType: DeviceType.Microphone,
  disable: true,
});

await disableAllMessages({ disable: true });

async function closeUserMicrophone(userId: string) {
  try {
    await closeParticipantDevice({
      userId,
      deviceType: DeviceType.Microphone,
    });
  } catch (error) {
    console.error('关闭成员麦克风失败', error);
  }
}

async function closeUserCamera(userId: string) {
  try {
    await closeParticipantDevice({
      userId,
      deviceType: DeviceType.Camera,
    });
  } catch (error) {
    console.error('关闭成员摄像头失败', error);
  }
}

async function closeUserScreenShare(userId: string) {
  try {
    await closeParticipantDevice({
      userId,
      deviceType: DeviceType.ScreenShare,
    });
  } catch (error) {
    console.error('关闭成员屏幕共享失败', error);
  }
}

function onParticipantDeviceClosed({ device, operator }) {
  console.warn('成员设备已被管理员关闭:', device, operator);
}

onMounted(() => {
  subscribeEvent(RoomParticipantEvent.onParticipantDeviceClosed, onParticipantDeviceClosed);
});

onUnmounted(() => {
  unsubscribeEvent(RoomParticipantEvent.onParticipantDeviceClosed, onParticipantDeviceClosed);
});
```

### 单成员关闭设备：更适合“纠正当前状态”，不等于持续禁用

`closeParticipantDevice()` 适合在成员已经打开麦克风、摄像头或屏幕共享时，房主或管理员立即把它关掉；它更像一次“纠正当前设备状态”的会控动作，而不是房间级长期限制。

如果业务想表达“普通成员后续也不能再自行打开该设备”，应优先使用 `disableAllDevices()` 做房间级禁用；如果只是单次关闭某个成员设备，则成员端应收到明确提示，并在没有房间级禁用的前提下允许再次主动开启。

## 调用时序
```
完成 login-auth 并进入会议
   │
   ▼
根据本地角色展示房主 / 管理员会控面板
   │
   ├─ 全员规则 → disableAllDevices / disableAllMessages
   ├─ 单成员控制 → closeParticipantDevice
   └─ 状态变更后 → 成员端 UI 立即感知并收口操作入口
```

## 平台特有注意事项
### 1. 创建期规则与会中会控要分层
默认禁麦、禁聊等会议初始规则属于 `conference/room-config`；会中动态控制才属于当前 slice。

### 2. 被控制方必须得到清晰反馈
远程关闭麦克风、摄像头或禁言后，被控制成员需要在本端看到明确提示，否则很容易误以为是本地故障。

### 3. 聊天与共享入口要和会控状态同步
全员禁言、禁共享或关闭设备的结果不应只靠接口返回值，页面层也必须立即响应状态变化。

### 4. 设备与共享入口要同时表达“不可用原因”和“恢复方式”
普通成员遇到 `disableAllDevices()` 触发的房间级禁用时，设备或共享按钮不应只做成不可点击的灰态而没有解释；至少应给出 toast 或等价提示，并根据产品需求决定是否在点击时调用 `requestToOpenDevice()`。这类房间级禁用通常不影响房主和管理员。如果只是被单独关闭设备或共享，则入口不应长期保持 disabled，而应允许成员在没有房间级禁用的前提下再次主动恢复对应能力。

## 代码生成约束
### 编译必要条件
- **通用条件**：见 [login-auth 平台 slice](../login-auth.md)。
- **额外导入**：至少需要导入 `useRoomParticipantState`，按需导入 `DeviceType`。
- **运行前提**：当前用户已在会议内，且具备房主或管理员权限。

### 生成规则
#### MUST（生成时必须包含）

1. **通过 `useRoomParticipantState` 执行会控动作** — 这样成员状态和 UI 感知才可统一收口。  
   **Verify**: 检查是否存在 `disableAllDevices` / `disableAllMessages` / `closeParticipantDevice`。
2. **把会控结果映射到成员端交互状态** — 否则页面会出现“按钮可点但实际被禁用”的错觉。  
   **Verify**: 检查聊天、设备或共享入口是否联动最新状态。
3. **按角色区分全体规则的实际生效范围** — `disableAllDevices` / `disableAllMessages` 的成员侧表现应与房主、管理员、普通成员的权限边界一致。  
   **Verify**: 检查普通成员与房主 / 管理员是否使用了不同的入口态或执行分支。

#### MUST NOT（生成时绝不能出现）

1. **不要把创建期配置当作会中控制复用** — 会造成语义混乱。  
   **Verify**: 检查会控逻辑是否仍与 `room-config` 分层。
2. **不要只在管理端成功提示，不处理成员端反馈** — 被控制用户会误判为本地故障。  
   **Verify**: 检查成员端是否有提示或状态联动逻辑。
3. **不要在房间级禁用仍生效时让普通成员直接调用开启能力** — 这会绕开会控语义，也会导致“点了没反应”的混乱体验。  
   **Verify**: 检查普通成员在全体禁设备或禁共享时是否优先走提示或申请链路，而不是直接 `open*` / `unmuteMicrophone()` / `startScreenShare()`。

### 集成检查点
- 当前 slice 常与 `conference/participant-management`、`conference/room-chat`、`conference/screen-share` 联动。
- 集成方式通常为新增房主管理面板和成员态提示，不需要修改底层会控实现。
- 如果业务还有企业合规或审计要求，建议把重要会控动作同步记录到业务日志系统。

## 验证矩阵
| 层级 | 检查项 | 验证手段 | 预期结果 |
|------|--------|----------|---------|
| 1. 编译级 | 已导入 `useRoomParticipantState` / `DeviceType` | 检查 `import` 语句 | 会控相关 API 可解析 |
| 2. 静态规则级 | 会控动作与成员端反馈都有体现 | 搜索会控 API 与状态联动逻辑 | 形成“控制 + 感知”闭环 |
| 3. 运行时级 | 房主 / 管理员可成功发起会控 | 在高权限账号下执行操作 | 会控成功并广播到成员端 |
| 4. 业务行为级 | 被控成员能明确感知限制 | 用被控制账号观察页面 | 设备 / 聊天 / 共享入口状态正确变化 |
