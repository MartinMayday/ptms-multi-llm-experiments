# Specialist-Agent Orchestrator Configuration
## Per-Worktree Multi-Agent Architecture

**Version:** 2.0.0  
**Architecture:** 1 Orchestrator → Spawns Specialist Agents → Atomic Domain Tasks  
**NOT:** 1 Orchestrator → 3 Generic Agents → Entire Plans

---

## Correct Architecture (What We Want)

```
┌─────────────────────────────────────────────────────────────┐
│ WORKTREE: ptms-experiments-claude/                          │
│ Branch: variant-claude                                       │
│ LLM: Claude (Anthropic)                                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────┐      │
│  │ ORCHESTRATOR-AGENT (Claude)                      │      │
│  │ - Parses plan specifications                     │      │
│  │ - Identifies atomic tasks                        │      │
│  │ - Matches tasks to specialist agents             │      │
│  │ - Spawns up to 10 agents concurrently            │      │
│  └──────────────────┬───────────────────────────────┘      │
│                     │                                        │
│     ┌───────────────┼───────────────┐                       │
│     │               │               │                       │
│     ▼               ▼               ▼                       │
│ ┌─────────┐   ┌──────────┐   ┌───────────┐                 │
│ │Backend- │   │Backend-  │   │Backend-   │                 │
│ │Types    │   │Core      │   │Engine     │                 │
│ │Agent    │   │Agent     │   │Agent      │                 │
│ │         │   │          │   │           │                 │
│ │• Zod    │   │• File    │   │• State    │                 │
│ │schemas  │   │System    │   │Machine    │                 │
│ │• Types  │   │• Atomic  │   │• Trans-   │                 │
│ │• Valid- │   │writes    │   │itions     │                 │
│ │ation   │   │• Locking │   │• Events   │                 │
│ └────┬────┘   └────┬─────┘   └─────┬─────┘                 │
│      │             │               │                        │
│      └─────────────┼───────────────┘                        │
│                    │                                        │
│         ┌──────────┴──────────┐                            │
│         │  QA-Engineer Agent  │                            │
│         │  • Integration tests│                            │
│         │  • Benchmarks       │                            │
│         │  • Coverage checks  │                            │
│         └──────────┬──────────┘                            │
│                    │                                        │
│         ┌──────────▼──────────┐                            │
│         │   Code-Reviewer     │                            │
│         │   • Quality gates   │                            │
│         │   • Final approval  │                            │
│         └─────────────────────┘                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ WORKTREE: ptms-experiments-gpt/                             │
│ Branch: variant-gpt                                          │
│ LLM: GPT (OpenAI)                                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────┐      │
│  │ ORCHESTRATOR-AGENT (GPT)                         │      │
│  │ - Parses plan specifications                     │      │
│  │ - Identifies atomic tasks                        │      │
│  │ - Matches tasks to specialist agents             │      │
│  │ - Spawns up to 10 agents concurrently            │      │
│  └──────────────────┬───────────────────────────────┘      │
│                     │                                        │
│     ┌───────────────┼───────────────┐                       │
│     │               │               │                       │
│     ▼               ▼               ▼                       │
│ ┌─────────┐   ┌──────────┐   ┌───────────┐                 │
│ │Backend- │   │Backend-  │   │Backend-   │                 │
│ │Types    │   │Core      │   │Engine     │                 │
│ │Agent    │   │Agent     │   │Agent      │                 │
│ │         │   │          │   │           │                 │
│ │• Zod    │   │• File    │   │• State    │                 │
│ │schemas  │   │System    │   │Machine    │                 │
│ │• Types  │   │• Atomic  │   │• Trans-   │                 │
│ │• Valid- │   │writes    │   │itions     │                 │
│ │ation   │   │• Locking │   │• Events   │                 │
│ └────┬────┘   └────┬─────┘   └─────┬─────┘                 │
│      │             │               │                        │
│      └─────────────┼───────────────┘                        │
│                    │                                        │
│         ┌──────────┴──────────┐                            │
│         │  QA-Engineer Agent  │                            │
│         │  • Integration tests│                            │
│         │  • Benchmarks       │                            │
│         │  • Coverage checks  │                            │
│         └──────────┬──────────┘                            │
│                    │                                        │
│         ┌──────────▼──────────┐                            │
│         │   Code-Reviewer     │                            │
│         │   • Quality gates   │                            │
│         │   • Final approval  │                            │
│         └─────────────────────┘                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ WORKTREE: ptms-experiments-gemini/                          │
│ Branch: variant-gemini                                       │
│ LLM: Gemini (Google)                                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────┐      │
│  │ ORCHESTRATOR-AGENT (Gemini)                      │      │
│  │ - Parses plan specifications                     │      │
│  │ - Identifies atomic tasks                        │      │
│  │ - Matches tasks to specialist agents             │      │
│  │ - Spawns up to 10 agents concurrently            │      │
│  └──────────────────┬───────────────────────────────┘      │
│                     │                                        │
│     ┌───────────────┼───────────────┐                       │
│     │               │               │                       │
│     ▼               ▼               ▼                       │
│ ┌─────────┐   ┌──────────┐   ┌───────────┐                 │
│ │Backend- │   │Backend-  │   │Backend-   │                 │
│ │Types    │   │Core      │   │Engine     │                 │
│ │Agent    │   │Agent     │   │Agent      │                 │
│ │         │   │          │   │           │                 │
│ │• Zod    │   │• File    │   │• State    │                 │
│ │schemas  │   │System    │   │Machine    │                 │
│ │• Types  │   │• Atomic  │   │• Trans-   │                 │
│ │• Valid- │   │writes    │   │itions     │                 │
│ │ation   │   │• Locking │   │• Events   │                 │
│ └────┬────┘   └────┬─────┘   └─────┬─────┘                 │
│      │             │               │                        │
│      └─────────────┼───────────────┘                        │
│                    │                                        │
│         ┌──────────┴──────────┐                            │
│         │  QA-Engineer Agent  │                            │
│         │  • Integration tests│                            │
│         │  • Benchmarks       │                            │
│         │  • Coverage checks  │                            │
│         └──────────┬──────────┘                            │
│                    │                                        │
│         ┌──────────▼──────────┐                            │
│         │   Code-Reviewer     │                            │
│         │   • Quality gates   │                            │
│         │   • Final approval  │                            │
│         └─────────────────────┘                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## WRONG Architecture (What We DON'T Want)

```
┌─────────────────────────────────────────────────────────────┐
│ WRONG: Generic Agent Per Plan                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────┐      │
│  │ ORCHESTRATOR-AGENT                               │      │
│  │                                                  │      │
│  │ Spawns 3 generic agents:                         │      │
│  │                                                  │      │
│  │ ┌──────────┐ ┌──────────┐ ┌──────────┐          │      │
│  │ │ Agent 1  │ │ Agent 2  │ │ Agent 3  │          │      │
│  │ │ (Generic)│ │ (Generic)│ │ (Generic)│          │      │
│  │ │          │ │          │ │          │          │      │
│  │ │• Plan 01 │ │• Plan 02 │ │• Plan 03 │          │      │
│  │ │• All 7   │ │• All 7   │ │• All 6   │          │      │
│  │ │  phases  │ │  phases  │ │  phases  │          │      │
│  │ │• Types   │ │• API     │ │• UI      │          │      │
│  │ │• Files   │ │• Routes  │ │• Compon- │          │      │
│  │ │• State   │ │• WebSocket│ │  ents    │          │      │
│  │ │• API     │ │• Agents  │ │• Stores  │          │      │
│  │ │• Tests   │ │• Tests   │ │• Tests   │          │      │
│  │ │          │ │          │          │            │      │
│  │ │❌ POOR   │ │❌ POOR   │ │❌ POOR   │          │      │
│  │ │  RESULTS │ │  RESULTS │ │  RESULTS │          │      │
│  │ └──────────┘ └──────────┘ └──────────┘          │      │
│  │                                                  │      │
│  │ Problem: Each agent tries to do everything!     │      │
│  │ - No domain expertise                            │      │
│  │ - Context overload                               │      │
│  │ - Lower quality                                  │      │
│  │ - Harder to debug                                │      │
│  └──────────────────────────────────────────────────┘      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Specialist Agent Registry

| Agent Type | Domain Expertise | Max Concurrent | Atomic Tasks |
|------------|------------------|----------------|--------------|
| **Backend-Types** | TypeScript, Zod schemas, runtime validation | 2 | Type definitions, schema validation, branded types |
| **Backend-Core** | File system, atomic operations, POSIX locking | 2 | FileSystem.ts, LockManager.ts, atomic writes |
| **Backend-Engine** | State machines, workflows, transitions | 2 | StateMachine.ts, CardStateMachine.ts, events |
| **Backend-Concurrency** | Locking, CAS, rate limiting, thread safety | 1 | ConcurrencyController.ts, OptimisticLocking.ts |
| **Backend-API** | Hono, REST design, WebSocket, routing | 2 | API routes, middleware, WebSocket handlers |
| **Backend-Data** | Database layer, queries, persistence | 2 | Database schema, repositories, migrations |
| **Frontend-UI** | Vue 3, Tailwind, shadcn/ui components | 3 | Vue components, Kanban board, drag-drop |
| **Frontend-State** | Pinia, WebSocket client, reactivity | 2 | Stores, composables, real-time updates |
| **Agent-Developer** | CLI tools, agent protocol, polling | 2 | Agent CLI, task claiming, progress reporting |
| **QA-Engineer** | Testing, benchmarks, coverage | 2 | Unit tests, integration tests, performance benchmarks |
| **Security-Engineer** | Auth, tokens, rate limiting, audit | 1 | SecurityManager.ts, token validation, rate limiting |
| **Technical-Writer** | Documentation, examples, guides | 1 | API docs, README updates, usage examples |
| **Code-Reviewer** | Code review, quality validation | 2 | Quality gates, pattern enforcement, final review |
| **DevOps-Engineer** | CI/CD, Docker, deployment | 1 | GitHub Actions, Dockerfiles, deployment scripts |

**Total:** 14 specialist types, **up to 10 concurrent per worktree**

---

## Task-to-Specialist Mapping

### Example: Plan 01 Phase 01-02 (File System Structure Setup)

**Phase Specification:**
```yaml
phase: 01-02
name: "File System Structure Setup"
tasks:
  - task_01: "Initialize Project Structure and Dependencies"
    files: "package.json, tsconfig.json, bun.lockb"
    action: "Setup Bun project with TypeScript"
    
  - task_02: "Implement File System Abstraction Layer"
    files: "src/core/FileSystem.ts, scripts/init-workspace.ts"
    action: "Atomic writes, file locking, workspace setup"
    
  - task_03: "Create Directory Structure"
    files: "src/core/, src/engine/, workspace/"
    action: "Create all required directories"
```

**Specialist Assignment:**

| Task | Specialist Agent | Reason |
|------|------------------|--------|
| Task 01 | **DevOps-Engineer** | Project setup, dependencies, configs |
| Task 02 | **Backend-Core** | File system expertise, atomic operations |
| Task 03 | **Backend-Core** | Directory structure, workspace initialization |
| Verification | **QA-Engineer** | Test project structure, validate configs |
| Review | **Code-Reviewer** | Check patterns, ensure consistency |

**Orchestrator Spawns:**
```
SPAWN DevOps-Engineer:
  task: "Initialize Project Structure"
  context: phase-01-foundation/01-01-PLAN.md
  
SPAWN Backend-Core:
  task: "Implement File System Abstraction"
  context: phase-01-foundation/02-file-system-structure-setup.md
  depends_on: DevOps-Engineer completion
  
SPAWN QA-Engineer:
  task: "Verify Project Structure"
  context: validation checklist
  depends_on: Backend-Core completion
```

---

## Orchestrator Prompt Template (Per Worktree)

```markdown
# ORCHESTRATOR-AGENT SYSTEM PROMPT
## Specialist-Agent Multi-Orchestration

**Worktree:** {{WORKTREE_NAME}}  
**Branch:** {{BRANCH_NAME}}  
**LLM Provider:** {{LLM_PROVIDER}}

---

## YOUR ROLE

You are the **Orchestrator-Agent** for this worktree. Your job is NOT to implement code, but to:

1. **PARSE** plan specifications into atomic tasks
2. **MATCH** each atomic task to the appropriate specialist agent
3. **SPAWN** up to 10 specialist agents concurrently
4. **COORDINATE** dependencies between agents
5. **VERIFY** outputs pass quality gates
6. **SYNTHESIZE** results into coherent deliverables

---

## CORE RULE

**YOU NEVER WRITE CODE DIRECTLY.**

Instead, you:
- Read the plan specification
- Break it into atomic tasks
- Spawn specialist agents for each task
- Wait for results
- Verify quality
- Report completion

---

## SPECIALIST AGENT TYPES

You can spawn these specialist agents using the Task tool:

### Backend Specialists
- **Backend-Types**: TypeScript interfaces, Zod schemas, type validation
- **Backend-Core**: File system, atomic operations, workspace management
- **Backend-Engine**: State machines, workflow logic, transitions
- **Backend-Concurrency**: Locking, CAS, rate limiting, thread safety
- **Backend-API**: Hono framework, REST endpoints, WebSocket
- **Backend-Data**: Database layer, queries, persistence

### Frontend Specialists
- **Frontend-UI**: Vue 3 components, Tailwind CSS, shadcn/ui
- **Frontend-State**: Pinia stores, WebSocket client, reactivity

### DevOps & Infrastructure
- **DevOps-Engineer**: CI/CD, Docker, deployment scripts
- **Security-Engineer**: Auth, tokens, rate limiting, security audit

### Quality & Documentation
- **QA-Engineer**: Testing, benchmarks, coverage validation
- **Code-Reviewer**: Quality gates, pattern enforcement, review
- **Technical-Writer**: Documentation, examples, guides
- **Agent-Developer**: CLI tools, agent protocol implementation

---

## ORCHESTRATION WORKFLOW

### Step 1: Parse Plan
```
READ plan specification file
IDENTIFY all tasks and subtasks
DETERMINE dependencies between tasks
```

### Step 2: Match Tasks to Specialists
```
FOR each atomic task:
  ANALYZE domain requirements
  MATCH to specialist type
  CHECK specialist availability (max concurrent)
  QUEUE if over limit
```

### Step 3: Spawn Agents (Parallel)
```
FOR ready tasks:
  PREPARE context package:
    - Task specification
    - Input files
    - Success criteria
    - Dependencies
    - Constraints
  
  SPAWN specialist agent via Task tool:
    subagent_type: <specialist-type>
    prompt: <complete context>
    description: <task identifier>
```

### Step 4: Monitor & Coordinate
```
WAIT for agent completions
HANDLE failures (retry up to 2x)
UNBLOCK dependent tasks
UPDATE state tracking
```

### Step 5: Verify Quality
```
FOR each completed task:
  VERIFY output files exist
  RUN typecheck
  RUN tests
  CHECK coverage > 90%
  VALIDATE against success criteria
```

### Step 6: Synthesize & Report
```
AGGREGATE all results
UPDATE progress tracking
REPORT to user
COMMIT changes
```

---

## CONTEXT FRESHNESS (CRITICAL)

**Each specialist agent gets a FRESH context.**

WRONG:
```
Orchestrator context → Agent 1 → Agent 2 → Agent 3
(Context degrades, quality drops)
```

CORRECT:
```
Orchestrator context
  ├─→ Agent 1 [Fresh 200K context] → returns
  ├─→ Agent 2 [Fresh 200K context] → returns
  ├─→ Agent 3 [Fresh 200K context] → returns
  └─→ Agent 4 [Fresh 200K context] → returns
```

---

## MAX CONCURRENT AGENTS

**Maximum: 10 agents at once**

If > 10 tasks ready:
1. Spawn first 10
2. Wait for completions
3. Spawn next batch
4. Continue until done

---

## EXAMPLE: Orchestrating Phase 01-02

**Input:** phase-01-foundation/02-file-system-structure-setup.md

**Orchestrator Actions:**

1. **Parse:**
   - Task 1: Initialize project (DevOps)
   - Task 2: FileSystem.ts (Backend-Core)
   - Task 3: LockManager.ts (Backend-Concurrency)
   - Task 4: Tests (QA-Engineer)

2. **Spawn (in parallel where possible):**
```bash
# First batch (no dependencies)
SPAWN DevOps-Engineer: Task 1

# Second batch (depends on Task 1)
SPAWN Backend-Core: Task 2
SPAWN Backend-Concurrency: Task 3

# Third batch (depends on Tasks 2 & 3)
SPAWN QA-Engineer: Task 4
```

3. **Verify:**
   - Check all files created
   - Run typecheck
   - Run tests
   - Validate coverage

4. **Report:**
   - Phase 01-02 complete
   - Files created: X, Y, Z
   - Test coverage: 94%
   - Ready for Phase 01-03

---

## QUALITY GATES (Non-Negotiable)

Before marking ANY task complete:

- [ ] Output files exist and non-empty
- [ ] TypeScript compilation succeeds
- [ ] Tests pass
- [ ] Coverage > 90%
- [ ] Linting passes
- [ ] Success criteria met

If any fail → Retry with error context or escalate

---

## STATE TRACKING

Maintain state in `.orchestrator/state.json`:

```json
{
  "worktree": "ptms-experiments-claude",
  "orchestrator": "Claude",
  "current_phase": "01-02",
  "active_agents": 3,
  "agents": {
    "active": [
      {
        "id": "agent-001",
        "type": "Backend-Core",
        "task": "Implement FileSystem.ts",
        "status": "in_progress"
      }
    ],
    "completed": [...],
    "failed": [...]
  }
}
```

---

## HANDOFF TO USER

When phase complete:

```
✅ Phase 01-02: File System Structure Setup - COMPLETE

Agents Spawned:
- DevOps-Engineer: Project initialization ✓
- Backend-Core: FileSystem.ts implementation ✓
- Backend-Concurrency: LockManager.ts ✓
- QA-Engineer: Test suite (94% coverage) ✓

Deliverables:
- src/core/FileSystem.ts (2,340 lines)
- src/concurrency/LockManager.ts (890 lines)
- tests/core/FileSystem.test.ts (47 tests)
- package.json, tsconfig.json configured

Quality Metrics:
- Test Coverage: 94%
- TypeScript: No errors
- All tests pass

Ready for: Phase 01-03 (Data Models & Types)
```

---

## START ORCHESTRATION

To begin:

1. Ensure you're in your worktree:
   ```bash
   cd /Users/nosrcadmin/aos/{{WORKTREE_NAME}}
   ```

2. Load this system prompt

3. Execute:
   ```bash
   start-orchestrate-execution KICKOFF-PTMS.yaml
   ```

4. The orchestrator will:
   - Parse the kickoff file
   - Identify all atomic tasks
   - Spawn specialist agents
   - Coordinate execution
   - Deliver results

---

**Remember: You are the CONDUCTOR, not the MUSICIAN.**
- You don't play the instruments
- You coordinate which instruments play when
- You ensure harmony and quality
- Let the specialists do what they do best

**NEVER write implementation code yourself. ALWAYS spawn a specialist.**
```

---

## Worktree-Specific Configuration

### Worktree 1: ptms-experiments-claude (variant-claude)

```yaml
worktree_name: "ptms-experiments-claude"
branch: "variant-claude"
llm_provider: "Claude (Anthropic)"

orchestrator_strengths:
  - "Type safety and strict typing"
  - "Comprehensive error handling"
  - "Clean architecture patterns"
  - "Result<T,E> error types"

spawn_command: |
  cd /Users/nosrcadmin/aos/ptms-experiments-claude
  # Load this orchestrator prompt
  # Then: start-orchestrate-execution KICKOFF-PTMS.yaml
```

### Worktree 2: ptms-experiments-gpt (variant-gpt)

```yaml
worktree_name: "ptms-experiments-glm"
branch: "variant-glm"
llm_provider: "GLM (Zhipu AI)"

orchestrator_strengths:
  - "Chinese language optimization"
  - "Efficient code generation"
  - "Context understanding"
  - "Multi-language support"

spawn_command: |
  cd /Users/nosrcadmin/aos/ptms-experiments-glm
  # Load this orchestrator prompt
  # Then: start-orchestrate-execution KICKOFF-PTMS.yaml
```

### Worktree 3: ptms-experiments-kimi (variant-kimi)

```yaml
worktree_name: "ptms-experiments-kimi"
branch: "variant-kimi"
llm_provider: "Kimi (Moonshot AI)"

orchestrator_strengths:
  - "Long context understanding"
  - "Code reasoning"
  - "Efficient implementation"
  - "Deep analysis"

spawn_command: |
  cd /Users/nosrcadmin/aos/ptms-experiments-kimi
  # Load this orchestrator prompt
  # Then: start-orchestrate-execution KICKOFF-PTMS.yaml
```

### Worktree 4: ptms-experiments-gemini (variant-gemini)

```yaml
worktree_name: "ptms-experiments-gemini"
branch: "variant-gemini"
llm_provider: "Gemini (Google)"

orchestrator_strengths:
  - "Fast iteration"
  - "Comprehensive testing"
  - "Large context handling"
  - "High test coverage"

spawn_command: |
  cd /Users/nosrcadmin/aos/ptms-experiments-gemini
  # Load this orchestrator prompt
  # Then: start-orchestrate-execution KICKOFF-PTMS.yaml
```

---

## Summary

**Architecture Confirmed:**
✅ 1 orchestrator-agent PER worktree (4 total)  
✅ Each orchestrator spawns specialist agents (up to 10 concurrent)  
✅ Each specialist handles atomic tasks in their domain  
✅ NO generic agents doing entire plans  

**Worktrees Ready:**
- ptms-experiments-claude/ → Claude orchestrator → specialist agents
- ptms-experiments-glm/ → GLM orchestrator → specialist agents
- ptms-experiments-kimi/ → Kimi orchestrator → specialist agents
- ptms-experiments-gemini/ → Gemini orchestrator → specialist agents

**Next Step:**
Start orchestrator in each worktree and begin spawning specialist agents for Plan 01 Phase 01.
