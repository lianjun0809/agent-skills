---
id: conference/virtual-background
platform: web
api_docs:
  - title: 虚拟背景
    url: https://cloud.tencent.com/document/product/647/126935
---

# 虚拟背景 — Web 实现

## 前置条件
**通用依赖**：见 [login-auth 平台 slice](../login-auth.md)。

**额外依赖**：
- 已安装 `tuikit-atomicx-vue3@latest`

**前置状态**：
- 已阅读 `conference/virtual-background`，明确当前能力的产品边界。
- 已完成 `conference/login-auth`，确保当前页面具备稳定登录态。
- 已根据业务流程接入会议上下文；需要房间状态时，优先通过 `conference/room-lifecycle` 统一承接。
- 当前能力涉及媒体采集、渲染或浏览器权限时，请在 `HTTPS` 或 `localhost` 安全上下文下调试。

## 代码示例
### 初始化模型资源并保存虚拟背景效果

```ts
import { onMounted } from 'vue';
import { useVirtualBackgroundState } from 'tuikit-atomicx-vue3/room';

const { isSupported, initVirtualBackground, setVirtualBackground, saveVirtualBackground } = useVirtualBackgroundState();

onMounted(async () => {
  if (!isSupported()) return;
  await initVirtualBackground({ assetsPath: 'https://cdn.example.com/assets' });
  await setVirtualBackground({ enable: true, type: 'blur', level: 0.5 });
  await saveVirtualBackground();
});
```

## 调用时序
```
完成 login-auth 并准备本地视频
   │
   ▼
调用 isSupported() 检查浏览器能力
   │
   ├─ 不支持 → 隐藏入口或走降级文案
   └─ 支持
       │
       ▼
initVirtualBackground({ assetsPath })
       │
       ▼
setVirtualBackground(...) 做预览
       │
       ├─ 用户确认 → saveVirtualBackground()
       └─ 用户取消 / 关闭 → 回退或关闭背景效果
```

## 平台特有注意事项
### 1. `assetsPath` 是虚拟背景能否启动的关键前置条件
必须先成功初始化 `assetsPath` 对应的模型或资源目录，否则虚拟背景功能无法正常工作。

### 2. 兼容性检查必须放在渲染前
浏览器对虚拟背景的支持差异明显，建议在页面入口或打开设置面板前先调用 `isSupported()` 做能力判断。

### 3. 虚拟背景比基础美颜更吃性能
在低性能设备上，虚拟背景更容易带来 CPU / GPU 压力；业务应允许快速关闭，并提供清晰的降级说明。

## 代码生成约束
### 编译必要条件
- **通用条件**：见 [login-auth 平台 slice](../login-auth.md)。
- **额外导入**：至少需要导入 `useVirtualBackgroundState`。
- **运行前提**：浏览器支持该能力、页面处于安全上下文，且 `assetsPath` 可访问。

### 生成规则
#### MUST（生成时必须包含）

1. **在启用虚拟背景前先做能力检查与初始化** — 否则会出现入口可见但能力不可用的假状态。  
   **Verify**: 检查是否存在 `isSupported()` 与 `initVirtualBackground(`。
2. **把预览与最终保存分开处理** — 用户需要有确认生效的动作，而不是一改即永久保存。  
   **Verify**: 检查是否同时存在 `setVirtualBackground(` 与 `saveVirtualBackground(`。

#### MUST NOT（生成时绝不能出现）

1. **不要跳过 `assetsPath` 初始化直接启用背景能力** — 功能无法正常启动。  
   **Verify**: 检查是否先调用 `initVirtualBackground({ assetsPath })`。
2. **不要忽略不支持浏览器的降级路径** — 低兼容环境会直接报错或白屏。  
   **Verify**: 检查是否存在 `isSupported()` 判断和降级处理。

### 集成检查点
- 当前 slice 常与 `conference/device-control`、`conference/beauty-effects` 联动。
- 集成方式通常是新增本地效果设置面板，不需要改动房间生命周期逻辑。
- 如果业务同时接入美颜和虚拟背景，建议在 UI 和性能策略上给出统一入口与优先级。

## 验证矩阵
| 层级 | 检查项 | 验证手段 | 预期结果 |
|------|--------|----------|---------|
| 1. 编译级 | 已导入 `useVirtualBackgroundState` | 检查 `import` 语句 | 虚拟背景 Hook 可解析 |
| 2. 静态规则级 | 存在能力检查、初始化、保存链路 | 搜索 `isSupported` / `initVirtualBackground` / `saveVirtualBackground` | 形成完整启用流程 |
| 3. 运行时级 | 支持浏览器可成功启用背景效果 | 在支持环境打开虚拟背景功能 | 能看到预览并保存效果 |
| 4. 业务行为级 | 不支持或低性能环境有清晰降级 | 在不支持环境或低性能设备联调 | 页面能正确隐藏入口或提示降级 |
