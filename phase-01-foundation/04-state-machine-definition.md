# Phase 04: State Machine Definition

## EARS Framework

### Event (Trigger)
- Data types are fully defined
- Need to implement state transition logic
- State machine must enforce valid transitions and trigger side effects

### Action (What We'll Do)
- Implement project state machine with validation
- Implement card state transitions
- Add transition side effects (notifications, logging)
- Create state machine tests
- Document all state rules

### Result (Outcome)
- Fully functional state machine
- All transitions validated against rules
- Side effects triggered automatically
- Comprehensive test coverage

### Success Criteria
- [ ] Invalid transitions are rejected with clear errors
- [ ] Valid transitions update state atomically
- [ ] All transitions emit notifications
- [ ] State changes are logged for audit trail
- [ ] Tests cover all transition paths

---

## Specification

### State Machine Architecture

```
State Machine (src/engine/StateMachine.ts)
├── ProjectStateMachine
│   ├── validateTransition()
│   ├── executeTransition()
│   ├── transitionInboxToBacklog()
│   ├── transitionBacklogToActive()
│   ├── transitionActiveToReview()
│   ├── transitionActiveToCompleted()
│   └── transitionToFailed()
├── CardStateMachine
│   ├── validateCardTransition()
│   ├── executeCardTransition()
│   └── handleDependencyBlocking()
└── TransitionOrchestrator
    ├── checkProjectReadyForTransition()
    ├── updateCardStates()
    └── emitTransitionEvent()
```

### Project State Machine

Create `src/engine/StateMachine.ts`:

```typescript
import {
  ProjectState,
  ProjectId,
  ProjectState as ProjectStateType,
  VALID_TRANSITIONS,
  TransitionRequest,
  TransitionResult,
  TransitionError,
  validateTransition as validateTransitionFn,
} from "../core/types";
import { FileSystem } from "../core/FileSystem";
import { NotificationManager } from "./NotificationManager";
import { Logger } from "../utils/Logger";

/**
 * Project state machine
 *
 * Responsibilities:
 * - Validate state transitions
 * - Execute transitions atomically
 * - Trigger side effects (notifications, logging)
 */
export class ProjectStateMachine {
  constructor(
    private fs: FileSystem,
    private notifications: NotificationManager,
    private logger: Logger
  ) {}

  /**
   * Execute a state transition
   */
  async transition(
    request: TransitionRequest
  ): Promise<TransitionResult> {
    const { project_id, from_state, to_state, reason, force } = request;

    // Log transition attempt
    this.logger.info("Transition attempt", {
      project_id,
      from_state,
      to_state,
      force,
    });

    // Validate transition
    let validationResult: TransitionResult;
    try {
      validationResult = validateTransitionFn(from_state, to_state, force);
    } catch (error) {
      if (error instanceof TransitionError) {
        throw error;
      }
      throw new TransitionError(from_state, to_state, "Unknown validation error");
    }

    // Read current project state
    const readResult = await this.fs.readProject(project_id);
    if (!readResult.success) {
      throw new Error(`Failed to read project: ${readResult.error.message}`);
    }

    const project = readResult.data;

    // Verify from_state matches current state
    if (project.state !== from_state) {
      throw new TransitionError(
        from_state,
        to_state,
        `Project is in ${project.state}, not ${from_state}`
      );
    }

    // Execute specific transition
    const handler = this.getTransitionHandler(from_state, to_state);
    await handler(project, reason);

    // Emit notification
    await this.notifications.emitStateTransition(project_id, from_state, to_state);

    // Log success
    this.logger.info("Transition successful", {
      project_id,
      from_state,
      to_state,
    });

    return validationResult;
  }

  /**
   * Get transition handler for specific transition
   */
  private getTransitionHandler(
    from: ProjectStateType,
    to: ProjectStateType
  ): (project: any, reason?: string) => Promise<void> {
    const handlers: Record<string, (p: any, r?: string) => Promise<void>> = {
      "inbox->backlog": this.transitionInboxToBacklog.bind(this),
      "backlog->active": this.transitionBacklogToActive.bind(this),
      "active->review": this.transitionActiveToReview.bind(this),
      "active->completed": this.transitionActiveToCompleted.bind(this),
      "any->failed": this.transitionToFailed.bind(this),
      "failed->backlog": this.transitionFailedToBacklog.bind(this),
    };

    const key = `${from}->${to}`;
    const handler = handlers[key];

    if (!handler) {
      throw new TransitionError(from, to, "No handler found");
    }

    return handler;
  }

  /**
   * Transition: inbox → backlog
   *
   * Triggers: Valid state.json created
   * Side effects: Initialize project, emit notification
   */
  private async transitionInboxToBacklog(
    project: any,
    reason?: string
  ): Promise<void> {
    // Validate project is ready to move to backlog
    if (!this.isProjectValidForBacklog(project)) {
      throw new Error("Project validation failed");
    }

    // Update state
    project.state = "backlog";
    project.updated_at = new Date().toISOString();

    // Write updated state
    await this.fs.writeProject(project.project_id, project);

    // Move folder from inbox/ to backlog/
    await this.fs.moveProject(
      project.project_id,
      "inbox",
      "backlog"
    );

    this.logger.info(`Project ${project.project_id} moved to backlog`, {
      reason,
    });
  }

  /**
   * Transition: backlog → active
   *
   * Triggers: Agent available, dependencies met
   * Side effects: Assign agent, start cards
   */
  private async transitionBacklogToActive(
    project: any,
    reason?: string
  ): Promise<void> {
    // Validate project can become active
    if (!this.areDependenciesMet(project)) {
      throw new Error("Dependencies not met");
    }

    // Update state
    project.state = "active";
    project.updated_at = new Date().toISOString();

    // Write updated state
    await this.fs.writeProject(project.project_id, project);

    // Move folder from backlog/ to active/unassigned/
    await this.fs.moveProject(
      project.project_id,
      "backlog",
      "active"
    );

    this.logger.info(`Project ${project.project_id} moved to active`, {
      reason,
    });
  }

  /**
   * Transition: active → review
   *
   * Triggers: All cards completed, checkpoint_required=true
   * Side effects: Emit review notification, wait for human approval
   */
  private async transitionActiveToReview(
    project: any,
    reason?: string
  ): Promise<void> {
    // Validate all cards completed
    if (!this.allCardsCompleted(project)) {
      throw new Error("Not all cards completed");
    }

    if (!project.checkpoint_required) {
      throw new Error("Checkpoint not required, should go to completed");
    }

    // Update state
    project.state = "review";
    project.updated_at = new Date().toISOString();

    // Write updated state
    await this.fs.writeProject(project.project_id, project);

    // Move folder from active/ to review/
    await this.fs.moveProject(
      project.project_id,
      "active",
      "review"
    );

    this.logger.info(`Project ${project.project_id} moved to review`, {
      reason,
    });
  }

  /**
   * Transition: active → completed
   *
   * Triggers: All cards completed, checkpoint_required=false
   * Side effects: Emit completion notification, archive
   */
  private async transitionActiveToCompleted(
    project: any,
    reason?: string
  ): Promise<void> {
    // Validate all cards completed
    if (!this.allCardsCompleted(project)) {
      throw new Error("Not all cards completed");
    }

    if (project.checkpoint_required && !project.checkpoint_approved) {
      throw new Error("Checkpoint required, go to review");
    }

    // Update state
    project.state = "completed";
    project.updated_at = new Date().toISOString();

    // Write updated state
    await this.fs.writeProject(project.project_id, project);

    // Move folder from active/ to completed/YYYY/MM/
    const now = new Date();
    const archivePath = `completed/${now.getFullYear()}/${now.getMonth() + 1}`;
    await this.fs.moveProject(
      project.project_id,
      "active",
      archivePath as any
    );

    this.logger.info(`Project ${project.project_id} completed`, {
      reason,
    });
  }

  /**
   * Transition: any → failed
   *
   * Triggers: Card failed, retry_max exceeded
   * Side effects: Log error, emit alert, move to failed/
   */
  private async transitionToFailed(
    project: any,
    reason?: string
  ): Promise<void> {
    // Update state
    project.state = "failed";
    project.updated_at = new Date().toISOString();

    // Write updated state
    await this.fs.writeProject(project.project_id, project);

    // Move folder from current state to failed/
    await this.fs.moveProject(
      project.project_id,
      project.state as any,
      "failed"
    );

    // Write error log
    const errorLog = `Error: ${reason || "Unknown error"}\nTimestamp: ${new Date().toISOString()}\nProject ID: ${project.project_id}`;
    await this.fs.writeFile(
      `failed/${project.project_id}/error.log`,
      errorLog
    );

    this.logger.error(`Project ${project.project_id} failed`, {
      reason,
    });
  }

  /**
   * Transition: failed → backlog
   *
   * Triggers: Manual retry command
   * Side effects: Reset retry counters, re-queue
   */
  private async transitionFailedToBacklog(
    project: any,
    reason?: string
  ): Promise<void> {
    // Reset retry counters
    project.retry_count = 0;
    for (const card of project.cards) {
      card.retry_count = 0;
      if (card.state === "failed") {
        card.state = "pending";
      }
    }

    // Update state
    project.state = "backlog";
    project.updated_at = new Date().toISOString();

    // Write updated state
    await this.fs.writeProject(project.project_id, project);

    // Move folder from failed/ to backlog/
    await this.fs.moveProject(
      project.project_id,
      "failed",
      "backlog"
    );

    this.logger.info(`Project ${project.project_id} requeued to backlog`, {
      reason,
    });
  }

  /**
   * Check if project is valid for backlog
   */
  private isProjectValidForBacklog(project: any): boolean {
    // Required fields present
    if (!project.project_id || !project.project_name) {
      return false;
    }

    // Valid priority
    if (project.priority < 1 || project.priority > 5) {
      return false;
    }

    // Valid state
    if (project.state !== "inbox") {
      return false;
    }

    return true;
  }

  /**
   * Check if all dependencies are met
   */
  private areDependenciesMet(project: any): boolean {
    // Build dependency graph
    const cardMap = new Map(project.cards.map((c: any) => [c.card_id, c]));

    // Check each card's dependencies
    for (const card of project.cards) {
      for (const depId of card.dependencies) {
        const depCard = cardMap.get(depId);
        if (!depCard || depCard.state !== "completed") {
          return false;
        }
      }
    }

    return true;
  }

  /**
   * Check if all cards completed
   */
  private allCardsCompleted(project: any): boolean {
    if (project.cards.length === 0) {
      return true;
    }

    return project.cards.every((card: any) => card.state === "completed");
  }
}
```

### Card State Machine

```typescript
import {
  CardState,
  CardId,
  CardState as CardStateType,
} from "../core/types";

/**
 * Card state machine
 *
 * Responsibilities:
 * - Validate card state transitions
 * - Handle dependency blocking
 * - Track retry counts
 */
export class CardStateMachine {
  /**
   * Valid card state transitions
   */
  private static readonly TRANSITIONS: Record<CardStateType, CardStateType[]> = {
    pending: ["in-progress", "blocked"],
    "in-progress": ["completed", "failed"],
    completed: [], // Terminal state
    failed: ["pending"], // Retry
    blocked: ["pending"], // Retry after unblocking
  };

  /**
   * Validate card state transition
   */
  validateTransition(
    from: CardStateType,
    to: CardStateType
  ): TransitionResult {
    const allowed = CardStateMachine.TRANSITIONS[from];

    if (!allowed.includes(to)) {
      throw new TransitionError(from, to, "Invalid card state transition");
    }

    return {
      success: true,
      from_state: from,
      to_state: to,
      timestamp: new Date().toISOString(),
    };
  }

  /**
   * Check if card should be blocked by dependencies
   */
  shouldBlock(
    cardId: CardId,
    projectCards: any[]
  ): { blocked: boolean; reason?: string } {
    const card = projectCards.find((c) => c.card_id === cardId);
    if (!card) {
      throw new Error(`Card not found: ${cardId}`);
    }

    // Check dependencies
    for (const depId of card.dependencies) {
      const depCard = projectCards.find((c) => c.card_id === depId);

      if (!depCard) {
        return { blocked: true, reason: `Dependency ${depId} not found` };
      }

      if (depCard.state !== "completed") {
        return {
          blocked: true,
          reason: `Dependency ${depId} is ${depCard.state}`,
        };
      }
    }

    return { blocked: false };
  }

  /**
   * Find all cards ready to execute (unblocked, pending)
   */
  findReadyCards(projectCards: any[]): CardId[] {
    const ready: CardId[] = [];

    for (const card of projectCards) {
      if (card.state !== "pending") {
        continue;
      }

      const { blocked } = this.shouldBlock(card.card_id, projectCards);
      if (!blocked) {
        ready.push(card.card_id);
      }
    }

    return ready;
  }
}
```

### Transition Orchestrator

```typescript
import { ProjectId, CardId } from "../core/types";
import { ProjectStateMachine } from "./StateMachine";
import { CardStateMachine } from "./CardStateMachine";

/**
 * Orchestrates state transitions
 *
 * Responsibilities:
 * - Coordinate project and card transitions
 * - Check readiness for transitions
 * - Handle dependency resolution
 */
export class TransitionOrchestrator {
  constructor(
    private projectStateMachine: ProjectStateMachine,
    private cardStateMachine: CardStateMachine
  ) {}

  /**
   * Check if project is ready to transition to specified state
   */
  async checkProjectReady(
    projectId: ProjectId,
    targetState: string
  ): Promise<{ ready: boolean; reason?: string }> {
    // Read project state
    const readResult = await this.fs.readProject(projectId);
    if (!readResult.success) {
      return { ready: false, reason: "Failed to read project" };
    }

    const project = readResult.data;

    // Check specific readiness conditions
    switch (targetState) {
      case "active":
        return this.checkReadyForActive(project);
      case "review":
        return this.checkReadyForReview(project);
      case "completed":
        return this.checkReadyForCompleted(project);
      default:
        return { ready: false, reason: "Unknown target state" };
    }
  }

  /**
   * Check if project ready to become active
   */
  private checkReadyForActive(project: any): { ready: boolean; reason?: string } {
    // Must have at least one pending card
    const pendingCards = project.cards.filter((c: any) => c.state === "pending");
    if (pendingCards.length === 0) {
      return { ready: false, reason: "No pending cards" };
    }

    // Dependencies must be met
    if (!this.projectStateMachine.areDependenciesMet(project)) {
      return { ready: false, reason: "Dependencies not met" };
    }

    return { ready: true };
  }

  /**
   * Check if project ready for review
   */
  private checkReadyForReview(project: any): { ready: boolean; reason?: string } {
    // Must require checkpoint
    if (!project.checkpoint_required) {
      return { ready: false, reason: "Checkpoint not required" };
    }

    // All cards must be completed
    if (!this.projectStateMachine.allCardsCompleted(project)) {
      return { ready: false, reason: "Not all cards completed" };
    }

    return { ready: true };
  }

  /**
   * Check if project ready for completion
   */
  private checkReadyForCompleted(project: any): { ready: boolean; reason?: string } {
    // All cards must be completed
    if (!this.projectStateMachine.allCardsCompleted(project)) {
      return { ready: false, reason: "Not all cards completed" };
    }

    // Checkpoint must not be required, or must be approved
    if (project.checkpoint_required && !project.checkpoint_approved) {
      return { ready: false, reason: "Checkpoint not approved" };
    }

    return { ready: true };
  }

  /**
   * Update card states based on project state
   */
  async updateCardStates(projectId: ProjectId): Promise<void> {
    const readResult = await this.fs.readProject(projectId);
    if (!readResult.success) {
      throw new Error(`Failed to read project: ${readResult.error.message}`);
    }

    const project = readResult.data;
    let changed = false;

    // Check each card for blocking/unblocking
    for (const card of project.cards) {
      const { blocked, reason } = this.cardStateMachine.shouldBlock(
        card.card_id,
        project.cards
      );

      if (blocked && card.state !== "blocked") {
        card.state = "blocked";
        changed = true;
      } else if (!blocked && card.state === "blocked") {
        card.state = "pending";
        changed = true;
      }
    }

    // Write if changed
    if (changed) {
      await this.fs.writeProject(projectId, project);
    }
  }
}
```

### Tests

Create `tests/engine/StateMachine.test.ts`:

```typescript
import { describe, it, expect } from "bun:test";
import { ProjectStateMachine } from "../../src/engine/StateMachine";
import { FileSystem } from "../../src/core/FileSystem";
import { NotificationManager } from "../../src/engine/NotificationManager";
import { Logger } from "../../src/utils/Logger";

describe("ProjectStateMachine", () => {
  let sm: ProjectStateMachine;

  beforeEach(() => {
    const fs = new FileSystem("test-workspace");
    const notifications = new NotificationManager(fs);
    const logger = new Logger("test");
    sm = new ProjectStateMachine(fs, notifications, logger);
  });

  describe("transition", () => {
    it("should validate inbox → backlog transition", async () => {
      const request = {
        project_id: "test-id",
        from_state: "inbox",
        to_state: "backlog",
        force: false,
      };

      const result = await sm.transition(request);
      expect(result.success).toBe(true);
    });

    it("should reject invalid transition", async () => {
      const request = {
        project_id: "test-id",
        from_state: "inbox",
        to_state: "completed", // Skip steps
        force: false,
      };

      await expect(sm.transition(request)).rejects.toThrow(
        "Transition not allowed"
      );
    });

    it("should allow forced transition", async () => {
      const request = {
        project_id: "test-id",
        from_state: "inbox",
        to_state: "completed",
        force: true, // Admin override
      };

      const result = await sm.transition(request);
      expect(result.success).toBe(true);
    });
  });
});
```

---

## Implementation Steps

1. **Create state machine classes**
   - ProjectStateMachine
   - CardStateMachine
   - TransitionOrchestrator

2. **Implement transition handlers**
   - Each transition has specific logic
   - Validate preconditions
   - Update state atomically
   - Move folders

3. **Add notification integration**
   - Emit events on transitions
   - Log all state changes

4. **Write comprehensive tests**
   - Test all valid transitions
   - Test invalid transitions are rejected
   - Test forced transitions
   - Test side effects

5. **Document state machine rules**
   - State diagrams
   - Transition tables
   - Side effect documentation

---

## Deliverables

1. **`src/engine/StateMachine.ts`** - Project state machine
2. **`src/engine/CardStateMachine.ts`** - Card state machine
3. **`src/engine/TransitionOrchestrator.ts`** - Transition coordinator
4. **`tests/engine/StateMachine.test.ts`** - Test suite
5. **`docs/STATE-MACHINE.md`** - State machine documentation

---

## Next Phase

After state machine is complete, proceed to **Phase 05: Concurrency & Security Design** to implement file locking, race condition prevention, and security measures.
