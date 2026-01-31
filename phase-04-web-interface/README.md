# Plan 04: Web Interface

## Overview

Implements the web-based Kanban UI for human oversight and manual intervention.

## Phases (6)

| Phase | Name | Description |
|--------|------|-------------|
| 01 | Project Setup | Vue 3 + Vite + Tailwind setup |
| 02 | Kanban Board Layout | Column-based UI with drag-and-drop |
| 03 | Card Management | View, edit, create cards |
| 04 | Real-time Updates | WebSocket integration |
| 05 | Agent Status Panel | Display agent activity |
| 06 | Responsive Design | Mobile and desktop layouts |

## Key Components

```
src/ui/
├── App.vue                # Root component
├── components/
│   ├── KanbanBoard.vue    # Main board
│   ├── KanbanColumn.vue    # Column component
│   ├── KanbanCard.vue      # Card component
│   ├── CardModal.vue       # Card detail modal
│   ├── AgentStatus.vue     # Agent monitoring
│   └── NotificationToast.vue # Alerts
├── composables/
│   ├── useProjects.ts      # Project data
│   ├── useWebSocket.ts     # WebSocket connection
│   └── useDragDrop.ts     # Drag-and-drop logic
└── stores/
    └── projectStore.ts     # Pinia store
```

## UI Structure

```
┌─────────────────────────────────────────────────────────────────┐
│  Header: Agentic Workflow Manager                             │
│  [New Project] [Filter] [Search]                           │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────┬─────────┬─────────────────┬─────────┬─────────┐
│  │  INBOX  │ BACKLOG │     ACTIVE      │ REVIEW  │ COMPLETED│
│  └─────────┴─────────┴─────────────────┴─────────┴─────────┘
│                                                               │
│  Active (grouped by agent):                                    │
│  ┌─────────────┐  ┌─────────────┐                           │
│  │ Agent A (3) │  │ Agent B (2) │                           │
│  │ [Card]      │  │ [Card]      │                           │
│  │ [Card]      │  │ [Card]      │                           │
│  │ [Card]      │  │ [Card]      │                           │
│  └─────────────┘  └─────────────┘                           │
│                                                               │
│  Agent Status Panel:                                            │
│  ┌─────────────┬─────────────┬─────────────┐                │
│  │ Agent A     │ Agent B     │ Agent C     │                │
│  │ ● Online    │ ● Busy (2)  │ ○ Offline   │                │
│  │ Load: 2/5  │ Load: 3/5  │ Load: 0/5  │                │
│  └─────────────┴─────────────┴─────────────┘                │
└─────────────────────────────────────────────────────────────────────┘
```

## Success Criteria

- [ ] Kanban board displays all project states
- [ ] Drag-and-drop between columns triggers state transitions
- [ ] Real-time updates via WebSocket
- [ ] Card modal shows full details with markdown rendering
- [ ] Agent status panel shows live activity
- [ ] Responsive design works on mobile (320px) and desktop (1920px+)

## Dependencies

- Plan 01 (Foundation) - required for types
- Plan 03 (Backend API) - required for REST/WebSocket
- ProjectState, Card, WebSocket API

## Tech Stack

- **Framework**: Vue 3 (Composition API)
- **Build**: Vite
- **Styling**: Tailwind CSS + shadcn/ui
- **State**: Pinia
- **Markdown**: markdown-it
- **Icons**: Lucide Vue

## Next Plan

Plan 05: Agent Interface (Agent protocol implementation)
