# /dev-cycle — 分阶段开发工作流

一个跨项目的 Claude Code 用户级 skill，将复杂开发任务分解为 4 个独立阶段，每个阶段有明确的产出和验收门禁。

## 流程概览

```
/dev-cycle "任务描述"
       │
       ▼
┌──────────────────────────────────────┐
│ S1: 方案设计 (brainstorming)          │  →  docs/dev-cycle/...-design.md
│    需求澄清 → 方案对比 → 设计定稿     │
└──────────────┬───────────────────────┘
               │ 用户审查 spec
               ▼
┌──────────────────────────────────────┐
│ S2: 设计对抗审查                      │  →  docs/dev-cycle/...-adv-review-d1.md
│    Saboteur / New Hire / Security    │
└──────────────┬───────────────────────┘
               │ 决定 findings 修复
               ▼
┌──────────────────────────────────────┐
│ S3: Karpathy 准则编码                 │  →  实际代码变更
│    清晰 > 巧妙, 小函数, 好命名        │
└──────────────┬───────────────────────┘
               │ 代码完成
               ▼
┌──────────────────────────────────────┐
│ S4: 代码对抗审查                      │  →  docs/dev-cycle/...-adv-review-code.md
│    线程安全 / 内存 / 错误处理          │
└──────────────┬───────────────────────┘
               │ 验收通过
               ▼
          ✅ 自动 git commit
            (不推送)
```

## 安装

```bash
# 克隆到 Claude Code 用户 skills 目录
mkdir -p ~/.claude/skills
git clone https://github.com/vintonliu/dev-cycle.git ~/.claude/skills/dev-cycle
```

安装后重启 Claude Code 即可使用 `/dev-cycle` 命令。

## 使用方法

### 开始新周期

```bash
/dev-cycle "添加用户认证模块"
```

Claude 自动进入 Stage 1，开始需求澄清和方案设计。

### 恢复进行中的周期

```bash
/dev-cycle --resume
```

读取项目根目录 `.dev-cycle.json`，从断点继续。

### 查看进度

```bash
/dev-cycle --status
```

显示当前阶段和已完成产出。

### 终止周期

```bash
/dev-cycle --abort
```

清除 `.dev-cycle.json`，可开始新周期。

## 4 个阶段

### Stage 1: 方案设计

使用内置 brainstorming 流程：对话式需求澄清、提出 2-3 个方案并对比优劣、逐段呈现设计并确认。设计文档保存到项目 `docs/dev-cycle/` 目录，必须包含：背景与目标、方案选择理由、架构设计、接口设计、错误处理、明确排除范围（YAGNI）。

### Stage 2: 设计对抗审查

调用 `engineering-skills:adversarial-reviewer`，从 3 个角色审查设计方案：

- **Saboteur** — 寻找一切可能在生产环境崩溃的路径
- **New Hire** — 评估可理解性和可维护性
- **Security Auditor** — 识别攻击面

用户决定哪些 Critical/Warning 发现需要修复，哪些延后处理。

### Stage 3: Karpathy 准则编码

调用 `andrej-karpathy-skills:karpathy-guidelines`，在以下准则指导下编码：
- 清晰优先于巧妙
- 小函数、好命名
- 适度抽象（不提前做接口层）
- 先阅读项目代码风格再写
- 尽早处理错误
- 注释说明"为什么"而非"是什么"

每完成一个模块做增量编译检查，全部变更后做完整构建验证。

### Stage 4: 代码对抗审查

再次调用 adversarial-reviewer，这次审查代码 diff 而非设计文档。重点：线程安全、内存管理、错误处理、遵循项目模式、安全问题。

发现问题可修复后循环 S4 直到用户审核通过。

### 完成

S4 通过后自动执行 `git commit`（不推送），commit SHA 记录到 `.dev-cycle.json`。

## 状态管理

项目根目录下的 `.dev-cycle.json` 文件跨会话跟踪进度：

```json
{
  "topic": "user-auth",
  "current_stage": 2,
  "stages": {
    "1": { "status": "done", "artifact": "docs/dev-cycle/2026-05-15-user-auth-design.md" },
    "2": { "status": "pending", "artifact": "docs/dev-cycle/2026-05-15-user-auth-adv-review-d1.md" },
    "3": { "status": "pending", "artifact": null },
    "4": { "status": "pending", "artifact": "docs/dev-cycle/2026-05-15-user-auth-adv-review-code.md" }
  },
  "decisions": {
    "s2_findings_to_fix": ["状态机线程安全问题"],
    "s2_findings_deferred": ["输入校验(低风险)"]
  },
  "started_at": "2026-05-15T10:00:00+08:00",
  "completed_at": null,
  "commit_sha": null
}
```

## 自动放行操作

为减少权限提示，以下操作无需用户逐次确认：

| 类别 | 操作 | 说明 |
|------|------|------|
| git 命令 | diff/status/log/add/commit/checkout/branch | 标准开发操作 |
| 构建命令 | cmake、构建脚本、代码格式化 | 仅本地构建 |
| 文件编辑 | 所有源码文件、`.claude/` 配置 | 项目内部操作 |

**仍需确认：** `git push`、批量删除、安装新依赖或系统工具

## 依赖的子 Skill

| Skill | 使用阶段 | 来源 |
|-------|----------|------|
| `engineering-skills:adversarial-reviewer` | S2, S4 | engineering-skills 插件 |
| `andrej-karpathy-skills:karpathy-guidelines` | S3 | andrej-karpathy-skills 插件 |

子 skill 不可用时，流程会回退为手动执行对应逻辑。

## 文件结构

```
~/.claude/skills/dev-cycle/
├── SKILL.md              # 主 skill 定义
├── README.md             # 英文说明
└── README-zh.md          # 中文说明

<项目根目录>/
├── .dev-cycle.json       # 周期状态（自动管理）
└── docs/dev-cycle/        # 阶段产出物
    ├── YYYY-MM-DD-<topic>-design.md
    ├── YYYY-MM-DD-<topic>-adv-review-d1.md
    └── YYYY-MM-DD-<topic>-adv-review-code.md
```

## 边界处理

| 场景 | 处理方式 |
|------|----------|
| S1 中用户退出会话 | resume 后从 S1 继续 |
| 设计有致命缺陷 | 修订 S1 设计 → 重新进入 S2 |
| 编码发现设计漏洞 | 停止编码，报告用户，修订 S1 |
| S4 发现严重问题 | 修复后循环 S4 直到通过 |
| 无变更可提交 | 跳过 commit，报告无变更 |
| 用户想修改 commit | 提供 SHA，建议 `git commit --amend` |
