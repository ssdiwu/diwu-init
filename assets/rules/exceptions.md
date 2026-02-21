# 异常处理

## 需要停止并请求人工帮助的情况

1. **缺少环境配置**:API 密钥、数据库连接、外部服务账号
2. **外部依赖不可用**:第三方 API 宕机、需人工授权的 OAuth、付费服务
3. **验证无法进行**:需要真实用户账号、依赖未部署的外部系统、需特定硬件
4. **需求存在问题**:发现原 acceptance 无法实现或存在矛盾

## 阻塞时的规则

**禁止**:
- 提交 git commit
- 将 `.claude/task.json` 状态设为 Done
- 假装任务已完成

**必须**:
- 在 `.claude/recording.md` 中记录当前进度和阻塞原因
- 将 `.claude/task.json` 任务状态退回 **InSpec**(不是 InDraft)
- 输出阻塞信息,说明需要人工做什么
- 停止任务,等待人工介入

## 阻塞信息格式

环境/依赖问题时输出 BLOCKED 格式。完整格式见 templates.md。

## Change Request 流程

当发现需求存在问题时,不使用 BLOCKED,而是提交 Change Request。

### 触发条件
- 发现原 acceptance 无法实现
- 发现 acceptance 存在矛盾

### CR 编号规则
CR 编号使用全局递增(跨任务连续),方便追踪。

### 输出说明
使用 CHANGE REQUEST 格式（见 templates.md）输出，输出后按 Change Request 记录格式追加到 recording.md。

## 阻塞恢复流程

### 环境阻塞恢复
1. 人工提供所需配置/权限
2. Agent 验证环境可用
3. 任务状态: InSpec → InProgress
4. 继续实施

### 需求变更恢复
1. 人工批准 Change Request
2. Agent 更新 task.json 中的 acceptance
3. 在 recording.md 记录 CR approved
4. 任务状态保持 InSpec → InProgress
5. 按新 acceptance 继续实施

## 超前实施

### 触发条件
当前任务的 blocked_by 中存在状态为 InReview 的任务时,可以超前实施。

### 超前规则
- **允许**: 最多超前 5 个任务
- **任务状态**: 超前完成的任务标记为 **InReview**
- **提交规则**: 超前任务完成时**立即创建 git commit**
- **标记**: 在 recording.md 中记录 "Task#N (blocked_by Task#M, 超前 X/5, commit: abc123)"
- **暂停**: 完成第 5 个超前任务后,输出 PENDING REVIEW,等待阻塞任务验收

超前暂停格式见 templates.md。

## 回退处理

### 触发条件
阻塞任务验收失败,已超前完成的任务可能受影响。

**阻塞任务验收通过后**:
1. 将阻塞任务标记 Done
2. 逐个将超前任务标记 Done(InReview → Done)
3. 在 recording.md 记录阻塞解除

**阻塞任务验收失败时**:
1. Agent 评估已超前完成的任务是否受影响
2. 在 recording.md 记录回退计划和选择的方式
3. **选择回退方式**(人工决定):
   - **方式 A**: `git revert` 超前 commit(完全不可复用)
   - **方式 B**: 保留 commit,在此基础上修改(部分可复用)
   - **方式 C**: `git reset --soft HEAD~N` 回退后重新实施
4. 修复阻塞任务并验收通过
5. 重新验证超前任务(如使用方式 B/C)
