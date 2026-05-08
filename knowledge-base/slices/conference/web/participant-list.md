---
id: conference/participant-list
platform: web
api_docs:
  - title: 成员管理
    url: https://cloud.tencent.com/document/product/647/126927
---

# 参会人列表与状态 — Web 实现

## 前置条件
**通用依赖**：见 [login-auth 平台 slice](../login-auth.md)。

**额外依赖**：
- 已安装 `tuikit-atomicx-vue3@latest`

**前置状态**：
- 已阅读 `conference/participant-list`，明确当前能力的产品边界。
- 已完成 `conference/login-auth`，确保当前页面具备稳定登录态。
- 已根据业务流程接入会议上下文；需要房间状态时，优先通过 `conference/room-lifecycle` 统一承接。

## 代码示例
### 基础接入：拉取参会人列表并按游标分页

```ts
import { useRoomParticipantState } from 'tuikit-atomicx-vue3/room';

const { participantList, participantListCursor, getParticipantList, speakingUsers } = useRoomParticipantState();

await getParticipantList({ cursor: '' });
if (participantListCursor.value) {
  await getParticipantList({ cursor: participantListCursor.value });
}
console.log(participantList.value, speakingUsers.value);
```

## 调用时序
```
完成 login-auth 并进入会议
   │
   ▼
调用 getParticipantList({ cursor: '' })
   │
   ├─ 首次成功 → 渲染 participantList
   ├─ 存在下一页 → 使用 participantListCursor 继续拉取
   └─ 失败 → 提示加载失败并允许重试
   │
   ▼
结合 speakingUsers / metaData / 角色态更新列表 UI
```

## 平台特有注意事项
### 1. 成员列表与管理动作应分层
成员列表负责展示“谁在会里、状态如何”；踢人、设管理员、转移房主等动作应继续交给 `conference/participant-management`。

### 2. 分页游标需要持续复用
当参会人数较多时，列表不应假设一次性全量返回；应根据 `participantListCursor` 持续分页拉取。

### 3. 业务扩展字段建议走 `metaData`
如果要展示身份标签、业务等级或组织信息，建议通过 `metaData` 同步，而不是在 UI 层拼接一套旁路数据源。

## 代码生成约束
### 编译必要条件
- **通用条件**：见 [login-auth 平台 slice](../login-auth.md)。
- **额外导入**：至少需要导入 `useRoomParticipantState`。
- **运行前提**：已进入会议，房间上下文可用。

### 生成规则
#### MUST（生成时必须包含）

1. **通过 `useRoomParticipantState` 承接成员列表数据** — 可以保持列表、发言态与角色信息同源。  
   **Verify**: 检查是否存在 `useRoomParticipantState()`。
2. **支持首屏加载与游标续拉** — 成员列表在多人场景下不能只写死首屏一次调用。  
   **Verify**: 检查是否存在 `getParticipantList({ cursor })` 与 `participantListCursor` 逻辑。

#### MUST NOT（生成时绝不能出现）

1. **不要把成员列表组件直接耦合管理操作权限** — 会让展示层和治理层边界混乱。  
   **Verify**: 检查踢人/转移房主逻辑是否放在独立治理能力中。
2. **不要忽略分页游标** — 在大房间场景会导致成员显示不完整。  
   **Verify**: 检查是否读取 `participantListCursor` 或提供加载更多逻辑。

### 集成检查点
- 当前 slice 常与 `conference/participant-management`、`conference/video-layout` 联动。
- 集成侵入性较低，通常新增一个侧栏、抽屉或列表区域即可。
- 若业务已有企业通讯录或组织树，需要明确“在线成员列表”和“组织架构列表”的来源边界。

## 验证矩阵
| 层级 | 检查项 | 验证手段 | 预期结果 |
|------|--------|----------|---------|
| 1. 编译级 | 已导入 `useRoomParticipantState` | 检查 `import` 语句 | 成员状态 Hook 可解析 |
| 2. 静态规则级 | 存在首屏加载与分页逻辑 | 搜索 `getParticipantList` 与 `participantListCursor` | 形成分页加载链路 |
| 3. 运行时级 | 成员列表能正常展示与刷新 | 进房后打开成员列表 | 可看到参会人和状态信息 |
| 4. 业务行为级 | 人数增多时列表仍完整 | 多人会议中滚动加载更多 | 列表可持续补齐更多成员 |
