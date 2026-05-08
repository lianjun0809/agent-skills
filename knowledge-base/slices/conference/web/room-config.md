---
id: conference/room-config
platform: web
api_docs:
  - title: 房间管理
    url: https://cloud.tencent.com/document/product/647/126919
---

# 会议配置与准入规则 — Web 实现

## 前置条件
**通用依赖**：见 [login-auth 平台 slice](../login-auth.md)。

**额外依赖**：
- 已安装 `tuikit-atomicx-vue3@latest`

**前置状态**：
- 已阅读 `conference/room-config`，明确当前能力的产品边界。
- 已完成 `conference/login-auth`，确保当前页面具备稳定登录态。
- 已根据业务流程接入会议上下文；创建 / 加入主路径优先通过 `conference/room-lifecycle` 承接。
- 如果当前是预约会议，时间、参会人和提醒等扩展字段应交由 `conference/room-schedule` 处理。

## 代码示例
### 维护房间配置草稿，并把 `options` 交给创建流程或会中更新流程消费

```ts
import { computed, reactive } from 'vue';
import { useRoomState } from 'tuikit-atomicx-vue3/room';

const { createAndJoinRoom, updateRoomInfo, currentRoom } = useRoomState();

const roomDraft = reactive({
  roomId: 'room_20260507_001',
  roomName: '产品评审会',
  password: '123456',
  isAllMicrophoneDisabled: true,
  isAllCameraDisabled: true,
  isAllScreenShareDisabled: true,
  isAllMessageDisabled: true,
});

const baseRoomOptions = computed(() => ({
  roomName: roomDraft.roomName.trim(),
  password: roomDraft.password || undefined,
  isAllMicrophoneDisabled: roomDraft.isAllMicrophoneDisabled,
  isAllCameraDisabled: roomDraft.isAllCameraDisabled,
  isAllScreenShareDisabled: roomDraft.isAllScreenShareDisabled,
  isAllMessageDisabled: roomDraft.isAllMessageDisabled,
}));

const toolbarUiState = computed(() => ({
  microphone: {
    disabled: baseRoomOptions.value.isAllMicrophoneDisabled,
    icon: baseRoomOptions.value.isAllMicrophoneDisabled ? 'mic-off-disabled' : 'mic-on',
    tooltip: baseRoomOptions.value.isAllMicrophoneDisabled ? '房主已开启全员静音' : '打开麦克风',
  },
}));

async function createConfiguredRoom() {
  await createAndJoinRoom({
    roomId: roomDraft.roomId,
    options: baseRoomOptions.value,
  });
}

async function updateCurrentRoomBaseInfo() {
  if (!currentRoom.value || currentRoom.value.roomId !== roomDraft.roomId) {
    console.warn('当前不在目标房间内，不能直接更新房间基础信息');
    return;
  }

  await updateRoomInfo({
    roomId: roomDraft.roomId,
    options: {
      roomName: roomDraft.roomName.trim(),
      password: roomDraft.password || undefined,
    },
  });
}
```

> **说明：**
> - `room-config` 负责维护 `roomDraft` 和 `options` 本身；
> - 当前是即时会议还是预约会议，不由当前 slice 决定，而由 `room-lifecycle` / `room-schedule` 选择消费方式；
> - 若当前是预约会议，可在 `room-schedule` 中复用 `baseRoomOptions`，再额外补充时间和参会人字段；
> - 当 `isAllMicrophoneDisabled` 为 `true` 时，前端不应只把它当成接口参数保存，还应同步让麦克风按钮切到禁用图标和禁用提示态。

## 调用时序
```text
完成 login-auth
   │
   ▼
维护 roomDraft 并完成字段校验
   │
   ▼
生成 baseRoomOptions
   │
   ├─ 即时会议 → 交给 room-lifecycle 的创建流程消费
   ├─ 预约会议 → 交给 room-schedule 复用，并追加时间 / 参会人字段
   ├─ 派生 toolbarUiState 等界面状态
   └─ 已在房间内且房主需要改名 / 改密码 → updateRoomInfo({ roomId, options })
```

## 平台特有注意事项
### 1. `room-config` 在 Web 侧更适合作为配置草稿与 `options` 的维护层
它回答的是“这场会议带什么配置被创建或更新”，而不是“当前该走创建、加入还是预约”。路径选择更适合交给 `room-lifecycle` 和 `room-schedule`。

### 2. `roomId` 更适合由业务后台统一生成并下发
这样既能降低冲突概率，也更利于快速会议、预约会议和服务端创建会议共用同一套会议标识。

### 3. 把基础字段收口到一个 `roomDraft` / `options` 模型里
房间名称、密码和默认规则最好由同一份草稿状态维护，避免快速会议和预约会议各自维护一套字段解释。
在 Web 侧，默认规则通常直接体现在 `options` 的四个布尔字段：`isAllMicrophoneDisabled`、`isAllCameraDisabled`、`isAllScreenShareDisabled`、`isAllMessageDisabled`。

### 4. 密码字段属于配置语义，但密码交互属于入会流程语义
`room-config` 负责写入 `password`；真正的密码输入、错误提示和重试逻辑，应由 `room-lifecycle` 的加入流程承接。

### 5. 预约会议应复用基础配置，而不是把时间字段塞回配置主模型
如果业务要预约会议，可在 `room-schedule` 中复用 `baseRoomOptions`，再额外补 `scheduleStartTime`、`scheduleEndTime`、`scheduleAttendees` 等字段。

### 6. `updateRoomInfo()` 更适合房主在会中执行
更新房间名称或密码前，应确认操作者仍在当前房间内；如果房主已经离房，通常需要重新入会后再修改。

### 7. 创建期默认规则与会中动态治理要分层
默认禁麦、禁画、禁共享、禁消息等规则适合在 `options` 中一次性声明；对应字段通常是 `isAllMicrophoneDisabled`、`isAllCameraDisabled`、`isAllScreenShareDisabled`、`isAllMessageDisabled`。会中临时禁言、关闭单成员设备等动作仍应由 `conference/room-moderation` 承接。

### 8. 默认规则生效后，按钮图标和禁用态也要同步
例如 `isAllMicrophoneDisabled` 为 `true` 时，麦克风按钮不应继续显示为“可开麦”图标；更合理的做法是同步切换到禁用态图标、禁用点击并展示“房主已开启全员静音”之类的提示。`isAllCameraDisabled`、`isAllScreenShareDisabled`、`isAllMessageDisabled` 也应同理映射到对应入口的禁用态。

## 代码生成约束
### 编译必要条件
- **通用条件**：见 [login-auth 平台 slice](../login-auth.md)。
- **额外导入**：直接消费配置或更新房间信息时，至少需要导入 `useRoomState`；若维护草稿状态，建议同时使用 `reactive` / `computed` 或等价状态管理方式。
- **运行前提**：当前用户具备创建会议或更新房间基础信息的权限。

### 生成规则
#### MUST（生成时必须包含）

1. **显式维护房间配置草稿或等价 `options` 模型** — 这是房间配置在 Web 侧最稳定的承载方式。  
   **Verify**: 检查是否存在 `roomDraft`、`baseRoomOptions` 或等价配置对象。
2. **通过 `options` 传递房间基础配置** — `roomName`、`password` 和默认规则都应从统一配置对象输出；若业务启用了全员规则，至少要明确处理 `isAllMicrophoneDisabled`、`isAllCameraDisabled`、`isAllScreenShareDisabled`、`isAllMessageDisabled` 这四类字段。  
   **Verify**: 检查是否存在 `options` 配置对象，而不是把字段散落在多个分支里。
3. **让默认规则同步驱动按钮图标、可点击态和提示文案** — 特别是 `isAllMicrophoneDisabled`，不能只在接口侧生效，而要让麦克风图标切换到禁用态。  
   **Verify**: 检查是否存在根据房间默认规则派生的按钮 `disabled`、图标或 tooltip 状态。
4. **把创建 / 加入路径选择留给 `room-lifecycle`，把时间扩展留给 `room-schedule`** — 当前 slice 只负责配置语义。  
   **Verify**: 检查是否没有在这里展开 `joinRoom` / `scheduleRoom` 的流程分流逻辑。
5. **调用 `updateRoomInfo()` 前确认当前房间上下文仍有效** — 房主离房后不应继续在脱离会中状态的页面直接修改房间信息。  
   **Verify**: 检查更新逻辑是否依赖 `currentRoom` 或等价房间态。
6. **让密码字段的创建侧与加入侧形成闭环** — 写入密码只是第一步，加入侧交互应明确交给其他 slice 承接。  
   **Verify**: 检查文档或代码是否明确把密码交互指向 `room-lifecycle`。

#### MUST NOT（生成时绝不能出现）

1. **不要在 `room-config` 里展开完整的创建 / 加入 / 退房流程** — 这会和 `room-lifecycle` 职责重叠。  
   **Verify**: 检查是否在当前 slice 中混入 `leaveRoom()`、`endRoom()` 或大量入会错误分流。
2. **不要把预约时间、参会人和提醒逻辑直接塞回基础配置模型** — 这会让 `room-config` 与 `room-schedule` 再次重叠。  
   **Verify**: 检查是否把 `scheduleStartTime`、`scheduleEndTime`、`scheduleAttendees` 直接当成当前 slice 的主字段。
3. **不要把 `updateRoomInfo()` 当成离房后仍可随时调用的全局配置接口** — 它更适合房主会中维护基础信息。  
   **Verify**: 检查更新前是否校验当前是否仍在目标房间。
4. **不要把会中动态治理动作塞回 `options` 草稿解释里** — 运行时禁言、关设备应回到 `room-moderation`。  
   **Verify**: 检查是否把会中单成员治理误写成当前 slice 的配置字段职责。

### 集成检查点
- 当前 slice 常与 `conference/room-lifecycle`、`conference/room-schedule`、`conference/room-moderation` 联动。
- 集成通常表现为创建会议表单、会议设置抽屉或会中基础信息编辑弹层。
- 若业务由后端创建会议，前端也应保持 `roomId`、会议名、密码和默认规则展示口径一致。

## 验证矩阵
| 层级 | 检查项 | 验证手段 | 预期结果 |
|------|--------|----------|---------|
| 1. 编译级 | 已导入 `useRoomState`，并维护配置草稿 | 检查 `import` 语句与配置对象 | 配置模型与更新逻辑可解析 |
| 2. 静态规则级 | 房间基础字段通过统一 `options` 输出，并派生按钮 UI 状态 | 搜索 `roomDraft` / `options` / `toolbarUiState` / `updateRoomInfo` | 配置字段来源集中且 UI 状态可追踪 |
| 3. 运行时级 | 创建时默认规则生效，会中可更新基础信息 | 创建一场带默认规则的会议并尝试改名 / 改密码 | 房间配置与房间信息更新都能正确工作 |
| 4. 业务行为级 | 默认规则与按钮图标、禁用态保持一致 | 验证全员静音后麦克风按钮是否同步变为禁用图标 | 用户不会看到“看起来可点、实际不可用”的错位体验 |
