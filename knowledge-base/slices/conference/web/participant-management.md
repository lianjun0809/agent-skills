---
id: conference/participant-management
platform: web
api_docs:
  - title: 成员管理
    url: https://cloud.tencent.com/document/product/647/126927
---

# 参会人管理与角色治理 — Web 实现

## 前置条件
**通用依赖**：见 [login-auth 平台 slice](../login-auth.md)。

**额外依赖**：
- 已安装 `tuikit-atomicx-vue3@latest`

**前置状态**：
- 已阅读 `conference/participant-management`，明确当前能力的产品边界。
- 已完成 `conference/login-auth`，确保当前页面具备稳定登录态。
- 已根据业务流程接入会议上下文；需要房间状态时，优先通过 `conference/room-lifecycle` 统一承接。

## 代码示例
### 管理操作：设管理员、撤销管理员、转移房主、移出成员

```ts
import { useRoomParticipantState } from 'tuikit-atomicx-vue3/room';

const { setAdmin, revokeAdmin, transferOwner, kickParticipant } = useRoomParticipantState();

await setAdmin({ userId: 'user_002' });
await revokeAdmin({ userId: 'user_002' });
await transferOwner({ userId: 'user_003' });
await kickParticipant({ userId: 'user_004' });
```

## 调用时序
```
完成 login-auth 并进入会议
   │
   ▼
读取 localParticipant.role 判断当前用户权限
   │
   ├─ 房主 / 管理员 → 展示治理入口
   └─ 普通成员 → 仅展示只读状态
   │
   ▼
用户执行 setAdmin / revokeAdmin / transferOwner / kickParticipant
   │
   ├─ 成功 → 刷新成员角色与 UI
   └─ 失败 → 提示权限不足或目标状态已变化
```

## 平台特有注意事项
### 1. 权限判断要前后端双重收口
前端应先隐藏无权限操作提升体验，但真正的权限校验仍要依赖 SDK / 服务端结果，不能只信任页面按钮状态。

### 2. 房主转移会影响后续治理边界
一旦执行 `transferOwner()`，结束会议、修改规则、全员会控等权限边界都会变化，页面状态必须同步切换。

### 3. 成员管理适合与业务角色标签联动
若业务有“主持人 / 嘉宾 / 观察员”等角色，建议通过 `metaData` 与会议治理能力协同，而不是在 UI 上维护一套脱节状态。

## 代码生成约束
### 编译必要条件
- **通用条件**：见 [login-auth 平台 slice](../login-auth.md)。
- **额外导入**：至少需要导入 `useRoomParticipantState`。
- **运行前提**：当前用户已在会议内，且具备足够角色权限。

### 生成规则
#### MUST（生成时必须包含）

1. **在展示治理动作前先读取本地角色状态** — 这样 UI 才能与权限边界一致。  
   **Verify**: 检查是否存在基于角色的显示判断。
2. **通过 `useRoomParticipantState` 执行治理动作** — 可保证成员状态与角色变化统一收口。  
   **Verify**: 检查是否存在 `setAdmin` / `revokeAdmin` / `transferOwner` / `kickParticipant` 调用。

#### MUST NOT（生成时绝不能出现）

1. **不要把角色治理动作混进纯展示型成员列表组件** — 会破坏列表层与治理层边界。  
   **Verify**: 检查治理逻辑是否独立于只读展示层。
2. **不要只在前端做权限判断而忽略失败回调处理** — 当目标状态变化或权限不足时会直接出错。  
   **Verify**: 检查是否存在错误提示或失败兜底。

### 集成检查点
- 当前 slice 常与 `conference/participant-list`、`conference/room-moderation` 配合使用。
- 集成方式通常是新增管理菜单、成员卡片操作或角色设置弹窗。
- 如果业务存在组织权限体系，需要把会议角色与业务角色的映射规则明确写在上层页面逻辑中。

## 验证矩阵
| 层级 | 检查项 | 验证手段 | 预期结果 |
|------|--------|----------|---------|
| 1. 编译级 | 已导入 `useRoomParticipantState` | 检查 `import` 语句 | 治理 Hook 可解析 |
| 2. 静态规则级 | 存在基于角色的入口显示控制 | 搜索角色判断或权限判断逻辑 | 无权限用户不显示治理操作 |
| 3. 运行时级 | 角色治理动作可执行并反馈结果 | 在房主 / 管理员账号下执行操作 | 成功后成员状态刷新，失败时有提示 |
| 4. 业务行为级 | 权限边界与 UI 一致 | 切换不同角色账号验证页面 | 展示和可执行动作符合角色权限 |
