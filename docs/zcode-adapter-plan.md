# ZCode 适配器实现规划

## 一、数据格式分析

### 1.1 目录结构

```
~/.zcode\                                     # Windows: C:\Users\{user}\.zcode
└── cli\
    ├── db\
    │   └── db.sqlite                         # 会话主存储（SQLite, WAL 模式）
    └── rollout\
        └── model-io-sess_<uuid>.jsonl        # 按会话的模型 IO 轨迹（调试用，非主存储）
```

### 1.2 数据库表结构

| 表 | 关键列 | 说明 |
|----|--------|------|
| `session` | `id`(sess_uuid)、`directory`(工作区路径)、`title`、`parent_id`、`time_created`/`time_updated`(毫秒)、`time_archived` | `parent_id` 非空的为子代理会话，列表时过滤 |
| `message` | `id`(msg_xxx)、`session_id`、`data`(JSON)、`sequence` | `data.role`: `user`/`assistant` |
| `part` | `id`(part_xxx)、`message_id`、`session_id`、`data`(JSON)、`sequence`、`time_created` | 消息内容层 |

注意：**`part.sequence` 是消息内序号（每条消息归零）**，时间线排序必须按
`message.time_created → part.time_created → id`，不能直接用 part.sequence。

### 1.3 part.data 类型

| type | 结构 | 适配器处理 |
|------|------|-----------|
| `text` | `{ text, time }` | 按所属消息 role 渲染为 user/assistant 块 |
| `reasoning` | `{ text, metadata, time }` | `thinking` 块（同 OpenCode 适配器惯例），可编辑 |
| `tool` | `{ callID, tool, state: { status, input, output, error } }` | 合并进本轮回答的 assistant 块 `toolCalls` |
| `step-start` / `step-finish` | 步骤边界与 token 统计 | 跳过 |
| `file` | 图片/文件附件（`zcode-artifact://` URL） | 跳过 |
| `timeline` | 时间线事件 | 跳过 |

### 1.4 合成 user 消息过滤

`message.data.semantics.kind` 区分消息来源：

- `user_prompt` — 真实用户输入
- `todo_reminder` / `background_notification` — 运行时注入的 user 角色消息（量大）

user 消息仅在 `semantics.kind` 缺失或等于 `user_prompt` 时渲染，避免时间线被噪音灌满。
assistant 消息不做 kind 过滤（99% 为 `assistant_response`，其余无 text part）。

## 二、适配器实现（已实现）

| 项 | 说明 |
|----|------|
| 平台 ID | `zcode` |
| 根路径 | `settings.zcodeHome`，默认 `~/.zcode`，DB 固定在 `cli/db/db.sqlite` |
| 会话 Key | `session.id`（全局唯一，无工作区前缀） |
| 列表 | `parent_id IS NULL OR ''`，按 `time_updated DESC`；`preview` 用会话标题 |
| 详情 | JOIN part + message，按时间排序；text → 消息 role 块；reasoning → thinking；tool → 挂到本轮 text 块，结尾未认领的冲刷成独立 assistant 块（同 claude/kiro-ide 模式） |
| 编辑 | `edit_target` 即 `part.id`；`text`/`reasoning` 改 `data.text`，`tool` 改 `state.output` |
| 搜索 | text/reasoning 正文 + tool 名称/输入输出 |
| 工具擦除 | 未实现（前端 erase 仅对 opencode 开放），trait 默认返回 Err |

## 三、已知限制

| 限制 | 说明 |
|------|------|
| 只读 `rollout/` | 模型 IO 轨迹未接入，仅作为后续 JSONL 导入/导出的候选来源 |
| 图片附件 | `file` part 的 `zcode-artifact://` URL 无法在时间线渲染，跳过 |
| 并发写入 | ZCode 运行中编辑 SQLite 理论上可能与 ZCode 自身写入竞争；WAL 模式下读写不互斥，风险可接受 |
