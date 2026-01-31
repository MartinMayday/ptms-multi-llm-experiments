# Plan 05: Agent Interface

## Overview

Implements the agent CLI protocol for programmatic interaction, including task claiming, progress reporting, and output generation.

## Phases (6)

| Phase | Name | Description |
|--------|------|-------------|
| 01 | Agent CLI Framework | CLI structure, argument parsing |
| 02 | Task Polling & Claiming | Poll for tasks, claim with CAS |
| 03 | Progress Reporting | Update task status, emit logs |
| 04 | Output Generation | Generate expected output files |
| 05 | Error Handling | Graceful failure, timeout handling |
| 06 | Agent Documentation | Protocol spec, examples |

## Key Components

```
src/agent/
├── cli.ts                   # Main CLI entry point
├── AgentClient.ts            # API client wrapper
├── TaskRunner.ts            # Task execution logic
├── OutputGenerator.ts        # Output file generation
├── ProgressReporter.ts       # Progress tracking
└── examples/
    └── example-agent.ts       # Sample agent implementation
```

## Agent Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                    Agent Lifecycle                             │
├─────────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Initialize                                               │
│     - Load agent manifest (capabilities, profiles)              │
│     - Register with system (POST /api/agents/register)          │
│     - Get authentication token                                  │
│                                                               │
│  2. Poll for Tasks                                           │
│     - GET /api/projects?state=active&unassigned=true            │
│     - Filter by agent profile (researcher, coder, etc.)        │
│     - Check dependencies, priority, due dates                    │
│                                                               │
│  3. Claim Task                                               │
│     - POST /api/cards/:id/claim                              │
│     - Include agent_id, capabilities                            │
│     - Receive claim confirmation or rejection                    │
│                                                               │
│  4. Execute Task                                             │
│     - Monitor max_duration_minutes                             │
│     - Handle SIGTERM gracefully (checkpoint on interrupt)         │
│     - Generate output files to outputs/ directory              │
│                                                               │
│  5. Report Progress                                          │
│     - PATCH /api/cards/:id (update state, logs)               │
│     - Emit progress events via WebSocket                       │
│     - Handle intermittent failures                               │
│                                                               │
│  6. Complete Task                                             │
│     - POST /api/cards/:id/complete                            │
│     - Include output files, metrics, duration                   │
│     - Handle checkpoint_after (wait for human approval)          │
│                                                               │
│  7. Handle Failure                                            │
│     - POST /api/cards/:id/fail                               │
│     - Include error details, logs                               │
│     - Wait for retry or escalation                              │
│                                                               │
│  8. Heartbeat (Keep-alive)                                   │
│     - POST /api/agents/heartbeat every 30s                     │
│     - Report current load, active tasks                          │
│                                                               │
└───────────────────────────────────────────────────────────────────┘
```

## Agent Manifest

```json
{
  "agent_id": "agent-researcher-01",
  "name": "Research Agent 01",
  "description": "Web research and document analysis",
  "version": "1.0.0",
  "capabilities": [
    "web-search",
    "document-analysis",
    "data-extraction"
  ],
  "profiles": [
    "researcher",
    "analyst"
  ],
  "max_concurrent": 2,
  "timeout_default": 30,
  "status": "online"
}
```

## CLI Commands

```bash
# Start agent (poll for tasks)
agent-cli start --manifest agent-manifest.json

# Claim specific task
agent-cli claim --card-id abc-123

# Report progress
agent-cli progress --card-id abc-123 --percent 50 --message "Researching..."

# Complete task
agent-cli complete --card-id abc-123 --outputs research-findings.json

# Fail task
agent-cli fail --card-id abc-123 --error "API rate limit exceeded"

# Heartbeat
agent-cli heartbeat --load 2 --active-tasks ["abc-123", "def-456"]
```

## Success Criteria

- [ ] Agents can poll for and claim tasks matching their profile
- [ ] Task execution respects max_duration_minutes
- [ ] Progress updates are reported in real-time
- [ ] Output files are generated correctly
- [ ] Failures are handled gracefully with retry/escalation
- [ ] Agent protocol is well-documented with examples

## Dependencies

- Plan 01 (Foundation) - required for types
- Plan 03 (Backend API) - required for agent endpoints
- CardFrontmatter, AgentManifest, WebSocket

## Tech Stack

- **Runtime**: Bun (for CLI)
- **CLI Framework**: Commander.js or similar
- **HTTP Client**: Bun's native fetch
- **WebSocket**: native WebSocket client

## Project Complete! 🎉

All 5 plans are now specified:

1. ✅ **Plan 01: Foundation & Architecture** (7 phases)
   - System architecture, file system, types, state machine, concurrency, file watching

2. ✅ **Plan 02: Workflow Engine** (7 phases)
   - Task executor, dependency resolver, retry handler, checkpoint handler, notifications

3. ✅ **Plan 03: Backend API** (6 phases)
   - REST endpoints, WebSocket server, agent protocol

4. ✅ **Plan 04: Web Interface** (6 phases)
   - Kanban UI, drag-and-drop, real-time updates

5. ✅ **Plan 05: Agent Interface** (6 phases)
   - Agent CLI, task claiming, progress reporting

**Total: 5 plans, 32 phases, comprehensive spec-driven-development**
