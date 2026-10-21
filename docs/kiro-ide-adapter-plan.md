# Kiro IDE 适配器实现规划

## 一、数据格式分析

### 1.1 目录结构

```
%APPDATA%\Kiro\User\
├── globalStorage\
│   └── kiro.kiroagent\
│       └── workspace-sessions\
│           ├── {base64(workspace_path)}\     # 每个工作区一个文件夹
│           │   ├── sessions.json             # 该工作区的会话索引
│           │   └── {session-uuid}.json       # 会话完整历史（单个大JSON）
│           └── ...
└── workspaceStorage\
    └── {workspace_hash}\
        └── state.vscdb                        # SQLite，存储 kiro.kiroAgent 等状态

```

### 1.2 工作区目录命名规则

目录名为 **Base64 编码的工作区路径**：

```
e:\workspace\20260402\vk-study-v4
  ↓ Base64
ZTpcd29ya3NwYWNlXDIwMjYwNDAyXHZrLXN0dWR5LXY0
```

### 1.3 sessions.json 格式

```json
[
  {
    "sessionId": "967d1224-cacd-4e44-b2d2-12b2fc769ac1",
    "title": "\\(^o^)/~",
    "dateCreated": "1776149767446",
    "workspaceDirectory": "e:\\workspace\\20260402\\vk-study-v4"
  }
]
```

- `dateCreated`：毫秒时间戳字符串
- `workspaceDirectory`：Windows 路径用 `\\` 分隔

### 1.4 Session JSON 格式

文件名：`{session-uuid}.json`

```json
{
  "history": [
    {
      "message": {
        "role": "user",
        "content": [
          {
            "type": "text",
            "text": "消息内容"
          }
        ],
        "id": "65da8818-ff5d-4b81-b29c-9361dc8924af"
      },
      "contextItems": [],
      "editorState": { ... }
    },
    {
      "message": {
        "role": "assistant",
        "content": [
          {
            "type": "text",
            "text": "回复内容"
          }
        ],
        "id": "..."
      }
    }
  ]
}
```

### 1.5 关键差异点（对比 Kiro CLI）

| 项目 | Kiro CLI | Kiro IDE |
|------|----------|----------|
| 路径 | `~/.kiro/sessions/cli/` | `%APPDATA%\Kiro\User\globalStorage\kiro.kiroagent\workspace-sessions\` |
| 文件格式 | `.json`（元数据）+ `.jsonl`（消息流） | **单个大 JSON 文件** |
| 组织方式 | 扁平目录 | **按工作区分组（Base64 编码目录名）** |
| 消息格式 | `kind: "Prompt"/"AssistantMessage"`, `data.content[].kind/data` | **`role: "user"/"assistant"`, `content[].type/text`** |
| 会话索引 | 无，直接扫描 `.json` 文件 | **`sessions.json` 索引文件** |
| CWD | 元数据 JSON 中 `cwd` 字段 | **`sessions.json` 中 `workspaceDirectory`** |

---

## 二、需要修改的文件清单

### 2.1 新增文件

| 文件 | 说明 |
|------|------|
| `src-tauri/src/platforms/kiro_ide.rs` | Kiro IDE 适配器实现 |

### 2.2 修改文件

| 文件 | 改动内容 |
|------|----------|
| `src-tauri/src/platforms/mod.rs` | 注册 `KiroIdePlatform`，在 `get_adapter()` 加 `"kiro-ide"` 分支 |
| `src-tauri/src/session_service.rs` | `dashboard_summary()` 循环加 `"kiro-ide"` |
| `src-tauri/src/settings.rs` | `AppSettings` 加 `kiro_ide_home: Option<String>`；`AppSettingsPatch` 同步 |
| `src/features/desktop/types.ts` | `DesktopSettings` / `DesktopSettingsPatch` 加 `kiroIdeHome` |
| `src/app/routes/settings.tsx` | 平台路径区块加 `PathRow`（`pickMode="directory"`） |
| `src/features/desktop/i18n/` | 加 `kiroIdeHomePath` 等 i18n key |

---

## 三、适配器实现设计

### 3.1 会话 Key 设计

```
{workspace_dir_base64}::{session_uuid}
// 例：ZTpcd29ya3NwYWNlXDIwMjYwNDAyXHZrLXN0dWR5LXY0::967d1224-cacd-4e44-b2d2-12b2fc769ac1
```

- `workspace_dir_base64`：工作区目录的 Base64 编码（与文件夹名一致）
- `session_uuid`：会话 UUID

### 3.2 list_sessions 逻辑

```
1. 确定 kiro_ide_home（优先 settings.kiro_ide_home，否则 %APPDATA%\Kiro\User\globalStorage\kiro.kiroagent）
2. 扫描 workspace-sessions/ 下所有子目录
3. 对每个工作区目录：
   a. 读取 sessions.json 获取会话列表
   b. 对每个会话：
      - 读取 {uuid}.json 的 history[0].message.content[0].text 作为 preview
      - 从 sessions.json 获取 title、dateCreated、workspaceDirectory
4. 合并所有工作区的会话，按 dateCreated 降序排列
5. 应用 alias_map、limit、offset
```

### 3.3 get_session_detail 逻辑

```
1. 从 session_key 解析 workspace_dir_base64 + session_uuid
2. 定位文件：{kiro_ide_home}/workspace-sessions/{workspace_dir_base64}/{session_uuid}.json
3. 读取 sessions.json 获取 title、workspaceDirectory（cwd）
4. 解析 history[]，转换为 TimelineBlock[]：
   - role="user"      → role=User,      content = message.content[0].text
   - role="assistant" → role=Assistant, content = message.content[0].text
5. edit_target 格式：{session_key}::{message_id}
```

### 3.4 update_message 逻辑

```
1. 解析 edit_target：session_key + message_id
2. 定位 JSON 文件
3. 读取 JSON → 找到 history 中 message.id == message_id 的条目
4. 更新 message.content[0].text = new_content
5. 写回文件（serde_json::to_string_pretty）
```

### 3.5 content_search 逻辑

```
1. 读取会话 JSON 文件
2. 遍历 history[]，提取 message.content[0].text
3. 搜索匹配 query 的消息
4. 返回 ContentMatch[]（snippet + match_index + role）
```

---

## 四、设置项

### 4.1 新增 settings 字段

```rust
// settings.rs - AppSettings
#[serde(default)]
pub kiro_ide_home: Option<String>,   // 默认: %APPDATA%\Kiro\User\globalStorage\kiro.kiroagent
```

默认行为：
- Windows: `%APPDATA%\Kiro\User\globalStorage\kiro.kiroagent`
- macOS/Linux: `~/.config/Kiro/User/globalStorage/kiro.kiroagent`（待确认）

### 4.2 前端 PathRow 配置

```tsx
<PathRow
  label={t("kiroIdeHomePath")}
  defaultHint="%APPDATA%\Kiro\User\globalStorage\kiro.kiroagent"
  pickMode="directory"
  value={snapshot.settings.kiroIdeHome ?? ""}
  onSave={(v) => updateSettings({ kiroIdeHome: v || null })}
/>
```

---

## 五、i18n Key 清单

需在中英文资源文件中新增：

| Key | 中文 | English |
|-----|------|---------|
| `kiroIdeHomePath` | Kiro IDE 数据目录 | Kiro IDE Home Path |
| `platform.kiro-ide` | Kiro IDE | Kiro IDE |

---

## 六、实现顺序

1. **`settings.rs`** — 加 `kiro_ide_home` 字段
2. **`kiro_ide.rs`** — 核心适配器
3. **`platforms/mod.rs`** — 注册适配器
4. **`session_service.rs`** — dashboard 加 `"kiro-ide"`
5. **前端 types.ts** — 加类型
6. **前端 settings.tsx** — 加 PathRow
7. **i18n** — 加翻译 key
8. **验收测试** — 确认多工作区会话正常列出和编辑

---

## 七、已知风险 & 待确认

| 风险 | 说明 | 处理方案 |
|------|------|----------|
| Base64 解码失败 | 目录名可能不是标准 Base64 | 解码失败时跳过该目录，记录警告 |
| content 数组为空 | `message.content` 可能为空数组 | 检查长度，空数组时 content = "" |
| 大文件性能 | 单个 JSON 文件可能很大（1.6MB+） | list 时只读 history[0]，detail 时全量读取 |
| macOS/Linux 路径 | 未确认非 Windows 平台的数据路径 | 先实现 Windows，其他平台待测试 |
| 工具调用消息 | content 可能包含 tool_use 等类型 | 只处理 `type: "text"`，其他类型跳过 |

---

## 八、2026-09 新版会话布局迁移（已实现）

新版 Kiro IDE 改变了会话存储位置：不再写 `globalStorage/kiro.kiroagent/workspace-sessions/`，
而是统一放到用户主目录的 `.kiro` 下。`globalStorage/kiro.kiroagent` 里只剩哈希命名的
（基本为空的）执行日志目录和 `profile.json`。第一~七章描述的旧布局仍保留兼容扫描。

### 8.1 新目录结构

```
~/.kiro\                                       # Windows: C:\Users\{user}\.kiro
├── sessions\
│   ├── {workspace_hash}\                      # 每个工作区一个文件夹（如 1807085860ef6468）
│   │   └── sess_{uuid}\                       # 每个会话一个文件夹
│   │       ├── session.json                   # 会话元数据
│   │       ├── messages.jsonl                 # 会话消息流（JSONL，一行一个事件）
│   │       └── snapshots\                     # 文件编辑快照
│   ├── _global\                               # 无工作区会话
│   └── cli\                                   # Kiro CLI 会话（归 platforms/kiro.rs 管，IDE 适配器跳过）
└── session-index\
    └── {workspace_hash}.jsonl                 # 索引操作日志 {"op":"add","sessionPath":...,"at":...}
```

注意：`workspace_hash` 是 Kiro 内部哈希，不是 Base64。工作区路径直接从 `session.json`
的 `workspacePaths` 读取，无需反解哈希。`state.vscdb` 仍是 VSCode 框架状态库，与聊天记录无关。

### 8.2 session.json 格式

```json
{
  "schemaVersion": "1.0.0",
  "id": "sess_6329c3e3-ecd6-40de-92b2-8b0d579136f3",
  "title": "sanguo",
  "agentMode": "vibe",
  "workspacePaths": ["e:\\workspace"],
  "rootPaths": ["e:\\workspace"],
  "createdAt": "2026-09-30T05:29:05.836Z",
  "lastModifiedAt": "2026-09-30T05:54:46.759Z",
  "modelId": "claude-opus-5.5",
  "autopilot": true
}
```

### 8.3 messages.jsonl 事件类型

| payload.type | 说明 | 适配器处理 |
|--------------|------|-----------|
| `user` | 用户消息，`content` 为字符串，含 `images`/`documents` | user 块 |
| `assistant` | 助手消息，`operationType`: `Say`（正式回复）/ `Reasoning`（思考）/ `Summary` | `Say`/`Summary` → assistant 块；`Reasoning` 跳过 |
| `tool_call` | 工具调用：`toolCallId`/`toolName`/`args`/`status`/`actionType` | 挂到本轮 assistant 块的 `toolCalls` |
| `tool_result` | 工具结果：`toolCallId`/`content`/`success` | 与 `tool_call` 按 id 合并 |
| `turn_start`/`turn_end` | 回合边界（`executionId`） | `turn_end` 时冲刷未认领的工具调用 |
| `session_start` | 系统提示词（单行可能很大） | 跳过 |
| `session_metadata` | `key`/`value` 状态（如 contextUsage） | 跳过 |
| `usage_summary`/`session_event`/`pending_interaction`/`interaction_resolved`/`sub_agent_start`/`sub_agent_complete`/`tombstone`/`steering_inclusion` | 其他运行时事件 | 跳过 |

消息内每行结构：`{"id", "timestamp", "payload": {...}}`。编辑按事件 `id` 定位
（无 id 时回退 `line:<index>`），替换 `payload.content` 后整文件重写（保持紧凑 JSONL）。

### 8.4 代码改动

| 文件 | 改动 |
|------|------|
| `kiro_ide.rs` | `KiroIdePlatform::new(kiro_home, legacy_agent_home)`；新增新布局扫描/详情/编辑/搜索；工具调用合并逻辑（参照 claude.rs 的 pending → assistant 块模式）；旧版逻辑整体保留为回退 |
| `platforms/mod.rs` | `"kiro-ide"` 分支传两个根：`settings.kiro_home`（默认 `~/.kiro`，与 Kiro CLI 共享）+ 旧版 `settings.kiro_ide_home`（默认 `%APPDATA%\Kiro\User\globalStorage\kiro.kiroagent`） |
| `default_agent_home` → `default_legacy_agent_home` | 仅作为旧版回退路径 |

会话 Key 两种布局统一为 `{workspace_key}::{session_id}`；新版 `session_id` 带 `sess_` 前缀，
按目录是否存在（而非前缀）判定走哪个布局。列表合并两个布局后按 `updated_at` 排序分页。

