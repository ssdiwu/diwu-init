# 文件布局

## .claude/ 目录结构

```
.claude/
├── CLAUDE.md                      # 全局 Agent 配置入口
├── task.json                      # 当前任务列表
├── task_archive_YYYY-MM.json      # 归档任务（归档后生成）
├── recording.md                   # Session 进度记录
├── recording_archive/             # 归档记录目录（归档后生成）
│   └── YYYY-MM-DD.md
├── rules/                         # 规则文件目录
│   ├── core-states.md
│   ├── core-workflow.md
│   ├── exceptions.md
│   ├── templates.md
│   └── file-layout.md
├── checks/                        # 验证脚本目录
│   ├── smoke.sh
│   └── task_<id>_verify.sh
└── init.sh                        # 环境初始化脚本（可选）
```

## 文件说明

| 路径 | 用途 | 读写方 |
|------|------|--------|
| `.claude/CLAUDE.md` | 全局配置、个人偏好、规则索引 | 共同维护 |
| `.claude/task.json` | 当前所有任务的状态和内容 | Agent 读写 |
| `.claude/task_archive_YYYY-MM.json` | 按月归档的 Done/Cancelled 任务，保留 id 序列 | Agent 写 |
| `.claude/recording.md` | Session 进度记录，每次追加 | Agent 写 |
| `.claude/recording_archive/YYYY-MM-DD.md` | 按天归档的历史 session 记录 | Agent 写 |
| `.claude/checks/smoke.sh` | 基线环境验证，session 启动时运行 | Agent 提供方案，人工确认后实施 |
| `.claude/checks/task_<id>_verify.sh` | 任务专属验证脚本，id 对应 task.json | Agent 创建并执行 |
| `.claude/init.sh` | 开发环境初始化，session 启动时按需运行 | 讨论后由 Agent 实施 |

## 归档触发条件

- **task_archive_YYYY-MM.json**：`task.json` 中 Done/Cancelled 任务超过 20 个时触发
- **recording_archive/YYYY-MM-DD.md**：recording.md 中 session 数超过 5 条时自动触发，无需人工确认，按天归档

## 归档执行步骤

**task.json 归档**：
1. 将 Done/Cancelled 任务移到 task_archive_YYYY-MM.json（当前月份）
2. 保留 id 序列（新任务继续递增）
3. 在 recording.md 记录归档操作

**recording.md 归档**：
1. 将最旧的 session 移入 recording_archive/YYYY-MM-DD.md（按天）
2. recording.md 始终保留最近 5 条 session
3. 无需人工确认，session 启动时自动执行

## 查找历史

- 最近任务：查 task.json
- 历史任务：查 task_archive_YYYY-MM.json（按月，grep 搜索）
- 最近进度：查 recording.md
- 历史进度：查 recording_archive/YYYY-MM-DD.md
