---
name: diwu-init
description: 初始化新项目的 Claude Code Agent 工作流结构。自动创建 CLAUDE.md、task.json、recording.md、init.sh,以及可选的架构约束文件。触发场景：(1) 新建项目或代码库时，(2) 用户说"init project"、"scaffold"、"新建项目"、"初始化项目"，(3) 为已有代码库搭建 Claude Code 工作流，(4) 用户提到需要创建 CLAUDE.md 或 task.json。
---

# 项目初始化

使用标准 Claude Code Agent 工作流结构初始化项目。

## 工作流程

### 1. 收集项目信息

询问用户:
- **项目名称**和**1-2 句描述**
- **技术栈**: 语言、框架、样式、数据库
- **常用命令**: dev、build、lint、test
- **关键目录**: 项目结构概览

### 2. 选择配置模式

询问用户在两种模式中选择:

**精简模式** (如果用户有 `~/.claude/rules/` 则推荐):
- 项目 `.claude/CLAUDE.md` 引用全局规则
- 文件体积更小
- 更易维护(全局规则可集中更新)
- 适合个人项目

**便携模式** (用于分享或独立项目时推荐):
- 将完整工作流规则嵌入 `.claude/CLAUDE.md`
- 项目自包含
- 无需 `~/.claude/rules/` 即可工作
- 适合分发给他人

检测逻辑:
- 检查 `~/.claude/rules/` 是否存在
- 存在则默认**精简模式**
- 不存在则默认**便携模式**

### 3. 创建项目文件

所有文件按照 Claude Code 官方结构创建(见 https://code.claude.com/docs/zh-CN/memory.md)。

#### .claude/CLAUDE.md

**精简模式:**
读取 `assets/claude-md-minimal.template`,使用步骤 1 的项目信息定制。

**便携模式:**
读取 `assets/claude-md-portable.template`,替换 `[RULES:filename.md]` 占位符为实际内容:
- 从 `assets/rules/` 读取每个规则文件
- 将占位符替换为文件内容
- 示例: `[RULES:task-states.md]` → `assets/rules/task-states.md` 的内容

使用步骤 1 的信息定制项目上下文部分。

编写指南(来自 `references/memory-best-practices.md`):
- 要具体: "使用 2 空格缩进" 而非 "正确格式化代码"
- 包含所有常用命令,避免重复搜索
- 记录命名约定和代码风格
- 使用 `@path/to/file` 引用文档,而非复制内容到 CLAUDE.md

#### AGENTS.md (项目根目录)

读取 `assets/agents-md.template` 并写入项目根目录。

这个精简文件作为重定向,供其他 AI 工具(Codex、Gemini 等)找到实际配置。

#### .claude/task.json

读取 `assets/task.json.template`,写入 `.claude/task.json`。如用户已定义需求,填充初始任务并设置 `"status": "InDraft"`,否则保持 tasks 数组为空。

任务结构:
```json
{
  "id": 1,
  "description": "实现用户登录功能",
  "status": "InDraft",
  "acceptance": [
    "输入正确密码后跳转首页",
    "错误密码显示提示"
  ],
  "steps": ["实现登录 API", "添加错误处理"],
  "category": "functional",
  "blocked_by": []
}
```

有效状态: `InDraft`、`InSpec`、`InProgress`、`InReview`、`Done`、`Cancelled`。

字段名称: `description`、`acceptance`、`steps`。`blocked_by` 为可选字段(无依赖时省略)。

#### init.sh

读取 `assets/init.sh.template`,根据技术栈定制:
- 设置正确的安装命令(npm install / pip install / cargo build / 等)
- 设置正确的开发服务器命令
- 添加可执行权限: `chmod +x init.sh`

#### .claude/recording.md

读取 `assets/recording.md.template`(原 progress.md.template)并写入 `.claude/recording.md`:
```markdown
# Session 记录

(Sessions will be recorded here)
```

注意: 旧项目可能有 `progress.txt` 或 `progress.md`,新标准为 `recording.md`。

#### .claude/checks/smoke.sh

读取 `assets/smoke.sh.template`,根据技术栈定制并写入 `.claude/checks/smoke.sh`。添加可执行权限 `chmod +x`。

### 4. 可选: 架构约束

对于架构复杂的项目,询问用户是否需要 `.claude/rules/constraints.md`。

如需要,读取 `references/constraint-template.md` 并引导用户使用五维框架定义约束:
- 业务约束: "100 年后还成立吗?"
- 时序约束: "能绕过这个状态转移吗?"
- 跨平台约束: "两端不同是 bug 吗?"
- 并发约束: "两个线程同时做会出问题吗?"
- 感知约束: "超过阈值用户会注意到吗?"

### 5. Git 初始化(如需要)

如果还不是 git 仓库:
```bash
git init
git add .
git commit -m "Initial project setup with Claude Code workflow"
```

### 6. 验证

确认所有文件已创建且有效:
- [ ] `.claude/CLAUDE.md` 已填充项目信息(精简或便携模式)
- [ ] 项目根目录有 `AGENTS.md`
- [ ] `.claude/task.json` 是有效 JSON
- [ ] `init.sh` 可执行
- [ ] `.claude/recording.md` 存在
- [ ] `.claude/checks/smoke.sh` 可执行
- [ ] (可选) `.claude/rules/constraints.md`

### 7. 规则文件说明(便携模式)

如选择便携模式,完整嵌入的工作流规则包含:

| 规则文件 | 内容概要 | 行数 |
|---------|---------|------|
| `core-states.md` | task.json 结构与字段定义、状态机与转移规则、blocked_by 规范 | ~137 行 |
| `core-workflow.md` | Session 流程、任务规划、实施、验证要求 | ~243 行 |
| `exceptions.md` | 阻塞、Change Request、超前实施、回退处理 | ~106 行 |
| `templates.md` | 所有格式模板、Git 规范、可调参数 | ~157 行 |
| `file-layout.md` | 所有文件路径定义、目录结构、归档触发条件 | ~45 行 |

**总计**: 约 688 行工作流规则

## 文件结构概览

```
project-root/
├── .claude/
│   ├── CLAUDE.md             # 项目配置(官方位置)
│   ├── task.json             # 任务跟踪
│   ├── recording.md          # 进度日志
│   ├── checks/
│   │   └── smoke.sh          # 基线验证
│   └── rules/
│       └── constraints.md    # 架构约束(可选)
├── AGENTS.md                 # 跨 AI 工具入口点
├── init.sh                   # 环境设置脚本
└── src/
```

## 技术说明

### 为什么使用 .claude/ 目录?

遵循 Claude Code 官方文档(https://code.claude.com/docs/zh-CN/memory.md):
- `.claude/CLAUDE.md` 优先级高于全局 `~/.claude/CLAUDE.md`
- 这是组织项目特定配置的推荐方式

### 为什么在根目录放置 AGENTS.md?

- **可发现性**: 其他 AI 工具可以轻松找到
- **简洁性**: 仅作为重定向,非完整文档
- **兼容性**: 适用于 Claude 之外的各种 AI 编程助手

### 规则嵌入策略

- **精简模式**: 无重复,引用全局规则
- **便携模式**: 使用 `assets/rules/` 中的内置规则作为默认模板
- 如用户在 `~/.claude/rules/` 有自定义规则,便携模式中优先使用这些规则
