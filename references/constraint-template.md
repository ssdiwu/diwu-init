# 架构约束模板

用于 `.claude/rules/constraints.md` 的模块设计模板。指导用户完成每个部分。

## 模板

```markdown
# [Module Name] Architecture Constraints

## Constraints

| Dimension | Constraint | Verification |
|-----------|-----------|--------------|
| Business | ... | "100 年后还成立吗？" |
| Temporal | ... | "能绕过这个状态转移吗？" |
| Cross-platform | ... | "两端不同是 bug 吗？" |
| Concurrency | ... | "两个线程同时做会出问题吗？" |
| Perception | ... | "超过阈值用户会注意到吗？" |

## State Machines (if applicable)

| Current State | Event | New State |
|--------------|-------|-----------|
| ... | ... | ... |

## Boundaries

| Type | Internal | External | Crossing Method |
|------|----------|----------|-----------------|
| Domain | ... | ... | Event / API |
| Platform | ... | ... | Interface / Adapter |
| Time | ... | ... | Projection function |

## Degradation Paths

| Dependency | Failure | Fallback | User Perception |
|-----------|---------|----------|-----------------|
| ... | ... | ... | ... |

## Data Ownership

Source of Truth → Projection chain → Local cache
Conflict strategy: [Server-wins / Last-write-wins / ...]
```

## 约束五维度发现问题

使用这些问题帮助用户发现约束:

1. **业务约束**: "这条规则 100 年后还成立吗？" — 区分本质约束和实现选择
2. **时序约束**: "有没有执行路径可以绕过？" — 如果无路径可绕过,就是时序约束
3. **跨平台约束**: "如果两个端不同,这是 bug 吗？" — 如果是,定义就是跨平台约束
4. **并发约束**: "两个线程同时做这件事会出问题吗？" — 必须在架构层面保证
5. **感知约束**: "超过这个阈值,用户会注意到吗？" — 不是优化目标,而是设计约束

## 检查清单

```
约束:
  □ 业务约束已识别(百年测试)
  □ 时序约束编码为状态机(无非法状态组合)
  □ 跨平台概念有单一定义
  □ 并发状态有序列化保护
  □ 感知阈值已定义

原子:
  □ 每个原子可用一句话描述,< 100 行
  □ 状态使用 sealed class/enum,不用布尔组合

组合:
  □ Pipeline 有统一流通货币
  □ 独立状态机正交
  □ 不同变化率分开存储

边界:
  □ 共享逻辑不 import 平台类
  □ 事实和投影区分

降级:
  □ 每个关键依赖有定义的回退方案
  □ 回退在代码中显式表达(不是 try/catch + print)
```
