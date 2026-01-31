# PTMS Multi-LLM Experiments

**Parallel Implementation of Project Task Management System using Multiple LLM Providers**

This repository contains parallel implementations of the Project Task Management System by three different LLM providers, allowing comparison and hybrid solution assembly.

---

## Quick Links

- **[Repository](https://github.com/MartinMayday/ptms-multi-llm-experiments)** - GitHub repo
- **[MULTI-LLM-WORKFLOW.md](MULTI-LLM-WORKFLOW.md)** - Complete workflow documentation
- **[SUMMARY.md](SUMMARY.md)** - Project specification and overview
- **[INDEX.md](INDEX.md)** - Project navigation and plan index

---

## Purpose

Compare how different LLMs approach the same complex software engineering task:
- **Claude** (Anthropic) - Strong typing, architecture patterns
- **GPT** (OpenAI) - Creative solutions, documentation
- **Gemini** (Google) - Fast iteration, comprehensive testing

Then merge the best components into a hybrid solution on `main`.

---

## Architecture: Git Worktrees

Each LLM works in an **isolated working directory** (worktree) linked to its own branch:

```
                    GitHub Repository
         ptms-multi-llm-experiments
                    │
        ┌───────────┼───────────┐
        │           │           │
    ┌───┴───┐   ┌───┴───┐   ┌───┴───┐
    │ main  │   │variant│   │variant│
    │(specs │   │-claude│   │-gpt   │
    │/ref)  │   │       │   │       │
    └───┬───┘   └───┬───┘   └───┬───┘
        │           │           │
   Worktrees (Isolated Directories)
        │           │           │
┌───────┴────┬──────┴─────┬─────┴──────┐
│            │            │            │
│ptms-     │ptms-      │ptms-      │ptms-
│experiments│experiments│experiments│experiments
│(main)     │-claude    │-gpt       │-gemini
│            │            │            │
│ Planning   │ Claude     │ GPT        │ Gemini
│ Files      │ Code       │ Code       │ Code
│            │            │            │
└────────────┴────────────┴────────────┴──────┘
```

---

## Worktree Structure

| Directory | Branch | LLM Provider | Status | Purpose |
|-----------|--------|--------------|--------|---------|
| `ptms-experiments/` | `main` | Reference | ✅ Stable | Planning files, specs, integration |
| `ptms-experiments-claude/` | `variant-claude` | **Claude** | 🔄 Ready | Claude-optimized implementation |
| `ptms-experiments-gpt/` | `variant-gpt` | **GPT** | 🔄 Ready | GPT-optimized implementation |
| `ptms-experiments-gemini/` | `variant-gemini` | **Gemini** | 🔄 Ready | Gemini-optimized implementation |

**Base Location:** `/Users/nosrcadmin/aos/`

---

## Quick Start

### 1. View All Worktrees
```bash
cd /Users/nosrcadmin/aos/ptms-experiments
git worktree list
```

### 2. Switch to Variant
```bash
# Work in Claude variant
cd /Users/nosrcadmin/aos/ptms-experiments-claude

# Work in GPT variant
cd /Users/nosrcadmin/aos/ptms-experiments-gpt

# Work in Gemini variant
cd /Users/nosrcadmin/aos/ptms-experiments-gemini

# Back to main (reference/specs)
cd /Users/nosrcadmin/aos/ptms-experiments
```

### 3. Check Status Across All
```bash
for dir in ptms-experiments ptms-experiments-claude ptms-experiments-gpt ptms-experiments-gemini; do
  echo "=== $dir ==="
  cd /Users/nosrcadmin/aos/$dir && git status --short
done
```

---

## Workflow: From Development to Merge

### Phase 1: Parallel Development
Each LLM works independently in their assigned directory:

```
Claude Session
  └─→ cd /Users/nosrcadmin/aos/ptms-experiments-claude
  └─→ Implements Plan 01 Phase 01-01
  └─→ git commit -m "01-01: Implement FileSystem [Claude]"
  └─→ Continues to next phase...

GPT Session  
  └─→ cd /Users/nosrcadmin/aos/ptms-experiments-gpt
  └─→ Implements Plan 01 Phase 01-01
  └─→ git commit -m "01-01: Implement FileSystem [GPT]"
  └─→ Continues to next phase...

Gemini Session
  └─→ cd /Users/nosrcadmin/aos/ptms-experiments-gemini
  └─→ Implements Plan 01 Phase 01-01
  └─→ git commit -m "01-01: Implement FileSystem [Gemini]"
  └─→ Continues to next phase...
```

### Phase 2: Validation & Testing

Run quality gates on EACH variant:

```bash
# In each worktree directory
cd /Users/nosrcadmin/aos/ptms-experiments-<variant>

# Quality Gates
bun run typecheck    # TypeScript compilation
cd /Users/nosrcadmin/aos/ptms-experiments-<variant>
bun test             # Test suite
cd /Users/nosrcadmin/aos/ptms-experiments-<variant>
bun test --coverage  # Coverage report (> 90%)
cd /Users/nosrcadmin/aos/ptms-experiments-<variant>
bun run lint         # Linting
cd /Users/nosrcadmin/aos/ptms-experiments-<variant>
bun run benchmark    # Performance tests
```

### Phase 3: Comparison

Compare implementations:

```bash
# View commits unique to each variant
git log variant-claude ^variant-gpt   # Claude-only commits
git log variant-gpt ^variant-claude   # GPT-only commits

# Compare specific files
diff ptms-experiments-claude/src/core/FileSystem.ts \
    ptms-experiments-gpt/src/core/FileSystem.ts
```

### Phase 4: Merge Best Components

#### Option A: Pick Winner (Full Merge)
```bash
cd /Users/nosrcadmin/aos/ptms-experiments
git checkout main
git merge variant-claude --no-ff -m "Merge: Claude wins Plan 01"
git push origin main
```

#### Option B: Cherry-Pick Best Parts (Hybrid)
```bash
cd /Users/nosrcadmin/aos/ptms-experiments
git checkout -b integration/plan-01-hybrid

# Claude's FileSystem
git cherry-pick <claude-fs-commit>

# GPT's StateMachine
git cherry-pick <gpt-sm-commit>

# Gemini's tests
git cherry-pick <gemini-test-commit>

# Test combined solution, then merge to main
git checkout main
git merge integration/plan-01-hybrid
```

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

## Current Status

### Plan 01: Foundation & Architecture

| Phase | Name | Claude | GPT | Gemini | Winner |
|-------|------|--------|-----|--------|--------|
| 01-01 | System Architecture | 🔄 | 🔄 | 🔄 | - |
| 01-02 | File System Setup | ⏳ | ⏳ | ⏳ | - |
| 01-03 | Data Models | ⏳ | ⏳ | ⏳ | - |
| 01-04 | State Machine | ⏳ | ⏳ | ⏳ | - |
| 01-05 | Concurrency | ⏳ | ⏳ | ⏳ | - |
| 01-06 | File Watching | ⏳ | ⏳ | ⏳ | - |
| 01-07 | Integration Tests | ⏳ | ⏳ | ⏳ | - |

Legend: ✅ Complete | 🔄 In Progress | ⏳ Pending

---

## Git Commands Reference

### Worktree Management
```bash
# List all worktrees
git worktree list

# Add new worktree
git worktree add <path> <branch>

# Remove worktree
git worktree remove <path>

# Prune stale worktrees
git worktree prune
```

### Compare Variants
```bash
# Compare two branches
git diff variant-claude variant-gpt

# View unique commits
git log variant-claude ^variant-gpt

# Compare specific file
git diff variant-claude variant-gpt -- src/core/FileSystem.ts
```

### Merge Workflow
```bash
# Merge full variant
git checkout main
git merge variant-claude

# Cherry-pick specific commit
git cherry-pick <commit-hash>

# Create integration branch
git checkout -b integration/plan-01
```

---

## Documentation

### Essential Reading

1. **[MULTI-LLM-WORKFLOW.md](MULTI-LLM-WORKFLOW.md)** - Complete workflow guide
   - Worktree setup and management
   - Validation procedures
   - Merge strategies
   - Emergency procedures

2. **[SUMMARY.md](SUMMARY.md)** - Project specification
   - 5 plans with 32 phases
   - Technology stack
   - Success criteria
   - Implementation order

3. **[INDEX.md](INDEX.md)** - Project overview
   - Quick start guide
   - Plan directory structure
   - Phase specifications

4. **[ORCHESTRATOR-AGENT-PROMPT.md](ORCHESTRATOR-AGENT-PROMPT.md)** - Orchestrator system prompt
   - Multi-agent orchestration
   - Task spawning
   - Quality gates

### Phase Specifications

- **phase-01-foundation/** - 7 detailed phase specs (COMPLETE)
- **phase-02-workflow-engine/** - Workflow engine overview
- **phase-03-backend-api/** - Backend API specification
- **phase-04-web-interface/** - Web interface specification
- **phase-05-agent-interface/** - Agent protocol specification

---

## LLM-Specific Guidelines

### Claude Variant (ptms-experiments-claude)

**Strengths:** Type safety, architecture patterns, error handling

**Focus:**
- Strict TypeScript typing (no `any`)
- Result<T,E> error types
- Atomic operations
- Clean architecture

**Entry:**
```bash
cd /Users/nosrcadmin/aos/ptms-experiments-claude
start-orchestrate-execution .planning/KICKOFF-PTMS.yaml
```

### GPT Variant (ptms-experiments-gpt)

**Strengths:** Creative solutions, documentation, innovation

**Focus:**
- Novel implementation patterns
- Performance optimizations
- Edge case handling
- Comprehensive docs

**Entry:**
```bash
cd /Users/nosrcadmin/aos/ptms-experiments-gpt
# Use GPT-4 with full system prompt
```

### Gemini Variant (ptms-experiments-gemini)

**Strengths:** Fast iteration, testing, large context

**Focus:**
- High test coverage
- Documentation completeness
- Integration scenarios
- Benchmark performance

**Entry:**
```bash
cd /Users/nosrcadmin/aos/ptms-experiments-gemini
# Use Gemini Pro/Ultra
```

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

### Common Issues

**Worktree not showing:**
```bash
git worktree prune  # Clean up stale entries
git worktree list   # Verify
```

**Merge conflicts:**
```bash
git merge --abort   # Start over
# Or resolve manually and commit
```

**Uncommitted work lost:**
```bash
git reflog          # Find lost commits
git cherry-pick <hash>  # Recover
```

### Full Documentation

- **Setup Issues:** MULTI-LLM-WORKFLOW.md Section 6 (Emergency Procedures)
- **Git Commands:** MULTI-LLM-WORKFLOW.md Section 4 (Commands Reference)
- **Validation:** MULTI-LLM-WORKFLOW.md Section 3 (Testing Workflow)
- **Merge Strategy:** MULTI-LLM-WORKFLOW.md Section 3 (Phase 5: Merge)

---

## Repository Info

- **URL:** https://github.com/MartinMayday/ptms-multi-llm-experiments
- **Created:** 2026-01-31
- **Status:** Ready for parallel development
- **Branches:** main, variant-claude, variant-gpt, variant-gemini
- **Worktrees:** 4 isolated working directories

---

## Next Steps

1. ✅ **Repository Created** - GitHub repo with worktrees
2. ✅ **Documentation Complete** - Workflow guide created
3. 🔄 **Start LLM Sessions:**
   - Open Claude → `cd ptms-experiments-claude`
   - Open GPT → `cd ptms-experiments-gpt`
   - Open Gemini → `cd ptms-experiments-gemini`
4. 🔄 **Execute Plan 01 Phase 01** in all variants
5. 🔄 **Validate** each variant independently
6. 🔄 **Compare** and merge best components

**Ready to orchestrate parallel LLM development! 🚀**
