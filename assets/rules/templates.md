# 格式模板与参数

## BLOCKED 格式

```
BLOCKED - 需要人工介入

当前任务: Task#N - [任务描述]

已完成的工作:
- [已完成的代码/配置]

阻塞原因:
- [具体说明为什么无法继续]

需人工帮助:
1. [具体的步骤]

解除阻塞后:
- 任务将从 InSpec 恢复到 InProgress
```

## CHANGE REQUEST 格式

```
CHANGE REQUEST - 需要修改需求

当前任务: Task#N - [任务描述]
当前状态: InSpec (保持)

已完成的工作:
- [已实现的部分]

发现的问题:
- [原 acceptance 为什么无法实现]

建议修改:
- acceptance 修改:
  - [删除] "xxx"
  - [新增] "yyy"
  - [修改] "zzz" → "www"

影响评估:
- 预计额外工作量: [X 小时/天]
- 影响其他任务: [是/否]

等待批准:
- 人工批准后,将更新 task.json 并继续实施
```

## PENDING REVIEW 格式

```
PENDING REVIEW - 超前实施已达上限

等待验收: Task#10
已超前完成: Task#11, Task#12, Task#13 (3/3)

请验收 Task#10:
- 通过 → 可继续超前实施 Task#14-16
- 失败 → 需评估 Task#11-13 的影响并修复
```

## REVIEW 请求格式

```
REVIEW - 请求人工审查

Task#N: [任务名称]
修改范围: [改动了哪些文件/功能]
验收验证: [逐条 acceptance 验证结果]

等待: 人工确认后将标记为 Done
```

## recording.md Session 格式

```markdown
---
## Session 2026-02-17 14:30:22

### 上下文恢复
- 上次任务: Task#1 (InProgress)
- Git 状态: clean
- 待批准 CR: 无

### Task#1: 用户登录功能 → Done

**实施内容**:
- 完成 src/auth/login.ts 核心逻辑
- 添加错误处理和 token 存储
- 添加单元测试

**验收验证** (自动化):
- [x] 输入正确密码后跳转首页 (npm test passed)
- [x] 错误密码显示提示 (npm test passed)
- [x] 登录状态保持 7 天 (手动验证通过)

**验证方法**: 运行 .claude/checks/task_1_verify.sh

**提交**: commit abc123f

### 下一步
Task#2: 密码重置功能

---
```

## Change Request 记录格式

```markdown
---
## Session 2026-02-18 10:00:35

### Task#2: 密码重置 → BLOCKED

**Change Request #2**:
- **原因**: 发现邮件服务依赖不稳定
- **建议**: 增加"手机验证码重置"作为备选方案
- **详细修改**:
  - [删除] "通过邮件发送重置链接"
  - [新增] "通过手机验证码验证身份"
  - [新增] "支持邮件和手机两种方式"
- **影响**: acceptance 需新增 2 条,预计额外 1 天工作量
- **状态**: pending

**需人工介入**:
请批准 CR#2,或提供稳定的邮件服务配置

---
```

## Git 规范

**提交时机**:
- 只有当任务状态到达 Done 时才提交
- 超前实施例外:超前任务完成时标记 InReview 且立即提交 commit(最多 3 次),阻塞解除后标记 Done

**commit message 格式**:
- 常规: `[Task#N] 任务描述 - completed`
- 超前: `[Task#N] 任务描述 - completed (超前实施 X/3, blocked_by Task#M)`

**提交内容**: 代码变更 + `.claude/task.json` 更新 + `.claude/recording.md` 更新,同一个 commit。

**force push 前置检查**: 执行 `git push --force` 前必须先 `git fetch origin` 并确认远端无未合并的变更（`git diff HEAD origin/<branch>`），有差异须先与用户确认再操作。

## 可调参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| 超前上限 | 5 | 最多同时超前实施的任务数 |
| 归档阈值 | 20 | task.json 中 Done/Cancelled 任务超过此数触发归档 |

## 验证脚本模板

**task_\<id\>_verify.sh**：
```bash
#!/bin/bash
npm run build || exit 1
npm test -- <test-file> || exit 1
echo "Task#N 验证通过"
exit 0
```

**smoke.sh**：
```bash
#!/bin/bash
[ -d "node_modules" ] || npm install
npm run build || exit 1
npm run lint || exit 1
echo "基线验证通过"
exit 0
```
