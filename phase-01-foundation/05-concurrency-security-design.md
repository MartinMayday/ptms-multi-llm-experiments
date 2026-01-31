# Phase 05: Concurrency & Security Design

## EARS Framework

### Event (Trigger)
- State machine is complete
- Need to handle concurrent agent access
- Need to prevent race conditions in multi-agent scenarios

### Action (What We'll Do)
- Implement file locking with POSIX flock()
- Add optimistic locking with version fields
- Create concurrency controller for task execution
- Design security model (authentication, authorization)
- Implement rate limiting and throttling

### Result (Outcome)
- Thread-safe file operations
- No data corruption under concurrent load
- Secure access control for agents and users
- Configurable rate limits

### Success Criteria
- [ ] Concurrent writes never corrupt state.json
- [ ] File locks resolve within 5 seconds or fail fast
- [ ] Race condition tests pass (100+ concurrent writers)
- [ ] Unauthorized requests are rejected (401/403)
- [ ] Rate limiting prevents abuse (429 responses)

---

## Specification

### Concurrency Threats

Identified concurrency issues:

1. **Race condition**: Two agents claim same card simultaneously
2. **Dirty read**: Agent reads stale state.json
3. **Lost update**: Agent overwrites another's changes
4. **Lock starvation**: Long-running operations hold locks too long
5. **Deadlock**: Multiple agents waiting on each other

### Concurrency Strategy

```
┌─────────────────────────────────────────────────────────┐
│           Concurrency Layer (src/concurrency/)         │
├─────────────────────────────────────────────────────────┤
│                                                       │
│  1. Lock Manager                                      │
│     - POSIX flock() for file locks                    │
│     - Lock timeout with exponential backoff             │
│     - Lock release with cleanup                        │
│                                                       │
│  2. Optimistic Locking                               │
│     - Version field in state.json                      │
│     - CAS (Compare-And-Swap) operations              │
│     - Retry on version mismatch                       │
│                                                       │
│  3. Concurrency Controller                            │
│     - Limit concurrent operations                     │
│     - Queue pending operations                        │
│     - Prioritize critical operations                  │
│                                                       │
└─────────────────────────────────────────────────────────┘
```

### Lock Manager

Create `src/concurrency/LockManager.ts`:

```typescript
import { open, close, flock, constants } from "fs/promises";
import { join } from "path";

/**
 * Lock configuration
 */
export interface LockConfig {
  lockDir: string;           // Directory for lock files
  timeoutMs: number;         // Max wait time for lock
  retryAttempts: number;      // Max retry attempts
  retryDelayMs: number;      // Base delay for exponential backoff
}

/**
 * Lock acquisition result
 */
export interface LockAcquisition {
  success: boolean;
  lockFileDescriptor?: number;
  error?: string;
}

/**
 * Lock manager using POSIX flock()
 */
export class LockManager {
  private locks = new Map<string, number>(); // resourceId -> fd

  constructor(private config: LockConfig) {}

  /**
   * Acquire lock on resource
   *
   * Uses exponential backoff with max 3 attempts:
   * - Attempt 1: 10ms wait
   * - Attempt 2: 40ms wait
   * - Attempt 3: 160ms wait
   */
  async acquire(resourceId: string): Promise<LockAcquisition> {
    const lockPath = join(this.config.lockDir, `${resourceId}.lock`);
    const startTime = Date.now();

    for (let attempt = 0; attempt < this.config.retryAttempts; attempt++) {
      try {
        // Open or create lock file
        const fd = await open(
          lockPath,
          constants.O_WRONLY | constants.O_CREAT,
          0644
        );

        // Try to acquire exclusive lock (non-blocking)
        await flock(fd, constants.LOCK_EX | constants.LOCK_NB);

        // Lock acquired!
        this.locks.set(resourceId, fd);
        return {
          success: true,
          lockFileDescriptor: fd,
        };
      } catch (error: any) {
        // Close fd if we opened it
        const fd = this.locks.get(resourceId);
        if (fd) {
          await close(fd);
        }

        // EWOULDBLOCK means lock is held
        if (error.code === "EWOULDBLOCK") {
          // Check timeout
          if (Date.now() - startTime > this.config.timeoutMs) {
            return {
              success: false,
              error: `Lock timeout after ${this.config.timeoutMs}ms`,
            };
          }

          // Exponential backoff
          const backoffMs = this.config.retryDelayMs * Math.pow(2, attempt);
          await new Promise((resolve) => setTimeout(resolve, backoffMs));
          continue;
        }

        // Other error (file system issue, etc.)
        return {
          success: false,
          error: error.message,
        };
      }
    }

    // All attempts exhausted
    return {
      success: false,
      error: `Failed to acquire lock after ${this.config.retryAttempts} attempts`,
    };
  }

  /**
   * Release lock on resource
   */
  async release(resourceId: string): Promise<void> {
    const fd = this.locks.get(resourceId);
    if (!fd) {
      console.warn(`No lock held for resource: ${resourceId}`);
      return;
    }

    try {
      // Release lock
      await flock(fd, constants.LOCK_UN);
      await close(fd);
      this.locks.delete(resourceId);
    } catch (error) {
      console.error(`Failed to release lock for ${resourceId}:`, error);
    }
  }

  /**
   * Check if resource is locked
   */
  isLocked(resourceId: string): boolean {
    return this.locks.has(resourceId);
  }

  /**
   * Release all locks (cleanup on shutdown)
   */
  async releaseAll(): Promise<void> {
    const releasePromises = Array.from(this.locks.entries()).map(
      async ([resourceId, fd]) => {
        try {
          await flock(fd, constants.LOCK_UN);
          await close(fd);
        } catch (error) {
          console.error(`Failed to release lock for ${resourceId}:`, error);
        }
      }
    );

    await Promise.all(releasePromises);
    this.locks.clear();
  }

  /**
   * Execute function with lock held
   *
   * Usage:
   * ```ts
   * await lockManager.withLock("project-123", async () => {
   *   // Safe operation
   * });
   * ```
   */
  async withLock<T>(
    resourceId: string,
    fn: () => Promise<T>
  ): Promise<T> {
    const acquisition = await this.acquire(resourceId);

    if (!acquisition.success) {
      throw new Error(acquisition.error || "Failed to acquire lock");
    }

    try {
      return await fn();
    } finally {
      await this.release(resourceId);
    }
  }
}
```

### Optimistic Locking

Create `src/concurrency/OptimisticLocking.ts`:

```typescript
import { FileSystem } from "../core/FileSystem";
import { Result } from "../core/types";

/**
 * Optimistic locking using version field
 *
 * Pattern:
 * 1. Read resource with version
 * 2. Compute new state
 * 3. Write with CAS: if version matches, update; else retry
 */
export class OptimisticLocking {
  constructor(private fs: FileSystem) {}

  /**
   * Atomic update with CAS (Compare-And-Swap)
   *
   * @returns Number of retries needed
   */
  async atomicUpdate<T extends { version: number }>(
    resourceId: string,
    updateFn: (current: T) => T | Promise<T>,
    maxRetries: number = 3
  ): Promise<Result<T>> {
    for (let attempt = 0; attempt < maxRetries; attempt++) {
      // Step 1: Read current version
      const readResult = await this.fs.readResource(resourceId);
      if (!readResult.success) {
        return { success: false, error: readResult.error };
      }

      const current = readResult.data as T;
      const currentVersion = current.version;

      // Step 2: Compute new state
      const updated = await updateFn(current);
      updated.version = currentVersion + 1;

      // Step 3: Write with CAS (only if version matches)
      const writeResult = await this.fs.writeResourceIfVersionMatches(
        resourceId,
        currentVersion,
        updated
      );

      if (writeResult.success) {
        return { success: true, data: updated };
      }

      // CAS failed: another process updated it, retry
      console.log(
        `CAS failed for ${resourceId}, retry ${attempt + 1}/${maxRetries}`
      );
    }

    return {
      success: false,
      error: new Error(
        `Atomic update failed after ${maxRetries} attempts (CAS conflicts)`
      ),
    };
  }

  /**
   * Atomic claim operation for card claiming
   *
   * Ensures only one agent can claim a card
   */
  async claimCard(
    projectId: string,
    cardId: string,
    agentId: string
  ): Promise<Result<{ success: boolean; reason?: string }>> {
    return await this.atomicUpdate(projectId, (project) => {
      // Find card
      const card = project.cards.find((c: any) => c.card_id === cardId);
      if (!card) {
        throw new Error(`Card not found: ${cardId}`);
      }

      // Check if already claimed
      if (card.assignee) {
        if (card.assignee === agentId) {
          return project; // Already claimed by same agent
        }
        throw new Error(`Card already claimed by ${card.assignee}`);
      }

      // Check if card is ready
      if (card.state !== "pending") {
        throw new Error(`Card is ${card.state}, cannot claim`);
      }

      // Check dependencies
      const cardMap = new Map(project.cards.map((c: any) => [c.card_id, c]));
      for (const depId of card.dependencies) {
        const dep = cardMap.get(depId);
        if (!dep || dep.state !== "completed") {
          throw new Error(`Dependency ${depId} not completed`);
        }
      }

      // Claim card
      card.assignee = agentId;
      card.state = "in-progress";
      card.started_at = new Date().toISOString();

      return project;
    });
  }
}
```

### Concurrency Controller

Create `src/concurrency/ConcurrencyController.ts`:

```typescript
/**
 * Concurrency configuration
 */
export interface ConcurrencyConfig {
  maxConcurrentOperations: number;    // Global limit
  maxPerAgentOperations: number;     // Per-agent limit
  queueSize: number;                // Max queued operations
  queueTimeoutMs: number;           // Max wait time in queue
}

/**
 * Queued operation
 */
interface QueuedOperation<T> {
  id: string;
  fn: () => Promise<T>;
  priority: number;                // 1 = highest, 10 = lowest
  enqueuedAt: number;
  resolve: (value: T) => void;
  reject: (error: any) => void;
  agentId?: string;                // For per-agent limits
}

/**
 * Concurrency controller
 *
 * Limits concurrent operations and queues pending requests
 */
export class ConcurrencyController {
  private queue: QueuedOperation<any>[] = [];
  private activeOps = new Map<string, number>(); // agentId -> active count
  private activeCount = 0;

  constructor(private config: ConcurrencyConfig) {}

  /**
   * Execute operation with concurrency control
   */
  async execute<T>(
    fn: () => Promise<T>,
    priority: number = 5,
    agentId?: string
  ): Promise<T> {
    // Check if we can execute immediately
    if (this.canExecute(agentId)) {
      return await this.executeImmediate(fn, agentId);
    }

    // Queue the operation
    return await this.enqueue(fn, priority, agentId);
  }

  /**
   * Check if operation can execute immediately
   */
  private canExecute(agentId?: string): boolean {
    // Check global limit
    if (this.activeCount >= this.config.maxConcurrentOperations) {
      return false;
    }

    // Check per-agent limit
    if (agentId) {
      const agentActive = this.activeOps.get(agentId) || 0;
      if (agentActive >= this.config.maxPerAgentOperations) {
        return false;
      }
    }

    return true;
  }

  /**
   * Execute operation immediately
   */
  private async executeImmediate<T>(
    fn: () => Promise<T>,
    agentId?: string
  ): Promise<T> {
    this.activeCount++;
    if (agentId) {
      this.activeOps.set(agentId, (this.activeOps.get(agentId) || 0) + 1);
    }

    try {
      return await fn();
    } finally {
      this.activeCount--;
      if (agentId) {
        const current = this.activeOps.get(agentId) || 0;
        if (current <= 1) {
          this.activeOps.delete(agentId);
        } else {
          this.activeOps.set(agentId, current - 1);
        }
      }

      // Process next queued operation
      this.processQueue();
    }
  }

  /**
   * Enqueue operation for later execution
   */
  private async enqueue<T>(
    fn: () => Promise<T>,
    priority: number,
    agentId?: string
  ): Promise<T> {
    // Check queue size
    if (this.queue.length >= this.config.queueSize) {
      throw new Error("Queue full, try again later");
    }

    return new Promise((resolve, reject) => {
      const id = `op-${Date.now()}-${Math.random()}`;
      const enqueuedAt = Date.now();

      this.queue.push({
        id,
        fn,
        priority,
        enqueuedAt,
        resolve,
        reject,
        agentId,
      });

      // Sort queue by priority (1 = highest), then by enqueue time
      this.queue.sort((a, b) => {
        if (a.priority !== b.priority) {
          return a.priority - b.priority;
        }
        return a.enqueuedAt - b.enqueuedAt;
      });

      // Set timeout for queue wait
      setTimeout(() => {
        const index = this.queue.findIndex((op) => op.id === id);
        if (index >= 0) {
          this.queue.splice(index, 1);
          reject(new Error("Queue timeout"));
        }
      }, this.config.queueTimeoutMs);
    });
  }

  /**
   * Process next queued operation
   */
  private processQueue(): void {
    while (this.queue.length > 0) {
      const next = this.queue[0];

      // Check if we can execute
      if (this.canExecute(next.agentId)) {
        this.queue.shift();

        // Execute
        next.fn()
          .then(next.resolve)
          .catch(next.reject)
          .finally(() => {
            // Process next
            this.processQueue();
          });

        return;
      }

      // Cannot execute, stop
      break;
    }
  }

  /**
   * Get current stats
   */
  getStats() {
    return {
      activeCount: this.activeCount,
      queueLength: this.queue.length,
      agentsActive: Array.from(this.activeOps.entries()),
    };
  }
}
```

### Security Model

Create `src/security/Security.ts`:

```typescript
import { z } from "zod";
import * as crypto from "crypto";

/**
 * Authentication token types
 */
export const TokenTypeSchema = z.enum(["agent", "user", "admin"]);
export type TokenType = z.infer<typeof TokenTypeSchema>;

/**
 * Authentication token payload
 */
export interface AuthToken {
  type: TokenType;
  id: string;           // agent_id or user_id
  issued_at: number;     // Unix timestamp
  expires_at: number;    // Unix timestamp
  scopes: string[];       // API scopes
}

/**
 * Security configuration
 */
export interface SecurityConfig {
  jwtSecret: string;
  tokenExpiryHours: {
    agent: number;
    user: number;
    admin: number;
  };
  rateLimits: {
    agent: number;    // requests per minute
    user: number;
    admin: number;
  };
}

/**
 * Security manager
 */
export class SecurityManager {
  constructor(private config: SecurityConfig) {}

  /**
   * Generate authentication token
   */
  generateToken(type: TokenType, id: string, scopes: string[]): string {
    const payload: AuthToken = {
      type,
      id,
      issued_at: Math.floor(Date.now() / 1000),
      expires_at:
        Math.floor(Date.now() / 1000) +
        this.config.tokenExpiryHours[type] * 3600,
      scopes,
    };

    const signature = crypto
      .createHmac("sha256", this.config.jwtSecret)
      .update(JSON.stringify(payload))
      .digest("hex");

    return `${Buffer.from(JSON.stringify(payload)).toString("base64")}.${signature}`;
  }

  /**
   * Validate authentication token
   */
  validateToken(token: string): { valid: boolean; payload?: AuthToken; error?: string } {
    try {
      // Split token
      const [payloadBase64, signature] = token.split(".");
      if (!payloadBase64 || !signature) {
        return { valid: false, error: "Invalid token format" };
      }

      // Decode payload
      const payload: AuthToken = JSON.parse(
        Buffer.from(payloadBase64, "base64").toString()
      );

      // Verify signature
      const expectedSignature = crypto
        .createHmac("sha256", this.config.jwtSecret)
        .update(JSON.stringify(payload))
        .digest("hex");

      if (signature !== expectedSignature) {
        return { valid: false, error: "Invalid signature" };
      }

      // Check expiration
      const now = Math.floor(Date.now() / 1000);
      if (payload.expires_at < now) {
        return { valid: false, error: "Token expired" };
      }

      return { valid: true, payload };
    } catch (error) {
      return { valid: false, error: "Token parse error" };
    }
  }

  /**
   * Check if token has required scope
   */
  hasScope(token: AuthToken, requiredScope: string): boolean {
    return token.scopes.includes(requiredScope) || token.scopes.includes("*");
  }

  /**
   * Authorize request
   */
  authorize(token: string, requiredScope: string): {
    authorized: boolean;
    token?: AuthToken;
    error?: string;
  } {
    const validation = this.validateToken(token);
    if (!validation.valid) {
      return { authorized: false, error: validation.error };
    }

    if (!this.hasScope(validation.payload!, requiredScope)) {
      return { authorized: false, error: "Insufficient scope" };
    }

    return { authorized: true, token: validation.payload };
  }
}
```

### Rate Limiting

Create `src/security/RateLimiter.ts`:

```typescript
/**
 * Rate limit bucket
 */
interface RateLimitBucket {
  count: number;
  resetAt: number;
}

/**
 * Rate limiter using token bucket algorithm
 */
export class RateLimiter {
  private buckets = new Map<string, RateLimitBucket>();

  constructor(
    private limits: Record<string, number>, // scopes -> requests per minute
    private windowMs: number = 60_000 // 1 minute
  ) {}

  /**
   * Check if request is allowed
   */
  check(scope: string, identifier: string): {
    allowed: boolean;
    remaining: number;
    resetAt: number;
  } {
    const key = `${scope}:${identifier}`;
    const now = Date.now();
    const limit = this.limits[scope] || this.limits["default"] || 60;

    // Get or create bucket
    let bucket = this.buckets.get(key);
    if (!bucket || now > bucket.resetAt) {
      bucket = {
        count: 0,
        resetAt: now + this.windowMs,
      };
      this.buckets.set(key, bucket);
    }

    // Check if allowed
    const allowed = bucket.count < limit;
    if (allowed) {
      bucket.count++;
    }

    return {
      allowed,
      remaining: Math.max(0, limit - bucket.count),
      resetAt: bucket.resetAt,
    };
  }

  /**
   * Get rate limit info (for headers)
   */
  getLimit(scope: string): number {
    return this.limits[scope] || this.limits["default"] || 60;
  }

  /**
   * Reset bucket (for testing or admin)
   */
  reset(scope: string, identifier: string): void {
    const key = `${scope}:${identifier}`;
    this.buckets.delete(key);
  }
}
```

### Tests

Create `tests/concurrency/LockManager.test.ts`:

```typescript
import { describe, it, expect, beforeEach } from "bun:test";
import { LockManager } from "../../src/concurrency/LockManager";
import { mkdir, rmdir } from "fs/promises";
import { join } from "path";

const TEST_LOCK_DIR = join(process.cwd(), "test-locks");

describe("LockManager", () => {
  let lm: LockManager;

  beforeEach(async () => {
    await mkdir(TEST_LOCK_DIR, { recursive: true });
    lm = new LockManager({
      lockDir: TEST_LOCK_DIR,
      timeoutMs: 1000,
      retryAttempts: 3,
      retryDelayMs: 10,
    });
  });

  afterEach(async () => {
    await lm.releaseAll();
    await rmdir(TEST_LOCK_DIR, { recursive: true });
  });

  it("should acquire and release lock", async () => {
    const acquisition = await lm.acquire("resource-1");
    expect(acquisition.success).toBe(true);

    await lm.release("resource-1");
    expect(lm.isLocked("resource-1")).toBe(false);
  });

  it("should prevent concurrent lock acquisition", async () => {
    const lock1 = await lm.acquire("resource-1");
    expect(lock1.success).toBe(true);

    const lock2 = await lm.acquire("resource-1");
    expect(lock2.success).toBe(false);

    await lm.release("resource-1");

    const lock3 = await lm.acquire("resource-1");
    expect(lock3.success).toBe(true);
  });

  it("should timeout after configured duration", async () => {
    const lock1 = await lm.acquire("resource-1");
    expect(lock1.success).toBe(true);

    // Try to acquire with short timeout
    const lm2 = new LockManager({
      lockDir: TEST_LOCK_DIR,
      timeoutMs: 50,
      retryAttempts: 1,
      retryDelayMs: 10,
    });
    const lock2 = await lm2.acquire("resource-1");
    expect(lock2.success).toBe(false);
    expect(lock2.error).toContain("timeout");
  });
});
```

---

## Implementation Steps

1. **Implement lock manager**
   - POSIX flock() wrapper
   - Exponential backoff
   - Cleanup on shutdown

2. **Implement optimistic locking**
   - CAS operations
   - Atomic card claiming
   - Retry logic

3. **Implement concurrency controller**
   - Global and per-agent limits
   - Priority queue
   - Queue timeout

4. **Implement security manager**
   - Token generation/validation
   - Scope-based authorization
   - Rate limiting

5. **Write concurrency tests**
   - Race condition tests
   - Lock contention tests
   - Deadlock prevention tests

---

## Deliverables

1. **`src/concurrency/LockManager.ts`**
2. **`src/concurrency/OptimisticLocking.ts`**
3. **`src/concurrency/ConcurrencyController.ts`**
4. **`src/security/Security.ts`**
5. **`src/security/RateLimiter.ts`**
6. **Comprehensive test suites**

---

## Next Phase

After concurrency and security are implemented, proceed to **Phase 06: File Watching Infrastructure** to set up real-time file system event handling.
