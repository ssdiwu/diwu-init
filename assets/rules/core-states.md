# 任务状态机

## task.json 结构

```json
{
  "tasks": [
    {
      "id": 1,
      "description": "实现用户登录功能",
      "status": "InDraft",
      "acceptance": [
        "Given 用户在登录页面且未登录 When 输入正确用户名密码并点击登录 Then 页面跳转到首页且 localStorage 存储 token",
        "Given 用户在登录页面 When 输入错误密码并点击登录 Then 页面停留在登录页且显示'用户名或密码错误'"
      ],
      "steps": ["实现登录 API 调用", "添加错误处理逻辑"],
      "category": "functional",
      "blocked_by": [2, 3]
    }
  ]
}
```

**字段说明**:

| 键名 | 中文含义 | 类型 | 说明 |
|------|---------|------|------|
| `id` | 任务编号 | 数字 | 从 1 开始递增，永不复用 |
| `description` | 任务描述 | 字符串 | 一句话描述任务内容 |
| `status` | 任务状态 | 字符串 | 见下方状态定义章节 |
| `acceptance` | 验收条件 | 数组 | Given/When/Then 格式的验收场景，见下方 acceptance 格式规范 |
| `steps` | 实施步骤 | 数组 | 实施过程的关键步骤 |
| `category` | 任务分类 | 字符串 | 见下方任务分类说明 |
| `blocked_by` | 前置任务 | 数组 | (可选) 见下方 blocked_by 规范章节 |

**任务分类说明**:

| 分类值 | 中文含义 | 适用场景 |
|--------|---------|---------|
| `functional` | 功能开发 | 新增业务功能、API、核心逻辑 |
| `ui` | 界面开发 | 页面、组件、交互、样式 |
| `bugfix` | 缺陷修复 | 修复已知 bug |
| `refactor` | 重构 | 不改变行为的代码结构优化 |
| `infra` | 基础设施 | 构建、部署、配置、脚本 |

**acceptance 格式规范（Given/When/Then）**:

`functional`、`ui`、`bugfix` 类任务**必须**使用 GWT 格式，`infra`、`refactor` 类任务**可选**。

格式：`"Given [前置条件] When [用户动作] Then [预期结果]"`
- 多个条件或结果用"且"连接：`"Then 页面跳转到首页且显示欢迎语"`
- 单条 acceptance 中"且"超过 3 个时，应拆分为多条
- `infra`/`refactor` 任务可使用简单描述：`"构建产物不超过 500KB"`

---

## 状态定义

| 状态 | 含义 |
|------|------|
| InDraft | 需求草稿中，Agent 可修改任务描述、验收条件、实施步骤 |
| InSpec | 需求已确认，锁定（Agent 只能修改 status 字段） |
| InProgress | 实施中，Agent 正在实现 |
| InReview | 验证中，实现完成，等待验证 |
| Done | 已完成，验证通过 |
| Cancelled | 已取消，任务不再需要 |

## 状态转移表

| 当前状态 | 事件 | 新状态 | 规则 |
|---------|------|--------|------|
| InDraft | 人工确认需求 | InSpec | Agent 不可再修改 acceptance |
| InDraft | 需求取消 | Cancelled | - |
| InSpec | Agent 开始实施 | InProgress | - |
| InSpec | 发现需求问题 | (保持 InSpec) | 提交 Change Request，等人工批准 |
| InProgress | 实现完成，准备验证 | InReview | - |
| InProgress | 遇到阻塞 | InSpec | 退回，记录阻塞原因 |
| InProgress | 需求取消 | Cancelled | - |
| InReview | acceptance 全部通过（小幅度修改） | Done | Agent 自审 |
| InReview | acceptance 全部通过（大幅度修改） | Done | 需人工确认 |
| InReview | 验证失败，需返工 | InProgress | - |
| InReview | 需求取消 | Cancelled | - |
| Done | (终态，不可转移) | — | - |
| Cancelled | 需求重新激活 | InSpec | 重新激活后直接锁定 |

**非法转移一律忽略**。例如 Done 状态下收到任何事件，保持 Done 不变。

## blocked_by 规范

### 语义
表示**阻塞关系**：前置任务未完成，当前任务无法开始。

### 修改权限
InDraft 自由修改；InSpec 可改但需在 recording.md 记录原因；InProgress 及之后只能通过 Change Request。

### 何时使用
前置任务未完成（InSpec/InProgress/InReview）且当前任务依赖其输出时使用。前置任务已 Done 或仅是代码调用关系时不使用。

### 合法性检查

Agent 修改 blocked_by 时必须验证：
1. **引用存在**：blocked_by 中的 ID 必须存在于 task.json 或 task_archive.json
2. **无循环依赖**：不存在 A→B→C→A 的循环
3. **非自引用**：任务不能阻塞自己
4. **状态合理**：
   - ✅ InSpec/InProgress/InReview
   - ⚠️ InDraft → 警告并提示先确认前置任务
   - ❌ Done → 提示在任务描述中说明即可
   - ❌ Cancelled → 拒绝

### 自动清理

当 blocked_by 中的任务变为 Done：
- Agent 自动从 blocked_by 中移除该 ID
- 在 recording.md 记录："Task#10 阻塞解除：Task#8 已完成"
