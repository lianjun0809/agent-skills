---
id: conference/login-auth
platform: web
api_docs:
  - title: 接入概览
    url: https://cloud.tencent.com/document/product/647/126917
---

# 登录与鉴权 — Web 实现

## 前置条件
**通用依赖**：已准备 `SDKAppID / UserID / UserSig`，并确认正式环境的 `UserSig` 由业务后端签发。

**额外依赖**：
- 已安装 `tuikit-atomicx-vue3@latest`

**前置状态**：
- 已阅读 `conference/login-auth`，明确当前能力的产品边界。
- 若后续需要采集摄像头、麦克风或屏幕，请直接在 `HTTPS` 或 `localhost` 环境联调。

## 代码示例
### 基础接入：启动时完成登录并写入用户资料

```ts
import { onMounted } from 'vue';
import { useLoginState } from 'tuikit-atomicx-vue3/room';

const { login, setSelfInfo } = useLoginState();

onMounted(async () => {
  await login({ sdkAppId: 1400000000, userId: 'user_001', userSig: 'YOUR_USERSIG' });
  await setSelfInfo({ userName: 'Alice', avatarUrl: '' });
});
```

### 事件处理：登录过期与被强制下线后收口登录态

```ts
import { LoginEvent, useLoginState } from 'tuikit-atomicx-vue3';

const { subscribeEvent } = useLoginState();

subscribeEvent(LoginEvent.onLoginExpired, () => {
  console.log('登录已过期，请重新登录');
  // 跳转到登录页面或刷新 userSig
});

subscribeEvent(LoginEvent.onKickedOffline, () => {
  console.log('账号在其他设备登录，已被强制下线');
  // 提示用户重新登录
});
```

## 调用时序
```
应用启动
   │
   ▼
准备 SDKAppID / UserID / UserSig
   │
   ▼
初始化全局上下文提供者
   │
   ▼
调用 login(...)
   │
   ├─ 失败 → 提示鉴权或签名错误，停止后续房间 / 设备初始化
   │
   └─ 成功
       │
       ▼
调用 setSelfInfo(...)
       │
       ▼
订阅 LoginEvent.onLoginExpired / LoginEvent.onKickedOffline
       │
       ├─ 触发过期事件 → 跳转登录页或刷新 userSig 后重新登录
       │
       ├─ 触发被踢下线事件 → 提示账号异地登录并收口当前登录态
       │
       ▼
再进入 room-lifecycle / device-control / room-chat 等后续能力
```

## 平台特有注意事项
### 1. `userSig` 只能由业务后端在正式环境签发
开发联调时可以临时生成 `userSig`，但线上不能把签名逻辑放在前端，否则会直接破坏账号安全边界。

### 2. 登录应早于房间与设备能力初始化
`room-lifecycle`、`device-control`、`room-chat` 等能力都依赖稳定登录态；未登录就先调房间或聊天接口，通常会导致调用失败或状态异常。

### 3. 需要显式处理登录过期与被强制下线事件
当登录凭证过期时，会触发 `LoginEvent.onLoginExpired`；收到此事件后应立即跳转登录页，或先刷新 `userSig` 再重新执行登录流程，不能只在控制台打印日志。

当账号在其他设备登录导致当前设备被强制下线时，会触发 `LoginEvent.onKickedOffline`；收到此事件后应立即提示用户当前登录态已失效，并引导重新登录，避免页面继续停留在失效会中状态。

### 4. Web 端媒体能力依赖安全上下文
如果页面后续需要摄像头、麦克风、屏幕共享或本地特效，部署环境必须满足 `HTTPS`、`localhost` 或浏览器认可的其他安全上下文条件。

## 代码生成约束
### 编译必要条件
- **通用条件**：当前 slice 是 Web 会议所有后续能力的统一登录入口。
- **额外导入**：至少需要从 `tuikit-atomicx-vue3/room` 导入 `useLoginState`；若需订阅登录事件，还应从 `tuikit-atomicx-vue3` 导入 `LoginEvent`。
- **运行前提**：页面必须能拿到有效的 `SDKAppID / UserID / UserSig`；若要承接 UI Kit 组件，还需在根节点挂载全局上下文。

### 生成规则
#### MUST（生成时必须包含）

1. **先完成 `login()`，再初始化其他会议能力** — 否则后续 room / device / chat 调用缺少基础登录态。  
   **Verify**: 检查代码中是否存在 `await login(`，且后续房间或设备逻辑位于其后。
2. **把用户资料写入 `setSelfInfo()` 或等价链路** — 否则远端成员列表与会中 UI 可能只显示默认 ID。  
   **Verify**: 检查是否存在 `setSelfInfo(` 或明确的资料同步逻辑。
3. **订阅登录过期与被强制下线事件并收口恢复逻辑** — 登录成功后仍要处理 `LoginEvent.onLoginExpired` 与 `LoginEvent.onKickedOffline`，避免后续房间与聊天状态静默失效。  
   **Verify**: 检查是否存在 `subscribeEvent(LoginEvent.onLoginExpired, ...)`、`subscribeEvent(LoginEvent.onKickedOffline, ...)` 或等价恢复逻辑。

#### MUST NOT（生成时绝不能出现）

1. **不要把正式环境 `UserSig` 生成逻辑写在前端** — 会直接泄露签名能力。  
   **Verify**: 搜索是否存在前端本地生成正式 `UserSig` 的实现。
2. **不要在未登录态下直接调用房间、聊天或设备接口** — 会造成初始化顺序错误。  
   **Verify**: 检查房间 / 设备 / 聊天逻辑是否依赖已完成登录的状态。
3. **不要在收到登录过期或被强制下线事件后只打印日志、不做恢复处理** — 这会让后续能力停留在失效登录态。  
   **Verify**: 检查 `onLoginExpired` 与 `onKickedOffline` 处理逻辑是否真正跳转登录页、刷新 `userSig` 或提示用户重新登录。

### 集成检查点
- 当前 slice 是 `conference/web` 目录下其他所有能力 slice 的公共前置条件。
- 集成方式通常是新增启动逻辑或根级状态管理，不应修改 SDK 内部实现。
- 若业务项目已有独立账号体系，应把用户登录与 TRTC/IM 登录桥接清楚，而不是直接复用匿名测试账号。

## 验证矩阵
| 层级 | 检查项 | 验证手段 | 预期结果 |
|------|--------|----------|---------|
| 1. 编译级 | 已正确导入 `useLoginState`、`LoginEvent` 与登录事件相关依赖 | 检查 `import` 语句 | 可解析登录 Hook 与 `LoginEvent` |
| 2. 静态规则级 | 登录先于其他会议能力初始化，且已监听过期与被踢下线事件 | 搜索 `await login(`、`setSelfInfo(`、`subscribeEvent(LoginEvent.onLoginExpired`、`subscribeEvent(LoginEvent.onKickedOffline` | 先登录、写入资料，并具备完整的登录态恢复入口 |
| 3. 运行时级 | 登录成功后可继续设置资料，并能响应过期或被强制下线事件 | 触发启动流程并观察日志或调试状态 | 登录成功且用户资料更新完成；过期时进入重登或刷新票据流程；被踢下线时提示重新登录 |
| 4. 业务行为级 | 后续会议能力可正常承接 | 完成登录后继续进入会议页面，并模拟异地登录顶替场景 | 房间、设备、聊天入口不再因未登录失败；登录过期或被强制下线后都能被及时收口 |
