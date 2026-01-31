# Phase 07: Core Interfaces & Integration Tests

## EARS Framework

### Event (Trigger)
- All foundation components are implemented
- Need to ensure components work together correctly
- Ready to handoff to workflow engine implementation

### Action (What We'll Do)
- Finalize all public interfaces
- Create comprehensive integration tests
- Implement utility functions
- Document foundation APIs
- Run full test suite and validate

### Result (Outcome)
- All components tested together
- Complete API documentation
- Foundation layer ready for next plans
- No integration issues detected

### Success Criteria
- [ ] All unit tests pass (> 90% coverage)
- [ ] Integration tests pass (end-to-end flows)
- [ ] API documentation complete
- [ ] No critical bugs in foundation layer
- [ ] Performance benchmarks meet targets

---

## Specification

### Integration Test Scenarios

Define key end-to-end scenarios to test:

```
┌─────────────────────────────────────────────────────────────┐
│              Integration Test Scenarios                  │
├─────────────────────────────────────────────────────────────┤
│                                                          │
│  1. Project Lifecycle                                    │
│     - Create project in inbox                              │
│     - Auto-transition to backlog                          │
│     - Claim task by agent                                 │
│     - Complete task                                       │
│     - Transition to completed                              │
│                                                          │
│  2. Concurrent Access                                     │
│     - Multiple agents claim different tasks                  │
│     - Two agents try to claim same task                   │
│     - Rapid state.json updates                            │
│                                                          │
│  3. File System Resilience                               │
│     - Corrupted state.json detection                       │
│     - Partial write recovery                               │
│     - Lock timeout handling                               │
│                                                          │
│  4. State Machine Validations                            │
│     - Invalid transitions rejected                         │
│     - Valid transitions execute                            │
│     - Checkpoint flow                                     │
│                                                          │
│  5. Dependency Resolution                                 │
│     - Card blocked until dependency completed                │
│     - Circular dependency detection                        │
│     - Dependency graph traversal                           │
│                                                          │
└─────────────────────────────────────────────────────────────┘
```

### Public API

Create `src/core/README.md` documenting all public APIs:

```markdown
# Foundation Layer Public API

## File System API (`src/core/FileSystem.ts`)

### Interface: IFileSystem

```typescript
interface IFileSystem {
  // Read operations
  readProject(projectId: string): Promise<Result<ProjectState>>;
  readCard(projectId: string, filename: string): Promise<Result<Card>>;

  // Write operations (atomic)
  writeProject(projectId: string, state: ProjectState): Promise<Result<void>>;
  writeCard(projectId: string, filename: string, card: Card): Promise<Result<void>>;

  // Folder operations
  moveProject(projectId: string, fromState, toState): Promise<Result<void>>;
  createProject(projectId, initialState, projectMd): Promise<Result<void>>;
  deleteProject(projectId: string): Promise<Result<void>>;

  // Query operations
  listProjects(state): Promise<Result<string[]>>;

  // Locking
  acquireLock(projectId, timeoutMs): Promise<Result<number>>;
  releaseLock(projectId, fd): Promise<Result<void>>;

  // Watching
  watchStateChanges(callback): () => void; // Returns cleanup
}
```

### Usage Example

```typescript
import { FileSystem } from "./core/FileSystem";

const fs = new FileSystem("workspace");

// Create new project
await fs.createProject(projectId, initialState, projectDescription);

// Claim card atomically
await fs.withLock(projectId, async () => {
  const project = await fs.readProject(projectId);
  // Update project...
  await fs.writeProject(projectId, updatedProject);
});

// Watch for changes
const cleanup = fs.watchStateChanges((event) => {
  console.log("State changed:", event.projectId);
});

// Cleanup later
cleanup();
```

---

## State Machine API (`src/engine/StateMachine.ts`)

### Class: ProjectStateMachine

```typescript
class ProjectStateMachine {
  constructor(fs: FileSystem, notifications: NotificationManager, logger: Logger);

  // Execute state transition
  async transition(request: TransitionRequest): Promise<TransitionResult>;

  // Check if transition is valid
  validateTransition(from, to): TransitionResult;

  // Check project readiness
  checkProjectReady(projectId, targetState): Promise<{ ready: boolean; reason? }>;

  // Update card states
  updateCardStates(projectId): Promise<void>;
}
```

### Usage Example

```typescript
import { ProjectStateMachine } from "./engine/StateMachine";

const sm = new ProjectStateMachine(fs, notifications, logger);

// Transition project
const result = await sm.transition({
  project_id: "project-123",
  from_state: "backlog",
  to_state: "active",
  reason: "Agent available",
  force: false,
});

if (result.success) {
  console.log("Transitioned to:", result.to_state);
}
```

---

## Concurrency API (`src/concurrency/`)

### Lock Manager

```typescript
class LockManager {
  constructor(config: LockConfig);

  // Acquire lock
  async acquire(resourceId: string): Promise<LockAcquisition>;

  // Release lock
  async release(resourceId: string): Promise<void>;

  // Execute with lock
  async withLock<T>(resourceId: string, fn: () => Promise<T>): Promise<T>;
}
```

### Optimistic Locking

```typescript
class OptimisticLocking {
  constructor(fs: FileSystem);

  // Atomic update with CAS
  async atomicUpdate<T extends { version: number }>(
    resourceId: string,
    updateFn: (current: T) => T | Promise<T>,
    maxRetries?: number
  ): Promise<Result<T>>;

  // Atomic card claim
  async claimCard(projectId, cardId, agentId): Promise<Result>;
}
```

### Concurrency Controller

```typescript
class ConcurrencyController {
  constructor(config: ConcurrencyConfig);

  // Execute with concurrency control
  async execute<T>(fn: () => Promise<T>, priority?: number, agentId?: string): Promise<T>;

  // Get stats
  getStats(): { activeCount, queueLength, agentsActive };
}
```

---

## Security API (`src/security/`)

### Security Manager

```typescript
class SecurityManager {
  constructor(config: SecurityConfig);

  // Generate token
  generateToken(type, id, scopes): string;

  // Validate token
  validateToken(token): { valid, payload?, error? };

  // Authorize request
  authorize(token, requiredScope): { authorized, token?, error? };
}
```

### Rate Limiter

```typescript
class RateLimiter {
  constructor(limits, windowMs?);

  // Check if allowed
  check(scope, identifier): { allowed, remaining, resetAt };

  // Get limit
  getLimit(scope): number;

  // Reset bucket
  reset(scope, identifier): void;
}
```

---

## Watching API (`src/watcher/`)

### File Watcher

```typescript
class FileWatcher extends EventEmitter {
  constructor(config: WatcherConfig);

  // Start watching
  async start(): Promise<void>;

  // Stop watching
  async stop(): Promise<void>;

  // Get stats
  getStats(): { watchedPaths, debounceQueueSize, ... };
}
```

### Watcher Service

```typescript
class WatcherService {
  // Get singleton instance
  static async getInstance(config, fs, stateMachine, logger): Promise<WatcherService>;

  // Start/stop
  async start(): Promise<void>;
  async stop(): Promise<void>;

  // Subscribe to events
  on(event, callback): void;
}
```

---

## Types (`src/core/types/`)

### Project Types

```typescript
type ProjectId = string & { __brand: unique symbol };
type ProjectState = "inbox" | "backlog" | "active" | "review" | "completed" | "failed";
type Priority = 1 | 2 | 3 | 4 | 5;

interface ProjectState {
  schema_version: "1.0";
  project_id: ProjectId;
  project_name: string;
  state: ProjectState;
  created_at: string; // ISO8601
  updated_at: string;
  assignee_agent: string | null;
  cards: CardState[];
  // ... other fields
}
```

### Card Types

```typescript
type CardId = string & { __brand: unique symbol };
type CardState = "pending" | "in-progress" | "completed" | "failed" | "blocked";
type AgentProfile = "researcher" | "coder" | "analyst" | "general";

interface CardFrontmatter {
  card_id: CardId;
  type: "task" | "meta" | "checkpoint" | "notification";
  state: CardState;
  priority: 1-5;
  atomic: boolean;
  max_duration_minutes: number;
  dependencies: CardId[];
  // ... other fields
}

interface Card {
  frontmatter: CardFrontmatter;
  content: string;
  path: string;
}
```

### Agent Types

```typescript
type AgentId = string & { __brand: unique symbol };
type Capability = "web-search" | "document-analysis" | "code-generation" | ...;

interface AgentManifest {
  agent_id: AgentId;
  name: string;
  capabilities: Capability[];
  profiles: AgentProfile[];
  max_concurrent: number;
  status: "online" | "offline" | "busy";
  // ... other fields
}
```

### Utility Types

```typescript
type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E };

type AsyncResult<T, E = Error> = Promise<Result<T, E>>;
```
```

### Integration Tests

Create `tests/integration/ProjectLifecycle.test.ts`:

```typescript
import { describe, it, expect, beforeAll, afterAll } from "bun:test";
import { FileSystem } from "../../src/core/FileSystem";
import { ProjectStateMachine } from "../../src/engine/StateMachine";
import { FileWatcher, WatcherConfig } from "../../src/watcher/FileWatcher";
import { WatcherService } from "../../src/watcher/WatcherService";
import { Logger } from "../../src/utils/Logger";
import { mkdir, rm } from "fs/promises";
import { join, tmpdir } from "path";
import { createProjectId } from "../../src/core/types";

const TEST_WORKSPACE = join(tmpdir(), "integration-test");

describe("Integration: Project Lifecycle", () => {
  let fs: FileSystem;
  let sm: ProjectStateMachine;
  let watcherService: WatcherService;
  let logger: Logger;

  beforeAll(async () => {
    // Setup workspace
    await mkdir(TEST_WORKSPACE, { recursive: true });

    // Initialize components
    fs = new FileSystem(TEST_WORKSPACE);
    logger = new Logger("integration-test");
    sm = new ProjectStateMachine(fs, { emitNotification: async () => {} } as any, logger);

    // Start file watcher
    const config: WatcherConfig = {
      workspaceRoot: TEST_WORKSPACE,
      debounceMs: 100,
      ignorePatterns: [/^\./],
      pollInterval: 10,
      stabilityThreshold: 50,
    };
    watcherService = await WatcherService.getInstance(config, fs, sm, logger);
    await watcherService.start();

    // Wait for watcher to be ready
    await new Promise((resolve) => setTimeout(resolve, 200));
  });

  afterAll(async () => {
    await watcherService.stop();
    await rm(TEST_WORKSPACE, { recursive: true, force: true });
  });

  it("should complete full project lifecycle", async () => {
    // Step 1: Create project in inbox
    const projectId = createProjectId("test-project-1");
    const initialState = {
      schema_version: "1.0" as const,
      project_id: projectId,
      project_name: "Integration Test Project",
      state: "inbox" as const,
      created_at: new Date().toISOString(),
      updated_at: new Date().toISOString(),
      assignee_agent: null,
      checkpoint_required: false,
      checkpoint_approved: null,
      retry_count: 0,
      cards_total: 2,
      cards_completed: 0,
      cards_failed: 0,
      cards: [
        {
          card_id: "card-1",
          filename: "001-task1.md",
          state: "pending" as const,
          assignee: null,
          started_at: null,
          completed_at: null,
          failed_at: null,
          retry_count: 0,
          dependencies: [],
          checkpoint_after: false,
        },
        {
          card_id: "card-2",
          filename: "002-task2.md",
          state: "pending" as const,
          assignee: null,
          started_at: null,
          completed_at: null,
          failed_at: null,
          retry_count: 0,
          dependencies: ["card-1"],
          checkpoint_after: false,
        },
      ],
    };

    await fs.createProject(projectId, initialState, "Test project");

    // Wait for auto-transition inbox → backlog
    await new Promise((resolve) => setTimeout(resolve, 300));

    // Verify project moved to backlog
    const backlogList = await fs.listProjects("backlog");
    expect(backlogList.success).toBe(true);
    expect(backlogList.data).toContain(projectId);

    // Step 2: Transition to active
    await sm.transition({
      project_id: projectId,
      from_state: "backlog",
      to_state: "active",
      reason: "Ready for execution",
      force: false,
    });

    // Step 3: Claim and complete card 1
    await sm.transition({
      project_id: projectId,
      from_state: "active",
      to_state: "active", // State unchanged, card updated
      reason: "Card 1 claimed and completed",
      force: false,
    });

    // Verify card 1 completed
    const project = await fs.readProject(projectId);
    expect(project.success).toBe(true);
    expect(project.data.cards[0].state).toBe("completed");

    // Step 4: Complete card 2 (dependency met)
    await sm.transition({
      project_id: projectId,
      from_state: "active",
      to_state: "completed",
      reason: "All cards completed",
      force: false,
    });

    // Verify project completed
    const completedProject = await fs.readProject(projectId);
    expect(completedProject.success).toBe(true);
    expect(completedProject.data.state).toBe("completed");

    console.log("✓ Full project lifecycle completed successfully");
  });
});
```

### Performance Benchmarks

Create `tests/benchmarks/FilePerformance.test.ts`:

```typescript
import { describe, it, expect, beforeAll, afterAll } from "bun:test";
import { FileSystem } from "../../src/core/FileSystem";
import { mkdir, rm } from "fs/promises";
import { join, tmpdir } from "path";

const TEST_WORKSPACE = join(tmpdir(), "benchmark-test");

describe("Benchmarks: File Performance", () => {
  let fs: FileSystem;

  beforeAll(async () => {
    await mkdir(TEST_WORKSPACE, { recursive: true });
    fs = new FileSystem(TEST_WORKSPACE);
  });

  afterAll(async () => {
    await rm(TEST_WORKSPACE, { recursive: true, force: true });
  });

  it("should read 100 projects in < 1000ms", async () => {
    // Create 100 projects
    for (let i = 0; i < 100; i++) {
      const projectId = `bench-project-${i}`;
      const state = createMockState(projectId);
      await fs.createProject(projectId, state, "Benchmark project");
    }

    // Measure read performance
    const start = Date.now();
    for (let i = 0; i < 100; i++) {
      const projectId = `bench-project-${i}`;
      const result = await fs.readProject(projectId);
      expect(result.success).toBe(true);
    }
    const duration = Date.now() - start;

    console.log(`Read 100 projects in ${duration}ms`);
    expect(duration).toBeLessThan(1000); // < 10ms per read
  });

  it("should handle 100 concurrent writes without corruption", async () => {
    const projectId = "concurrent-write-test";
    await fs.createProject(projectId, createMockState(projectId), "Test");

    // Concurrent writes
    const promises = Array.from({ length: 100 }, async (_, i) => {
      const state = await fs.readProject(projectId);
      if (!state.success) throw state.error;
      state.data.project_name = `Update ${i}`;
      return fs.writeProject(projectId, state.data);
    });

    const results = await Promise.allSettled(promises);

    // Verify final state is valid
    const final = await fs.readProject(projectId);
    expect(final.success).toBe(true);
    expect(final.data.project_name).toMatch(/^Update \d+$/);

    console.log(`✓ All writes completed, ${results.filter(r => r.status === 'fulfilled').length}/100 succeeded`);
  });
});
```

### Final Validation Script

Create `scripts/validate-foundation.ts`:

```typescript
import { FileSystem } from "../src/core/FileSystem";
import { ProjectStateMachine } from "../src/engine/StateMachine";
import { Logger } from "../src/utils/Logger";

const logger = new Logger("validate-foundation");

async function validateFoundation() {
  console.log("=== Foundation Layer Validation ===\n");

  let passed = 0;
  let failed = 0;

  // Test 1: File system operations
  try {
    logger.info("Test 1: File system operations");
    const fs = new FileSystem("workspace");
    const projectId = "validation-project-1";

    const state = createMockState(projectId);
    await fs.createProject(projectId, state, "Validation test");

    const readResult = await fs.readProject(projectId);
    if (readResult.success) {
      logger.success("✓ File system operations working");
      passed++;
    } else {
      throw readResult.error;
    }
  } catch (error) {
    logger.error("✗ File system operations failed", error);
    failed++;
  }

  // Test 2: State machine
  try {
    logger.info("Test 2: State machine");
    const fs = new FileSystem("workspace");
    const sm = new ProjectStateMachine(fs, { emitNotification: async () => {} } as any, logger);

    const result = await sm.transition({
      project_id: "validation-project-1",
      from_state: "backlog",
      to_state: "active",
      reason: "Validation",
      force: false,
    });

    if (result.success) {
      logger.success("✓ State machine working");
      passed++;
    } else {
      throw new Error("Transition failed");
    }
  } catch (error) {
    logger.error("✗ State machine failed", error);
    failed++;
  }

  // Test 3: Concurrency
  try {
    logger.info("Test 3: Concurrency");
    const fs = new FileSystem("workspace");
    const lm = new LockManager({
      lockDir: "locks",
      timeoutMs: 1000,
      retryAttempts: 3,
      retryDelayMs: 10,
    });

    const lock1 = await lm.acquire("resource-1");
    if (lock1.success) {
      await lm.release("resource-1");
      logger.success("✓ Concurrency working");
      passed++;
    } else {
      throw lock1.error;
    }
  } catch (error) {
    logger.error("✗ Concurrency failed", error);
    failed++;
  }

  // Summary
  console.log(`\n=== Validation Summary ===`);
  console.log(`Passed: ${passed}`);
  console.log(`Failed: ${failed}`);

  if (failed === 0) {
    console.log("\n✓ Foundation layer is ready for next phase!");
    process.exit(0);
  } else {
    console.log("\n✗ Foundation layer has issues, fix before proceeding.");
    process.exit(1);
  }
}

function createMockState(projectId: string) {
  return {
    schema_version: "1.0" as const,
    project_id: projectId,
    project_name: "Validation Test",
    state: "backlog" as const,
    created_at: new Date().toISOString(),
    updated_at: new Date().toISOString(),
    assignee_agent: null,
    checkpoint_required: false,
    checkpoint_approved: null,
    retry_count: 0,
    cards_total: 0,
    cards_completed: 0,
    cards_failed: 0,
    cards: [],
  };
}

// Run validation
validateFoundation();
```

---

## Implementation Steps

1. **Document all public APIs**
   - Create API documentation
   - Add usage examples
   - Document interfaces

2. **Write integration tests**
   - Project lifecycle test
   - Concurrent access test
   - Dependency resolution test

3. **Write performance benchmarks**
   - File read/write benchmarks
   - Lock contention benchmarks
   - Concurrency benchmarks

4. **Create validation script**
   - Automated foundation layer validation
   - Test all core functionality
   - Generate validation report

5. **Run full test suite**
   - Unit tests
   - Integration tests
   - Benchmarks

6. **Generate final report**
   - Test coverage report
   - Performance metrics
   - Known issues and limitations

---

## Deliverables

1. **`src/core/README.md`** - Complete API documentation
2. **`tests/integration/ProjectLifecycle.test.ts`** - Integration test suite
3. **`tests/benchmarks/FilePerformance.test.ts`** - Performance benchmarks
4. **`scripts/validate-foundation.ts`** - Validation script
5. **Test coverage report** (HTML + JSON)
6. **Performance metrics report**

---

## Foundation Plan Complete! 🎉

The foundation layer is now complete with:

✓ System architecture designed
✓ File system structure implemented
✓ Data models and types defined
✓ State machine implemented
✓ Concurrency and security measures in place
✓ File watching infrastructure running
✓ All components tested together

### Next Steps

Proceed to **Plan 02: Workflow Engine** to implement:
- Task execution engine
- Dependency resolution
- Retry logic
- Checkpoint handling
- Notification system

---

## Plan 01 Summary

**Phase 01: System Architecture Design**
- Documented high-level system architecture
- Defined component boundaries and responsibilities

**Phase 02: File System Structure Setup**
- Created workspace directory structure
- Implemented file system abstraction layer
- Added atomic file operations with locking

**Phase 03: Data Model & Types Definition**
- Defined TypeScript interfaces for all domain objects
- Created Zod schemas for validation
- Generated types from schemas

**Phase 04: State Machine Definition**
- Implemented project state machine with validation
- Implemented card state transitions
- Added transition side effects

**Phase 05: Concurrency & Security Design**
- Implemented file locking with POSIX flock()
- Added optimistic locking with version fields
- Created security model with authentication and rate limiting

**Phase 06: File Watching Infrastructure**
- Set up chokidar file watcher
- Implemented event debouncing
- Created event dispatcher for state changes

**Phase 07: Core Interfaces & Integration Tests**
- Finalized all public interfaces
- Created comprehensive integration tests
- Documented foundation APIs

**Total:** 7 phases, ~30 tasks, foundation layer complete
