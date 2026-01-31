# Phase 01: System Architecture Design

## EARS Framework

### Event (Trigger)
- Project initiation requires architectural blueprint before any implementation
- Need to define system boundaries, components, and data flow

### Action (What We'll Do)
- Document high-level system architecture
- Define component boundaries and responsibilities
- Specify data flow between components
- Identify integration points and interfaces

### Result (Outcome)
- Complete system architecture document
- Clear component diagram showing all subsystems
- Interface contracts between core components
- Technology decisions with rationale

### Success Criteria
- [ ] Architecture document approved by stakeholders
- [ ] All component boundaries clearly defined
- [ ] Technology stack decisions documented with rationale
- [ ] Data flow diagram shows complete end-to-end paths
- [ ] Integration points identified with protocols

---

## Specification

### System Overview

The Agentic Workflow Manager is a **file-system-first** system with the following layers:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Presentation Layer                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Web Kanban  │  │   Agent CLI  │  │   Admin API  │      │
│  │   Interface   │  │   Protocol    │  │   Endpoints  │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
└─────────┼──────────────────┼──────────────────┼───────────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────┐
│                  API & WebSocket Layer                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │           REST API (Bun HTTP Server)                   │   │
│  │   - Projects CRUD, Transitions, Cards, Search          │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │           WebSocket Server (Real-time)                   │   │
│  │   - Push state changes, agent events, notifications     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┼────────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────┐
│                  Core Business Logic                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  State       │  │  Task        │  │  Dependency  │      │
│  │  Machine     │  │  Executor    │  │  Resolver    │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Notification │  │  Retry       │  │  Concurrency  │      │
│  │  Manager     │  │  Handler     │  │  Controller   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└────────────────────────────┼────────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────┐
│               Data Access Layer                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │           File System Abstraction                        │   │
│  │   - Atomic writes, file locking, watching, queries     │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │           State Cache (Optional SQLite)                 │   │
│  │   - Fast queries, indexing, search optimization         │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┼────────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────────┐
│              Persistence Layer (Source of Truth)                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │            Workspace File System                       │   │
│  │   inbox/ | backlog/ | active/ | review/ | completed/ │   │
│  │   failed/ | notifications/                              │   │
│  └──────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────┘
```

### Component Descriptions

#### 1. Presentation Layer

**Web Kanban Interface**
- Purpose: Human oversight and manual intervention
- Technology: Vue 3 + Tailwind CSS + shadcn/ui
- Communication: REST API + WebSocket
- Key Features:
  - Drag-and-drop card management
  - Real-time state updates
  - Card creation/editing/viewing
  - Agent status monitoring
  - Filter/search functionality

**Agent CLI Protocol**
- Purpose: Programmatic agent interaction
- Technology: TypeScript CLI (executed by agents)
- Communication: File system + optional WebSocket
- Key Features:
  - Task polling/claiming
  - Progress reporting
  - Output file generation
  - Error handling

**Admin API**
- Purpose: System administration and overrides
- Technology: REST endpoints (same as public API)
- Authentication: Admin token-based
- Key Features:
  - Force state transitions
  - View all system state
  - Cancel/retry tasks
  - Configuration management

#### 2. API & WebSocket Layer

**REST API Server**
- Runtime: Bun HTTP Server (native)
- Framework: Hono (lightweight, TypeScript-native)
- Endpoints:
  - `GET /api/projects` - List all projects
  - `GET /api/projects/:id` - Get project details
  - `POST /api/projects` - Create new project
  - `PUT /api/projects/:id/state` - Update project state
  - `GET /api/projects/:id/cards` - List project cards
  - `GET /api/cards/:id` - Get card content
  - `PUT /api/cards/:id` - Update card
  - `POST /api/cards/:id/claim` - Agent claims task
  - `POST /api/cards/:id/complete` - Agent completes task
  - `POST /api/cards/:id/fail` - Agent fails task
- Middleware:
  - Request validation (Zod schemas)
  - Rate limiting (Bun's built-in)
  - CORS configuration
  - Error handling

**WebSocket Server**
- Runtime: Bun native WebSocket
- Channels:
  - `projects/:id` - Project-specific updates
  - `global:state` - System-wide state changes
  - `notifications` - Push notifications to clients
  - `agents:*` - Agent status updates
- Events:
  - `state_transition` - Project moved to new state
  - `card_updated` - Card content changed
  - `card_claimed` - Task claimed by agent
  - `card_completed` - Task completed
  - `card_failed` - Task failed
  - `notification` - New notification

#### 3. Core Business Logic

**State Machine**
- Purpose: Enforce valid project state transitions
- States: `inbox → backlog → active → review → completed/failed`
- Responsibilities:
  - Validate transition rules
  - Execute transition atomically
  - Trigger side effects (notifications, logging)

**Task Executor**
- Purpose: Execute atomic tasks within time limits
- Responsibilities:
  - Spawn worker processes for task execution
  - Monitor task duration (enforce max_duration_minutes)
  - Handle task timeouts gracefully
  - Capture task outputs

**Dependency Resolver**
- Purpose: Determine task execution order
- Algorithm: Topological sort with cycle detection
- Responsibilities:
  - Build dependency graph from card definitions
  - Identify ready tasks (no pending dependencies)
  - Block tasks with unmet dependencies
  - Detect and report circular dependencies

**Notification Manager**
- Purpose: Dispatch notifications to appropriate channels
- Channels: WebSocket, Webhook, Email, File log
- Responsibilities:
  - Queue notification events
  - Format messages per channel
  - Retry failed deliveries
  - Aggregate related notifications

**Retry Handler**
- Purpose: Manage failed task retries
- Responsibilities:
  - Track retry count per task
  - Exponential backoff for retries
  - Escalate after retry_max exceeded
  - Move project to failed state if unrecoverable

**Concurrency Controller**
- Purpose: Prevent race conditions in multi-agent scenarios
- Mechanisms:
  - POSIX file locks (`flock()`)
  - Optimistic locking (version fields)
  - Atomic file operations (write-temp + rename)
  - Lock timeout handling

#### 4. Data Access Layer

**File System Abstraction**
- Purpose: Provide clean interface to file system operations
- Key Operations:
  - `readProject(id)` - Read project folder + state.json
  - `writeProject(id, state)` - Atomic write with lock
  - `moveProject(id, fromState, toState)` - Move folder atomically
  - `watchStateChanges(callback)` - File system events
  - `listProjectsByState(state)` - Scan folders efficiently
- Implementation: Node.js `fs` + `chokidar` for watching

**State Cache (Optional)**
- Purpose: Fast queries without scanning file system
- Technology: SQLite (bun:sqlite)
- Indexes: project_id, state, assignee, created_at, priority
- Sync: Update cache on every file system write
- Cache Invalidation: File watcher triggers rebuild

#### 5. Persistence Layer

**Workspace File System**
- Purpose: Single source of truth for all state
- Structure:
  ```
  workspace/
  ├── inbox/                    # New projects (unprocessed)
  ├── backlog/                  # Ready for execution
  ├── active/                   # Currently executing
  │   ├── unassigned/           # Tasks waiting for agent
  │   └── assigned/            # Agent-owned
  │       ├── agent-01/
  │       └── agent-02/
  ├── review/                   # Human approval pending
  ├── completed/                # Successful completions
  │   └── 2026/01/            # Organized by date
  ├── failed/                   # Failed with error logs
  └── notifications/            # Event logs
  ```
- File Format:
  - `state.json` - Project state (atomic write target)
  - `project.md` - Project description (human-readable)
  - `cards/` - Task cards (Markdown with YAML frontmatter)

### Data Flow Examples

#### Flow 1: Human Creates New Project
```
Human (Web UI)
  → POST /api/projects
  → API Layer: Validate request, generate UUID
  → Business Logic: Create initial state.json
  → Data Layer: Write to workspace/inbox/{uuid}/
  → File System: Creates folder + files
  → File Watcher: Detects CLOSE_WRITE on state.json
  → Business Logic: Auto-transition inbox → backlog
  → Data Layer: Move to workspace/backlog/{uuid}/
  → WebSocket: Publish `state_transition` event
  → Web UI: Updates Kanban (new card in Backlog column)
```

#### Flow 2: Agent Claims Task
```
Agent (CLI)
  → POST /api/cards/:id/claim (with agent_id)
  → API Layer: Validate agent exists, card is pending
  → Business Logic: Update card.assignee, card.state=in-progress
  → Data Layer: Atomic write state.json with lock
  → File System: state.json updated
  → File Watcher: Detects change
  → Concurrency Controller: Check no other agent claimed it
  → WebSocket: Publish `card_claimed` event
  → Web UI: Updates card to show agent assignment
```

#### Flow 3: Task Execution Fails
```
Task Executor (monitoring)
  → Task exceeds max_duration_minutes OR process exits with error
  → Business Logic: Increment card.retry_count
  → Data Layer: Update card.state=failed, add error log
  → File System: state.json updated
  → File Watcher: Detects change
  → Retry Handler: Check if retry_count < retry_max
    → If yes: Reset card.state=pending, queue for retry
    → If no: Check if project can continue without this card
      → If yes: Continue with remaining cards
      → If no: Move project to failed/ state
  → Notification Manager: Send alert
  → WebSocket: Publish `card_failed` event
  → Web UI: Update card status, show error
```

### Technology Decisions (with Rationale)

| Technology | Choice | Rationale |
|-----------|--------|-----------|
| **Runtime** | Bun | 3x faster than Node.js, native TypeScript, built-in HTTP/WS server |
| **Frontend** | Vue 3 | Reactive, TypeScript-first, smaller bundle than React, good ecosystem |
| **UI Library** | Tailwind + shadcn/ui | Utility-first styling, accessible components, no runtime overhead |
| **API Framework** | Hono | Lightweight, TypeScript-native, faster than Express, Bun-compatible |
| **File Watching** | chokidar | Battle-tested, cross-platform (inotify, FSEvents, ReadDirectoryChangesW) |
| **Locking** | POSIX flock() | Simple, atomic, OS-level guarantee, no external dependencies |
| **Cache** | SQLite (bun:sqlite) | Embedded, zero config, fast queries, can sync from file system |
| **Task Workers** | Bun Worker Threads | Parallel execution without process overhead, shared memory |
| **Markdown** | frontmatter + remark-parse | Standard format, easy to edit, supports YAML metadata |

### Integration Points & Interfaces

#### Interface 1: File System → File Watcher
- **Protocol**: OS file system events (inotify, FSEvents)
- **Event Types**: `CLOSE_WRITE` on state.json
- **Debounce**: 100ms to batch rapid changes
- **Contract**: Watcher guarantees at-least-once delivery

#### Interface 2: API Layer → Business Logic
- **Protocol**: TypeScript function calls
- **Contract**: All functions return `Result<T, Error>` (never throw)
- **Validation**: Zod schemas at entry point
- **Error Handling**: Convert domain errors to HTTP status codes

#### Interface 3: Business Logic → Data Layer
- **Protocol**: Async functions with file paths as input
- **Contract**: All writes are atomic (lock + write temp + rename)
- **Locking**: Caller must not hold lock across multiple operations
- **Retries**: Exponential backoff for transient failures

#### Interface 4: WebSocket Server → Clients
- **Protocol**: JSON over WebSocket
- **Message Format**: `{event: string, data: any, timestamp: ISO8601}`
- **Reconnection**: Client implements exponential backoff
- **Heartbeat**: 30-second pings to detect dead connections

#### Interface 5: Agent CLI → Backend
- **Protocol**: REST API (primary) + direct file writes (fallback)
- **Fallback**: If API unavailable, agent can write directly to state.json
- **Contract**: Agent must check lock before writing, respect max_duration_minutes
- **Timeout**: Agent must respond within 5s to API requests

### Non-Functional Requirements

#### Performance
- File system operations: < 10ms for single project read
- WebSocket message latency: < 50ms p99
- API response time: < 200ms p99 (non-compute endpoints)
- Concurrent agents: Support 50+ agents claiming tasks simultaneously

#### Scalability
- Projects: Support 10,000+ projects in workspace
- File watching: Handle 100+ watched folders without performance degradation
- WebSocket clients: Support 200+ concurrent connections
- Horizontal scaling: Distribute workers across multiple processes if needed

#### Reliability
- Data consistency: Never corrupt state.json (atomic writes always)
- Lock contention: Resolve within 5 seconds or fail fast
- Recovery: Graceful restart from any crash state
- Durability: All state persisted to disk before acknowledgment

#### Observability
- Logging: Structured JSON logs for all operations
- Metrics: Counters for state transitions, task durations, errors
- Tracing: Request ID passed through all layers
- Health checks: `/health` endpoint reports system status

---

## Deliverables

1. **ARCHITECTURE.md** - This document
2. **COMPONENT-DIAGRAM.png** - Visual component diagram
3. **DATA-FLOW-DIAGRAMS.md** - Sequence diagrams for key flows
4. **INTERFACE-CONTRACTS.md** - TypeScript interfaces for all integration points
5. **TECH-DECISIONS.md** - Technology choice rationales

---

## Next Phase

After architecture is finalized, proceed to **Phase 02: File System Structure Setup** to implement the workspace directories and file watching infrastructure.
