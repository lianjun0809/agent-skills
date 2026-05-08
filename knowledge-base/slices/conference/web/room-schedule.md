---
id: conference/room-schedule
platform: web
api_docs:
  - title: 预定房间
    url: https://cloud.tencent.com/document/product/647/126931
---

# 预约会议 — Web 实现

## 前置条件
**通用依赖**：见 [login-auth 平台 slice](../login-auth.md)。

**额外依赖**：
- 已安装 `tuikit-atomicx-vue3@latest`

**前置状态**：
- 已阅读 `conference/room-schedule`，明确当前能力的产品边界。
- 已完成 `conference/login-auth`，确保当前页面具备稳定登录态。
- 已根据业务流程接入会议上下文；需要房间状态时，优先通过 `conference/room-lifecycle` 统一承接。

## 代码示例
### 预约与取消：创建未来会议并分页读取列表

```ts
import { useRoomState } from 'tuikit-atomicx-vue3/room';

const { scheduleRoom, getScheduledRoomList, cancelScheduledRoom } = useRoomState();
const startTime = Math.floor(Date.now() / 1000) + 3600;

await scheduleRoom({
  roomId: 'schedule_room',
  options: {
    roomName: '周会',
    scheduleStartTime: startTime,
    scheduleEndTime: startTime + 1800,
    scheduleAttendees: ['user_a', 'user_b'],
  },
});

await getScheduledRoomList({ cursor: '' });
await cancelScheduledRoom({ roomId: 'schedule_room' });
```

## 调用时序
```
完成 login-auth
   │
   ▼
准备预约时间与房间信息
   │
   ▼
scheduleRoom({ roomId, options })
   │
   ├─ 成功 → 写入预约列表
   ├─ 需要查看更多 → getScheduledRoomList({ cursor })
   └─ 用户取消预约 → cancelScheduledRoom({ roomId })
   │
   ▼
会议临近时结合提醒事件与 room-lifecycle 引导入会
```

## 平台特有注意事项
### 1. 时间参数通常使用秒级时间戳
Web 业务常用毫秒时间戳，但预约会议能力通常要求秒级时间戳；传参前必须显式转换。

### 2. `roomId` 更适合由业务后台生成并保证唯一性
预约会议的 `roomId` 最好由业务后台统一生成或分配，避免未来会议与即时会议在不同入口下发生标识冲突。

### 3. 预约时可同步写入 `scheduleAttendees`
如果业务希望把会议信息同步到参会人的会议列表或提醒入口，应在预约阶段明确传入受邀参会人集合，而不是依赖后续临时补录。

### 4. 预约列表通常需要分页拉取
不要默认一次返回所有预约记录；应根据返回的 `cursor` 持续拉取后续列表数据。

### 5. 预约会议最终仍要回到房间生命周期
无论预约由前端创建还是后端排期系统预生成，临近开会时都应回到 `conference/room-lifecycle` 的真实入会链路。

## 代码生成约束
### 编译必要条件
- **通用条件**：见 [login-auth 平台 slice](../login-auth.md)。
- **额外导入**：至少需要导入 `useRoomState`。
- **运行前提**：当前用户具备预约会议权限，且时间参数经过单位校验。

### 生成规则
#### MUST（生成时必须包含）

1. **在预约创建时显式传入秒级时间戳** — 时间单位错误会直接导致会议时间错位。  
   **Verify**: 检查是否存在 `Math.floor(Date.now() / 1000)` 或等价转换。
2. **在预约场景中明确会议标识与参会人集合** — 未来会议记录应能稳定映射到唯一房间和对应参会人入口。  
   **Verify**: 检查是否提供稳定 `roomId`，并在需要时传入 `scheduleAttendees`。
3. **支持预约列表分页读取** — 长列表场景不能只写死第一页。  
   **Verify**: 检查是否存在 `getScheduledRoomList({ cursor })` 调用。

#### MUST NOT（生成时绝不能出现）

1. **不要直接把毫秒时间戳传给预约接口** — 会议开始结束时间会错乱。  
   **Verify**: 检查时间值是否经过秒级转换。
2. **不要把预约成功等价为已经进入会议** — 预约与真实入会是两条不同链路。  
   **Verify**: 检查是否仍通过 `room-lifecycle` 承接正式开会。

### 集成检查点
- 当前 slice 常与 `conference/room-config`、`conference/room-lifecycle`、业务提醒系统联动。
- 集成方式通常是新增日程表单、预约列表和提醒入口。
- 如果业务已有企业日历或排期系统，需要提前约定字段映射与同步策略。

## 验证矩阵
| 层级 | 检查项 | 验证手段 | 预期结果 |
|------|--------|----------|---------|
| 1. 编译级 | 已导入 `useRoomState` | 检查 `import` 语句 | 预约相关 API 可解析 |
| 2. 静态规则级 | 时间参数、`roomId` 与参会人集合处理正确 | 搜索 `Date.now()`、`roomId`、`scheduleAttendees` | 传参为秒级时间戳，会议标识稳定 |
| 3. 运行时级 | 可创建、读取和取消预约会议 | 在预约页面走完整流程 | 预约列表与取消结果正确 |
| 4. 业务行为级 | 临近会议时可顺畅跳转入会 | 从预约记录点击进入会议 | 进入正式入会链路而非停留在预约状态 |
