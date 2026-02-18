# 文件布局

## .claude/ 目录结构

```
.claude/
├── CLAUDE.md                      # 全局 Agent 配置入口
├── task.json                      # 当前任务列表
├── task_archive_YYYY-MM.json      # 归档任务（按月分割，归档后生成）
├── recording.md                   # Session 进度记录（滚动窗口，保留最近 5 条）
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
| `.claude/task_archive_YYYY-MM.json` | Done/Cancelled 任务按月归档，保留 id 序列 | Agent 写 |
| `.claude/recording.md` | Session 进度记录，滚动窗口保留最近 5 条 | Agent 写 |
| `.claude/recording_archive/YYYY-MM-DD.md` | 按天归档的历史 session 记录 | Agent 写 |
| `.claude/checks/smoke.sh` | 基线环境验证，session 启动时运行 | Agent 提供方案，人工确认后实施 |
| `.claude/checks/task_<id>_verify.sh` | 任务专属验证脚本，id 对应 task.json | Agent 创建并执行 |
| `.claude/init.sh` | 开发环境初始化，session 启动时按需运行 | 讨论后由 Agent 实施 |

## 归档触发条件

- **task_archive_YYYY-MM.json**：`task.json` 中 Done/Cancelled 任务超过 20 个时触发，按月分割归档
- **recording_archive/YYYY-MM-DD.md**：recording.md 中 session 数超过 5 条时自动触发，按天归档
