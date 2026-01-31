# Project Task Management System - Orchestration Package

Complete multi-agent orchestration system for building the Agentic Workflow Management System.

## Package Contents

1. **ORCHESTRATOR-AGENT-PROMPT.md** - Master orchestrator system prompt
2. **KICKOFF-PTMS.yaml** - Ready-to-use kickoff configuration
3. **SUMMARY.md** - Complete project specification
4. **INDEX.md** - Project overview and navigation
5. **phase-01-foundation/** - Plan 01 (Foundation) with 7 detailed phases
6. **phase-02-workflow-engine/** - Plan 02 (Workflow Engine)
7. **phase-03-backend-api/** - Plan 03 (Backend API)
8. **phase-04-web-interface/** - Plan 04 (Web Interface)
9. **phase-05-agent-interface/** - Plan 05 (Agent Interface)

## Quick Start

### Step 1: Prepare New Repository

```bash
# Create new project directory
mkdir -p ~/projects/ptms-orchestrated
cd ~/projects/ptms-orchestrated

# Initialize git
git init

# Copy orchestration files
cp -r /Users/nosrcadmin/aos/COSMO/.planning/project-task-management-system .
```

### Step 2: Start opencode Session

```bash
# Start opencode CLI in project directory
opencode

# The orchestrator prompt will load automatically
```

### Step 3: Execute Orchestration

```bash
# Hand off to orchestrator with kickoff file
start-orchestrate-execution project-task-management-system/KICKOFF-PTMS.yaml
```

## What Happens Next

The Orchestrator-Agent will:

1. **Parse** the kickoff file and understand the project structure
2. **Display** the execution roadmap (5 plans, 32 phases)
3. **Begin** with Plan 01: Foundation & Architecture
4. **Spawn** specialist agents (up to 10 concurrent) for each phase
5. **Manage** dependencies and ensure quality gates pass
6. **Report** progress at each plan boundary
7. **Deliver** complete, tested, documented system

## Specialist Agents Available

| Agent | Purpose | Max Concurrent |
|-------|---------|----------------|
| Architect | System design | 1 |
| Backend-Core | File system, atomic operations | 2 |
| Backend-Types | TypeScript, Zod schemas | 2 |
| Backend-Engine | State machines, workflows | 2 |
| Backend-Concurrency | Locking, CAS, rate limiting | 1 |
| Backend-API | Hono, REST, WebSocket | 2 |
| Frontend-UI | Vue 3, Tailwind, components | 3 |
| Frontend-State | Pinia, real-time updates | 2 |
| Agent-Developer | CLI, agent protocol | 2 |
| QA-Engineer | Testing, benchmarks | 2 |
| Technical-Writer | Documentation | 1 |
| Code-Reviewer | Quality validation | 2 |

## Execution Timeline

- **Plan 01** (Foundation): 5-7 days - Sequential execution, all other plans depend on this
- **Plan 02** (Workflow Engine): 5-7 days - Starts after Plan 01
- **Plan 03** (Backend API): 4-6 days - Starts after Plans 01 & 02
- **Plan 04** (Web Interface): 4-6 days - Starts after Plan 03, can parallel with Plan 05
- **Plan 05** (Agent Interface): 3-5 days - Starts after Plan 03, can parallel with Plan 04

**Total**: 15-22 days with full parallelization

## Key Features

### Context Freshness (gsd-workflow)
- Every task spawns in fresh 200K context
- Prevents context rot and quality degradation
- Atomic commits for each task

### Dependency Management
- DAG validation (no circular dependencies)
- Topological sort for execution order
- Parallel execution where dependencies allow

### Quality Gates
- TypeScript compilation mandatory
- Test coverage > 90% required
- All success criteria must pass
- Performance benchmarks enforced

### State Tracking
- `.orchestrator/state.json` - Execution state
- `.orchestrator/progress.md` - Human-readable progress
- `.orchestrator/logs/` - Detailed execution logs

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

## Customization

### Modifying Agent Assignments

Edit `KICKOFF-PTMS.yaml`:

```yaml
phases:
  phase_01_02:
    agent: "Backend-Core"  # Change to different agent type
    max_concurrent: 3       # Override default
```

### Adding New Plans

Add to `KICKOFF-PTMS.yaml`:

```yaml
plans:
  plan_06:
    id: "06"
    name: "DevOps & Deployment"
    agent: "DevOps-Engineer"
    dependencies: ["05"]
```

### Custom Success Criteria

Modify `special_instructions` in kickoff file:

```yaml
success_criteria:
  custom_metric:
    description: "My custom requirement"
    threshold: 100
```

## Troubleshooting

### Agent Consistently Fails

```
# Orchestrator will automatically:
1. Retry with error context (2 attempts)
2. Spawn Code-Reviewer for analysis
3. Present findings to user
```

### Circular Dependency Detected

```
# Orchestrator will:
1. Halt execution
2. Display dependency graph
3. Request user resolution
```

### Performance Issues

```
# Check benchmarks in:
- benchmarks/metrics.json
- .orchestrator/progress.md
# Orchestrator will flag phases exceeding targets
```

## File Structure After Completion

```
project-root/
├── .orchestrator/
│   ├── state.json
│   ├── progress.md
│   └── logs/
├── src/
│   ├── core/          # Foundation layer
│   ├── engine/        # Workflow engine
│   ├── api/           # Backend API
│   ├── ui/            # Web interface
│   ├── agent/         # Agent CLI
│   ├── concurrency/   # Concurrency control
│   ├── security/      # Security layer
│   └── watcher/       # File watching
├── tests/
│   ├── unit/
│   ├── integration/
│   └── benchmarks/
├── workspace/         # Runtime state
│   ├── inbox/
│   ├── backlog/
│   ├── active/
│   ├── review/
│   ├── completed/
│   └── failed/
├── docs/              # Documentation
├── package.json
├── tsconfig.json
└── README.md
```

## Support

### Documentation
- **ORCHESTRATOR-AGENT-PROMPT.md** - Complete orchestrator specification
- **SUMMARY.md** - Project specification overview
- **INDEX.md** - Project navigation

### Phase Specifications
- **phase-01-foundation/** - 7 detailed phase specs
- **phase-02-workflow-engine/** - Workflow engine overview
- **phase-03-backend-api/** - API specification
- **phase-04-web-interface/** - UI specification
- **phase-05-agent-interface/** - Agent protocol spec

### Questions?

1. Check phase-specific READMEs in each plan directory
2. Review ORCHESTRATOR-AGENT-PROMPT.md Section 12 (Troubleshooting)
3. Examine `.orchestrator/progress.md` for current status

---

**Ready to orchestrate?** Copy these files to a new repo and run:

```bash
start-orchestrate-execution KICKOFF-PTMS.yaml
```

The orchestrator will handle the rest!
