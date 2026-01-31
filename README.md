# PTMS Multi-LLM Experiments

**Parallel Implementation of Project Task Management System using Multiple LLM Providers**

This repository contains parallel implementations of the Project Task Management System by four different LLM providers, allowing comparison and hybrid solution assembly.

---

## Quick Links

- **[Repository](https://github.com/MartinMayday/ptms-multi-llm-experiments)** - GitHub repo
- **[SPECIALIST-ORCHESTRATOR-CONFIG.md](SPECIALIST-ORCHESTRATOR-CONFIG.md)** - Specialist-agent orchestration config
- **[MULTI-LLM-WORKFLOW.md](MULTI-LLM-WORKFLOW.md)** - Complete workflow documentation
- **[SUMMARY.md](SUMMARY.md)** - Project specification and overview
- **[INDEX.md](INDEX.md)** - Project navigation and plan index

---

## Purpose

Compare how different LLMs approach the same complex software engineering task:
- **Claude** (Anthropic) - Strong typing, architecture patterns
- **GLM** (Zhipu AI) - Chinese optimization, efficient generation
- **Kimi** (Moonshot AI) - Long context, deep analysis
- **Gemini** (Google) - Fast iteration, comprehensive testing

Then merge the best components into a hybrid solution on `main`.

---

## Architecture: Git Worktrees with Specialist Agents

Each LLM works in an **isolated working directory** (worktree) with its own orchestrator that spawns **specialist agents**:

```
                    GitHub Repository
         ptms-multi-llm-experiments
                    │
        ┌───────────┼───────────┐
        │           │           │
    ┌───┴───┐   ┌───┴───┐   ┌───┴───┐
    │ main  │   │variant│   │variant│
    │(specs │   │-claude│   │-glm   │
    │/ref)  │   │       │   │       │
    └───┬───┘   └───┬───┘   └───┬───┘
        │           │           │
   Worktrees (Isolated Directories)
        │           │           │
┌───────┴────┬──────┴─────┬─────┴──────┐
│            │            │            │
│ptms-     │ptms-      │ptms-      │ptms-
│experiments│experiments│experiments│experiments
│(main)     │-claude    │-glm       │-gemini
│            │            │            │
│ Planning   │ Claude     │ GLM        │ Gemini
│ Files      │ Code       │ Code       │ Code
│            │            │            │
└────────────┴────────────┴────────────┴──────┘

WITHIN EACH WORKTREE:
┌─────────────────────────────────────────┐
│ Orchestrator Agent                      │
│  ├─ Backend-Types Specialist            │
│  ├─ Backend-Core Specialist             │
│  ├─ Backend-Engine Specialist           │
│  ├─ Backend-API Specialist              │
│  ├─ Frontend-UI Specialist              │
│  ├─ QA-Engineer Specialist              │
│  └─ Up to 10 concurrent specialists     │
└─────────────────────────────────────────┘
```

**Key Point:** 1 orchestrator per worktree → spawns specialist agents (NOT generic agents doing entire plans)

See [SPECIALIST-ORCHESTRATOR-CONFIG.md](SPECIALIST-ORCHESTRATOR-CONFIG.md) for detailed configuration.

---

## Worktree Structure

| Directory | Branch | LLM Provider | Status | Purpose |
|-----------|--------|--------------|--------|---------|
| `ptms-experiments/` | `main` | Reference | ✅ Stable | Planning files, specs, integration |
| `ptms-experiments-claude/` | `variant-claude` | **Claude** (Anthropic) | 🔄 Ready | Claude orchestrator + specialists |
| `ptms-experiments-glm/` | `variant-glm` | **GLM** (Zhipu AI) | 🔄 Ready | GLM orchestrator + specialists |
| `ptms-experiments-kimi/` | `variant-kimi` | **Kimi** (Moonshot AI) | 🔄 Ready | Kimi orchestrator + specialists |
| `ptms-experiments-gemini/` | `variant-gemini` | **Gemini** (Google) | 🔄 Ready | Gemini orchestrator + specialists |

**Base Location:** `/Users/nosrcadmin/aos/`

---

## Quick Start

### 1. View All Worktrees
```bash
cd /Users/nosrcadmin/aos/ptms-experiments
git worktree list
```

### 2. Start Orchestrator in Worktree
```bash
# Claude worktree - spawns specialist agents
cd /Users/nosrcadmin/aos/ptms-experiments-claude

# GLM worktree - spawns specialist agents
cd /Users/nosrcadmin/aos/ptms-experiments-glm

# Kimi worktree - spawns specialist agents  
cd /Users/nosrcadmin/aos/ptms-experiments-kimi

# Gemini worktree - spawns specialist agents
cd /Users/nosrcadmin/aos/ptms-experiments-gemini
```

Each orchestrator will:
1. Parse plan specifications
2. Break into atomic tasks
3. Spawn specialist agents (Backend-Types, Backend-Core, etc.)
4. Coordinate up to 10 agents concurrently
5. Deliver results

---

## Architecture: Specialist-Agent Multi-Orchestration

**CORRECT APPROACH:**
```
Worktree: ptms-experiments-kimi/
├─ Orchestrator (Kimi)
│  ├─ Backend-Types Specialist → Type definitions, Zod schemas
│  ├─ Backend-Core Specialist → File system, atomic operations
│  ├─ Backend-Engine Specialist → State machines, workflows
│  ├─ Backend-API Specialist → Hono, REST, WebSocket
│  ├─ Frontend-UI Specialist → Vue 3, Tailwind, components
│  ├─ QA-Engineer Specialist → Tests, benchmarks
│  └─ Up to 10 concurrent specialists
```

**NOT:** 1 orchestrator → 3 generic agents → each does entire plan

See [SPECIALIST-ORCHESTRATOR-CONFIG.md](SPECIALIST-ORCHESTRATOR-CONFIG.md) for:
- Specialist agent registry (14 types)
- Task-to-specialist mapping
- Orchestrator prompt template
- Quality gates

---

## Project Specification

### What You're Building

An **Agentic Workflow Management System** with:

- ✅ **File system as source of truth** (git-like versioning)
- ✅ **Atomic task operations** with defined inputs/outputs
- ✅ **State machine** with validated transitions
- ✅ **Kanban UI** for human oversight
- ✅ **Agent protocol** for programmatic access
- ✅ **5 plans, 32 phases, 21-31 days estimated effort**

### Plans Overview

| Plan | Name | Phases | Status |
|------|------|--------|--------|
| 01 | Foundation & Architecture | 7 | 🔵 Ready |
| 02 | Workflow Engine | 7 | ⏳ Pending |
| 03 | Backend API | 6 | ⏳ Pending |
| 04 | Web Interface | 6 | ⏳ Pending |
| 05 | Agent Interface | 6 | ⏳ Pending |

Legend: ✅ Complete | 🔵 Ready | 🔄 In Progress | ⏳ Pending | ❌ Failed

### Technology Stack

**Backend:**
- Runtime: Bun (3x faster than Node.js)
- Language: TypeScript 5.x
- Framework: Hono (lightweight, fast)
- File Watching: chokidar (cross-platform)
- Validation: Zod
- Locking: POSIX flock()

**Frontend:**
- Framework: Vue 3 (Composition API)
- Build: Vite
- Styling: Tailwind CSS + shadcn/ui
- State: Pinia
- Real-time: Native WebSocket

**Testing:**
- Framework: bun:test (built-in)
- Coverage Target: > 90%
- Benchmarks: Performance metrics

---

## Workflow: LLM Variant Development

### Phase 1: Parallel Development
Each worktree runs its own orchestrator spawning specialist agents:

```
Claude Worktree
  └─ Orchestrator spawns:
     ├─ Backend-Types agent → Type definitions
     ├─ Backend-Core agent → FileSystem.ts
     ├─ Backend-Engine agent → StateMachine.ts
     └─ QA-Engineer agent → Tests

GLM Worktree
  └─ Orchestrator spawns:
     ├─ Backend-Types agent → Type definitions
     ├─ Backend-Core agent → FileSystem.ts
     ├─ Backend-Engine agent → StateMachine.ts
     └─ QA-Engineer agent → Tests

Kimi Worktree
  └─ Orchestrator spawns:
     ├─ Backend-Types agent → Type definitions
     ├─ Backend-Core agent → FileSystem.ts
     ├─ Backend-Engine agent → StateMachine.ts
     └─ QA-Engineer agent → Tests

Gemini Worktree
  └─ Orchestrator spawns:
     ├─ Backend-Types agent → Type definitions
     ├─ Backend-Core agent → FileSystem.ts
     ├─ Backend-Engine agent → StateMachine.ts
     └─ QA-Engineer agent → Tests
```

### Phase 2: Validation & Testing

Each variant independently runs quality gates:
```bash
cd /Users/nosrcadmin/aos/ptms-experiments-<variant>
bun run typecheck    # TypeScript compilation
bun test             # Test suite
bun test --coverage  # Coverage report (> 90%)
bun run lint         # Linting
bun run benchmark    # Performance tests
```

### Phase 3: Comparison & Selection

Compare specialist implementations across variants:
```bash
# Compare Backend-Core implementations
diff ptms-experiments-claude/src/core/FileSystem.ts \
    ptms-experiments-kimi/src/core/FileSystem.ts

# Compare test coverage
# Claude: 94%, Kimi: 96%, GLM: 92%, Gemini: 95%
```

### Phase 4: Merge Best Components

Cherry-pick winning specialist implementations:
```bash
cd /Users/nosrcadmin/aos/ptms-experiments

# Kimi's Backend-Core won
git cherry-pick <kimi-backend-core-commit>

# Claude's Backend-Types won
git cherry-pick <claude-backend-types-commit>

# Gemini's QA suite won
git cherry-pick <gemini-qa-commit>
```

---

## Documentation

### Essential Reading

1. **[SPECIALIST-ORCHESTRATOR-CONFIG.md](SPECIALIST-ORCHESTRATOR-CONFIG.md)** - Complete orchestrator configuration
   - Specialist agent registry (14 types)
   - Task-to-specialist mapping
   - Orchestrator prompt template
   - Quality gates and constraints

2. **[MULTI-LLM-WORKFLOW.md](MULTI-LLM-WORKFLOW.md)** - Workflow guide
   - Worktree management
   - Validation procedures
   - Merge strategies
   - Emergency procedures

3. **[SUMMARY.md](SUMMARY.md)** - Project specification
   - 5 plans with 32 phases
   - Technology stack
   - Success criteria

4. **[INDEX.md](INDEX.md)** - Project navigation
   - Quick start
   - Plan directory structure

### Phase Specifications

- **phase-01-foundation/** - 7 detailed phase specs (COMPLETE)
- **phase-02-workflow-engine/** - Workflow engine overview
- **phase-03-backend-api/** - Backend API specification
- **phase-04-web-interface/** - Web interface specification
- **phase-05-agent-interface/** - Agent protocol specification

---

## Success Criteria

The system is successful when:

- ✅ **File System**: All state in files, atomic writes never corrupt data
- ✅ **State Machine**: Valid transitions only, side effects trigger correctly
- ✅ **Concurrency**: No race conditions, locks resolve within 5s
- ✅ **Observability**: Kanban UI for human oversight
- ✅ **Agent-Native**: Agents can claim/execute/report programmatically
- ✅ **Resilience**: Graceful failure handling with retries
- ✅ **Performance**: File ops < 10ms, transitions < 50ms, API < 200ms
- ✅ **Testing**: > 90% coverage, all tests pass
- ✅ **Documentation**: Complete API docs and agent protocol

---

## Support & Troubleshooting

### Worktree Issues
```bash
# List all worktrees
git worktree list

# Clean up stale worktrees
git worktree prune

# Check worktree status
for dir in ptms-experiments ptms-experiments-claude ptms-experiments-glm ptms-experiments-kimi ptms-experiments-gemini; do
  echo "=== $dir ==="
  cd /Users/nosrcadmin/aos/$dir && git status --short
done
```

### Full Documentation

- **Setup Issues:** MULTI-LLM-WORKFLOW.md Section 6 (Emergency Procedures)
- **Orchestrator Config:** SPECIALIST-ORCHESTRATOR-CONFIG.md
- **Validation:** MULTI-LLM-WORKFLOW.md Section 3 (Testing Workflow)
- **Merge Strategy:** MULTI-LLM-WORKFLOW.md Section 3 (Phase 5: Merge)

---

## Repository Info

- **URL:** https://github.com/MartinMayday/ptms-multi-llm-experiments
- **Created:** 2026-01-31
- **Status:** Ready for parallel specialist-agent development
- **Worktrees:** 5 (main + 4 LLM variants)
- **Branches:** main, variant-claude, variant-glm, variant-kimi, variant-gemini

---

## Next Steps

1. ✅ **Repository Created** - GitHub repo with 5 worktrees
2. ✅ **Specialist Config** - Orchestrator spawns domain specialists
3. 🔄 **Start Orchestrators:**
   - Claude → `cd ptms-experiments-claude`
   - GLM → `cd ptms-experiments-glm`
   - Kimi → `cd ptms-experiments-kimi`
   - Gemini → `cd ptms-experiments-gemini`
4. 🔄 **Execute Plan 01** - All orchestrators spawn specialists for Phase 01
5. 🔄 **Validate** - Each variant independently tested
6. 🔄 **Compare & Merge** - Best specialist implementations cherry-picked

**Ready for parallel specialist-agent orchestration! 🚀**

---

**Key Principle:** Each worktree has 1 orchestrator that spawns specialist agents (Backend-Types, Backend-Core, etc.), NOT generic agents doing entire plans. This maximizes quality through domain expertise.
