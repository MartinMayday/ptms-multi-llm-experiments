# Project Task Management System

## Overview

A file-system-based project management system for orchestrating AI agents performing atomic tasks in long-running workflows, with human oversight via Kanban interface.

## Vision Statement

**To provide a resilient, observable, and agent-native workflow management system where:**

- File system is the single source of truth (git-like versioning)
- Every task is atomic with defined inputs/outputs
- Humans can intervene at any workflow stage
- Agents can interact programmatically (UI is secondary)
- Failures are handled gracefully with retry mechanisms

## Core Principles

1. **File-System as Source of Truth** - All state in files, enabling git-like versioning
2. **Atomic Tasks** - Each task is independently executable with defined inputs/outputs
3. **Observable** - Humans can view and intervene at any workflow stage
4. **Agent-Native** - Designed for programmatic interaction first, UI second
5. **Resilient** - Graceful handling of failures, retries, and interruptions

## Project Plans

This project is organized into **5 execution plans**, each with 5-8 phases:

| Plan | Name | Description | Phases |
|------|------|-------------|--------|
| 01 | Foundation & Architecture | Core system architecture, file system structure, data models | 7 |
| 02 | Workflow Engine | State transitions, task execution, dependency resolution, notifications | 7 |
| 03 | Backend API | REST API, WebSocket, authentication, agent endpoints | 6 |
| 04 | Web Interface | Kanban UI, real-time updates, card management | 6 |
| 05 | Agent Interface | Agent protocol, task claiming, progress reporting | 6 |

## Technology Stack

### Backend
- **Runtime**: Bun (fast, TypeScript-native)
- **Language**: TypeScript 5.x
- **File System**: Node.js `fs` with `chokidar` for watching
- **Concurrency**: Worker threads for parallel task execution
- **WebSocket**: Bun's native WebSocket server

### Frontend
- **Framework**: Vue 3 + TypeScript
- **Build**: Vite
- **UI Library**: Tailwind CSS + shadcn/ui components
- **Real-time**: Native WebSocket client
- **State Management**: Pinia

### Infrastructure
- **File Storage**: Local file system (source of truth)
- **Optional Index**: SQLite for faster queries (cache, not source of truth)
- **Locking**: POSIX `flock()` for concurrency control

## Development Workflow

This project follows **Spec-Driven Development** with **EARS Framework**:

1. **E**vent: What triggers this work?
2. **A**ction: What will be done?
3. **R**esult: What will be achieved?
4. **S**uccess Criteria: How do we know it's done?

Each plan file contains detailed specifications that specialist agents can execute without ambiguity.

## Directory Structure (After Implementation)

```
project-task-management-system/
├── .planning/              # This directory (spec files)
│   ├── phase-01-foundation/
│   ├── phase-02-workflow-engine/
│   ├── phase-03-backend-api/
│   ├── phase-04-web-interface/
│   └── phase-05-agent-interface/
├── src/
│   ├── core/               # Foundation types, interfaces
│   ├── engine/             # Workflow state machine
│   ├── api/                # REST API endpoints
│   ├── ws/                 # WebSocket handlers
│   ├── agents/             # Agent protocol implementation
│   ├── ui/                 # Vue frontend
│   └── utils/             # Shared utilities
├── tests/                  # Test suites
├── workspace/              # Runtime workspace (state folders)
│   ├── inbox/
│   ├── backlog/
│   ├── active/
│   ├── review/
│   ├── completed/
│   ├── failed/
│   └── notifications/
└── package.json
```

## Getting Started

To build this system, execute plans in order:

```bash
# Plan 1: Foundation & Architecture
./execute-plan.sh phase-01-foundation/01-PLAN.md

# Plan 2: Workflow Engine
./execute-plan.sh phase-02-workflow-engine/01-PLAN.md

# ... and so on
```

## Specification Files

Each plan directory contains detailed phase specifications:

- **PROJECT-STATE.md**: Overall project state and decisions
- **[NN]-PLAN.md**: Detailed plan for each phase
- **[NN]-SPEC.md**: Technical specification artifacts

## Success Criteria

The system is successful when:

- [ ] File system accurately reflects all project and task states
- [ ] State transitions are atomic and never corrupt data
- [ ] Multiple agents can work concurrently without conflicts
- [ ] Human can observe and intervene at any point via Kanban UI
- [ ] Agents can claim, execute, and report tasks programmatically
- [ ] All failures are recoverable or escalated appropriately
- [ ] System remains responsive under load (100+ concurrent projects)

## Contributing

This is a spec-driven project. All changes must:
1. Follow EARS framework for phase planning
2. Include clear success criteria
3. Maintain backward compatibility with file system schema
4. Pass all existing tests

## License

[To be determined]
