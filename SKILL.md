---
name: dev-cycle
version: 1.1.0
description: >
  分阶段开发工作流。将复杂开发任务分解为4个独立阶段：
  S1: 方案设计(brainstorming) → S2: 设计方案对抗审查(adversarial-reviewer) →
  S3: Karpathy准则编码(karpathy-guidelines) → S4: 代码质量对抗审查(adversarial-reviewer)。
  每个阶段产出落地，用户验收后进入下一阶段。跨项目可用。
trigger: /dev-cycle
author: user
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Bash
  - AskUserQuestion
  - Skill
  - TaskCreate
  - TaskUpdate
---

# /dev-cycle — 分阶段开发工作流

## 概述

将复杂开发任务分解为 4 个独立阶段，阶段间有明确产出和验收门禁。
用户级 skill，适用于任意项目目录。

### 流程一览

```
Stage 1: brainstorming       → docs/dev-cycle/...-design.md
Stage 2: adversarial-review  → docs/dev-cycle/...-adv-review-d1.md  (设计审查)
Stage 3: karpathy-guidelines → 实际代码变更
Stage 4: adversarial-review  → docs/dev-cycle/...-adv-review-code.md (代码审查)
```

每个阶段之间**用户验收**后才能推进下一阶段。

## 操作权限

本 skill 在执行流程中会对以下操作自动放行，无需用户逐次确认：

- **git 命令**: `git diff`、`git status`、`git log`、`git add`、`git commit`、`git checkout`、`git branch` — 这些是开发流程中的标准操作，不涉及远程推送风险
- **构建命令**: `cmake`、`build_*.sh` 脚本、`clang-format` — 本地构建和代码格式化
- **文件操作**: 项目所有源码文件（`src/`、`demo/`、`docs/`、`CMakeLists.txt` 等）及 `.claude/` 配置文件的读写编辑 — 全部自动放行

**例外 — 以下操作仍需用户确认**:
- `git push` — 推送到远程仓库
- 删除大量文件或目录
- 安装新依赖或全局工具

## 状态管理

项目根目录维护 `.dev-cycle.json`，记录当前周期进度和跨阶段决策。

```json
{
  "topic": "简短主题",
  "current_stage": 1,
  "stages": {
    "1": { "status": "done", "artifact": "docs/dev-cycle/2026-05-15-xxx-design.md" },
    "2": { "status": "pending", "artifact": "docs/dev-cycle/2026-05-15-xxx-adv-review-d1.md" },
    "3": { "status": "pending", "artifact": null },
    "4": { "status": "pending", "artifact": "docs/dev-cycle/2026-05-15-xxx-adv-review-code.md" }
  },
  "s1_brainstorming_notes": "",  // 恢复 S1 时的快速上下文
  "decisions": {
    "s2_findings_to_fix": [],
    "s2_findings_deferred": [],
    "s4_findings_to_fix": [],
    "s4_findings_deferred": []
  },
  "started_at": "2026-05-15T10:00:00+08:00",
  "completed_at": null,
  "commit_sha": null // 完成后自动提交的 git commit hash
}
```

## 入口逻辑

当用户运行 `/dev-cycle [task描述]` 或 `/dev-cycle --resume`:

1. **不存在 `.dev-cycle.json`** → 开始新周期，进入 **Stage 1**
2. **存在且 `current_stage` 有未完成 stage** → 从该 stage 继续
3. **所有 stage 已完成** → 报告完成（含 commit SHA），提示运行 `/dev-cycle --abort` 开始新周期
4. **参数 `--status`** → 仅显示当前进度概览，不执行任何 stage
5. **参数 `--abort`** → 删除 `.dev-cycle.json`，清除状态

首次调用时 `task描述` 为必填。后续 resume 时可选。

---

## Stage 1: 方案设计 (brainstorming)

**Skill:** 内置 brainstorming 流程（不单独调用子 skill，按规则执行）
**产出:** `docs/dev-cycle/YYYY-MM-DD-<topic>-design.md`
**门禁:** 用户审查 spec 文件并 approve

### 行为

1. 读取 CLAUDE.md、项目 README 等上下文文件
2. 读取项目根目录 `.dev-cycle.json`（刚创建，S1 pending）
3. 创建任务列表跟踪子步骤：
   - `[ ] 探索项目上下文` — 检查相关源码、最近提交
   - `[ ] 需求澄清` — 一次一问，理解任务目标、约束、成功标准
   - `[ ] 提出 2-3 方案` — 带权衡分析和推荐
   - `[ ] 逐段呈现设计` — 每段确认后继续（架构、组件、数据流、错误处理）
   - `[ ] 写 design doc` — 保存并提交 git
   - `[ ] Spec 自检` — 检查占位符、内部矛盾、模糊表述
4. **设计方案时不可引入当前协议栈以外的第三方依赖**，除非用户明确要求
5. 设计文档必须包含：
   - **背景与目标** — 为什么做、解决什么问题
   - **方案选择** — 对比过的方案及选择理由
   - **架构设计** — 组件图/数据流描述
   - **接口设计** — 函数签名、数据结构
   - **错误处理** — 预期异常及处置策略
   - **不做的事项** — 明确排除的范围（YAGNI）
6. 文档写入后，运行 **Spec 自检**：
   - 是否有 "TBD"/"TODO"/"略" 等占位符？→ 修复
   - 各节是否矛盾？→ 修复
   - 是否有模糊可两解的表述？→ 明确化
7. 更新 `.dev-cycle.json`：`current_stage = 2`, `stages[1].status = "done"`
8. 输出总结，提示用户审查 spec：
   > "设计文档已写入 `<path>`，请审查后运行 `/dev-cycle --resume` 进入 Stage 2。"

### 关键规则

- 一次只问一个问题，不要多问
- 偏好选择题（A/B/C），但开放问题也可
- 每个 section 呈现后询问"这部分是否合适？"
- YAGNI 原则：从设计中移除不必要的功能

---

## Stage 2: 设计方案对抗审查 (adversarial-reviewer)

**Skill:** `engineering-skills:adversarial-reviewer`
**输入:** Stage 1 产出的 design.md + CLAUDE.md + 相关源码上下文
**产出:** `docs/dev-cycle/YYYY-MM-DD-<topic>-adv-review-d1.md`
**门禁:** 用户决定哪些 findings 修/不修

### 行为

1. 读取 `.dev-cycle.json`，确认 S1 已完成
2. 读取 design.md，读取 CLAUDE.md 和相关源码头文件了解上下文
3. **调用 Skill 工具:** `engineering-skills:adversarial-reviewer`，传入审查范围：
   - 审查对象: `docs/dev-cycle/...-design.md` 设计文档
   - 角色: Saboteur / New Hire / Security Auditor
4. 等待 adversarial-reviewer 完成后，获取其产出
5. 将审查报告写入 `docs/dev-cycle/YYYY-MM-DD-<topic>-adv-review-d1.md`
6. 呈现给用户，逐一询问每个 CRITICAL/WARNING finding 是否修复：
   - "Finding #N [CRITICAL]: xxx — 是否修复？(y/n)"
   - 用户答复后记录到 `.dev-cycle.json` 的 `decisions.s2_findings_to_fix` / `s2_findings_deferred`
7. 更新 `.dev-cycle.json`：`current_stage = 3`, `stages[2].status = "done"`
8. 提示用户运行 `/dev-cycle --resume` 进入 Stage 3

### 注意

- 如果 adversarial-reviewer skill 不可用，使用其方法（3 角色分析）手动执行
- NOTE 级别的 finding 自动 defer，不询问用户

---

## Stage 3: Karpathy 准则编码实施

**Skill:** `andrej-karpathy-skills:karpathy-guidelines`
**输入:** design.md + s2_findings_to_fix + 项目现有代码
**产出:** 实际代码变更 (git diff)

### 行为

1. 读取 `.dev-cycle.json`，确认 S2 已完成
2. 读取 design.md 及 `decisions.s2_findings_to_fix`
3. **调用 Skill 工具:** `andrej-karpathy-skills:karpathy-guidelines`
4. 在 karpathy-guidelines 的指导下编码，核心准则：

   - **清晰 > 巧妙** — 代码可读性优先，不做聪明但晦涩的实现
   - **小函数** — 每个函数做一件事，函数名描述做什么而非怎么做
   - **好命名** — 变量/函数/类名表达意图，不加匈牙利/无意义前缀
   - **适度抽象** — 不需要的抽象比缺失的抽象更糟；不要提前做接口层
   - **先读后写** — 先阅读项目现有代码风格和模式，严格遵循
   - **错误处理** — 尽早返回错误，不在深层嵌套中隐藏
   - **注释应说明"为什么"** — 代码本身说明"是什么"，注释解释"为什么这么写"

5. 每次编辑前先 `git diff` 确认当前工作区干净
6. 每完成一个独立功能模块后做增量编译检查（`cmake --build`），有错立即修
7. 所有变更完成后，运行一次完整编译验证
8. 更新 `.dev-cycle.json`：`current_stage = 4`, `stages[3].status = "done"`
9. 输出变更摘要（改了什么、改了哪些文件），提示用户运行 `/dev-cycle --resume` 进入 Stage 4

### 编码顺序

- 接口/数据结构定义 → 核心逻辑 → 集成与对接 → 清理（旧代码移除）
- 如果 design.md 有 Phase 划分，按 Phase 顺序实施

### Rollback 策略

- 如果实施中途发现设计有重大缺陷，**立即停止编码**，输出问题描述，要求用户决定：修订设计后再编码 / 终止当前周期
- 更新 `.dev-cycle.json` 并在 decisions 中记录问题

---

## Stage 4: 代码质量对抗审查 (adversarial-reviewer)

**Skill:** `engineering-skills:adversarial-reviewer`（代码审查模式）
**输入:** Stage 3 的完整 git diff + 上下文代码
**产出:** `docs/dev-cycle/YYYY-MM-DD-<topic>-adv-review-code.md`
**门禁:** 用户决定哪些 issues 修复（修复后可重启 S4）

### 行为

1. 读取 `.dev-cycle.json`，确认 S3 已完成
2. 运行 `git diff <base-branch>...HEAD` 获取完整变更
3. 读取所有变更文件的上下文（改了什么、改了哪些行）
4. **调用 Skill 工具:** `engineering-skills:adversarial-reviewer`，传入：
   - 审查范围: git diff
   - 角色: Saboteur / New Hire / Security Auditor
   - 审查重点: 代码质量、线程安全、内存管理、错误处理、遵循现有模式、安全问题
5. 等待审查完成
6. 将审查报告写入 `docs/dev-cycle/YYYY-MM-DD-<topic>-adv-review-code.md`
7. 呈现给用户，逐一询问每个 CRITICAL/WARNING finding 是否修复
8. 更新 `.dev-cycle.json`：
   - 记录 decisions (s4_findings_to_fix / s4_findings_deferred)
   - `current_stage = 4`（保持在 4 直到用户确认所有 fixes 完成）

### 修复迭代

- 如果用户选择修复某些 findings：
  - 修复代码
  - 更新 `.dev-cycle.json.stages[4].status = "pending"` 重置状态
  - 再次运行 S4 审查更新后的代码
- 如果用户确认所有 findings 已处理或 defer：
  - 更新 `.dev-cycle.json.stages[4].status = "done"`，`current_stage = 5`, `completed_at = now()`

### 完成（含自动提交）

当 S4 所有 findings 已处理或 defer，进入完成阶段：

1. 更新 `.dev-cycle.json.stages[4].status = "done"`，`current_stage = 5`，`completed_at = now()`
2. **自动 git commit** — 不询问用户，直接执行：
   - `git add -A` 暂存所有变更
   - `git commit -m "<类型>: <topic> — <变更摘要>"`，参考项目 commit 风格
   - 获取 commit SHA 并写入 `.dev-cycle.json.commit_sha`
   - **不执行 `git push`**
3. 输出最终总结：变更文件列表、commit SHA、审查结果摘要

---

## 内置辅助指令

### `/dev-cycle --status`

显示当前进度表格：

```
┌─────────┬──────────────┬──────────────────────────────────────┐
│ Stage   │ 状态         │ 产出                                  │
├─────────┼──────────────┼──────────────────────────────────────┤
│ S1 设计  │ ✅ done      │ docs/dev-cycle/...-design.md         │
│ S2 审查  │ ✅ done      │ docs/dev-cycle/...-adv-review-d1.md  │
│ S3 编码  │ 🔄 in progress │ (进行中)                            │
│ S4 审查  │ ⏳ pending   │ -                                    │
└─────────┴──────────────┴──────────────────────────────────────┘
决定: S2 修复 3/5 findings
```

### `/dev-cycle --abort`

- 删除 `.dev-cycle.json`
- 输出 "开发周期已终止。运行 `/dev-cycle <task>` 开始新周期。"

### 找不到 `.dev-cycle.json` 时

如果运行 `/dev-cycle --resume` 但 `.dev-cycle.json` 不存在，报错：
"没有正在进行的开发周期。运行 `/dev-cycle <任务描述>` 开始新周期。"

---

## 边界/异常处理

| 场景 | 处理 |
|------|------|
| S1 中用户退出会话 | 重新运行 `/dev-cycle --resume` 从 S1 继续（状态已存） |
| S2 审查到设计有致命缺陷 | 回到 S1 修订 design.md，然后重新进入 S2 |
| S3 编码遇到设计漏洞 | 立即停止，向用户报告，回 S1 修订 |
| S4 发现严重代码问题 | 修复后循环 S4 直到通过 |
| decision 冲突（用户改主意） | 支持手动编辑 `.dev-cycle.json.decisions` |
| 项目没有 docs/dev-cycle 目录 | 自动创建 |
| 周期完成时工作区有未跟踪文件 | 一并 git add + commit |
| git commit 失败（如无变更） | 跳过 commit，输出说明 |
| 用户想在完成后手动修改 commit | 告知 commit SHA，使用 `git commit --amend` 修改 |
