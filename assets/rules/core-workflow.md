# Session 工作流

## Session 启动（必须按顺序执行）

每个新 session 必须先完成以下步骤，然后才能开始实现任务：

### 1. Preflight 检查
- 运行 `.claude/checks/smoke.sh`（如存在），验证基线环境
- 检查 recording.md 中是否有 pending Change Requests
- 运行 `git status`，确认工作区状态

### 2. 上下文恢复
- 读取 `.claude/recording.md`，了解上一个 session 的工作和注意事项
- 运行 `git log --oneline -20`，了解最近的代码变更历史

### 3. 任务选择策略
- 读取 `.claude/task.json`，了解所有任务的当前状态
- **优先恢复** `status: "InProgress"` 的任务（中断的任务）
- 否则选择第一个 `status: "InSpec"` 的任务，并检查 blocked_by：
  - **blocked_by 为空或不存在** → 无阻塞，可开始
  - **blocked_by 全部 Done** → 阻塞已解除，可开始
  - **blocked_by 存在 InReview 且超前<3** → 可超前实施（标记为超前）
  - **其他情况** → 跳过，选择下一个任务
- **禁止选择** `status: "InDraft"` 的任务（需人工先确认）
- 选择优先级：无 blocked_by 的任务优先，基础功能优先

### 4. 环境初始化（可选）
- 运行 `.claude/init.sh` 启动开发环境（如需要）
- 运行项目的基线测试（build/lint），确认系统处于正常状态
- 如果基线测试失败，优先修复，然后再开始新任务

---

## 任务规划

### 触发条件
- 用户在讨论想法、需求、功能设计
- `.claude/task.json` 为空或用户明确要求分解任务
- 用户描述了一个较大的功能，需要拆分为可执行步骤

### 分解流程

**需求精炼（任务描述模糊时触发）**：
- 每次只问一个问题，理解目的、约束、成功标准
- 提出 2-3 个方案并给出推荐，获得确认后再写 task.json

**草案阶段（InDraft）**：
1. **提炼**：从讨论中识别可执行的功能点
2. **拆分**：每个任务一句话可描述、一个 session 可完成
3. **定义验收**：为每个任务明确验收条件（可验证的标准）
4. **排序**：识别依赖关系，基础功能排前面
5. **展示**：展示分解结果，包含任务描述、验收条件、实施步骤
6. **写入**：写入 `.claude/task.json`，状态为 `InDraft`

**锁定阶段（InSpec）**：
- **确认**：等人工确认后，Agent 将状态改为 `InSpec`
- **锁定**：从此刻起，Agent 只能修改 status 字段
- **变更**：如需修改需求，走 Change Request 流程

### 任务粒度标准
- 预估修改代码 < 2000 行（超过则继续拆分）
- 有明确的验收标准（`acceptance` 字段，GWT 格式，可证据化验证）
- 有清晰的实施步骤（`steps` 字段）
- 不依赖未完成的其他任务，或依赖已在 blocked_by 字段中标注

### task.json 写入规则
- 新任务状态一律为 `InDraft`
- ID 递增，不复用已有 ID
- category 可选值：functional / ui / bugfix / refactor / infra
- 如果 `.claude/task.json` 已有任务，新任务追加到列表末尾
- 必须包含 `acceptance` 字段（验收条件数组，格式规范见 core-states.md）
- `functional`/`ui`/`bugfix` 类任务的 acceptance 必须使用 Given/When/Then 格式
- `infra`/`refactor` 类任务的 acceptance 可使用简单描述
- 如有依赖关系，使用 `blocked_by` 字段标注（可选）

- steps 必须自包含：外部系统凭据写明来源文件路径（如"凭据见 `doc/runbook.md §1.1`"），代码路径写绝对路径，不依赖会话上下文

---

## 任务实施

InSpec → InProgress → InReview → Done 完整流程：

1. 将选定任务状态改为 `InProgress`
2. 仔细阅读任务的任务描述、验收条件、实施步骤
3. 按照项目既有的代码模式实现功能（如有测试框架，建议顺序：先写验收测试框架 → 单元测试 → 实现）
4. 实现完成后，按本文件"验证要求"章节进行验证
5. 验证通过后，将状态改为 `InReview`
6. 根据分层审查规则：
   - 小幅度修改：Agent 自审后标记 Done
   - 大幅度修改：输出 REVIEW 请求，等待人工确认

**大幅度修改判定**：满足以下任一条件即为大幅度修改：修改了原有 API 规范或字段定义；单次任务修改代码行数超过 2000 行。其余 Agent 自审即可。

**Change Request 流程**（执行 InSpec 任务时发现需求问题）：
1. **创建 CR**：在 recording.md 中记录 Change Request
2. **保持状态**：任务状态保持 InSpec（不退回 InDraft）
3. **停止实施**：输出 CHANGE_REQUEST 信息，等待批准
4. **批准后**：更新 task.json 中的验收条件，继续实施

### 子代理策略
- **派生子代理**：多个独立子任务可并行 / 有明确边界的探索任务（搜索代码结构、调研某模式）
- **保持单一上下文**：顺序依赖任务（每步结果决定下一步方向）/ 实施类任务需要积累上下文

---

## 验证要求

**核心原则**：**未验证 = 未完成**。所有 acceptance 条件逐条验证通过后才能标记 InReview，验证方法和结果记录在 recording.md。

### 1. 自动化验证（推荐）
脚本位置：`.claude/checks/task_<id>_verify.sh`，返回 0 成功 / 非0 失败。模板见 templates.md。

### 2. 手动验证（不可自动化时）
场景：实机测试、UI 交互、需要真实账号。Agent 提供步骤清单 → 人工执行反馈 PASS/FAIL → Agent 标记状态。

### 3. 人工审查（大幅度修改）
Agent 完成实现和验证后输出 REVIEW 请求，等待人工确认。

### 基线验证（smoke.sh）
位置：`.claude/checks/smoke.sh`，验证基线环境（依赖/构建/lint）。模板见 templates.md。

### 测试分层
- 大幅度修改：浏览器测试工具（Playwright/Puppeteer MCP）验证功能和 UI
- 小幅度修改：lint/build/单元测试

### 验证失败处理
验证失败时任务状态保持 InProgress，修复后重新验证。

---

## Session 结束

在 session 结束前（包括 context window 接近上限时）：

1. 确保所有代码变更已提交
2. 更新 `.claude/recording.md`，记录本次 session：
   - Session 时间戳（运行 `date '+%Y-%m-%d %H:%M:%S'` 获取真实时间，禁止伪造）
   - 处理的任务及状态
   - 验收验证结果
   - Change Request（如有）
   - 下次应该做什么
3. 确保 `.claude/task.json` 反映最新的任务状态

### Context Window 管理
- context window 会在接近上限时自动压缩，允许无限期继续工作
- 不要因为 token 预算担忧而提前停止任务
- 在感觉接近上限时，优先保存进度到 .claude/recording.md
- 始终保持最大程度的自主性，完整完成任务

归档触发条件与执行步骤见 `file-layout.md`。
