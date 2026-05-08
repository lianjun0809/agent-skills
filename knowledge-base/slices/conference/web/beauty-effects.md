---
id: conference/beauty-effects
platform: web
api_docs:
  - title: 基础美颜
    url: https://cloud.tencent.com/document/product/647/126937
---

# 美颜效果 — Web 实现

## 前置条件
**通用依赖**：见 [login-auth 平台 slice](../login-auth.md)。

**额外依赖**：
- 已安装 `tuikit-atomicx-vue3@latest`

**前置状态**：
- 已阅读 `conference/beauty-effects`，明确当前能力的产品边界。
- 已完成 `conference/login-auth`，确保当前页面具备稳定登录态。
- 已根据业务流程接入会议上下文；需要房间状态时，优先通过 `conference/room-lifecycle` 统一承接。
- 当前能力涉及媒体采集、渲染或浏览器权限时，请在 `HTTPS` 或 `localhost` 安全上下文下调试。

## 代码示例
### 基础接入：实时预览并显式保存美颜设置

```ts
import { useFreeBeautyState } from 'tuikit-atomicx-vue3/room';

const { setFreeBeauty, saveBeautySetting } = useFreeBeautyState();

await setFreeBeauty({ beautyLevel: 60, whitenessLevel: 40, ruddinessLevel: 20 });
await saveBeautySetting();
```

## 调用时序
```
完成 login-auth 并打开本地视频
   │
   ▼
进入美颜设置面板
   │
   ▼
调用 setFreeBeauty(...) 做本地预览
   │
   ├─ 用户继续调节 → 重复预览
   ├─ 用户取消 → 回退未保存的本地预览
   └─ 用户确认 → saveBeautySetting()
   │
   ▼
远端看到最终保存后的美颜效果
```

## 平台特有注意事项
### 1. `setFreeBeauty()` 主要作用于本地预览
只有在调用 `saveBeautySetting()` 后，当前选择才会作为最终效果稳定生效；不要把两者当成同一步。

### 2. 无摄像头或摄像头关闭时应禁用入口
美颜直接依赖本地视频流，若摄像头未开启，则继续暴露调节入口会产生无效操作体验。

### 3. 低性能设备要预留降级路径
美颜会增加本地渲染负担，低性能终端可能出现掉帧、发热或风扇噪声上升，业务层应允许快速关闭。

## 代码生成约束
### 编译必要条件
- **通用条件**：见 [login-auth 平台 slice](../login-auth.md)。
- **额外导入**：至少需要从 `tuikit-atomicx-vue3/room` 导入 `useFreeBeautyState`。
- **运行前提**：本地摄像头可用，且页面已处于安全上下文。

### 生成规则
#### MUST（生成时必须包含）

1. **使用 `useFreeBeautyState` 承接美颜状态与保存动作** — 这样 UI 预览与最终生效状态才一致。  
   **Verify**: 检查是否存在 `useFreeBeautyState()`。
2. **将预览与保存拆成两段链路** — 否则用户每次调节参数都可能直接影响最终效果。  
   **Verify**: 检查是否同时存在 `setFreeBeauty(` 与 `saveBeautySetting(`。

#### MUST NOT（生成时绝不能出现）

1. **不要在无本地视频的前提下强行启用美颜操作** — 会造成空操作或异常体验。  
   **Verify**: 检查入口是否与摄像头状态联动。
2. **不要把滑杆变化直接等价为永久保存** — 会破坏“先预览、再确认”的交互预期。  
   **Verify**: 检查参数变化处理是否无条件调用 `saveBeautySetting()`。

### 集成检查点
- 当前 slice 常与 `conference/device-control`、`conference/video-layout` 联动。
- 通常只需要新增一个设置面板或弹层，不应修改视频渲染底层实现。
- 若业务还接入了 `conference/virtual-background`，需明确二者在 UI 上的优先级和入口关系。

## 验证矩阵
| 层级 | 检查项 | 验证手段 | 预期结果 |
|------|--------|----------|---------|
| 1. 编译级 | 已导入 `useFreeBeautyState` | 检查 `import` 语句 | 美颜 Hook 可解析 |
| 2. 静态规则级 | 预览与保存动作分离 | 搜索 `setFreeBeauty` 与 `saveBeautySetting` | 两段链路同时存在 |
| 3. 运行时级 | 参数调整后可看到本地效果变化 | 开启摄像头并调节美颜参数 | 本地预览立即变化 |
| 4. 业务行为级 | 用户确认后效果稳定生效 | 调整参数并点击保存 | 最终效果符合保存结果 |
