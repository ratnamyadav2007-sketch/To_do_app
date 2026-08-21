# ApexTask — Phase 1 Detailed Plan: Core Engine & Essential CRUD

**Duration:** Weeks 1–3
**Goal:** A rock-solid, responsive single-user todo list with modern UI/UX, built on a schema that won't need to be re-architected when multi-tenancy, search, and collaboration are added in later phases.

---

## Week 1 — Foundation & Data Layer

### 1. Repo & Tooling Setup
- [ ] Monorepo structure (Turborepo or Nx) — `apps/web`, `apps/api` (or single Next.js app with API routes if going full-stack Next.js), `packages/db`, `packages/ui`
- [ ] TypeScript strict mode, ESLint, Prettier, Husky pre-commit hooks
- [ ] CI pipeline (GitHub Actions): lint → typecheck → test → build on every PR
- [ ] Environment config: `.env.example`, secrets via GitHub Actions secrets / Vercel env
- [ ] Design system primitives: color tokens, spacing scale, typography scale in Tailwind config; base components (Button, Input, Modal, Dropdown, Toast) using shadcn/ui as a starting point

### 2. Database Schema (Postgres + Prisma)
Design every table with `workspaceId` from day one, even though Phase 1 is single-user — this avoids a painful migration in Phase 4.

```prisma
model User {
  id            String    @id @default(cuid())
  email         String    @unique
  name          String?
  avatarUrl     String?
  passwordHash  String?   // null if OAuth-only
  createdAt     DateTime  @default(now())
  workspaces    WorkspaceMember[]
}

model Workspace {
  id        String   @id @default(cuid())
  name      String
  ownerId   String
  createdAt DateTime @default(now())
  members   WorkspaceMember[]
  tasks     Task[]
  categories Category[]
  tags      Tag[]
}

model WorkspaceMember {
  id          String   @id @default(cuid())
  workspaceId String
  userId      String
  role        Role     @default(OWNER) // OWNER only in Phase 1; EDITOR/VIEWER added Phase 4
  workspace   Workspace @relation(fields: [workspaceId], references: [id])
  user        User      @relation(fields: [userId], references: [id])
  @@unique([workspaceId, userId])
}

model Task {
  id           String    @id @default(cuid())
  workspaceId  String
  title        String
  description  String?   // markdown
  status       TaskStatus @default(TODO) // TODO, IN_PROGRESS, DONE
  priority     Priority?  // P1, P2, P3, P4
  dueDate      DateTime?
  completedAt  DateTime?
  categoryId   String?
  parentTaskId String?    // self-relation for subtasks
  position     Int        // for manual ordering
  createdAt    DateTime   @default(now())
  updatedAt    DateTime   @updatedAt
  createdById  String

  category     Category?  @relation(fields: [categoryId], references: [id])
  parentTask   Task?      @relation("Subtasks", fields: [parentTaskId], references: [id])
  subtasks     Task[]     @relation("Subtasks")
  tags         TaskTag[]

  @@index([workspaceId, status])
  @@index([workspaceId, dueDate])
}

model Category {
  id          String  @id @default(cuid())
  workspaceId String
  name        String
  color       String
  tasks       Task[]
}

model Tag {
  id          String   @id @default(cuid())
  workspaceId String
  name        String
  color       String
  tasks       TaskTag[]
  @@unique([workspaceId, name])
}

model TaskTag {
  taskId String
  tagId  String
  task   Task @relation(fields: [taskId], references: [id])
  tag    Tag  @relation(fields: [tagId], references: [id])
  @@id([taskId, tagId])
}

enum TaskStatus { TODO IN_PROGRESS DONE }
enum Priority   { P1 P2 P3 P4 }
enum Role       { OWNER EDITOR VIEWER }
```

- [ ] Set up Prisma migrations, seed script with sample data for local dev
- [ ] Add full-text search column (Postgres `tsvector`, generated column on `title || description`) with a GIN index — costs nothing now, saves a migration in Phase 2

### 3. Authentication
- [ ] NextAuth.js (or Clerk if you want to move faster and pay for it) — Email/Password + Google + GitHub OAuth providers
- [ ] On first login: auto-create a personal `Workspace` for the user, add them as `OWNER`
- [ ] Session strategy: JWT for stateless scaling; store `userId` + active `workspaceId` in session
- [ ] Protected route middleware for all `/api/*` and app routes

---

## Week 2 — Core Task API & CRUD UI

### 4. API Layer (REST or tRPC — tRPC recommended for a TypeScript monorepo, less boilerplate)
Endpoints (or tRPC procedures):

| Action | Endpoint | Notes |
|---|---|---|
| List tasks | `GET /api/tasks?status=&categoryId=&tagId=` | paginated, filterable |
| Get task | `GET /api/tasks/:id` | includes subtasks, tags |
| Create task | `POST /api/tasks` | validate with Zod |
| Update task | `PATCH /api/tasks/:id` | partial update |
| Delete task | `DELETE /api/tasks/:id` | soft-delete recommended (add `deletedAt`) |
| Toggle complete | `PATCH /api/tasks/:id/complete` | sets `completedAt`, `status=DONE` |
| Reorder tasks | `PATCH /api/tasks/reorder` | batch position update |
| CRUD categories | `/api/categories` | |
| CRUD tags | `/api/tags` | |

- [ ] Zod schemas for all input validation, shared between client and server
- [ ] Soft-delete pattern (`deletedAt` timestamp) instead of hard delete — needed later for activity logs/undo
- [ ] Optimistic concurrency: `updatedAt` check on PATCH to avoid silent overwrite races (useful groundwork for Phase 4 collab)

### 5. Frontend — Task CRUD UI
- [ ] Task creation: quick-add input (title only) + expanded form (description, due date, priority, tags, category, subtasks)
- [ ] Task detail panel/modal: markdown-rendered description, inline editing
- [ ] Task list item component: checkbox, title, due date badge, priority flag, tag chips
- [ ] Delete confirmation (with soft-delete "Undo" toast — cheap win for UX polish)
- [ ] Subtask UI: nested checklist under parent task, progress indicator (e.g. "3/5 done")
- [ ] Category & Tag management UI (create/edit/delete, color picker)

### 6. State Management & Data Fetching
- [ ] TanStack Query (React Query) for server state — caching, optimistic updates, background refetch
- [ ] Optimistic UI on create/update/delete/toggle-complete (instant feedback, rollback on error)

---

## Week 3 — List View, Filtering, Sorting, Polish

### 7. List View
- [ ] Smart views: **Today**, **Upcoming** (next 7 days), **Completed**, **All Tasks**
  - Implement as saved filter presets, not hardcoded pages — this makes "custom smart filters" in Phase 2 trivial to add later
- [ ] Filtering: by status, priority, category, tag, due date range
- [ ] Sorting: by due date, priority, created date, manual (drag-to-reorder using `position` field)
- [ ] Empty states for each view (e.g. "Nothing due today 🎉")

### 8. Offline Persistence (basic, Phase 2 will deepen this)
- [ ] IndexedDB cache of last-fetched task list via TanStack Query's persister plugin
- [ ] Read-only offline mode for Phase 1 (full write-sync + CRDTs come in Phase 4) — just make sure the app doesn't blank-screen with no network

### 9. Testing & Quality Gates
- [ ] Unit tests: Zod schemas, task business logic (e.g. recurrence-free due date handling, position reordering)
- [ ] Integration tests: API routes (Vitest + Supertest or similar)
- [ ] E2E smoke test (Playwright): sign up → create task → complete task → delete task
- [ ] Lighthouse/perf check on task list with 500+ seeded tasks (catch N+1 queries early)

### 10. Deployment
- [ ] Vercel (frontend + API routes) or Railway/Render for a split Node backend
- [ ] Managed Postgres (Neon, Supabase, or Railway) + Redis (Upstash) provisioned even if unused until Phase 2/3
- [ ] Staging environment separate from production, seeded with demo data

---

## Definition of Done for Phase 1
- User can sign up/log in via email or OAuth
- User can create, read, update, delete tasks with title, description (markdown), due date, priority, tags, category
- User can create and manage subtasks
- User can view tasks in a filterable, sortable list across Today/Upcoming/Completed/All views
- Deleted tasks are recoverable via undo (soft delete)
- All API inputs validated; core flows covered by automated tests
- App deployed to staging and usable end-to-end by a real user

## Explicit Non-Goals for Phase 1 (deferred on purpose)
- Kanban/Calendar/Matrix views (Phase 2)
- Any AI features (Phase 3)
- Sharing, comments, real-time sync (Phase 4)
- CRDT-based offline write sync (Phase 4) — Phase 1 offline is read-only cache
- Billing/multi-tenant beyond the `workspaceId` schema field (Phase 5)
