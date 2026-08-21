# ApexTask — Phase 2 Detailed Plan: Views Engine & Productivity Tools

**Duration:** Weeks 4–6
**Prerequisite:** Phase 1 complete (auth, task CRUD, list view, filtering/sorting all working)
**Goal:** Expand the single List view into a full multi-view engine, and add the daily-workflow tools that make the app sticky enough to open every day.

---

## Week 4 — Kanban Board View

### 1. Data model additions
Kanban needs a concept of "column" that's more flexible than the fixed `TaskStatus` enum (TODO/IN_PROGRESS/DONE) — teams want custom columns like "Backlog," "In Review," "Blocked."

```prisma
model Board {
  id          String   @id @default(cuid())
  workspaceId String
  name        String   @default("Board")
  columns     BoardColumn[]
  workspace   Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
}

model BoardColumn {
  id        String  @id @default(cuid())
  boardId   String
  name      String
  color     String  @default("#6B6862")
  position  Int
  board     Board   @relation(fields: [boardId], references: [id], onDelete: Cascade)
  tasks     Task[]
}
```

- [ ] Add `boardColumnId String?` to `Task` (nullable — tasks not yet on a board just use `status`)
- [ ] Migration: on first Kanban view load, auto-create a default Board with 3 columns (To Do / In Progress / Done) mapped to the existing `TaskStatus` values, and backfill `boardColumnId` from `status` — so users don't lose their Phase 1 data
- [ ] Decide: does moving a card between columns update `status` too (for Today/Upcoming filtering to stay correct), or do columns become the new source of truth? Recommendation: keep `status` as the canonical field, columns declare which `status` they represent, so List/Kanban/Matrix views stay consistent

### 2. API
- [ ] `GET/POST /api/boards`, `PATCH/DELETE /api/boards/[id]/columns/[id]`
- [ ] `PATCH /api/tasks/[id]/move` — `{ boardColumnId, position }`, updates both column and derived `status`
- [ ] Reuse the existing reorder-transaction pattern from Phase 1's `/api/tasks/reorder`

### 3. Frontend — Drag & Drop
- [ ] Use `@dnd-kit/core` + `@dnd-kit/sortable` (already in target stack)
- [ ] `KanbanBoard` component: horizontal-scrolling columns, each a `SortableContext`
- [ ] `KanbanCard`: condensed version of `TaskRow` — priority stamp, title, due badge, tag chips, avatar placeholder (real avatars come in Phase 4)
- [ ] Optimistic move: update local state immediately on drop, PATCH in background, roll back on error (same TanStack Query pattern as Phase 1)
- [ ] Column header: task count, "+ Add task" quick-add scoped to that column
- [ ] Keyboard-accessible drag (dnd-kit supports this out of the box) — don't ship a mouse-only interaction

---

## Week 5 — Calendar View & Eisenhower Matrix

### 4. Calendar View (Day/Week/Month)
- [ ] Library choice: build custom on top of `date-fns` (lighter, matches existing dependency) rather than pulling in FullCalendar — Phase 1's date utilities already establish this pattern
- [ ] `CalendarView` component with three modes:
  - Month: grid of day cells, tasks shown as compact pills (max 3 visible + "+N more")
  - Week: 7-column layout with hour rows for tasks that have a time component (add optional `dueTime` or keep `dueDate` as full DateTime — recommend the latter, simpler)
  - Day: single column, hour-by-hour
- [ ] Drag-to-reschedule: dragging a task pill to a new day/hour calls `PATCH /api/tasks/[id]` with new `dueDate` — dnd-kit again, different drop-zone shape than Kanban
- [ ] Unscheduled tasks tray (tasks with no `dueDate`) alongside the calendar so they can be dragged onto a date

### 5. Eisenhower Matrix View
- [ ] No new data needed — this is a pure derived view: Urgent = due within 48h or overdue, Important = priority P1/P2
- [ ] 2×2 CSS grid: Urgent+Important / Not Urgent+Important / Urgent+Not Important / Neither
- [ ] Tasks drggable between quadrants — dropping into a quadrant sets `priority` (P1 for Important, P3 for Not Important) and optionally prompts to set/clear a near-term due date for the Urgent axis, since "urgent" isn't a stored field, it's derived from `dueDate`
- [ ] This view doubles as a good onboarding moment — empty quadrants can carry instructional copy ("Drag a task here to mark it Important + Urgent")

### 6. View Switcher
- [ ] Persist the user's last-used view per workspace (simple `localStorage` key is fine for Phase 2; server-persisted view preferences can wait)
- [ ] Tab/segmented-control UI in the header: List | Kanban | Calendar | Matrix — reuse the existing dashboard layout's header slot

---

## Week 6 — Productivity Tools & Offline Foundation

### 7. Pomodoro Timer
```prisma
model PomodoroSession {
  id          String    @id @default(cuid())
  userId      String
  taskId      String?
  workspaceId String
  startedAt   DateTime  @default(now())
  endedAt     DateTime?
  durationSec Int       @default(1500) // 25 min default
  completed   Boolean   @default(false)
  task        Task?     @relation(fields: [taskId], references: [id])
}
```
- [ ] Floating timer widget (persists across route navigation — put it in the dashboard layout, not per-page)
- [ ] Start a session optionally linked to the active task; on completion, log a `PomodoroSession` row (used for analytics in Phase 4)
- [ ] Simple states: Focus (25m) → Short Break (5m) → repeat, Long Break (15m) after 4 cycles — configurable later, hardcoded now
- [ ] Browser notification + sound on session end (respect notification permission prompts — don't request on page load, request on first "Start Timer" click)

### 8. Command Palette (Cmd+K) & Keyboard Shortcuts
- [ ] `cmdk` library (lightweight, purpose-built for this)
- [ ] Actions: quick-add task, jump to view (List/Kanban/Calendar/Matrix), jump to Today/Upcoming/Completed, search tasks by title (client-side filter over already-fetched tasks is enough for Phase 2 — server-side full-text search is a Phase 3+ concern)
- [ ] Global shortcuts: `Cmd+K` open palette, `N` quick-add (when not focused in an input), `E` toggle completed on focused task row, `?` shortcuts help overlay
- [ ] Shortcut handling via a small custom hook (`useHotkeys`) rather than a heavy library — the shortcut set is small

### 9. Offline Persistence (IndexedDB fallback)
Phase 1 shipped a read-only cache. Phase 2 makes writes work offline too, as a stepping stone toward full CRDT sync in Phase 4.

- [ ] `idb` library wrapping IndexedDB
- [ ] TanStack Query's persister (`@tanstack/query-sync-storage-persister` swapped for an IndexedDB persister) to survive reloads
- [ ] Outbox pattern: writes made while offline go into an `outbox` IndexedDB store; a `navigator.onLine` listener + periodic retry flushes the outbox when connectivity returns
- [ ] This is intentionally simple last-write-wins for Phase 2 — real conflict resolution (CRDTs) is Phase 4 scope. Document this limitation in-app if a sync conflict is detected (e.g. "This task changed elsewhere — showing the latest version")

---

## Definition of Done for Phase 2
- User can switch between List, Kanban, Calendar, and Matrix views of the same underlying tasks, with changes in one view reflected in the others
- Kanban supports custom columns with drag-and-drop reordering and cross-column moves
- Calendar supports Day/Week/Month with drag-to-reschedule
- Matrix view lets tasks be dragged between quadrants, updating priority/urgency
- Pomodoro timer works standalone or linked to a task, logs completed sessions
- Cmd+K command palette covers quick-add, view switching, and task search
- App remains usable (read + write) with no network connection; writes sync once reconnected

## Explicit Non-Goals for Phase 2 (deferred on purpose)
- Natural language task parsing, AI daily planner (Phase 3)
- Real recurrence engine — recurring tasks stay manual for now (Phase 3)
- CRDT-based conflict-free sync — Phase 2's offline support is last-write-wins (Phase 4)
- Multi-user presence on Kanban/Calendar (Phase 4)
- Server-persisted view preferences, cross-device view sync (nice-to-have, not blocking)

## Carry-forward risks to flag before starting
1. **Task `status` vs `boardColumnId` duality** — decide the source-of-truth rule in week 4 before building Calendar/Matrix, since both derive from `status`/`dueDate`/`priority`. Getting this wrong means rework in week 5.
2. **dnd-kit across three different drop-zone shapes** (Kanban columns, Calendar cells, Matrix quadrants) — budget time to build one shared `useDragAndDrop` abstraction rather than three bespoke implementations.
3. **IndexedDB outbox** is genuinely fiddly to get right (ordering, retries, partial failures) — timebox it and accept a simpler version if week 6 runs long; better to ship a working read-only-offline app than a buggy write-sync one.
