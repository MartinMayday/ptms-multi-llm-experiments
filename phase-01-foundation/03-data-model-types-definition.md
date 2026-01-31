# Phase 03: Data Model & Types Definition

## EARS Framework

### Event (Trigger)
- File system layer is complete
- Need to define TypeScript types for all domain objects
- Validation schemas required for API and data layer

### Action (What We'll Do)
- Define TypeScript interfaces for all domain types
- Create Zod schemas for validation
- Generate types from schemas
- Document type relationships and constraints
- Create utility types for common patterns

### Result (Outcome)
- Complete type system for entire project
- Runtime validation with Zod schemas
- Type-safe file I/O with schema validation
- Comprehensive type documentation

### Success Criteria
- [ ] All domain objects have TypeScript interfaces
- [ ] Zod schemas match TypeScript types exactly
- [ ] Runtime validation catches invalid data
- [ ] No `any` types in business logic layer
- [ ] Type documentation covers all relationships

---

## Specification

### Type System Architecture

```
Domain Types (src/core/types/)
├── Project.ts          - Project and ProjectState types
├── Card.ts             - Card and CardState types
├── Agent.ts            - Agent manifest and status types
├── Notification.ts      - Notification event types
├── Transition.ts       - State transition types
├── ValidationError.ts   - Validation error types
└── index.ts            - Public exports
```

### Project Types

Create `src/core/types/Project.ts`:

```typescript
import { z } from "zod";

/**
 * Unique identifier for a project
 */
export type ProjectId = string & { readonly __brand: unique symbol };
export const ProjectIdSchema = z.string().uuid();

/**
 * Create a ProjectId from string (runtime validation)
 */
export function createProjectId(id: string): ProjectId {
  const result = ProjectIdSchema.safeParse(id);
  if (!result.success) {
    throw new Error(`Invalid project ID: ${id}`);
  }
  return id as ProjectId;
}

/**
 * Project state machine states
 */
export const ProjectStateSchema = z.enum([
  "inbox",
  "backlog",
  "active",
  "review",
  "completed",
  "failed",
]);
export type ProjectState = z.infer<typeof ProjectStateSchema>;

/**
 * Valid state transitions
 * Key = from state, Value = array of allowed to states
 */
export const VALID_TRANSITIONS: Record<ProjectState, ProjectState[]> = {
  inbox: ["backlog", "failed"],
  backlog: ["active", "failed"],
  active: ["review", "completed", "failed"],
  review: ["completed", "active", "failed"],
  completed: [], // Terminal state
  failed: ["backlog"], // Can retry
};

/**
 * Priority levels (1 = highest)
 */
export const PrioritySchema = z.number().int().min(1).max(5);
export type Priority = z.infer<typeof PrioritySchema>;

/**
 * Project state file (state.json)
 */
export const ProjectStateSchema = z.object({
  schema_version: z.literal("1.0"),
  project_id: ProjectIdSchema,
  project_name: z.string().min(1).max(255),

  state: ProjectStateSchema,
  created_at: z.string().datetime(),
  updated_at: z.string().datetime(),

  assignee_agent: z.string().uuid().nullable(),
  checkpoint_required: z.boolean(),
  checkpoint_approved: z.boolean().nullable(),

  retry_count: z.number().int().min(0),

  cards_total: z.number().int().min(0),
  cards_completed: z.number().int().min(0),
  cards_failed: z.number().int().min(0),

  cards: z.array(CardStateSchema),
});

export type ProjectState = z.infer<typeof ProjectStateSchema>;

/**
 * Project metadata (from project.md)
 */
export interface ProjectMetadata {
  title: string;
  description: string;
  created_by: string;
  tags: string[];
  estimated_completion: string | null; // ISO8601 datetime or null
}

/**
 * Complete project (state + metadata)
 */
export interface Project {
  state: ProjectState;
  metadata: ProjectMetadata;
}
```

### Card Types

Create `src/core/types/Card.ts`:

```typescript
import { z } from "zod";
import { ProjectId } from "./Project";

/**
 * Unique identifier for a card
 */
export type CardId = string & { readonly __brand: unique symbol };
export const CardIdSchema = z.string().uuid();

/**
 * Card types
 */
export const CardTypeSchema = z.enum([
  "task",
  "meta",
  "checkpoint",
  "notification",
]);
export type CardType = z.infer<typeof CardTypeSchema>;

/**
 * Card state
 */
export const CardStateSchema = z.enum([
  "pending",
  "in-progress",
  "completed",
  "failed",
  "blocked",
]);
export type CardState = z.infer<typeof CardStateSchema>;

/**
 * Agent profile types (agent capabilities)
 */
export const AgentProfileSchema = z.enum([
  "researcher",
  "coder",
  "analyst",
  "general",
]);
export type AgentProfile = z.infer<typeof AgentProfileSchema>;

/**
 * Failure handling strategy
 */
export const FailureHandlingSchema = z.enum([
  "retry",
  "escalate",
  "notify",
]);
export type FailureHandling = z.infer<typeof FailureHandlingSchema>;

/**
 * Card frontmatter schema
 */
export const CardFrontmatterSchema = z.object({
  card_id: CardIdSchema,
  type: CardTypeSchema,
  project_id: ProjectIdSchema,
  sequence: z.number().int().min(1),
  state: CardStateSchema,
  priority: z.number().int().min(1).max(5),
  created: z.string().datetime(),
  started: z.string().datetime().nullable(),
  completed: z.string().datetime().nullable(),
  due: z.string().datetime().nullable(),
  assignee: z.string().uuid().nullable(),
  atomic: z.boolean(),
  estimated_duration: z.string().regex(/^\d+[mhd]$/), // e.g., "15m", "2h", "1d"
  max_duration_minutes: z.number().int().min(1),
  tags: z.array(z.string().max(50)),
  dependencies: z.array(CardIdSchema).default([]),
  outputs_expected: z.array(z.string()).default([]),
  failure_handling: FailureHandlingSchema,
  retry_count: z.number().int().min(0).default(0),
  retry_max: z.number().int().min(0).default(3),
  agent_profile: AgentProfileSchema,
  checkpoint_after: z.boolean(),
});

export type CardFrontmatter = z.infer<typeof CardFrontmatterSchema>;

/**
 * Card state (embedded in project state.json)
 */
export const EmbeddedCardStateSchema = z.object({
  card_id: CardIdSchema,
  filename: z.string().min(1), // e.g., "001-research.md"
  state: CardStateSchema,
  assignee: z.string().uuid().nullable(),
  started_at: z.string().datetime().nullable(),
  completed_at: z.string().datetime().nullable(),
  failed_at: z.string().datetime().nullable(),
  retry_count: z.number().int().min(0),
  dependencies: z.array(CardIdSchema).default([]),
  checkpoint_after: z.boolean(),
});

export type EmbeddedCardState = z.infer<typeof EmbeddedCardStateSchema>;

/**
 * Complete card (frontmatter + markdown content)
 */
export interface Card {
  frontmatter: CardFrontmatter;
  content: string;
  path: string;
}

/**
 * Parse card from markdown string
 */
export function parseCard(markdown: string): Card {
  const frontmatterMatch = markdown.match(/^---\n([\s\S]+?)\n---\n([\s\S]*)$/);

  if (!frontmatterMatch) {
    throw new Error("Card must have YAML frontmatter delimited by ---");
  }

  const [_, frontmatterYaml, content] = frontmatterMatch;
  const frontmatter = CardFrontmatterSchema.parse(parseYaml(frontmatterYaml));

  return {
    frontmatter,
    content: content.trim(),
    path: "", // Set by caller
  };
}

/**
 * Serialize card to markdown string
 */
export function serializeCard(card: Card): string {
  const yaml = stringifyYaml(card.frontmatter);
  return `---\n${yaml}---\n\n${card.content}`;
}
```

### Agent Types

Create `src/core/types/Agent.ts`:

```typescript
import { z } from "zod";
import { AgentProfile } from "./Card";

/**
 * Unique identifier for an agent
 */
export type AgentId = string & { readonly __brand: unique symbol };
export const AgentIdSchema = z.string().regex(/^agent-[a-z0-9-]+$/);

/**
 * Agent capabilities
 */
export const CapabilitySchema = z.enum([
  "web-search",
  "document-analysis",
  "code-generation",
  "code-review",
  "data-processing",
  "api-integration",
  "text-generation",
  "image-generation",
]);
export type Capability = z.infer<typeof CapabilitySchema>;

/**
 * Agent manifest (agent registration)
 */
export const AgentManifestSchema = z.object({
  agent_id: AgentIdSchema,
  name: z.string().min(1).max(100),
  description: z.string().min(1).max(500),
  capabilities: z.array(CapabilitySchema),
  profiles: z.array(AgentProfileSchema),
  max_concurrent: z.number().int().min(1).max(10),
  timeout_default: z.number().int().min(1).max(120), // minutes
  version: z.string().regex(/^\d+\.\d+\.\d+$/),
  status: z.enum(["online", "offline", "busy"]),
  last_seen: z.string().datetime(),
});

export type AgentManifest = z.infer<typeof AgentManifestSchema>;

/**
 * Agent current status
 */
export interface AgentStatus {
  agent_id: AgentId;
  status: "online" | "offline" | "busy";
  current_load: number; // Number of active tasks
  current_tasks: string[]; // Card IDs
}

/**
 * Agent task claim
 */
export interface AgentClaim {
  agent_id: AgentId;
  card_id: string;
  claimed_at: string; // ISO8601
  expires_at: string; // ISO8601 (timeout)
}
```

### Notification Types

Create `src/core/types/Notification.ts`:

```typescript
import { z } from "zod";
import { ProjectId } from "./Project";
import { CardId } from "./Card";

/**
 * Notification ID
 */
export type NotificationId = string & { readonly __brand: unique symbol };
export const NotificationIdSchema = z.string().uuid();

/**
 * Notification types
 */
export const NotificationTypeSchema = z.enum([
  "project_created",
  "project_started",
  "project_completed",
  "project_failed",
  "card_claimed",
  "card_completed",
  "card_failed",
  "card_timeout",
  "review_required",
  "agent_offline",
  "agent_error",
]);
export type NotificationType = z.infer<typeof NotificationTypeSchema>;

/**
 * Notification severity
 */
export const SeveritySchema = z.enum(["info", "warning", "error", "critical"]);
export type Severity = z.infer<typeof SeveritySchema>;

/**
 * Notification schema
 */
export const NotificationSchema = z.object({
  notification_id: NotificationIdSchema,
  timestamp: z.string().datetime(),
  type: NotificationTypeSchema,
  project_id: ProjectIdSchema,
  card_id: CardIdSchema.nullable(),
  agent_id: z.string().uuid().nullable(),

  message: z.string().min(1).max(1000),
  severity: SeveritySchema,

  // Actionable items
  actions: z.array(
    z.object({
      label: z.string().min(1).max(50),
      url: z.string().url().nullable(),
      command: z.string().nullable(),
    })
  ).default([]),

  // Additional context
  metadata: z.record(z.any()).default({}),
});

export type Notification = z.infer<typeof NotificationSchema>;

/**
 * Notification delivery channels
 */
export type NotificationChannel = "websocket" | "webhook" | "email" | "file";

/**
 * Delivery result
 */
export interface NotificationDeliveryResult {
  channel: NotificationChannel;
  success: boolean;
  error?: string;
  delivered_at?: string;
}
```

### State Transition Types

Create `src/core/types/Transition.ts`:

```typescript
import { z } from "zod";
import { ProjectId } from "./Project";
import { ProjectState } from "./Project";
import { CardId } from "./Card";

/**
 * State transition request
 */
export const TransitionRequestSchema = z.object({
  project_id: ProjectIdSchema,
  from_state: z.custom<ProjectState>(),
  to_state: z.custom<ProjectState>(),
  reason: z.string().optional(),
  force: z.boolean().default(false), // Admin override
});

export type TransitionRequest = z.infer<typeof TransitionRequestSchema>;

/**
 * State transition result
 */
export interface TransitionResult {
  success: boolean;
  from_state: ProjectState;
  to_state: ProjectState;
  reason?: string;
  timestamp: string;
}

/**
 * Transition validation error
 */
export class TransitionError extends Error {
  constructor(
    public from_state: ProjectState,
    public to_state: ProjectState,
    reason: string
  ) {
    super(`Invalid transition ${from_state} → ${to_state}: ${reason}`);
    this.name = "TransitionError";
  }
}

/**
 * Validate state transition
 */
export function validateTransition(
  from: ProjectState,
  to: ProjectState,
  force: boolean = false
): TransitionResult {
  // Admin force override
  if (force) {
    return {
      success: true,
      from_state: from,
      to_state: to,
      reason: "Force transition by admin",
      timestamp: new Date().toISOString(),
    };
  }

  // Check if transition is valid
  const allowedStates = VALID_TRANSITIONS[from];
  if (!allowedStates.includes(to)) {
    throw new TransitionError(from, to, "Transition not allowed");
  }

  return {
    success: true,
    from_state: from,
    to_state: to,
    timestamp: new Date().toISOString(),
  };
}
```

### Validation Error Types

Create `src/core/types/ValidationError.ts`:

```typescript
/**
 * Validation error codes
 */
export enum ErrorCode {
  // Project errors
  INVALID_PROJECT_ID = "INVALID_PROJECT_ID",
  INVALID_PROJECT_STATE = "INVALID_PROJECT_STATE",
  INVALID_TRANSITION = "INVALID_TRANSITION",

  // Card errors
  INVALID_CARD_ID = "INVALID_CARD_ID",
  INVALID_CARD_STATE = "INVALID_CARD_STATE",
  CIRCULAR_DEPENDENCY = "CIRCULAR_DEPENDENCY",
  UNMET_DEPENDENCY = "UNMET_DEPENDENCY",

  // Agent errors
  INVALID_AGENT_ID = "INVALID_AGENT_ID",
  AGENT_OFFLINE = "AGENT_OFFLINE",
  AGENT_OVERLOAD = "AGENT_OVERLOAD",

  // File system errors
  LOCK_TIMEOUT = "LOCK_TIMEOUT",
  FILE_CORRUPTED = "FILE_CORRUPTED",
  WRITE_FAILED = "WRITE_FAILED",

  // General errors
  VALIDATION_ERROR = "VALIDATION_ERROR",
  INTERNAL_ERROR = "INTERNAL_ERROR",
}

/**
 * Validation error details
 */
export interface ValidationError {
  code: ErrorCode;
  message: string;
  field?: string;
  value?: any;
}

/**
 * Create validation error
 */
export function createValidationError(
  code: ErrorCode,
  message: string,
  field?: string,
  value?: any
): ValidationError {
  return { code, message, field, value };
}

/**
 * Check if error is recoverable
 */
export function isRecoverableError(error: ValidationError): boolean {
  const recoverableCodes = [
    ErrorCode.LOCK_TIMEOUT,
    ErrorCode.AGENT_OFFLINE,
    ErrorCode.AGENT_OVERLOAD,
  ];
  return recoverableCodes.includes(error.code);
}
```

### Utility Types

Create `src/core/types/Utility.ts`:

```typescript
/**
 * Result type for fallible operations (like Rust's Result<T, E>)
 */
export type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E };

/**
 * Extract success type from Result
 */
export type SuccessType<R> = R extends Result<infer T, any> ? T : never;

/**
 * Extract error type from Result
 */
export type ErrorType<R> = R extends Result<any, infer E> ? E : never;

/**
 * Async result type
 */
export type AsyncResult<T, E = Error> = Promise<Result<T, E>>;

/**
 * Deep partial type (all properties optional, including nested)
 */
export type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object
    ? DeepPartial<T[P]>
    : T[P];
};

/**
 * Extract keys by value type
 */
export type KeysByType<T, V> = {
  [K in keyof T]: T[K] extends V ? K : never;
}[keyof T];

/**
 * Require at least one of specified keys
 */
export type RequireAtLeastOne<T, Keys extends keyof T = keyof T> = T &
  Required<Pick<T, Pick<Keys, T[Keys]>>>;

/**
 * Pick optional keys
 */
export type OptionalKeys<T> = {
  [K in keyof T]-?: {} extends Pick<T, K> ? K : never;
}[keyof T];

/**
 * Pick required keys
 */
export type RequiredKeys<T> = {
  [K in keyof T]-?: {} extends Pick<T, K> ? never : K;
}[keyof T];
```

### Public Exports

Create `src/core/types/index.ts`:

```typescript
// Project types
export * from "./Project";

// Card types
export * from "./Card";

// Agent types
export * from "./Agent";

// Notification types
export * from "./Notification";

// Transition types
export * from "./Transition";

// Validation error types
export * from "./ValidationError";

// Utility types
export * from "./Utility";
```

### Type Documentation

Create `docs/TYPES.md`:

```markdown
# Type System Documentation

## Type Hierarchy

```
Project
├── ProjectState (state.json)
│   ├── schema_version
│   ├── project_id (UUID)
│   ├── state (enum: inbox|backlog|active|review|completed|failed)
│   ├── cards[]
│   │   └── EmbeddedCardState
│   │       ├── card_id (UUID)
│   │       ├── state (enum: pending|in-progress|completed|failed|blocked)
│   │       └── dependencies (CardId[])
│   └── ...metadata
└── ProjectMetadata (project.md)

Card
├── CardFrontmatter (YAML)
│   ├── card_id (UUID)
│   ├── type (enum: task|meta|checkpoint|notification)
│   ├── state (enum)
│   ├── priority (1-5)
│   ├── atomic (boolean)
│   ├── max_duration_minutes (number)
│   └── ...metadata
└── Content (Markdown body)

Agent
├── AgentManifest (registration)
│   ├── agent_id (pattern: agent-*)
│   ├── capabilities (Capability[])
│   └── profiles (AgentProfile[])
└── AgentStatus (runtime)

Notification
├── notification_id (UUID)
├── type (enum)
├── severity (enum: info|warning|error|critical)
└── actions (Action[])
```

## Type Validation

All types have corresponding Zod schemas for runtime validation:

- `ProjectStateSchema` - Validates state.json
- `CardFrontmatterSchema` - Validates card YAML frontmatter
- `AgentManifestSchema` - Validates agent registration
- `NotificationSchema` - Validates notification payload

## State Machine Rules

### Project State Transitions

```
inbox → backlog → active → review → completed
  ↓        ↓        ↓         ↓
 failed ←───────────────────────┘
```

### Card State Transitions

```
pending → in-progress → completed
   ↓          ↓
 blocked ←─────────┘
   ↓
 failed
```

## Branding for Type Safety

Critical IDs use branded types to prevent accidental mixing:

```typescript
type ProjectId = string & { readonly __brand: unique symbol };
type CardId = string & { readonly __brand: unique symbol };
type AgentId = string & { readonly __brand: unique symbol };
type NotificationId = string & { readonly __brand: unique symbol };
```

This prevents assigning a CardId to a ProjectId variable at compile time.
```

---

## Implementation Steps

1. **Create type files**
   - Create `src/core/types/` directory
   - Create each type file with Zod schemas
   - Export types via index.ts

2. **Generate types from schemas**
   - Use Zod's `z.infer<>` to generate TypeScript types
   - Ensure all types are exported

3. **Create utility functions**
   - Helper functions for parsing/serializing cards
   - Transition validation function
   - Error creation helpers

4. **Write tests**
   - Test type validation with invalid data
   - Test branded type safety
   - Test transition validation

5. **Document types**
   - Create comprehensive type documentation
   - Document state machine rules
   - Document type relationships

---

## Deliverables

1. **Type files** in `src/core/types/`:
   - Project.ts
   - Card.ts
   - Agent.ts
   - Notification.ts
   - Transition.ts
   - ValidationError.ts
   - Utility.ts
   - index.ts

2. **Type documentation** at `docs/TYPES.md`

3. **Type tests** at `tests/core/types/`

---

## Next Phase

After types are defined, proceed to **Phase 04: State Machine Definition** to implement the state transition logic with validation.
