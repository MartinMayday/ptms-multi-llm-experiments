# Plan 02: Workflow Engine

## Overview

Implements the core workflow execution engine that manages task execution, dependency resolution, retry logic, checkpoint handling, and notification delivery.

## Phases (7)

| Phase | Name | Description |
|--------|------|-------------|
| 01 | Task Executor | Spawns worker processes for task execution, enforces time limits |
| 02 | Dependency Resolver | Topological sort, cycle detection, blocking/unblocking |
| 03 | Retry Handler | Exponential backoff, retry counting, escalation |
| 04 | Checkpoint Handler | Human review workflow, approval/rejection |
| 05 | Notification Manager | Multi-channel notifications (WebSocket, Webhook, Email) |
| 06 | Error Recovery | Graceful degradation, crash recovery, state restoration |
| 07 | Workflow Integration Tests | End-to-end workflow scenarios |

## Key Components

```
src/engine/
├── TaskExecutor.ts         # Worker process management
├── DependencyResolver.ts    # Dependency graph algorithms
├── RetryHandler.ts         # Retry logic and escalation
├── CheckpointHandler.ts     # Human review workflow
└── NotificationManager.ts   # Notification dispatch
```

## Success Criteria

- [ ] Tasks execute in isolated worker processes
- [ ] Dependency resolution prevents circular dependencies
- [ ] Retry logic follows exponential backoff
- [ ] Checkpoints require human approval before continuation
- [ ] Notifications delivered to all configured channels
- [ ] Errors are recovered gracefully

## Dependencies

- Plan 01 (Foundation) - must be complete
- FileSystem, StateMachine, LockManager, FileWatcher

## Next Plan

Plan 03: Backend API (REST endpoints, WebSocket)
