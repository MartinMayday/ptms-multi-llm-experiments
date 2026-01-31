# Phase 06: File Watching Infrastructure

## EARS Framework

### Event (Trigger)
- Concurrency layer is complete
- Need real-time detection of file system changes
- State transitions must trigger automatically on state.json changes

### Action (What We'll Do)
- Set up chokidar file watcher
- Implement event debouncing
- Create event dispatcher for state changes
- Add file integrity checking
- Implement watcher resilience (auto-restart)

### Result (Outcome)
- Real-time file system event detection
- Debounced events (no duplicates)
- Automatic state transition triggers
- Resilient to file system issues

### Success Criteria
- [ ] State.json CLOSE_WRITE events fire within 200ms
- [ ] Rapid changes (100ms apart) are debounced to single event
- [ ] Watcher restarts automatically after crash
- [ ] File hash verification detects corrupted writes
- [ ] Event dispatcher processes events in order

---

## Specification

### File Watching Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              File Watching Layer (src/watcher/)            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. FileWatcher (chokidar)                                │
│     - Watch workspace directories                             │
│     - Filter for state.json files                           │
│     - Handle errors gracefully                               │
│                                                             │
│  2. Event Debouncer                                        │
│     - Batch rapid changes (100ms stability threshold)          │
│     - Deduplicate events                                    │
│     - Preserve event order                                  │
│                                                             │
│  3. Event Dispatcher                                       │
│     - Parse file changes                                    │
│     - Route to appropriate handlers                         │
│     - Emit to subscribers                                  │
│                                                             │
│  4. Integrity Checker                                      │
│     - Compute file hashes                                   │
│     - Detect corrupted files                                │
│     - Validate JSON structure                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### File Watcher Implementation

Create `src/watcher/FileWatcher.ts`:

```typescript
import chokidar from "chokidar";
import { EventEmitter } from "events";

/**
 * Watch event types
 */
export type WatchEventType = "add" | "change" | "unlink";

/**
 * Watch event
 */
export interface WatchEvent {
  type: WatchEventType;
  path: string;
  projectId?: string; // Extracted from path
  state?: string;     // inbox|backlog|active|review|completed|failed
  timestamp: number;
}

/**
 * Watcher configuration
 */
export interface WatcherConfig {
  workspaceRoot: string;
  debounceMs: number;
  ignorePatterns: RegExp[];
  pollInterval: number;
  stabilityThreshold: number;
}

/**
 * File watcher using chokidar
 */
export class FileWatcher extends EventEmitter {
  private watcher: chokidar.FSWatcher | null = null;
  private debounceTimeouts = new Map<string, NodeJS.Timeout>();
  private eventQueue: WatchEvent[] = [];
  private processingQueue = false;

  constructor(private config: WatcherConfig) {
    super();
  }

  /**
   * Start watching workspace
   */
  async start(): Promise<void> {
    if (this.watcher) {
      throw new Error("Watcher already started");
    }

    console.log(`[FileWatcher] Starting watcher for ${this.config.workspaceRoot}`);

    const watchPaths = [
      `${this.config.workspaceRoot}/inbox`,
      `${this.config.workspaceRoot}/backlog`,
      `${this.config.workspaceRoot}/active`,
      `${this.config.workspaceRoot}/review`,
      `${this.config.workspaceRoot}/completed`,
      `${this.config.workspaceRoot}/failed`,
    ];

    this.watcher = chokidar.watch(watchPaths, {
      ignored: this.config.ignorePatterns,
      persistent: true,
      ignoreInitial: true, // Don't fire events for existing files
      awaitWriteFinish: {
        stabilityThreshold: this.config.stabilityThreshold, // ms
        pollInterval: this.config.pollInterval,
      },
      usePolling: false, // Use native file system events (faster)
    });

    // Bind event handlers
    this.watcher
      .on("add", (path) => this.handleRawEvent("add", path))
      .on("change", (path) => this.handleRawEvent("change", path))
      .on("unlink", (path) => this.handleRawEvent("unlink", path))
      .on("error", (error) => this.handleError(error))
      .on("ready", () => {
        console.log("[FileWatcher] Ready and watching");
      });

    // Wait for watcher to be ready
    await new Promise<void>((resolve) => {
      this.watcher!.once("ready", () => resolve());
    });

    console.log("[FileWatcher] Started successfully");
  }

  /**
   * Stop watching
   */
  async stop(): Promise<void> {
    if (!this.watcher) {
      return;
    }

    console.log("[FileWatcher] Stopping watcher");

    // Clear all debounce timeouts
    this.debounceTimeouts.forEach((timeout) => clearTimeout(timeout));
    this.debounceTimeouts.clear();

    // Close watcher
    await this.watcher.close();
    this.watcher = null;

    console.log("[FileWatcher] Stopped");
  }

  /**
   * Handle raw file system event
   */
  private handleRawEvent(type: WatchEventType, path: string): void {
    // Only process state.json files
    if (!path.endsWith("/state.json")) {
      return;
    }

    // Extract project ID and state from path
    const parts = path.split("/");
    const state = parts[parts.length - 2];
    const projectId = parts[parts.length - 3];

    if (!projectId || !state) {
      console.warn(`[FileWatcher] Invalid path format: ${path}`);
      return;
    }

    // Debounce: clear existing timeout for this file
    const existingTimeout = this.debounceTimeouts.get(path);
    if (existingTimeout) {
      clearTimeout(existingTimeout);
    }

    // Schedule debounced event
    const timeout = setTimeout(() => {
      const event: WatchEvent = {
        type,
        path,
        projectId,
        state: state as any,
        timestamp: Date.now(),
      };

      this.queueEvent(event);
      this.debounceTimeouts.delete(path);
    }, this.config.debounceMs);

    this.debounceTimeouts.set(path, timeout);
  }

  /**
   * Queue event for processing
   */
  private queueEvent(event: WatchEvent): void {
    this.eventQueue.push(event);

    // Start processing if not already
    if (!this.processingQueue) {
      this.processQueue = true;
      setImmediate(() => this.processEventQueue());
    }
  }

  /**
   * Process event queue (in order)
   */
  private async processEventQueue(): Promise<void> {
    while (this.eventQueue.length > 0) {
      const event = this.eventQueue.shift()!;

      // Emit event
      this.emit("change", event);

      // Small delay between events to prevent flooding
      await new Promise((resolve) => setTimeout(resolve, 10));
    }

    this.processingQueue = false;
  }

  /**
   * Handle watcher error
   */
  private handleError(error: Error): void {
    console.error("[FileWatcher] Error:", error);

    // Emit error event
    this.emit("error", error);

    // Attempt to restart after fatal errors
    if (this.watcher) {
      this.restart();
    }
  }

  /**
   * Restart watcher (after error)
   */
  private async restart(): Promise<void> {
    console.log("[FileWatcher] Restarting after error...");

    try {
      await this.stop();
      await this.start();
      console.log("[FileWatcher] Restarted successfully");
    } catch (error) {
      console.error("[FileWatcher] Restart failed:", error);

      // Retry after delay
      setTimeout(() => this.restart(), 5000);
    }
  }

  /**
   * Get watcher stats
   */
  getStats() {
    return {
      watchedPaths: this.watcher?.getWatched() || {},
      debounceQueueSize: this.debounceTimeouts.size,
      eventQueueLength: this.eventQueue.length,
      processing: this.processingQueue,
    };
  }
}
```

### Integrity Checker

Create `src/watcher/IntegrityChecker.ts`:

```typescript
import { readFileSync } from "fs";
import { createHash } from "crypto";
import { join } from "path";
import { ProjectStateSchema } from "../core/types";

/**
 * File integrity check result
 */
export interface IntegrityCheck {
  valid: boolean;
  hash?: string;
  error?: string;
}

/**
 * Integrity checker for state.json files
 */
export class IntegrityChecker {
  /**
   * Compute SHA256 hash of file
   */
  async computeHash(filePath: string): Promise<string> {
    const content = readFileSync(filePath, "utf-8");
    const hash = createHash("sha256");
    hash.update(content);
    return hash.digest("hex");
  }

  /**
   * Validate JSON structure
   */
  async validateJSON(filePath: string): Promise<IntegrityCheck> {
    try {
      const content = readFileSync(filePath, "utf-8");
      const json = JSON.parse(content);

      // Validate against Zod schema
      const result = ProjectStateSchema.safeParse(json);

      if (!result.success) {
        return {
          valid: false,
          error: `Validation failed: ${result.error.message}`,
        };
      }

      const hash = await this.computeHash(filePath);
      return { valid: true, hash };
    } catch (error) {
      return {
        valid: false,
        error: `Invalid JSON: ${(error as Error).message}`,
      };
    }
  }

  /**
   * Check if file changed since last check
   */
  async hasChanged(
    filePath: string,
    previousHash: string
  ): Promise<boolean> {
    const currentHash = await this.computeHash(filePath);
    return currentHash !== previousHash;
  }
}
```

### Event Dispatcher

Create `src/watcher/EventDispatcher.ts`:

```typescript
import { FileWatcher, WatchEvent } from "./FileWatcher";
import { FileSystem } from "../core/FileSystem";
import { IntegrityChecker } from "./IntegrityChecker";
import { StateMachine } from "../engine/StateMachine";
import { Logger } from "../utils/Logger";

/**
 * Event dispatcher
 *
 * Routes file system events to appropriate handlers
 */
export class EventDispatcher {
  private processedHashes = new Map<string, string>();

  constructor(
    private fileWatcher: FileWatcher,
    private fs: FileSystem,
    private integrityChecker: IntegrityChecker,
    private stateMachine: StateMachine,
    private logger: Logger
  ) {
    this.setupEventHandlers();
  }

  /**
   * Set up event handlers
   */
  private setupEventHandlers(): void {
    this.fileWatcher.on("change", async (event: WatchEvent) => {
      await this.handleStateChangeEvent(event);
    });

    this.fileWatcher.on("error", (error: Error) => {
      this.logger.error("File watcher error", { error: error.message });
    });
  }

  /**
   * Handle state.json change event
   */
  private async handleStateChangeEvent(event: WatchEvent): Promise<void> {
    const { path, projectId } = event;

    this.logger.info("State change detected", { projectId, path });

    try {
      // Validate file integrity
      const integrityCheck = await this.integrityChecker.validateJSON(path);

      if (!integrityCheck.valid) {
        this.logger.error("File integrity check failed", {
          projectId,
          path,
          error: integrityCheck.error,
        });
        return;
      }

      // Check if actually changed (avoid duplicate processing)
      const previousHash = this.processedHashes.get(path);
      if (previousHash && previousHash === integrityCheck.hash) {
        this.logger.debug("File not actually changed (duplicate event)", {
          projectId,
        });
        return;
      }

      // Read project state
      const readResult = await this.fs.readProject(projectId);
      if (!readResult.success) {
        this.logger.error("Failed to read project", {
          projectId,
          error: readResult.error.message,
        });
        return;
      }

      const project = readResult.data;

      // Check if state transition occurred
      const previousHashKey = `${path}:state`;
      const previousState = this.processedHashes.get(previousHashKey);

      if (previousState && previousState !== project.state) {
        this.logger.info("State transition detected", {
          projectId,
          from: previousState,
          to: project.state,
        });

        // Trigger state machine transition
        // (State machine will handle validation and side effects)
        try {
          await this.stateMachine.transition({
            project_id: projectId,
            from_state: previousState as any,
            to_state: project.state,
            reason: "State file updated",
            force: false,
          });
        } catch (error) {
          this.logger.error("State transition failed", {
            projectId,
            error: (error as Error).message,
          });
        }
      }

      // Store current hash
      this.processedHashes.set(path, integrityCheck.hash);
      this.processedHashes.set(previousHashKey, project.state);

      this.logger.info("State change processed", { projectId });
    } catch (error) {
      this.logger.error("Error processing state change", {
        projectId,
        error: (error as Error).message,
        stack: (error as Error).stack,
      });
    }
  }

  /**
   * Clear processed hashes (for testing or manual intervention)
   */
  clearProcessedHashes(): void {
    this.processedHashes.clear();
  }

  /**
   * Get processed hashes (for debugging)
   */
  getProcessedHashes(): Map<string, string> {
    return new Map(this.processedHashes);
  }
}
```

### Watcher Service

Create `src/watcher/WatcherService.ts`:

```typescript
import { FileWatcher, WatcherConfig } from "./FileWatcher";
import { EventDispatcher } from "./EventDispatcher";
import { FileSystem } from "../core/FileSystem";
import { IntegrityChecker } from "./IntegrityChecker";
import { StateMachine } from "../engine/StateMachine";
import { Logger } from "../utils/Logger";

/**
 * Watcher service (singleton)
 *
 * Manages the entire file watching infrastructure
 */
export class WatcherService {
  private static instance: WatcherService;
  private fileWatcher: FileWatcher;
  private eventDispatcher: EventDispatcher;

  private constructor(
    config: WatcherConfig,
    fs: FileSystem,
    stateMachine: StateMachine,
    logger: Logger
  ) {
    const integrityChecker = new IntegrityChecker();
    this.fileWatcher = new FileWatcher(config);

    this.eventDispatcher = new EventDispatcher(
      this.fileWatcher,
      fs,
      integrityChecker,
      stateMachine,
      logger
    );
  }

  /**
   * Get or create watcher service instance
   */
  static async getInstance(
    config: WatcherConfig,
    fs: FileSystem,
    stateMachine: StateMachine,
    logger: Logger
  ): Promise<WatcherService> {
    if (!WatcherService.instance) {
      WatcherService.instance = new WatcherService(
        config,
        fs,
        stateMachine,
        logger
      );
    }

    return WatcherService.instance;
  }

  /**
   * Start watcher service
   */
  async start(): Promise<void> {
    await this.fileWatcher.start();
  }

  /**
   * Stop watcher service
   */
  async stop(): Promise<void> {
    await this.fileWatcher.stop();
  }

  /**
   * Get watcher stats
   */
  getStats() {
    return {
      fileWatcher: this.fileWatcher.getStats(),
    };
  }

  /**
   * Subscribe to change events
   */
  on(event: "change", callback: (event: any) => void): void {
    this.fileWatcher.on(event, callback);
  }
}

### Tests

Create `tests/watcher/FileWatcher.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from "bun:test";
import { FileWatcher, WatchEvent, WatcherConfig } from "../../src/watcher/FileWatcher";
import { mkdir, rmdir, writeFile, rm } from "fs/promises";
import { join, tmpdir } from "path";

const TEST_DIR = join(tmpdir(), "watcher-test");
const STATE_FILE = join(TEST_DIR, "inbox", "test-project", "state.json");

describe("FileWatcher", () => {
  let watcher: FileWatcher;
  let events: WatchEvent[] = [];

  beforeEach(async () => {
    // Create test directory structure
    await mkdir(join(TEST_DIR, "inbox", "test-project"), {
      recursive: true,
    });

    // Create initial state.json
    const state = {
      schema_version: "1.0",
      project_id: "test-project-123",
      project_name: "Test Project",
      state: "inbox",
      created_at: new Date().toISOString(),
      updated_at: new Date().toISOString(),
      cards_total: 0,
      cards_completed: 0,
      cards_failed: 0,
      cards: [],
    };
    await writeFile(STATE_FILE, JSON.stringify(state, null, 2));

    // Create watcher
    const config: WatcherConfig = {
      workspaceRoot: TEST_DIR,
      debounceMs: 100,
      ignorePatterns: [/^\./],
      pollInterval: 10,
      stabilityThreshold: 50,
    };

    watcher = new FileWatcher(config);
    watcher.on("change", (event) => events.push(event));
    await watcher.start();

    // Wait for watcher to be ready
    await new Promise((resolve) => setTimeout(resolve, 100));
  });

  afterEach(async () => {
    await watcher.stop();
    events = [];
    await rm(TEST_DIR, { recursive: true, force: true });
  });

  it("should detect state.json changes", async () => {
    // Update state.json
    const state = JSON.parse(await Bun.file(STATE_FILE).text());
    state.project_name = "Updated Name";
    await writeFile(STATE_FILE, JSON.stringify(state, null, 2));

    // Wait for debounced event
    await new Promise((resolve) => setTimeout(resolve, 200));

    expect(events.length).toBeGreaterThan(0);
    expect(events[0].projectId).toBe("test-project");
    expect(events[0].path).toContain("state.json");
  });

  it("should debounce rapid changes", async () => {
    // Rapidly write multiple times
    for (let i = 0; i < 5; i++) {
      const state = JSON.parse(await Bun.file(STATE_FILE).text());
      state.project_name = `Name ${i}`;
      await writeFile(STATE_FILE, JSON.stringify(state, null, 2));
      await new Promise((resolve) => setTimeout(resolve, 20));
    }

    // Wait for debounced event
    await new Promise((resolve) => setTimeout(resolve, 200));

    // Should only fire once (debounced)
    expect(events.length).toBe(1);
  });
});
```

---

## Implementation Steps

1. **Implement file watcher**
   - Set up chokidar
   - Filter for state.json files
   - Handle errors gracefully

2. **Implement debouncing**
   - Stability threshold (50ms)
   - Delay timeout (100ms)
   - Preserve event order

3. **Implement integrity checker**
   - SHA256 hash computation
   - JSON validation
   - Change detection

4. **Implement event dispatcher**
   - Route events to handlers
   - Check for actual changes
   - Trigger state machine transitions

5. **Create watcher service**
   - Singleton instance
   - Start/stop methods
   - Stats monitoring

6. **Write comprehensive tests**
   - Test event detection
   - Test debouncing
   - Test error handling
   - Test watcher restart

---

## Deliverables

1. **`src/watcher/FileWatcher.ts`** - File watcher implementation
2. **`src/watcher/IntegrityChecker.ts`** - File integrity validation
3. **`src/watcher/EventDispatcher.ts`** - Event routing
4. **`src/watcher/WatcherService.ts`** - Service wrapper
5. **`tests/watcher/FileWatcher.test.ts`** - Test suite

---

## Next Phase

After file watching is complete, proceed to **Phase 07: Core Interfaces & Integration Tests** to complete foundation layer with comprehensive integration testing and finalize all interfaces.
