# /dev-cycle — Multi-Stage Development Workflow

A cross-project user-level skill for Claude Code that breaks complex development tasks into 4 independent stages, each with clear artifacts and user review gates.

## Flow Overview

```
/dev-cycle "task description"
       │
       ▼
┌──────────────────────────────────────┐
│ S1: Solution Design (brainstorming)   │  →  docs/dev-cycle/...-design.md
│    Requirements → Options → Design    │
└──────────────┬───────────────────────┘
               │ User reviews spec
               ▼
┌──────────────────────────────────────┐
│ S2: Design Adversarial Review         │  →  docs/dev-cycle/...-adv-review-d1.md
│    Saboteur / New Hire / Security     │
└──────────────┬───────────────────────┘
               │ Decide findings to fix
               ▼
┌──────────────────────────────────────┐
│ S3: Karpathy-Principled Coding        │  →  Code changes
│    Clarity > cleverness, small funcs  │
└──────────────┬───────────────────────┘
               │ Code complete
               ▼
┌──────────────────────────────────────┐
│ S4: Code Adversarial Review           │  →  docs/dev-cycle/...-adv-review-code.md
│    Thread safety / Memory / Errors    │
└──────────────┬───────────────────────┘
               │ Approved
               ▼
          ✅ Auto git commit
            (no push)
```

## Installation

```bash
# Clone into the Claude Code user skills directory
mkdir -p ~/.claude/skills
git clone https://github.com/vintonliu/dev-cycle.git ~/.claude/skills/dev-cycle
```

Restart Claude Code after installation — the `/dev-cycle` command will be available.

## Usage

### Start a new cycle

```bash
/dev-cycle "Add user authentication module"
```

Claude enters Stage 1 automatically to clarify requirements and design the solution.

### Resume an ongoing cycle

```bash
/dev-cycle --resume
```

Reads `.dev-cycle.json` from the project root and resumes from the current stage.

### Check progress

```bash
/dev-cycle --status
```

Shows which stage is active and what artifacts have been produced.

### Abort a cycle

```bash
/dev-cycle --abort
```

Clears `.dev-cycle.json` — start a new cycle anytime.

## The 4 Stages

### Stage 1: Solution Design

Uses the built-in brainstorming process: conversational requirement clarification, 2-3 solution proposals with trade-off analysis, and section-by-section design validation. The design document is saved to the project's `docs/dev-cycle/` directory and must include: background & goals, options considered, architecture, interface design, error handling, and explicit scope exclusions (YAGNI).

### Stage 2: Design Adversarial Review

Invokes `engineering-skills:adversarial-reviewer` to examine the design from 3 perspectives:

- **Saboteur** — seeks every possible production crash path
- **New Hire** — evaluates understandability and maintainability
- **Security Auditor** — identifies attack surfaces

The user decides which Critical/Warning findings to fix and which to defer.

### Stage 3: Karpathy-Principled Coding

Invokes `andrej-karpathy-skills:karpathy-guidelines` and codes under these principles:
- Clarity over cleverness
- Small functions, good names
- Just-right abstraction (no premature interfaces)
- Read existing project style before writing
- Handle errors early
- Comments explain "why", not "what"

Each module is incrementally compiled. A full build verification runs after all changes.

### Stage 4: Code Adversarial Review

Invokes adversarial-reviewer again, this time reviewing the code diff rather than the design document. Focus areas: thread safety, memory management, error handling, project pattern adherence, security issues.

Issues found can be fixed in a loop — S4 reruns until the user approves all findings.

### Completion

When S4 passes, an automatic `git commit` is executed (no push). The commit SHA is recorded in `.dev-cycle.json`.

## State Management

A `.dev-cycle.json` file in the project root tracks progress across sessions:

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
    "s2_findings_to_fix": ["Thread safety in state machine"],
    "s2_findings_deferred": ["Input validation (low risk)"]
  },
  "started_at": "2026-05-15T10:00:00+08:00",
  "completed_at": null,
  "commit_sha": null
}
```

## Auto-Approved Operations

To reduce permission prompts, these operations run without user confirmation:

| Category | Operations | Notes |
|----------|-----------|-------|
| git commands | diff, status, log, add, commit, checkout, branch | Standard dev flow |
| Build commands | cmake, build scripts, formatters | Local build only |
| File edits | All source files, `.claude/` config | Project-internal |

**Still requires confirmation:** `git push`, bulk deletion, installing new dependencies/system tools

## Prerequisite Skills

These sub-skills are called during the workflow and must be installed:

| Skill | Used in | Source |
|-------|---------|--------|
| `engineering-skills:adversarial-reviewer` | S2, S4 | engineering-skills plugin |
| `andrej-karpathy-skills:karpathy-guidelines` | S3 | andrej-karpathy-skills plugin |

If a sub-skill is unavailable, the workflow falls back to executing its logic manually.

## File Structure

```
~/.claude/skills/dev-cycle/
├── SKILL.md              # Main skill definition
├── README.md             # This file (English)
└── README-zh.md          # Chinese translation

<project-root>/
├── .dev-cycle.json       # Cycle state (auto-managed)
└── docs/dev-cycle/        # Stage artifacts
    ├── YYYY-MM-DD-<topic>-design.md
    ├── YYYY-MM-DD-<topic>-adv-review-d1.md
    └── YYYY-MM-DD-<topic>-adv-review-code.md
```

## Edge Case Handling

| Scenario | Handling |
|----------|----------|
| User exits mid-S1 | Resume from S1 on next run |
| Design has fatal flaws | Revise S1 design → re-enter S2 |
| Coding reveals design gaps | Stop coding, report to user, revise S1 |
| S4 finds critical issues | Fix → re-run S4 until clean |
| No changes to commit | Skip commit, report no-op |
| User wants to amend commit | Provide SHA, suggest `git commit --amend` |
