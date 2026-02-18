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
- 有明确的验收标准（`acceptance` 字段，可证据化验证）
- 有清晰的实施步骤（`steps` 字段）
- 不依赖未完成的其他任务，或依赖已在 blocked_by 字段中标注

### task.json 写入规则
- 新任务状态一律为 `InDraft`
- ID 递增，不复用已有 ID
- category 可选值：functional / ui / bugfix / refactor / infra
- 如果 `.claude/task.json` 已有任务，新任务追加到列表末尾
- 必须包含 `acceptance` 字段（验收条件数组）
- 如有依赖关系，使用 `blocked_by` 字段标注（可选）

```json
{
  "tasks": [
    {
      "id": 1,
      "description": "实现用户登录功能",
      "status": "InDraft",
      "acceptance": [
        "输入正确用户名密码后跳转到首页",
        "输入错误密码显示错误提示",
        "登录状态保持 7 天"
      ],
      "steps": [
        "实现登录 API 调用",
        "添加错误处理逻辑",
        "实现本地 token 存储"
      ],
      "category": "functional"
    },
    {
      "id": 2,
      "description": "实现用户个人资料页面",
      "status": "InDraft",
      "acceptance": ["显示用户头像、昵称、邮箱", "可编辑昵称并保存"],
      "steps": ["创建个人资料组件", "实现编辑功能"],
      "category": "ui",
      "blocked_by": [1]
    }
  ]
}
```

---

## 任务实施

InSpec → InProgress → InReview → Done 完整流程：

1. 将选定任务状态改为 `InProgress`
2. 仔细阅读任务的任务描述、验收条件、实施步骤
3. 按照项目既有的代码模式实现功能
4. 实现完成后，按本文件"验证要求"章节进行验证
5. 验证通过后，将状态改为 `InReview`
6. 根据分层审查规则：
   - 小幅度修改：Agent 自审后标记 Done
   - 大幅度修改：输出 REVIEW 请求，等待人工确认

**Change Request 流程**（执行 InSpec 任务时发现需求问题）：
1. **创建 CR**：在 recording.md 中记录 Change Request
2. **保持状态**：任务状态保持 InSpec（不退回 InDraft）
3. **停止实施**：输出 CHANGE_REQUEST 信息，等待批准
4. **批准后**：更新 task.json 中的验收条件，继续实施

---

## 验证要求

**核心原则**：**未验证 = 未完成**。不要将任务标记为 InReview，除非所有 acceptance 条件已验证通过。

无论使用哪种验证方式，必须：
1. task.json 中的 acceptance 逐条验证
2. 验证方法和结果记录在 recording.md
3. 未验证 = 未完成，不可标记 Done

### 1. 自动化验证（推荐，如可行）
- **位置**：`.claude/checks/task_<id>_verify.sh`
- **要求**：脚本返回 0（成功）或 非0（失败）
- **记录**：脚本输出写入 recording.md

```bash
#!/bin/bash
# .claude/checks/task_1_verify.sh
npm run build || exit 1
npm test -- login.test.ts || exit 1
echo "Task#1 验证通过"
exit 0
```

### 2. 手动验证（不可自动化时）
- **场景**：实机 App 测试、UI 交互、需要真实账号等
- **流程**：
  1. Agent 提供明确的验证步骤清单
  2. 人工执行并反馈结果（PASS/FAIL）
  3. Agent 根据反馈标记任务状态
- **记录**：在 recording.md 中注明"手动验证通过"

```markdown
## 验证步骤（请在真机执行）
1. 打开 App，进入录音页面
2. 点击"开始录音"按钮
3. 说话 10 秒后点击"停止"
4. 检查文件管理器中是否有新文件
5. 点击播放，确认录音内容

### 验收对照
- [ ] 点击录音按钮后开始录音 → 步骤 2-3
- [ ] 录音文件保存到本地 → 步骤 4
- [ ] 停止后可播放 → 步骤 5
```

### 3. 人工审查（大幅度修改）
- **场景**：架构变更、API 契约修改、超过 2000 行代码
- **流程**：Agent 完成实现和验证后，输出 REVIEW 请求，等待人工确认
- **记录**：recording.md 中注明"已人工审查"

### 基线验证（smoke.sh）
- **位置**：`.claude/checks/smoke.sh`
- **职责**：验证基线环境，不包含具体任务验证
- **检查项**：依赖安装完整 / 基础构建通过 / 核心服务可启动 / lint 无全局错误

```bash
#!/bin/bash
# .claude/checks/smoke.sh
echo "检查依赖..."
[ -d "node_modules" ] || npm install

echo "检查构建..."
npm run build || exit 1

echo "检查 lint..."
npm run lint || exit 1

echo "基线验证通过"
exit 0
```

### 测试分层
- **大幅度修改**：使用浏览器测试工具（Playwright/Puppeteer MCP）验证功能，验证页面正确加载和渲染，验证交互功能（表单提交、按钮点击等），截图确认 UI 正确显示
- **小幅度修改**：可用 lint/build/单元测试验证；如有疑虑，仍建议进行浏览器测试
- **所有修改必须通过**：lint 检查无错误、build 构建成功、无类型错误（TypeScript 等）、功能在实际环境中正常工作

### 验证失败处理
- 验证失败时，任务状态保持 InProgress
- 修复问题后重新验证
- 不要跳过失败的验证

---

## Session 结束

在 session 结束前（包括 context window 接近上限时）：

1. 确保所有代码变更已提交
2. 更新 `.claude/recording.md`，记录本次 session：
   - Session 时间戳
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

### 自动归档触发条件

**task.json 归档**（触发条件：Done/Cancelled 任务超过 20 个）：
1. 将 Done/Cancelled 任务移到 task_archive_YYYY-MM.json
2. 保留 id 序列（新任务继续递增）
3. 在 recording.md 记录归档操作

**recording.md 归档**（触发条件：session 数超过 5 条）：
1. 自动将最旧的 session 归档到 recording_archive/YYYY-MM-DD.md
2. recording.md 保留最近 5 条 session

**查找历史**：
- 最近任务：查 task.json
- 历史任务：查 task_archive_YYYY-MM.json（按 id 搜索）
- 最近进度：查 recording.md
- 历史进度：查 recording_archive/YYYY-MM-DD.md
