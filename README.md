# /dev-cycle — 分阶段开发工作流

一个跨项目的 Claude Code 用户级 skill，将复杂开发任务分解为 4 个独立阶段，每个阶段有明确的产出和验收门禁。

## 流程概览

```
/dev-cycle "任务描述"
       │
       ▼
┌──────────────────────────────────┐
│ S1: 方案设计 (brainstorming)      │  →  docs/dev-cycle/...-design.md
│    用户需求澄清 → 方案对比 → 设计  │
└──────────────┬───────────────────┘
               │ 用户审查 spec
               ▼
┌──────────────────────────────────┐
│ S2: 设计对抗审查 (adversarial)     │  →  docs/dev-cycle/...-adv-review-d1.md
│    Saboteur / New Hire / Security │
└──────────────┬───────────────────┘
               │ 决定 findings 修复
               ▼
┌──────────────────────────────────┐
│ S3: Karpathy 准则编码             │  →  实际代码变更
│    清晰 > 巧妙, 小函数, 好命名     │
└──────────────┬───────────────────┘
               │ 代码完成
               ▼
┌──────────────────────────────────┐
│ S4: 代码对抗审查 (adversarial)     │  →  docs/dev-cycle/...-adv-review-code.md
│    线程安全 / 内存 / 错误处理       │
└──────────────┬───────────────────┘
               │ approved
               ▼
          ✅ 自动 git commit
            (不 push)
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
/dev-cycle "为 RTC SDK 添加远程摄像头 PTZ 控制"
```

Claude 会自动进入 Stage 1，开始需求澄清和方案设计。

### 继续进行中周期

```bash
/dev-cycle --resume
```

读取项目根目录 `.dev-cycle.json`，从断点继续。

### 查看进度

```bash
/dev-cycle --status
```

显示当前处于哪个阶段、已完成哪些产出。

### 终止周期

```bash
/dev-cycle --abort
```

清除 `.dev-cycle.json`，可开始新周期。

## 4 个阶段

### Stage 1: 方案设计

调用内置 brainstorming 流程，与用户对话式澄清需求，提出 2-3 个方案对比，逐段呈现设计并获取确认。设计文档落地到项目 `docs/dev-cycle/` 目录，包含：背景与目标、方案选择、架构设计、接口设计、错误处理、不做事项（YAGNI）。

### Stage 2: 设计对抗审查

调用 `engineering-skills:adversarial-reviewer`，以 3 个角色审查设计方案：
- **Saboteur** — 最挑剔的工程师，寻找一切可能在生产环境崩溃的路径
- **New Hire** — 新入职成员，关注可理解性和可维护性
- **Security Auditor** — 安全审核员，寻找攻击面

用户决定哪些 Critical/Warning 发现需要修复。

### Stage 3: Karpathy 准则编码

调用 `andrej-karpathy-skills:karpathy-guidelines`，在以下准则指导下编码：
- 清晰 > 巧妙
- 小函数、好命名
- 适度抽象（不做提前接口层）
- 先读项目代码风格后写
- 尽早处理错误
- 注释说明"为什么"而非"是什么"

每完成一个模块做增量编译检查，所有变更后做完整编译验证。

### Stage 4: 代码对抗审查

再次调用 adversarial-reviewer，这次审查代码 diff 而非设计文档。重点：线程安全、内存管理、错误处理、遵循现有模式、安全问题。

发现问题可修复后循环 S4 直到通过。

### 完成

S4 通过后自动执行 `git commit`（不 push），commit SHA 记录到 `.dev-cycle.json`。

## 状态管理

项目根目录 `.dev-cycle.json` 记录完整周期状态：

```json
{
  "topic": "ptz-control",
  "current_stage": 2,
  "stages": {
    "1": { "status": "done", "artifact": "docs/dev-cycle/2026-05-15-ptz-control-design.md" },
    "2": { "status": "pending", "artifact": "docs/dev-cycle/2026-05-15-ptz-control-adv-review-d1.md" },
    "3": { "status": "pending", "artifact": null },
    "4": { "status": "pending", "artifact": "docs/dev-cycle/2026-05-15-ptz-control-adv-review-code.md" }
  },
  "decisions": {
    "s2_findings_to_fix": ["线程安全问题"],
    "s2_findings_deferred": ["SDP校验(低风险)"]
  },
  "started_at": "2026-05-15T10:00:00+08:00",
  "completed_at": null,
  "commit_sha": null
}
```

跨会话通过此文件恢复进度。

## 操作权限

本 skill 在流程中自动放行的操作：

| 类别 | 操作 | 说明 |
|------|------|------|
| git 命令 | diff/status/log/add/commit/checkout/branch | 开发标准操作 |
| 构建命令 | cmake、build_*.sh、clang-format | 本地构建和格式化 |
| 文件编辑 | 所有源码 + `.claude/` 配置 | 项目内读写 |

**需用户确认的操作：** `git push`、批量删除、安装新依赖

## 前置依赖 Skill

本 skill 在运行过程中会调用以下子 skill，需已安装：

| Skill | 使用阶段 | 来源 |
|-------|----------|------|
| `engineering-skills:adversarial-reviewer` | S2, S4 | engineering-skills 插件 |
| `andrej-karpathy-skills:karpathy-guidelines` | S3 | andrej-karpathy-skills 插件 |

如果子 skill 不可用，会回退为手动执行对应流程。

## 文件结构

```
~/.claude/skills/dev-cycle/
├── SKILL.md              # 主 skill 定义
└── README.md             # 本文件

<项目根目录>/
├── .dev-cycle.json       # 周期状态（自动管理）
└── docs/dev-cycle/        # 每阶段产出物
    ├── YYYY-MM-DD-<topic>-design.md
    ├── YYYY-MM-DD-<topic>-adv-review-d1.md
    └── YYYY-MM-DD-<topic>-adv-review-code.md
```

## 边界处理

| 场景 | 处理方式 |
|------|----------|
| S1 中用户退出会话 | resume 后从 S1 继续 |
| 设计有致命缺陷 | 回 S1 修订 → 重新 S2 |
| 编码发现设计漏洞 | 停止编码，回 S1 修订 |
| S4 发现严重问题 | 修复后循环 S4 直到通过 |
| commit 时无变更 | 跳过 commit，输出说明 |
