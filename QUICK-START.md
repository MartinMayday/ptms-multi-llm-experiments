# Quick Start Guide: Project Task Management System

## 🎯 Recommended Approach

Use **GSD Workflow** for Plan 01 + **Agent Orchestration** for Plans 02-05

---

## Option 1: Quick & Simple (Recommended)

```bash
# Step 1: Start with GSD Workflow
/gsd-workflow

# Step 2: When GSD asks what to do, say:
"Execute Plan 01: Foundation from .planning/project-task-management-system/phase-01-foundation/01-01-PLAN.md"

# Step 3: GSD will:
# - Load plan file
# - Execute tasks 1-7 with fresh context per task
# - Verify each task with provided verify commands
# - Create atomic commits
# - Pause for context refresh between tasks

# To continue after a pause:
/execute-plan .planning/project-task-management-system/phase-01-foundation/01-01-PLAN.md
```

**Time Estimate**: 5-7 days for Plan 01

---

## Option 2: Parallel Specialist Agents (After Plan 01 Complete)

Once Foundation is ready, launch parallel experts:

```bash
# Step 1: Use Agents skill
/Agents

# Step 2: Request parallel execution:
"Launch 4 specialist agents to execute Plans 02-05 in parallel:
- Agent 1 (Backend): Execute Plan 02 (Workflow Engine)
- Agent 2 (Backend): Execute Plan 03 (Backend API)
- Agent 3 (Frontend): Execute Plan 04 (Web Interface)
- Agent 4 (Protocol): Execute Plan 05 (Agent Interface)"

# Step 3: Each agent will:
# - Load EngineerContext (or appropriate context)
# - Read their plan's README.md and phases
# - Execute tasks sequentially with fresh context
# - Report completion
```

**Time Estimate**: 3-5 days for Plans 02-05 (parallel execution)

---

## Option 3: Session Management (Alternative)

```bash
# Execute Plan 01 with context breaks
/execute-plan .planning/project-task-management-system/phase-01-foundation/01-01-PLAN.md

# After each task completes:
# - Session pauses automatically
# - Context is refreshed
# - Review task completion
# - Continue with: /execute-plan .planning/project-task-management-system/phase-01-foundation/01-01-PLAN.md
```

---

## 📊 Decision Matrix

| Situation | Recommended Approach | Why |
|-----------|---------------------|------|
| **Starting fresh** | GSD Workflow | Built-in context freshness |
| **Foundation layer critical** | GSD Workflow | Fresh context prevents bugs |
| **Multiple plans ready** | Agent Orchestration | Parallel execution saves time |
| **Small team** | GSD Workflow | Less coordination overhead |
| **Large team with specialists** | Agent Orchestration | Leverage all expertise |
| **Need checkpoints** | Session Management | Explicit pause/resume |

---

## 🔧 What Actually Happens Under the Hood

### GSD Workflow Execution

```
Your Input: "Execute Plan 01 with GSD"

┌─────────────────────────────────────────────────────────┐
│  GSD Skill: Load gsd-workflow/SKILL.md         │
├─────────────────────────────────────────────────────────┤
│                                                       │
│  1. Load plan file (.planning/*/01-01-PLAN.md)     │
│  2. Parse tasks (7 tasks in Plan 01)                │
│  3. Start DISCUSS phase (skip if specs done)       │
│  4. Start PLAN phase (skip if tasks defined)          │
│  5. Start EXECUTE phase:                               │
│     For each task (1-7):                               │
│       a. Launch fresh context subagent                   │
│          → Load EngineerContext                         │
│          → Load phase spec (01-system-architecture.md)    │
│          → Execute task                               │
│          → Verify completion                           │
│          → Atomic commit                               │
│       b. Return result to main context                  │
│       c. Pause for context refresh                      │
│       d. Continue with /execute-plan or auto-resume    │
│  6. Start VERIFY phase:                               │
│     - Run integration tests                             │
│     - Validate against success criteria                 │
│     - Generate coverage report                         │
│                                                       │
└─────────────────────────────────────────────────────────┘
```

### Agent Orchestration Execution

```
Your Input: "Launch parallel agents for Plans 02-05"

┌─────────────────────────────────────────────────────────┐
│  Agents Skill: Load Agents/SKILL.md               │
├─────────────────────────────────────────────────────────┤
│                                                       │
│  1. Load SpawnParallelAgents workflow                  │
│  2. For each requested agent:                          │
│     a. Compose agent personality                        │
│        → Load EngineerContext (or specialized)            │
│        → Add traits (Backend, Frontend, Protocol)       │
│        → Assign voice mapping                           │
│     b. Launch agent in isolated context                  │
│        → Load agent's plan README.md                     │
│        → Load detailed phase specs                       │
│        → Execute tasks sequentially                      │
│        → Each task in fresh context                    │
│        → Report progress                               │
│     c. Collect results when complete                   │
│  3. Return summary of all agent completions            │
│                                                       │
└─────────────────────────────────────────────────────────┘
```

---

## ✅ Skills Being Used

### gsd-workflow
**Path**: `~/.claude/skills/gsd-workflow/SKILL.md`

**What it does:**
- Context-fresh development (200K context per task)
- DISCUSS → PLAN → EXECUTE → VERIFY phases
- Prevents quality degradation from context rot

**Key Feature**: Each task gets FRESH context, so quality stays consistent.

### Agents
**Path**: `~/.claude/skills/Agents/SKILL.md`

**What it does:**
- Dynamic agent composition (traits + personality + voice)
- Parallel agent orchestration
- Custom agent creation

**Key Feature**: Spawn up to 10 specialist agents in parallel.

### session-management
**Path**: `~/.claude/skills/session-management/SKILL.md`

**What it does:**
- `/execute-plan` command
- Pause after each task for context refresh
- Session rollback capability

**Key Feature**: Explicit control over execution flow.

---

## 🚀 Getting Started Right Now

### Step 1: Verify Files

```bash
# Check that spec files exist
ls -la .planning/project-task-management-system/

# Should see:
# - README.md
# - INDEX.md
# - SUMMARY.md
# - phase-01-foundation/ (with 01-01-PLAN.md and phases 01-07)
# - phase-02-workflow-engine/ (README.md)
# - phase-03-backend-api/ (README.md)
# - phase-04-web-interface/ (README.md)
# - phase-05-agent-interface/ (README.md)
```

### Step 2: Review Overview (2 minutes)

```bash
# Read the quick navigation guide
cat .planning/project-task-management-system/INDEX.md

# Or the full summary
cat .planning/project-task-management-system/SUMMARY.md
```

### Step 3: Start Execution

**If you want GSD to handle everything:**
```bash
/gsd-workflow
# Then say: "Execute Plan 01 from .planning/project-task-management-system/phase-01-foundation/01-01-PLAN.md"
```

**If you want to control execution manually:**
```bash
# Start session management
/execute-plan .planning/project-task-management-system/phase-01-foundation/01-01-PLAN.md

# Continue after each task:
/execute-plan .planning/project-task-management-system/phase-01-foundation/01-01-PLAN.md
```

---

## 📖 What About Orchestrator-Agent?

**Good question!** The spec files I created **don't explicitly mention orchestrator-agent**. Here's why:

### Current Structure
The PLAN.md files are designed for **GSD workflow** which:
- Uses `/execute-plan` command (session-management skill)
- Automatically spawns fresh-context subagents via Task tool
- Tracks state across tasks
- Provides built-in orchestration

### Alternative: Custom Orchestrator

If you want a custom orchestrator-agent that manages 10 specialists:

```bash
# You would need to create:
.planning/project-task-management-system/orchestrator.md

# Which would define:
- Orchestrator agent responsibilities
- Specialist agent types (10 types)
- Task assignment logic
- Result aggregation
- Error handling and escalation

# Then use Agents skill to:
/Agents
# And say:
"Create custom orchestrator agent with these responsibilities..."
```

### Recommendation

**For now, use GSD + Agents skills** instead of custom orchestrator:

1. **GSD Workflow** handles foundation (Plan 01)
2. **Agent Orchestration** (SpawnParallelAgents) handles Plans 02-05 in parallel

This leverages **existing, tested skills** without needing custom orchestrator development.

---

## 🎓 Learning More About Skills

To understand each skill in depth:

```bash
# GSD Workflow
cat ~/.claude/skills/gsd-workflow/SKILL.md

# Agent Orchestration
cat ~/.claude/skills/Agents/SKILL.md

# Session Management
cat ~/.claude/skills/session-management/SKILL.md

# Agent Testing (for Plan 04 validation)
cat ~/.claude/skills/Evals/SKILL.md

# Code Structure Search (for finding patterns)
cat ~/.claude/skills/ast-grep/SKILL.md
```

---

## ❓ Common Questions

### Q: Should I use `/gsd:` commands or handoff INDEX.md?

**A:** Use `/gsd-workflow` skill, not `/gsd:` commands directly.

The `/gsd:` commands (like `/gsd:new-project`, `/gsd:plan-phase`) are **slash command triggers** that automatically load the gsd-workflow skill.

**Just use:**
```bash
/gsd-workflow
```

Then tell it what you want to do (execute plan, discuss, plan, verify).

### Q: What if I want to pause and review work?

**A:** Use session-management's `/execute-plan`:

```bash
# This pauses after each task
/execute-plan .planning/project-task-management-system/phase-01-foundation/01-01-PLAN.md

# Review the work, then continue:
/execute-plan .planning/project-task-management-system/phase-01-foundation/01-01-PLAN.md
```

### Q: Can I run Plans 02-05 in parallel with Plan 01?

**A:** No, Plan 01 must complete first (Plans 02-05 depend on it).

But you **can** run Plans 02-05 in parallel with each other once Plan 01 is done:

```bash
/Agents

# Say:
"Launch 4 agents in parallel:
- Backend Agent 1: Execute Plan 02
- Backend Agent 2: Execute Plan 03
- Frontend Agent: Execute Plan 04
- Protocol Agent: Execute Plan 05"
```

### Q: What about MCP servers and automation?

**A:** Available but optional:

- **n8n-mcp-tools-expert**: Workflow automation (use after foundation is complete)
- **ast-grep**: Code structure search (useful during implementation)
- **Evals**: Agent testing (use for validating specialists)

**Don't start with these.** Focus on getting the system working first.

---

## 🎯 Summary: Recommended Path

```
Day 1-7:   Plan 01 (Foundation)
             └─► Use GSD Workflow
             └─► Execute with fresh context per task
             └─► Verify integration tests
             └─► Generate coverage report

Day 8-12:   Plans 02-05 (Parallel Execution)
             └─► Use Agent Orchestration
             └─► Launch 4 specialist agents
             └─► Execute in parallel
             └─► Each agent uses fresh context per task

Day 13-15:   Integration & Testing
             └─► End-to-end workflows
             └─► Performance benchmarks
             └─► Security audit
             └─► Deployment preparation
```

**Total Time: 13-15 days with parallel execution**

---

**Ready to start? Just run:**
```bash
/gsd-workflow
```

And tell it what you want to execute!
