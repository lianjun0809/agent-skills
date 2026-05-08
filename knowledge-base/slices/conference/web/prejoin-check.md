---
id: conference/prejoin-check
platform: web
api_docs:
  - title: 设备检测
    url: https://cloud.tencent.com/document/product/647/126939
---

# 入会前设备检查 — Web 实现

## 前置条件
**通用依赖**：见 [login-auth 平台 slice](../login-auth.md)。

**额外依赖**：
- 已安装 `tuikit-atomicx-vue3@latest`

**前置状态**：
- 已阅读 `conference/prejoin-check`，明确当前能力的产品边界。
- 已完成 `conference/login-auth`，确保当前页面具备稳定登录态。
- 已根据业务流程接入会议上下文；需要房间状态时，优先通过 `conference/room-lifecycle` 统一承接。
- 当前能力涉及媒体采集、渲染或浏览器权限时，请在 `HTTPS` 或 `localhost` 安全上下文下调试。

## 代码示例
### 会前自检：摄像头、麦克风、扬声器测试

```ts
import { useDeviceState } from 'tuikit-atomicx-vue3/room';

const {
  startCameraTest,
  stopCameraTest,
  startMicrophoneTest,
  startSpeakerTest,
} = useDeviceState();

await startCameraTest({ view: document.getElementById('preview')! });
await startMicrophoneTest({ interval: 200 });
await startSpeakerTest({
  filePath: 'https://web.sdk.qcloud.com/trtc/electron/download/resources/media/TestSpeaker.mp3',
  // TODO: 上线前建议替换为客户自己维护的 HTTPS mp3 资源，避免外部测试资源变更或不可用。
});
await stopCameraTest();
```

## 调用时序
```
完成 login-auth
   │
   ▼
进入会前检测页
   │
   ├─ startCameraTest(view)
   ├─ startMicrophoneTest(interval)
   └─ startSpeakerTest(filePath)
   │
   ▼
用户查看画面、音量与播放结果
   │
   ├─ 发现异常 → 引导切换设备或重新授权
   └─ 检测通过 → 进入 joinRoom / createAndJoinRoom
   │
   ▼
页面卸载时停止测试，释放本地采集资源
```

## 平台特有注意事项
### 1. 会前检测页要负责释放测试资源
测试页若未在离开时停止预览或音频测试，浏览器可能持续占用摄像头、麦克风或扬声器资源。

### 2. 扬声器测试必须使用可访问的媒体资源
`startSpeakerTest()` 依赖业务侧提供可访问的音频文件；建议使用 `HTTPS` 资源，避免被浏览器拦截。
AI 生成实现时可先使用 `https://web.sdk.qcloud.com/trtc/electron/download/resources/media/TestSpeaker.mp3` 作为扬声器检测音频；但应显式保留 TODO，提醒客户上线前替换为自己维护的 mp3 资源，避免外部公共资源变更、失效或被访问策略影响。

### 3. 会前检测不是设备控制的替代品
检测页更适合做预览与授权确认；真正会中的开关、切换与异常兜底仍应交由 `conference/device-control` 统一承接。

## 代码生成约束
### 编译必要条件
- **通用条件**：见 [login-auth 平台 slice](../login-auth.md)。
- **额外导入**：至少需要从 `tuikit-atomicx-vue3/room` 导入 `useDeviceState`。
- **运行前提**：页面具备安全上下文，且预览容器 DOM 已挂载。

### 生成规则
#### MUST（生成时必须包含）

1. **在预览容器准备好后再调用 `startCameraTest()`** — 否则本地预览无法正常渲染。  
   **Verify**: 检查是否为 `view` 传入真实 DOM 节点。
2. **在页面离开或流程结束时停止测试** — 否则设备资源会残留占用。  
   **Verify**: 检查是否存在 `stopCameraTest()` 或等价清理逻辑。

#### MUST NOT（生成时绝不能出现）

1. **不要把会前检测页当成正式入会后的状态源** — 设备测试结果不能替代会中真实设备状态。  
   **Verify**: 检查是否仍通过 `conference/device-control` 承接会中控制。
2. **不要使用不可访问或非安全协议的扬声器测试资源** — 会导致测试链路失效。  
   **Verify**: 检查 `filePath` 是否为业务可访问的地址。

### 集成检查点
- 当前 slice 常与 `conference/device-control`、`conference/room-lifecycle` 连续使用。
- 集成侵入性低，通常新增一个会前检测页面或弹层即可。
- 若业务需要在会前页完成设备选择，建议把最终选择结果同步给正式入会流程。

## 验证矩阵
| 层级 | 检查项 | 验证手段 | 预期结果 |
|------|--------|----------|---------|
| 1. 编译级 | 已导入 `useDeviceState` | 检查 `import` 语句 | 会前检测 Hook 可解析 |
| 2. 静态规则级 | 预览与测试都有显式启动 / 停止逻辑 | 搜索 `startCameraTest` / `stopCameraTest` / `startMicrophoneTest` | 形成完整自检链路 |
| 3. 运行时级 | 摄像头与麦克风测试可执行 | 打开会前检测页并授权设备 | 能看到预览或音量变化 |
| 4. 业务行为级 | 用户可在入会前完成自检 | 走完整个会前页流程 | 检测完成后可顺畅进入会议 |
