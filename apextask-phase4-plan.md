# ApexTask — Phase 4 Detailed Plan: Real-Time Collaboration & Analytics

**Duration:** Weeks 10–12
**Prerequisite:** Phase 1 + 2 + 3 complete (multi-view engine, AI features, recurring tasks, notifications all working)
**Goal:** Turn ApexTask from a single-user app into a team-ready collaborative workspace, and give users a reason to look back at what they've done, not just what's ahead.

This is the highest-risk phase so far — it's the first one that changes who can see and touch a piece of data, not just how that data is displayed or automated. Get the permission model right before writing any real-time code.

---

## Week 10 — Workspace Sharing & Access Control

### 1. Extending the permission model
Phase 1 already put `WorkspaceMember.role` (OWNER/EDITOR/VIEWER) in the schema specifically so this week wouldn't require a migration — now it needs to actually be enforced.

- [ ] `requireUserAndWorkspace` (Phase 1's helper) needs to become workspace-*selectable*, not workspace-*singular*. Right now it grabs the user's first membership; Phase 4 needs an explicit `workspaceId` on every request (header, query param, or path segment — recommend a `/api/workspaces/[workspaceId]/tasks` style path so it's impossible to forget) plus a check that the membership exists and has sufficient role for the action
  ```ts
  async function requireWorkspaceAccess(workspaceId: string, minRole: "VIEWER" | "EDITOR" | "OWNER") {
    // VIEWER < EDITOR < OWNER — check membership.role meets or exceeds minRole
  }
  ```
- [ ] Every mutating route (task create/update/delete, category/tag create, board changes) needs a minimum-role check added — this touches nearly every Phase 1–3 API route, so budget real time for this, not just the new Phase 4 endpoints
- [ ] Read routes need VIEWER-or-above; write routes need EDITOR-or-above; workspace settings (invite/remove members, delete workspace) need OWNER

### 2. Invitations
```prisma
model WorkspaceInvite {
  id          String   @id @default(cuid())
  workspaceId String
  email       String
  role        Role     @default(EDITOR)
  invitedById String
  token       String   @unique
  expiresAt   DateTime
  acceptedAt  DateTime?
  createdAt   DateTime @default(now())
}
```
- [ ] `POST /api/workspaces/[id]/invites` (OWNER only) — creates invite, sends email (reuse Phase 3's Resend integration) with a tokenized accept link
- [ ] `POST /api/invites/[token]/accept` — if the invited email matches an existing account, add `WorkspaceMember`; if not, redirect to register-then-accept
- [ ] Workspace settings page: pending invites list (with revoke), member list (with role change/remove, OWNER only)
- [ ] Decide workspace-switching UX now even though most users will only have one or two workspaces: a workspace switcher in the sidebar header, persisting last-active workspace in a cookie so `requireWorkspaceAccess` has a sensible default when no explicit `workspaceId` is passed

### 3. Guest / external access (stretch — only if week 10 goes smoothly)
- [ ] View-only shareable link per task or list, no login required (`GET /api/share/[token]` returns read-only rendering data) — genuinely optional for Phase 4; don't let it block the real-time work in weeks 11–12 if time is short

---

## Week 11 — Real-Time Sync

### 4. WebSocket infrastructure
- [ ] Socket.io server — this needs a persistent Node process, which breaks the "just API routes" simplicity of Phases 1–3. Options: a small standalone Socket.io server deployed alongside the Next.js app (Railway/Render, same place as the Phase 3 BullMQ worker — good opportunity to consolidate both into one long-running process), or a managed realtime provider (Pusher, Ably, or Supabase Realtime) if standing up Socket.io infra isn't worth it for the team's size
- [ ] Room-per-workspace: clients join `workspace:{workspaceId}` on connect (after verifying session + membership over the socket handshake, not just trusting a client-sent workspaceId)
- [ ] Events to broadcast: `task:created`, `task:updated`, `task:deleted`, `task:moved` (Kanban), `comment:created`, `member:joined`, `presence:update`
- [ ] Every existing mutation route (task CRUD, move, reorder) needs to emit its event after a successful DB write — this is the other place, besides permissions, where Phase 1–3 routes all need a small addition, so plan the diff across ~15 existing route files, not just new code

### 5. Client-side real-time integration
- [ ] Socket connection managed in a new `RealtimeProvider` (same pattern as Phase 2's `PomodoroProvider`/`CommandPaletteProvider`) — connects once, joins the active workspace's room, reconnects on drop
- [ ] On receiving `task:updated` etc., patch TanStack Query's cache directly (`queryClient.setQueryData`) rather than invalidating and refetching — refetching on every teammate's keystroke would be both slow and janky; direct cache patches keep the UI responsive
- [ ] Reconcile with Phase 2's optimistic updates: a local optimistic update and an incoming real-time event for the *same* mutation (your own change echoing back) need to be deduplicated — tag outgoing mutations with a client-generated `mutationId` and ignore incoming events that echo an in-flight one
- [ ] This is also where Phase 2's offline outbox (deliberately left shallow) starts to matter more — a write made offline and replayed later needs to not stomp a teammate's concurrent edit. Phase 4 scope is still last-write-wins (full CRDT conflict resolution is explicitly out of scope until the offline-sync item later in this phase), but the real-time layer should at least show a "this was just changed by someone else" toast if an incoming event contradicts a pending local mutation

### 6. Live presence
- [ ] Lightweight presence: on socket connect, broadcast `presence:update` with `{ userId, name, avatarUrl, currentView }`; server tracks connected users per workspace room in memory (Redis-backed if running multiple server instances — reuse Phase 3's Redis)
- [ ] UI: small avatar stack in the header showing who else is currently viewing the workspace, and on Kanban cards, a subtle indicator if someone else has that card open

---

## Week 12 — Comments, Activity Log, and Analytics Dashboard

### 7. Comments & @mentions
```prisma
model Comment {
  id        String   @id @default(cuid())
  taskId    String
  authorId  String
  body      String   // markdown, same rendering path as Task.description
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  deletedAt DateTime?
  task      Task     @relation(fields: [taskId], references: [id], onDelete: Cascade)
  mentions  CommentMention[]
}

model CommentMention {
  id         String  @id @default(cuid())
  commentId  String
  userId     String
  comment    Comment @relation(fields: [commentId], references: [id], onDelete: Cascade)
}
```
- [ ] Comment thread in the task detail panel (Phase 1 never actually built a full detail panel — this is the forcing function to finally add one; a slide-over panel, consistent with the ledger-row design direction, showing full description edit, comments, and activity for that task)
- [ ] `@mention` autocomplete pulling from workspace members; on submit, create `CommentMention` rows and a `Notification` (reuse Phase 3's `Notification` model, add `MENTION` to `NotificationType`)
- [ ] Mentioned users get an in-app notification immediately (via the real-time layer, not just the next digest) and are included in the email digest if they have it enabled

### 8. Activity Log
```prisma
model ActivityLogEntry {
  id          String   @id @default(cuid())
  workspaceId String
  taskId      String?
  actorId     String
  action      String   // "task.created", "task.completed", "task.status_changed", "comment.created", etc.
  metadata    String?  // JSON blob with before/after for diffable fields
  createdAt   DateTime @default(now())
}
```
- [ ] Write an entry from the same mutation routes that now emit WebSocket events — same touch points, so implement both together rather than two separate passes over ~15 route files
- [ ] Per-task activity tab in the new task detail panel ("Sarah changed priority P3 → P1", "Marked complete", "Comment added")
- [ ] Workspace-level activity feed (optional, time permitting) for a team's "what happened since I was last here" view

### 9. Productivity Analytics Dashboard
- [ ] New `/analytics` view: completion velocity (tasks completed per day/week, trailing 30 days), Pomodoro session count and focus-minutes trend (data already exists from Phase 2's `PomodoroSession`), habit-adjacent streak view if recurring-task completion rates are tracked (derive from Phase 3's `recurrenceParentId` — % of instances completed vs. skipped per template)
- [ ] Charting: `recharts` (already in the target stack's available libraries) for line/bar charts — completion velocity as a line chart, time-per-category as a stacked bar
- [ ] Keep this read-only and workspace-scoped for Phase 4 — per-person leaderboards or comparisons across teammates are a easy way to make a productivity tool feel surveillance-y; if that's wanted later, make it opt-in per user, not a default

---

## Definition of Done for Phase 4
- Workspace owners can invite teammates by email with a chosen role (Viewer/Editor/Admin); invited users can accept and see the shared workspace
- Every API route enforces the invited user's role — Viewers cannot mutate, Editors cannot manage membership
- Task changes make by one teammate appear in another teammate's open browser within roughly a second, without a manual refresh
- A live presence indicator shows who's currently viewing the workspace
- Every task has a comment thread supporting @mentions, which generate real-time + digest notifications
- An activity log shows a human-readable history of changes per task
- An analytics dashboard shows completion velocity and focus-time trends over the trailing 30 days

## Explicit Non-Goals for Phase 4 (deferred on purpose)
- CRDT-based conflict-free offline sync — real-time sync (week 11) assumes an online connection; full offline-write conflict resolution is still out of scope, tracked as its own follow-up item, not silently absorbed into this phase
- Granular per-task or per-list sharing (only workspace-level roles for now) — narrower sharing scopes are a natural Phase 5+ extension once workspace-level roles are proven out
- Cross-workspace analytics or team leaderboards — explicitly avoided per the risk note in step 9
- Video/voice presence, live cursors on the Kanban board — presence here means "who's around," not full multiplayer cursors

## Carry-forward risks to flag before starting
1. **Permission enforcement is a full-codebase pass, not a new-feature pass.** Nearly every route from Phases 1–3 needs a role check added. Do this in week 10 as its own tracked pass across the route list, not opportunistically while building new Phase 4 endpoints — it's the kind of change that's easy to miss a file on, and a missed check here is a real security bug, not a UX rough edge.
2. **The same "touch every route" problem applies again in week 11** for WebSocket event emission, and again in week 12 for activity logging — by week 12 some routes will have been touched three separate times for three different concerns. Consider, once all three concerns are understood, refactoring toward a single mutation wrapper (permission check → DB write → emit event → log activity) that new Phase 5 routes can use from day one, rather than continuing to hand-add all four concerns to every route individually.
3. **Real-time + optimistic UI + offline outbox is three different "what's the current truth" mechanisms overlapping.** The dedup strategy in week 11 (client-generated `mutationId`) needs to be designed before writing the `RealtimeProvider`, not discovered through bug reports after teammates start colliding on the same task.
