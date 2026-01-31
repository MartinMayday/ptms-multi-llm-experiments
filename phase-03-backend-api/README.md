# Plan 03: Backend API

## Overview

Implements REST API endpoints, WebSocket server, and agent protocol for system interaction.

## Phases (6)

| Phase | Name | Description |
|--------|------|-------------|
| 01 | API Design & Framework Setup | Hono framework setup, route structure |
| 02 | Project Endpoints | CRUD operations for projects |
| 03 | Card Endpoints | Card operations, claiming, completion |
| 04 | WebSocket Implementation | Real-time updates, event streaming |
| 05 | Agent Protocol | Agent-specific endpoints, authentication |
| 06 | API Documentation | OpenAPI spec, interactive docs |

## Key Components

```
src/api/
├── server.ts              # Hono HTTP server
├── routes/
│   ├── projects.ts        # Project endpoints
│   ├── cards.ts          # Card endpoints
│   ├── agents.ts         # Agent protocol
│   └── admin.ts         # Admin operations
├── middleware/
│   ├── auth.ts           # JWT validation
│   ├── rateLimit.ts      # Rate limiting
│   └── error.ts         # Error handling
└── websocket/
    └── server.ts         # WebSocket server
```

## API Endpoints

```
Projects:
  GET    /api/projects              # List all projects
  GET    /api/projects/:id         # Get project details
  POST   /api/projects              # Create new project
  PUT    /api/projects/:id/state   # Update state
  DELETE /api/projects/:id         # Delete project

Cards:
  GET    /api/projects/:id/cards   # List project cards
  GET    /api/cards/:id           # Get card content
  PUT    /api/cards/:id           # Update card
  POST   /api/cards/:id/claim    # Agent claims task
  POST   /api/cards/:id/complete  # Agent completes task
  POST   /api/cards/:id/fail     # Agent fails task

Agent Protocol:
  GET    /api/agents/status       # Agent status
  POST   /api/agents/register    # Register agent
  POST   /api/agents/heartbeat  # Keep-alive

WebSocket:
  ws://localhost:3000/ws         # WebSocket endpoint
  - projects/:id                  # Project-specific updates
  - global:state                  # System-wide state changes
  - notifications                  # Push notifications
```

## Success Criteria

- [ ] All REST endpoints functional with proper error handling
- [ ] WebSocket server delivers real-time updates
- [ ] Agent authentication with JWT tokens
- [ ] Rate limiting prevents abuse
- [ ] OpenAPI documentation complete

## Dependencies

- Plan 01 (Foundation) - required
- Plan 02 (Workflow Engine) - required for task execution
- StateMachine, FileSystem, SecurityManager

## Next Plan

Plan 04: Web Interface (Kanban UI)
