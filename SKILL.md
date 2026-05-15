---
name: dev-cycle
version: 1.1.0
description: >
  4-stage development workflow for Claude Code:
  S1: Solution Design (brainstorming) → S2: Design Adversarial Review (adversarial-reviewer) →
  S3: Karpathy-Principled Coding (karpathy-guidelines) → S4: Code Adversarial Review (adversarial-reviewer).
  Each stage produces artifacts with user review gates. Cross-project user-level skill.
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

# /dev-cycle — Multi-Stage Development Workflow

## Overview

Breaks complex development tasks into 4 independent stages. Each stage produces concrete artifacts and requires user approval before proceeding.

```
Stage 1: brainstorming       → docs/dev-cycle/...-design.md
Stage 2: adversarial-review  → docs/dev-cycle/...-adv-review-d1.md  (design review)
Stage 3: karpathy-guidelines → actual code changes
Stage 4: adversarial-review  → docs/dev-cycle/...-adv-review-code.md (code review)
```

**User approval is required** between every stage.

## Auto-Approved Operations

The following operations run without user confirmation during the workflow:

- **git commands**: `git diff`, `git status`, `git log`, `git add`, `git commit`, `git checkout`, `git branch` — standard development operations, no remote push risk
- **Build commands**: `cmake`, `build_*.sh` scripts, `clang-format` — local build and formatting
- **File edits**: All project source files (`src/`, `demo/`, `docs/`, `CMakeLists.txt`, etc.) and `.claude/` configuration files — fully auto-approved

**Exceptions — still require confirmation**:
- `git push` — pushing to remote repository
- Bulk file deletion
- Installing new dependencies or global tools

## State Management

A `.dev-cycle.json` file in the project root tracks progress across sessions.

```json
{
  "topic": "short-topic",
  "current_stage": 1,
  "stages": {
    "1": { "status": "done", "artifact": "docs/dev-cycle/2026-05-15-xxx-design.md" },
    "2": { "status": "pending", "artifact": "docs/dev-cycle/2026-05-15-xxx-adv-review-d1.md" },
    "3": { "status": "pending", "artifact": null },
    "4": { "status": "pending", "artifact": "docs/dev-cycle/2026-05-15-xxx-adv-review-code.md" }
  },
  "s1_brainstorming_notes": "",
  "decisions": {
    "s2_findings_to_fix": [],
    "s2_findings_deferred": [],
    "s4_findings_to_fix": [],
    "s4_findings_deferred": []
  },
  "started_at": "2026-05-15T10:00:00+08:00",
  "completed_at": null,
  "commit_sha": null
}
```

## Entry Logic

When the user runs `/dev-cycle [task description]` or `/dev-cycle --resume`:

1. **No `.dev-cycle.json` exists** → start a new cycle, enter **Stage 1**
2. **File exists with unfinished stages** → resume from the current stage
3. **All stages complete** → report completion (with commit SHA), prompt `/dev-cycle --abort` to start a new cycle
4. **`--status` flag** → show progress overview only, no stage execution
5. **`--abort` flag** → delete `.dev-cycle.json`, clear state

Task description is required on first call. Optional on resume.

---

## Stage 1: Solution Design (brainstorming)

**Skill:** Built-in brainstorming process (no sub-skill invocation)
**Artifact:** `docs/dev-cycle/YYYY-MM-DD-<topic>-design.md`
**Gate:** User reviews and approves the spec file

### Behavior

1. Read CLAUDE.md, project README, and other context files
2. Read `.dev-cycle.json` from project root (freshly created, S1 pending)
3. Create task list tracking substeps:
   - `[ ] Explore project context` — relevant source code, recent commits
   - `[ ] Clarify requirements` — one question at a time, understand goals, constraints, success criteria
   - `[ ] Propose 2-3 approaches` — with trade-off analysis and recommendation
   - `[ ] Present design section by section` — confirm each before proceeding (architecture, components, data flow, error handling)
   - `[ ] Write design doc` — save and git commit
   - `[ ] Spec self-review` — check for placeholders, contradictions, ambiguity
4. **Do not introduce third-party dependencies outside the current tech stack** unless explicitly requested
5. Design document must include:
   - **Background & Goals** — why this change, what problem it solves
   - **Options Considered** — alternatives compared and selection rationale
   - **Architecture** — component diagram / data flow description
   - **Interface Design** — function signatures, data structures
   - **Error Handling** — expected exceptions and mitigation strategies
   - **Out of Scope** — explicitly excluded items (YAGNI)
6. After writing, run **Spec Self-Review**:
   - Any "TBD"/"TODO"/"略" placeholders? → fix
   - Any contradictions between sections? → fix
   - Any ambiguous phrasing? → clarify
7. Update `.dev-cycle.json`: `current_stage = 2`, `stages[1].status = "done"`
8. Output summary, ask user to review spec:
   > "Design doc written to `<path>`. Please review it, then run `/dev-cycle --resume` to proceed to Stage 2."

### Key Rules

- One question at a time, never batch multiple questions
- Prefer multiple choice (A/B/C) when possible
- Ask "does this look right?" after each section
- YAGNI: remove unnecessary features from the design

---

## Stage 2: Design Adversarial Review

**Skill:** `engineering-skills:adversarial-reviewer`
**Input:** Stage 1 design.md + CLAUDE.md + relevant source context
**Artifact:** `docs/dev-cycle/YYYY-MM-DD-<topic>-adv-review-d1.md`
**Gate:** User decides which findings to fix or defer

### Behavior

1. Read `.dev-cycle.json`, confirm S1 is complete
2. Read design.md, CLAUDE.md, and relevant header files for context
3. **Invoke Skill:** `engineering-skills:adversarial-reviewer`, specify scope:
   - Target: `docs/dev-cycle/...-design.md`
   - Roles: Saboteur / New Hire / Security Auditor
4. Wait for adversarial-reviewer to complete, capture its output
5. Write review report to `docs/dev-cycle/YYYY-MM-DD-<topic>-adv-review-d1.md`
6. Present findings to user, ask about each CRITICAL/WARNING finding:
   - "Finding #N [CRITICAL]: xxx — fix this? (y/n)"
   - Record decisions to `.dev-cycle.json` `decisions.s2_findings_to_fix` / `s2_findings_deferred`
7. Update `.dev-cycle.json`: `current_stage = 3`, `stages[2].status = "done"`
8. Prompt user to run `/dev-cycle --resume` to proceed to Stage 3

### Notes

- If adversarial-reviewer skill is unavailable, execute its method manually (3-role analysis)
- NOTE-level findings are automatically deferred without asking

---

## Stage 3: Karpathy-Principled Coding

**Skill:** `andrej-karpathy-skills:karpathy-guidelines`
**Input:** design.md + s2_findings_to_fix + existing project code
**Output:** Actual code changes (git diff)

### Behavior

1. Read `.dev-cycle.json`, confirm S2 is complete
2. Read design.md and `decisions.s2_findings_to_fix`
3. **Invoke Skill:** `andrej-karpathy-skills:karpathy-guidelines`
4. Code guided by these core principles:

   - **Clarity > cleverness** — readability first, no clever-but-obscure implementations
   - **Small functions** — each function does one thing, name describes what not how
   - **Good naming** — variables/functions/classes express intent, no Hungarian notation or meaningless prefixes
   - **Just-right abstraction** — unnecessary abstraction is worse than missing abstraction; no premature interfaces
   - **Read before writing** — read existing project code style and patterns first, follow them strictly
   - **Error handling** — return errors early, don't bury them in deep nesting
   - **Comments explain "why"** — code says "what", comments say "why"

5. Run `git diff` before each edit to confirm clean working tree
6. Run incremental build check (`cmake --build`) after each module, fix errors immediately
7. Run full build verification after all changes
8. Update `.dev-cycle.json`: `current_stage = 4`, `stages[3].status = "done"`
9. Output change summary (what changed, which files), prompt user to run `/dev-cycle --resume` for Stage 4

### Coding Order

- Interface/data structure definitions → core logic → integration → cleanup (remove old code)
- If design.md has Phase divisions, follow Phase order

### Rollback Strategy

- If a major design flaw is discovered during coding, **stop immediately**, describe the problem, and ask the user: revise design then continue / abort current cycle
- Update `.dev-cycle.json` and record the issue in decisions

---

## Stage 4: Code Adversarial Review

**Skill:** `engineering-skills:adversarial-reviewer` (code review mode)
**Input:** Full git diff from Stage 3 + context code
**Artifact:** `docs/dev-cycle/YYYY-MM-DD-<topic>-adv-review-code.md`
**Gate:** User decides which issues to fix (S4 can be re-run after fixes)

### Behavior

1. Read `.dev-cycle.json`, confirm S3 is complete
2. Run `git diff <base-branch>...HEAD` to get full change set
3. Read context around all changed files
4. **Invoke Skill:** `engineering-skills:adversarial-reviewer`, specify:
   - Scope: git diff
   - Roles: Saboteur / New Hire / Security Auditor
   - Focus: code quality, thread safety, memory management, error handling, project pattern adherence, security
5. Wait for review to complete
6. Write review report to `docs/dev-cycle/YYYY-MM-DD-<topic>-adv-review-code.md`
7. Present findings to user, ask about each CRITICAL/WARNING finding
8. Update `.dev-cycle.json`:
   - Record decisions (s4_findings_to_fix / s4_findings_deferred)
   - `current_stage = 4` (stay at 4 until user confirms all fixes done)

### Fix Loop

- If user chooses to fix findings:
  - Apply fixes
  - Reset `.dev-cycle.json.stages[4].status = "pending"`
  - Re-run S4 review on updated code
- When user confirms all findings handled or deferred:
  - Set `.dev-cycle.json.stages[4].status = "done"`, `current_stage = 5`, `completed_at = now()`

### Completion (with auto-commit)

When S4 all findings are handled or deferred:

1. Update `.dev-cycle.json.stages[4].status = "done"`, `current_stage = 5`, `completed_at = now()`
2. **Auto git commit** — no user prompt:
   - `git add -A` stage all changes
   - `git commit -m "<type>: <topic> — <summary>"`, follow project commit style
   - Record commit SHA in `.dev-cycle.json.commit_sha`
   - **Do NOT `git push`**
3. Output final summary: changed file list, commit SHA, review result summary

---

## Built-in Commands

### `/dev-cycle --status`

Show progress table:

```
┌─────────┬──────────────┬──────────────────────────────────────┐
│ Stage   │ Status       │ Artifact                              │
├─────────┼──────────────┼──────────────────────────────────────┤
│ S1      │ ✅ done      │ docs/dev-cycle/...-design.md          │
│ S2      │ ✅ done      │ docs/dev-cycle/...-adv-review-d1.md   │
│ S3      │ 🔄 in progress │ (working)                            │
│ S4      │ ⏳ pending   │ -                                     │
└─────────┴──────────────┴──────────────────────────────────────┘
Decisions: S2 fix 3/5 findings
```

### `/dev-cycle --abort`

- Delete `.dev-cycle.json`
- Output: "Cycle aborted. Run `/dev-cycle <task>` to start a new cycle."

### Missing `.dev-cycle.json`

If `/dev-cycle --resume` is run but `.dev-cycle.json` doesn't exist:
"No active development cycle found. Run `/dev-cycle <task description>` to start a new one."

---

## Edge Case Handling

| Scenario | Handling |
|----------|----------|
| User exits mid-S1 | Resume from S1 on next `/dev-cycle --resume` |
| S2 reveals fatal design flaws | Revise design.md in S1, then re-enter S2 |
| S3 coding uncovers design gaps | Stop immediately, report to user, revise S1 |
| S4 finds critical code issues | Fix → loop S4 until clean |
| Decision conflict (user changes mind) | Support manual `.dev-cycle.json.decisions` editing |
| Project has no docs/dev-cycle dir | Auto-create |
| Untracked files at cycle completion | Include in git add + commit |
| git commit fails (no changes) | Skip commit, report no-op |
| User wants to amend final commit | Provide commit SHA, suggest `git commit --amend` |
