---
id: conference/room-invite
platform: web
api_docs:
  - title: 会中呼叫
    url: https://cloud.tencent.com/document/product/647/126929
---

# 会中邀请入会 — Web 实现

## 前置条件
**通用依赖**：见 [login-auth 平台 slice](../login-auth.md)。

**额外依赖**：
- 已安装 `tuikit-atomicx-vue3@latest`

**前置状态**：
- 已阅读 `conference/room-invite`，明确当前能力的产品边界。
- 已完成 `conference/login-auth`，确保当前页面具备稳定登录态。
- 已根据业务流程接入会议上下文；需要房间状态时，优先通过 `conference/room-lifecycle` 统一承接。

## 代码示例
### 邀请链路：发起呼叫、接听邀请并进入会议

```ts
import { useRoomState, RoomEvent } from 'tuikit-atomicx-vue3/room';

const { callUserToRoom, acceptCall, joinRoom, subscribeEvent } = useRoomState();

await callUserToRoom({ roomId: 'demo_room', userIdList: ['user_002'], timeout: 60 });

subscribeEvent(RoomEvent.onCallReceived, async ({ roomInfo }) => {
  await acceptCall({ roomId: roomInfo.roomId });
  await joinRoom({ roomId: roomInfo.roomId });
});
```

## 调用时序
```
【邀请端】
在会议中调用 callUserToRoom({ roomId, userIdList, timeout })
   │
   ▼
被邀请人收到 RoomEvent.onCallReceived
   │
   ├─ 接受 → acceptCall({ roomId }) → joinRoom({ roomId })
   ├─ 拒绝 → rejectCall({ roomId })
   ├─ 超时 → onCallTimeout 收口 UI
   └─ 其他设备已处理 → onCallHandledByOtherDevice 收口 UI
```

## 平台特有注意事项
### 1. `acceptCall()` 不等于已经进入会议
接受邀请只是信令确认；真正进入房间仍需要继续调用 `joinRoom()`，两步不能省略其中任何一步。

### 2. 邀请弹层应统一监听取消、超时与多设备收口
如果只处理收到邀请和接受，不处理 `onCallCancelled`、`onCallTimeout`、`onCallHandledByOtherDevice`，邀请 UI 很容易残留。

### 3. `extensionInfo` 适合透传业务字段
邀请原因、来源场景、优先级、会议主题摘要等字段建议统一放在 `extensionInfo` 或业务补充结构中，而不是在 UI 层硬编码。

## 代码生成约束
### 编译必要条件
- **通用条件**：见 [login-auth 平台 slice](../login-auth.md)。
- **额外导入**：至少需要导入 `useRoomState`；如需事件监听，还需导入 `RoomEvent`。
- **运行前提**：当前用户已在会中，且被邀请方具备有效登录态。

### 生成规则
#### MUST（生成时必须包含）

1. **把邀请确认与真正入会拆成两步** — 才能正确覆盖呼叫确认与实际进房两个阶段。  
   **Verify**: 检查是否同时存在 `acceptCall(` 与 `joinRoom(`。
2. **在常驻层监听邀请事件并收口状态** — 任何页面都应能响应呼叫。  
   **Verify**: 检查是否存在 `subscribeEvent(RoomEvent.onCallReceived, ...)` 或等价监听逻辑。

#### MUST NOT（生成时绝不能出现）

1. **不要把 `acceptCall()` 当作最终进房动作** — 会造成“已接听但未进房”的悬空状态。  
   **Verify**: 检查接受邀请后是否继续调用 `joinRoom()`。
2. **不要忽略取消、超时和多设备处理事件** — 邀请弹窗容易残留。  
   **Verify**: 检查是否存在对应事件或兜底清理说明。

### 集成检查点
- 当前 slice 常与 `conference/room-lifecycle`、`conference/participant-management` 联动。
- 集成通常表现为会中邀请弹层和常驻呼叫监听组件。
- 若业务侧还有站内信或电话提醒，需要明确多种通知渠道之间的优先级和收口策略。

## 验证矩阵
| 层级 | 检查项 | 验证手段 | 预期结果 |
|------|--------|----------|---------|
| 1. 编译级 | 已导入 `useRoomState` 与 `RoomEvent` | 检查 `import` 语句 | 邀请相关 API 与事件可解析 |
| 2. 静态规则级 | 接受邀请后继续进房 | 搜索 `acceptCall` 与 `joinRoom` | 存在两步链路 |
| 3. 运行时级 | 收到邀请后可接受并进入会议 | 双账号联调邀请流程 | 接收端可成功进房 |
| 4. 业务行为级 | 取消 / 超时 / 多设备处理时 UI 正确收口 | 模拟各种邀请结果 | 邀请弹层不会残留 |
