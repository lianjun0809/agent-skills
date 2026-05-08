---
id: conference/network-quality
platform: web
api_docs:
  - title: 设备及网络
    url: https://cloud.tencent.com/document/product/647/126923
---

# 网络质量 — Web 实现

## 前置条件
**通用依赖**：见 [login-auth 平台 slice](../login-auth.md)。

**额外依赖**：
- 已安装 `tuikit-atomicx-vue3@latest`

**前置状态**：
- 已阅读 `conference/network-quality`，明确当前能力的产品边界。
- 已完成 `conference/login-auth`，确保当前页面具备稳定登录态。
- 已根据业务流程接入会议上下文；需要房间状态时，优先通过 `conference/room-lifecycle` 统一承接。

## 代码示例
### 基础接入：把网络质量映射成可读文案

```ts
import { computed } from 'vue';
import { useDeviceState, NetworkQuality } from 'tuikit-atomicx-vue3/room';

const { networkInfo } = useDeviceState();

const networkText = computed(() => {
  switch (networkInfo.value?.quality) {
    case NetworkQuality.Excellent: return '优秀';
    case NetworkQuality.Good: return '良好';
    case NetworkQuality.Poor: return '一般';
    default: return '未知';
  }
});
```

## 调用时序
```
完成 login-auth 并进入会议
   │
   ▼
读取 useDeviceState().networkInfo
   │
   ├─ quality 优秀 / 良好 → 展示常规状态
   ├─ quality 一般 / 较差 → 展示弱网提示与恢复建议
   └─ 数据为空 → 等待进房与媒体状态稳定后再展示
   │
   ▼
结合 room-lifecycle 的重连 / 离会状态做统一反馈
```

## 平台特有注意事项
### 1. `networkInfo` 通常在进房且媒体链路建立后更稳定
仅完成登录但未进房时，网络质量数据往往不完整；展示层应允许初始为空或未知状态。

### 2. 弱网提示应与房间生命周期联动
如果只在顶部展示一个图标，却不联动重连、掉线、离会等状态，用户很难理解当前问题来源。

### 3. 网络状态适合做常驻感知，不适合做高频弹窗
建议把状态放到顶部工具栏、布局状态栏或参会者信息区，避免高频通知打断会议流程。

## 代码生成约束
### 编译必要条件
- **通用条件**：见 [login-auth 平台 slice](../login-auth.md)。
- **额外导入**：至少需要导入 `useDeviceState`；如需映射等级，按需导入 `NetworkQuality`。
- **运行前提**：页面已进入会议流程，网络状态来源可用。

### 生成规则
#### MUST（生成时必须包含）

1. **通过 `networkInfo` 作为网络质量单一状态源** — 可避免 UI 与底层状态割裂。  
   **Verify**: 检查是否存在 `useDeviceState()` 与 `networkInfo` 读取逻辑。
2. **把质量等级映射为用户可理解的文案或图标** — 直接暴露枚举值不利于会议中快速感知。  
   **Verify**: 检查是否存在 `computed` / 映射表 / 文案转换逻辑。

#### MUST NOT（生成时绝不能出现）

1. **不要把弱网提示做成与房间状态无关的孤立展示** — 用户无法判断是否需要重试或重连。  
   **Verify**: 检查是否与 `room-lifecycle` 或页面状态联动。
2. **不要假设 `networkInfo` 在所有阶段都立即可用** — 初始空值必须可处理。  
   **Verify**: 检查是否对空值或未知状态做了兜底。

### 集成检查点
- 当前 slice 常与 `conference/video-layout`、`conference/room-lifecycle` 联动展示。
- 一般只需要新增状态条、角标或弱网提示组件，侵入性较低。
- 若业务侧已有独立网络探测能力，需要明确展示优先级，避免两套网络状态互相矛盾。

## 验证矩阵
| 层级 | 检查项 | 验证手段 | 预期结果 |
|------|--------|----------|---------|
| 1. 编译级 | 已导入 `useDeviceState` / `NetworkQuality` | 检查 `import` 语句 | 网络状态相关导入可解析 |
| 2. 静态规则级 | 网络枚举被映射为可读状态 | 搜索 `networkInfo` 和文案映射逻辑 | 存在从枚举到 UI 的转换 |
| 3. 运行时级 | 进房后可观测到网络状态变化 | 在会议中查看弱网提示区域 | 可看到网络等级或未知兜底 |
| 4. 业务行为级 | 弱网时用户能获得清晰反馈 | 模拟网络波动或弱网环境 | 页面出现明确的状态感知与提示 |
