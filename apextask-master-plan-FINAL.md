# ApexTask — Final Master Plan (Phases 1–5, Revised)

**Timeline:** 15 weeks, 5 phases · **Source:** consolidated + corrected from the 5 individual phase plans

This document is the original 5-phase plan merged into one, with the inconsistencies and deferred
technical debt that the original plans kept flagging (but never resolved) fixed at the point where
they're cheapest to fix. Section 0 explains every change. Sections 1–5 are the full phase plans with
those changes folded in — anything changed from the original is marked **[REVISED]**.

---

## 0. What Was Wrong and What Changed

### 0.1 API style: tRPC vs REST — pick REST, once, in Phase 1

Phase 1 recommended tRPC ("less boilerplate") but then specified a REST-shaped endpoint table
(`GET /api/tasks`, `PATCH /api/tasks/:id`, etc.). Every later phase — Phase 4's
`/api/workspaces/[workspaceId]/tasks` path convention, Phase 5's versioned `/api/v1/*` public API and
Zod→OpenAPI doc generation — assumes REST semantics that don't map cleanly onto tRPC procedures.
**Fix:** drop the tRPC recommendation. Standardize on REST from Week 2 of Phase 1, explicitly
*because* Phase 5 needs a public, documentable, versionable API — building on tRPC would mean either
a rewrite in Phase 5 or bolting a REST shim on top of tRPC procedures. REST from day one avoids both.
**Update (0.9):** with Phase 1 now built on FastAPI, this decision gets stronger, not weaker — FastAPI
generates OpenAPI documentation automatically from the same Pydantic models used for validation, which
is exactly what Phase 5 Week 14 wants ("API documentation... generated from schemas already defined"),
just arriving three phases early instead of needing to be built from scratch in Phase 5.

### 0.2 The "touch every route" problem — build the mutation wrapper in Phase 4, not never

Phase 4's own risk notes flagged that permission checks (week 10), WebSocket emission (week 11), and
activity logging (week 12) all get hand-added to the same ~15 mutation routes — three passes over the
same files in the same phase — and suggested consolidating into one wrapper "once all three concerns
are understood." Phase 5 then re-flagged the exact same unresolved problem when webhook firing became
a *fourth* concern on those routes, saying week 14 is "the last reasonable point" to fix it. Two phases
in a row identified the fix and neither one did it.
**Fix:** build the mutation wrapper in **Phase 4, Week 10**, immediately after the permission-check
pattern is designed, before week 11's realtime emit or week 12's activity log are written:

```ts
async function mutate<T>(ctx: {
  workspaceId: string;
  minRole: "EDITOR" | "OWNER";
  action: string; // "task.created", etc.
}, fn: () => Promise<T>): Promise<T> {
  await requireWorkspaceAccess(ctx.workspaceId, ctx.minRole);
  const result = await fn();
  await emitRealtimeEvent(ctx.workspaceId, ctx.action, result);   // no-op until Week 11 ships
  await logActivity(ctx.workspaceId, ctx.action, result);          // no-op until Week 12 ships
  await fireOutboundWebhooks(ctx.workspaceId, ctx.action, result); // no-op until Phase 5 Week 14 ships
  return result;
}
```
Weeks 11, 12, and Phase 5 Week 14 then each just implement one previously-stubbed function instead of
re-touching every route file. This turns three-to-four full-codebase passes into one.

### 0.3 Push subscriptions: Phase 5 assumes a field Phase 3 never created

Phase 5 Week 15 says "Phase 3's `NotificationPreference.pushSubscription` field already anticipates
this" — but Phase 3's actual `NotificationPreference` model has no such field; Web Push (VAPID) requires
storing a subscription object (endpoint + keys) per browser/device, and a user can have several.
**Fix:** add a proper model in Phase 3, Week 9, not a single field on the preference row:

```prisma
model PushSubscription {
  id          String   @id @default(cuid())
  userId      String
  endpoint    String   @unique
  p256dh      String
  auth        String
  createdAt   DateTime @default(now())
}
```
This is what Phase 5's PWA work actually subscribes into.

### 0.4 Full-text search infra is built in Phase 1 and never wired up

Phase 1 adds a `tsvector`/GIN index specifically to save a migration later. Phase 2 explicitly defers
search ("server-side full-text search is a Phase 3+ concern"). Phase 3, 4, and 5 never mention it again
— the index sits unused for the entire project.
**Fix:** wire it up where it was deferred to — **Phase 2, Week 6**, alongside the command palette (which
currently does client-side filtering over already-fetched tasks). Add `GET /api/tasks/search?q=` backed
by the Phase 1 `tsvector` column, and have the command palette call it instead of filtering client-side
once the task list exceeds what's already fetched. This is a half-day addition given the index already
exists, and it's the reason the index was built in the first place.
**Update (0.9):** with Phase 1's data layer now JSON files instead of Postgres, there is no `tsvector`
column to build on. Phase 1's `GET /api/tasks/search?q=` is now a plain Python substring match instead
(see the Phase 1 section below) — same endpoint contract, same Phase-2-wires-it-up plan, cheaper
implementation, and a documented signal (search getting slow) for when to reconsider the storage layer.

### 0.5 `WorkspaceMember.role` defaults to `OWNER` — that's a privilege-escalation footgun

Phase 1's schema sets `role Role @default(OWNER)` on `WorkspaceMember`. That default only matters for
code paths that create a membership without explicitly setting a role — and Phase 4 adds exactly such a
path (invite acceptance). Phase 4's invite flow does set `role` explicitly from `WorkspaceInvite.role`
(default `EDITOR`), so it isn't exploited in the plan as written, but a schema default of `OWNER` is a
silent trap for any future code path (e.g. a Phase 5 API-driven member-add) that forgets to pass a role.
**Fix:** change the schema default to `EDITOR` in Phase 1, and require `WorkspaceMember` creation during
signup/personal-workspace-creation to pass `role: "OWNER"` explicitly rather than relying on the default.
A permission model should never have its safest-looking default be its most dangerous value.

### 0.6 Two sources of truth for workspace ownership

`Workspace.ownerId` and `WorkspaceMember.role === OWNER` both encode "who owns this," with no stated
rule for what happens if they disagree (e.g. after an ownership transfer, which Phase 4/5 never define).
**Fix:** `WorkspaceMember.role === OWNER` is the single source of truth for current ownership. Keep
`Workspace.ownerId` only as an immutable "created by" audit field and rename it `createdById` in Phase 1
so it's never mistaken for "current owner" by later phases. Add an explicit (stretch, Phase 4+) "transfer
ownership" action that moves the `OWNER` role between members — not in this plan's critical path, but the
naming fix prevents the ambiguity from calcifying into the schema.

### 0.7 Dependency introduced without being flagged as new

Phase 2 says `@dnd-kit/core` is "already in target stack," but no earlier phase lists it as a dependency
— Phase 1's tooling setup doesn't mention it. **Fix:** Phase 1's Week 1 tooling list now explicitly adds
`@dnd-kit/core` + `@dnd-kit/sortable` as a base dependency (it costs nothing to install early and Phase
2 needs it in Week 4), so Phase 2 accurately inherits it rather than silently introducing it.
**Update (0.9):** moot as originally framed — `@dnd-kit` is a React-only library, and Phase 1's
frontend is no longer React. Phase 1 now standardizes on the native HTML5 Drag and Drop API (or
SortableJS as a fallback) for its own drag-to-reorder, and flags this as the dependency Phase 2's
Kanban/Calendar/Matrix drag-and-drop should build on instead, if Phase 2 stays framework-free too.

### 0.8 Everything else

No other schema, sequencing, or scope conflicts were found — the phase-to-phase handoffs (workspaceId
scoping from Phase 1 paying off in Phase 4, RRULE templates from Phase 3 feeding the Phase 4 activity
log and Phase 5 analytics, the offline-outbox → realtime-dedup → calendar-sync-loop-prevention lesson
carried across Phases 2/4/5) were already well-designed and are kept as-is.

---

## 1. Phase 1 — Core Engine & Essential CRUD (Weeks 1–3) **[REVISED — stack change 0.9]**

**Goal:** unchanged — a rock-solid single-user todo app on a schema that survives Phases 2–5 without re-architecture.

> **Stack change (0.9):** Phase 1 now uses a **Python backend**, a **plain HTML/CSS/JS frontend**
> (no React, no build step required), and **JSON files** as the data store, replacing
> Next.js/TypeScript/Prisma/Postgres. This is a deliberate simplification for getting Phase 1 running
> fast with zero external services. It has real consequences for later phases — see the callout at
> the end of this section.

### Week 1 — Foundation & Data Layer

**1. Repo & tooling setup**
- [ ] `apextask/backend` (Python) + `apextask/frontend` (static HTML/CSS/JS) as sibling folders — no
  monorepo tooling (Turborepo/Nx) needed since there's no shared TypeScript package to build
- [ ] Backend: **FastAPI** (not Flask) — its automatic OpenAPI generation directly supports the REST
  decision in §0.1 and gives Phase 5's public-API/OpenAPI-docs goal a running start for free
- [ ] `requirements.txt`: `fastapi`, `uvicorn`, `pydantic`, `python-jose` (JWT), `passlib[bcrypt]`
  (password hashing), `filelock` (safe concurrent JSON writes), `pytest`, `httpx` (test client)
- [ ] Frontend: no framework, no bundler — `index.html` + native ES modules (`<script type="module">`)
  loaded directly by the browser; plain CSS with custom properties for the design-token equivalent
  (color/spacing/typography scale) instead of a Tailwind config
- [ ] `ruff` + `black` for lint/format (Python equivalent of ESLint/Prettier), pre-commit hooks via
  `pre-commit`
- [ ] CI pipeline (GitHub Actions): lint → typecheck (`mypy`, optional) → pytest → build/package on
  every PR
- [ ] `.env.example` for secrets (JWT signing key, OAuth client IDs if used)

**2. Data layer (JSON files, not Postgres)**
Each "table" from the original Prisma schema becomes a JSON file of records under
`backend/app/data/`, accessed only through a small storage module — nothing in route code touches the
files directly.

```python
# backend/app/storage.py
import json, uuid
from pathlib import Path
from filelock import FileLock
from datetime import datetime, timezone

DATA_DIR = Path(__file__).parent / "data"

def _paths(name: str):
    f = DATA_DIR / f"{name}.json"
    return f, FileLock(str(f) + ".lock")

def read_all(collection: str) -> list[dict]:
    f, _ = _paths(collection)
    if not f.exists():
        return []
    return json.loads(f.read_text())

def write_all(collection: str, records: list[dict]) -> None:
    f, lock = _paths(collection)
    with lock:
        f.write_text(json.dumps(records, indent=2, default=str))

def new_id() -> str:
    return uuid.uuid4().hex

def now() -> str:
    return datetime.now(timezone.utc).isoformat()
```

Collections, mirroring the original Prisma models field-for-field (just as plain dicts validated by
Pydantic instead of Prisma types): `users.json`, `workspaces.json`, `workspace_members.json`,
`tasks.json`, `categories.json`, `tags.json`, `task_tags.json`.

```python
# backend/app/models.py  (Pydantic — replaces Zod/Prisma types)
from pydantic import BaseModel
from typing import Optional, Literal

class Task(BaseModel):
    id: str
    workspaceId: str
    title: str
    description: Optional[str] = None
    status: Literal["TODO", "IN_PROGRESS", "DONE"] = "TODO"
    priority: Optional[Literal["P1", "P2", "P3", "P4"]] = None
    dueDate: Optional[str] = None
    completedAt: Optional[str] = None
    categoryId: Optional[str] = None
    parentTaskId: Optional[str] = None
    position: int
    createdAt: str
    updatedAt: str
    createdById: str
    deletedAt: Optional[str] = None   # soft-delete, same pattern as the original plan

class WorkspaceMember(BaseModel):
    id: str
    workspaceId: str
    userId: str
    role: Literal["OWNER", "EDITOR", "VIEWER"] = "EDITOR"   # default fixed per §0.5
```

- [ ] Seed script (`scripts/seed.py`) writes sample `users.json`/`workspaces.json`/`tasks.json` for
  local dev, same purpose as the original Prisma seed script
- [ ] **[REVISED — 0.4, adapted]** No Postgres `tsvector`/GIN index is available in this stack, so
  full-text search is a plain Python function over the in-memory task list (case-insensitive substring
  match on `title` + `description`), exposed as `GET /api/tasks/search?q=`. This is fine at JSON-file
  scale; flagged in the stack-change callout below as the first thing to reconsider if task counts grow
  large enough that linear scan becomes slow.
- [ ] **Known limitation, stated plainly:** JSON-file storage has no real transactions and no
  concurrent-write safety beyond the `filelock` wrapper above — reads are cheap, but a write locks the
  whole collection file. This is acceptable for Phase 1's single-user scope. It is **not** acceptable
  once Phase 4 adds real multi-user concurrent writes to the same workspace — see the callout at the
  end of this section.

**3. Authentication**
- [ ] Email/password only for Phase 1 (OAuth via `authlib` is a clean add later if needed, but isn't
  required to hit Phase 1's Definition of Done)
- [ ] `passlib[bcrypt]` for password hashing, stored in `users.json` as `passwordHash`
- [ ] **Stateless JWT session** (via `python-jose`), signed with a server secret, stored in an
  `httpOnly` cookie — no server-side session store needed, which fits a JSON-file backend well (no
  extra collection to keep consistent)
- [ ] On first signup: auto-create a personal `Workspace` (JSON record) and a `WorkspaceMember` row
  with `role="OWNER"` set **explicitly** in code, not relied on as a schema default (§0.5)
- [ ] FastAPI dependency (`Depends(get_current_user)`) protecting every route that needs auth —
  the Python equivalent of the original plan's Next.js middleware

### Week 2 — Core Task API & CRUD UI

**4. API layer (FastAPI, REST + Pydantic)** — same endpoint table as the original plan, same REST
decision as §0.1, just implemented in Python instead of Next.js route handlers:

| Action | Endpoint | Notes |
|---|---|---|
| List tasks | `GET /api/tasks?status=&categoryId=&tagId=` | paginated, filterable |
| Search tasks | `GET /api/tasks/search?q=` | linear substring match — see note above |
| Get task | `GET /api/tasks/{id}` | includes subtasks, tags |
| Create task | `POST /api/tasks` | validated by the Pydantic `Task` model |
| Update task | `PATCH /api/tasks/{id}` | partial update |
| Delete task | `DELETE /api/tasks/{id}` | soft-delete (`deletedAt`), same as original plan |
| Toggle complete | `PATCH /api/tasks/{id}/complete` | sets `completedAt`, `status="DONE"` |
| Reorder tasks | `PATCH /api/tasks/reorder` | batch position update |
| CRUD categories / tags | `/api/categories`, `/api/tags` | same shape as original plan |

- [ ] Pydantic models are the single source of validation truth on the backend (replaces Zod); the
  frontend re-implements the same field constraints in a small `validate.js` module rather than sharing
  types across a language boundary — this is the one place the "shared Zod schema" convenience from the
  original TypeScript-only plan is genuinely lost, and it's an accepted tradeoff of this stack
- [ ] Optimistic-concurrency check on `PATCH` (compare submitted `updatedAt` to the stored one, reject
  with `409` on mismatch) — same pattern as the original plan, still worth keeping even without a DB
  enforcing it, since it's groundwork for Phase 4 collaboration either way

**5. Frontend — Task CRUD UI (vanilla HTML/CSS/JS)**
- [ ] `frontend/js/api.js`: a thin `fetch()` wrapper (base URL, JSON headers, cookie credentials,
  error handling) — the vanilla-JS equivalent of a typed API client
- [ ] `frontend/js/components/`: small, framework-free UI modules built with plain DOM APIs
  (`taskList.js`, `taskForm.js`, `modal.js`, `toast.js`) rendering via template strings or
  `<template>` elements — no JSX, no virtual DOM
- [ ] Task creation: quick-add input (title only) + expandable form (description, due date, priority,
  tags, category, subtasks) — same UX as the original plan
- [ ] Task detail: a modal (native `<dialog>` element) with markdown-rendered description
  (`marked.js` via CDN, small and dependency-light) and inline editing
- [ ] Task list item: checkbox, title, due-date badge, priority flag, tag chips — plain DOM elements
  with CSS classes, no component library
- [ ] Delete confirmation with soft-delete "Undo" toast — same UX goal as the original plan
- [ ] Subtask UI: nested checklist with a progress indicator ("3/5 done")
- [ ] Category & tag management UI (create/edit/delete, color picker via `<input type="color">`)

**6. State management & data fetching**
- [ ] No TanStack Query (React-only) — a small hand-rolled `store.js` holding the current task list in
  memory, with a `subscribe()`/`notify()` pattern so components re-render on change; optimistic
  update helpers (`optimisticCreate`, `optimisticUpdate`, `optimisticDelete`) that mutate the store
  immediately and roll back on a failed `fetch()` — same UX goal as the original plan's optimistic UI,
  implemented without a library

### Week 3 — List View, Filtering, Sorting, Polish

**7. List view** — unchanged in behavior from the original plan: Today / Upcoming / Completed / All as
saved filter presets (not hardcoded pages, so Phase 2's custom filters are still easy to add);
filtering by status/priority/category/tag/due-date range; sorting including manual drag-to-reorder.

- [ ] **[REVISED — 0.7, adapted]** Manual drag-to-reorder uses the native HTML5 Drag and Drop API
  directly, or **SortableJS** (a small, framework-agnostic library) if the native API's touch-device
  support proves too rough — **not** `@dnd-kit`, which is a React-only library and doesn't apply to a
  vanilla-JS frontend. This is the dependency Phase 2's Kanban board should also plan around (see
  callout below).

**8. Offline persistence (basic)**
- [ ] `localStorage` cache of the last-fetched task list (simpler than IndexedDB at this data volume,
  and avoids pulling in `idb` before Phase 2 actually needs write-outbox semantics)
- [ ] Read-only offline mode: on `fetch()` failure, fall back to the cached list and show a small
  "offline — showing last synced data" banner instead of blank-screening

**9. Testing & quality gates**
- [ ] Unit tests: `pytest` for Pydantic validation and task business logic (position reordering,
  due-date handling)
- [ ] Integration tests: FastAPI's `TestClient` (built on `httpx`) against the real route handlers and
  a temp JSON data directory
- [ ] E2E smoke test: Playwright (language-agnostic, works fine against a plain HTML frontend) —
  sign up → create task → complete task → delete task
- [ ] Basic perf check: seed 500+ tasks into the JSON files and confirm `GET /api/tasks` still responds
  quickly — this is the first practical signal for when JSON-file storage needs to be reconsidered

**10. Deployment**
- [ ] Backend: `uvicorn` behind a process manager, deployed to a host with a **persistent disk** (Render,
  Fly.io, a small VPS, Railway) — **not** a serverless/edge platform with an ephemeral filesystem, since
  the JSON files must survive between requests and deploys
- [ ] Frontend: static files served either directly by FastAPI (`StaticFiles` mount) for simplicity, or
  from any static host (Netlify, Vercel static, GitHub Pages) pointed at the backend's API URL
- [ ] Back up the `data/` directory on a schedule (even a simple cron `cp` to object storage) — there's
  no managed-database backup safety net in this stack, so this replaces what Neon/Supabase would have
  given for free
- [ ] Staging environment: a second deployment with its own `data/` directory seeded with demo data

**Definition of Done for Phase 1:** unchanged in substance from the original plan (signup/login,
task CRUD with description/due date/priority/tags/category, subtasks, filterable/sortable list across
Today/Upcoming/Completed/All, soft-delete-with-undo, validated inputs, automated tests, deployed and
usable end-to-end) — only the implementation stack changed, not the user-facing scope.

**Non-Goals:** unchanged from the original plan (Kanban/Calendar/Matrix in Phase 2, AI in Phase 3,
sharing/comments/real-time in Phase 4, CRDT offline sync in Phase 4, billing/multi-tenant in Phase 5).

> ### ⚠️ Ripple effects of this stack change on Phases 2–5
> Phases 2–5 in this master plan were written assuming Prisma/Postgres, Next.js/TypeScript, and React
> (`@dnd-kit`, TanStack Query, `RealtimeProvider`, shadcn/ui, etc.). Switching Phase 1 to
> Python/JSON/vanilla-JS doesn't break Phase 1 itself, but it means Phases 2–5 **as currently written
> are no longer accurate** and would need their own pass before being built:
> - Every later-phase Prisma model addition (Kanban's `Board`/`BoardColumn`, Phase 3's
>   `PomodoroSession`/recurrence fields, Phase 4's `Comment`/`ActivityLogEntry`, Phase 5's `ApiKey`)
>   becomes a new JSON collection instead of a migration.
> - Phase 4's real-time collaboration and Phase 4/5's concurrent multi-user writes are the point where
>   JSON-file storage genuinely stops being appropriate — file-level locking does not give you the
>   row-level concurrency multiple simultaneous editors need. **A real move to Postgres (or similar)
>   should happen no later than Phase 4, Week 10**, before workspace sharing introduces real concurrent
>   writers, not after.
> - Every React-specific dependency in Phases 2–5 (`@dnd-kit`, TanStack Query, shadcn/ui,
>   `RealtimeProvider` as a React context) needs a vanilla-JS or different-framework equivalent if the
>   frontend stays framework-free, or Phase 2 becomes the point where a frontend framework gets
>   introduced deliberately (rather than assumed).
> This plan does not rewrite Phases 2–5 to match — say the word and I'll do that pass next.

---

## 2. Phase 2 — Views Engine & Productivity Tools (Weeks 4–6)

**Goal:** unchanged.

**Week 4 (Kanban):** unchanged — `Board`/`BoardColumn` models, `status` remains canonical source of
truth with columns declaring which status they represent, dnd-kit drag-and-drop (now a Phase 1
dependency, not newly introduced — see 0.7), optimistic move with rollback.

**Week 5 (Calendar & Matrix):** unchanged — custom calendar on `date-fns`, `dueDate` stays a full
`DateTime` rather than adding a separate `dueTime` field, Eisenhower Matrix as a pure derived view,
view switcher with `localStorage`-persisted last-used view.

**Week 6 (Productivity tools) — one addition:**
- **[REVISED — 0.4]** Command palette's task search now hits `GET /api/tasks/search?q=` backed by
  Phase 1's `tsvector`/GIN index, instead of filtering only the already-fetched client-side task list.
  This finally uses the search infrastructure Phase 1 built for this exact purpose. (Client-side
  filtering can remain as an instant-feedback layer while the debounced server search resolves, but
  server search is now the actual match source, not a "Phase 3+ concern.")
- Everything else (Pomodoro timer + `PomodoroSession` model, Cmd+K shortcuts, IndexedDB outbox pattern
  for offline writes, last-write-wins with in-app conflict notice) unchanged.

**Definition of Done:** unchanged, plus: "Command palette task search is server-backed via full-text
search, not limited to already-fetched tasks."
**Non-Goals:** unchanged.
**Carry-forward risks:** unchanged (status/boardColumnId duality, shared drag-and-drop abstraction,
IndexedDB outbox timeboxing).

---

## 3. Phase 3 — AI Capabilities & Smart Automation (Weeks 7–9)

**Goal:** unchanged.

**Week 7 (NL input, auto-categorization):** unchanged — BullMQ + Redis job infra, heuristic pre-filter
before calling the AI endpoint, `generateObject`-style structured parsing with the schema shown in the
original plan, cheap list-based category/tag suggestion (embeddings explicitly deferred), dismissible
suggestion chips, `estimatedMinutes` field added to `Task`.

**Week 8 (Recurring engine + Daily Planner):** unchanged — RRULE string on `Task.recurrenceRule`,
template + materialized-instance design with `recurrenceParentId`, 30-day generation window via
repeatable BullMQ job, occurrence-vs-template edit prompt, Smart Daily Planner as a dismissible card
that never silently reorders the task list.

**Week 9 (Notifications) — one addition:**
- **[REVISED — 0.3]** Add a dedicated `PushSubscription` model (endpoint, p256dh, auth, per-user,
  supporting multiple devices) rather than a single field bolted onto `NotificationPreference`. This
  is what Web Push actually subscribes into, and what Phase 5's PWA work builds on.
- Everything else (`Notification`/`NotificationPreference` models, in-app bell + unread badge,
  contextual — not on-load — push permission prompt, quiet hours, Resend-based daily/weekly digest
  with mandatory unsubscribe link) unchanged.

**Definition of Done:** unchanged, plus: "Push notifications are backed by a proper per-device
subscription model, supporting a user with multiple browsers/devices."
**Non-Goals / carry-forward risks:** unchanged (AI cost control front-loaded to week 7, template/instance
split as the highest-risk schema decision, push-permission UX can't be re-prompted once denied).

---

## 4. Phase 4 — Real-Time Collaboration & Analytics (Weeks 10–12)

**Goal:** unchanged — still the highest-risk phase because it's the first one that changes who can see
and touch data.

**Week 10 (Sharing & access control) — the key structural change:**
- Permission model (`requireWorkspaceAccess`, VIEWER < EDITOR < OWNER, workspace-scoped paths) as
  originally planned.
- **[REVISED — 0.2]** Immediately after the permission-check pattern is working, build the `mutate()`
  wrapper described in §0.2, with `emitRealtimeEvent`, `logActivity`, and `fireOutboundWebhooks` as
  stubbed no-ops. Migrate all ~15 existing mutation routes to call through the wrapper **once**, in
  Week 10, instead of being touched again in Weeks 11 and 12 (and again in Phase 5 Week 14).
- Invitations (`WorkspaceInvite` model, tokenized accept link, workspace settings page) unchanged —
  note invite acceptance already explicitly sets `role` from the invite, so it isn't affected by the
  Phase 1 default-role fix in §0.5, but now benefits from it as defense-in-depth.
- Guest/view-only share links: unchanged, still explicitly a stretch goal.

**Week 11 (Real-time sync):**
- **[REVISED — 0.2]** Implementing `task:created`/`task:updated`/etc. is now "fill in the
  `emitRealtimeEvent` stub" rather than "add an emit call to ~15 route files." Socket.io (or managed
  alternative), room-per-workspace, client-generated `mutationId` dedup, and `RealtimeProvider` are
  otherwise unchanged from the original plan.

**Week 12 (Comments, activity log, analytics):**
- **[REVISED — 0.2]** Activity logging is now "fill in the `logActivity` stub," not a third pass over
  the route list.
- Comment model, `@mention` autocomplete + notification, task detail slide-over panel, analytics
  dashboard (completion velocity, Pomodoro trends, recurring-task completion rate, workspace-scoped
  and non-comparative per the original risk note) all unchanged.

**Definition of Done:** unchanged, plus: "All mutation routes go through a single wrapper that handles
permission check, DB write, realtime emit, and activity log — not four hand-maintained concerns per
route."
**Non-Goals:** unchanged.
**Carry-forward risks:** risk #2 (the "touch every route" problem) is resolved by the Week 10 change
above rather than carried forward again. Risks #1 (permission enforcement is a full pass) and #3
(three overlapping truth mechanisms needing a dedup strategy before `RealtimeProvider` is written)
are unchanged and still apply.

---

## 5. Phase 5 — Platform & Scale (Weeks 13–15)

**Goal:** unchanged — billing, public API, calendar sync, browser extension, mobile presence.

**Week 13 (Billing):** unchanged — Stripe Customer per workspace, Checkout + Customer Portal,
webhook handler with signature verification, plan-tier limits with soft-warning-before-hard-block,
audit log reusing `ActivityLogEntry`, data export, soft-delete-with-grace-period for workspace deletion.

**Week 14 (Public API & integrations):**
- **[REVISED — 0.2]** Outbound webhook firing is now "fill in the `fireOutboundWebhooks` stub" set up
  in Phase 4 Week 10 — this is the point the original plan itself said was "the last reasonable point"
  to consolidate the mutation-handling logic; that consolidation already happened two phases earlier,
  so this week is pure feature work (webhook model, HMAC signing, BullMQ-backed retry with backoff),
  not a fifth hand-added concern on every route.
- `ApiKey` model, `/api/v1/*` versioned namespace, per-key rate limiting via Redis, OpenAPI docs
  generated from the existing Zod schemas — unchanged, and now more directly justified given §0.1
  locked in REST from Phase 1 specifically for this reason.
- Zapier/Make support built on the public API — unchanged.

**Week 15 (Calendar sync, extension, mobile):**
- **[REVISED — 0.3]** PWA push subscription flow now stores into Phase 3's `PushSubscription` model
  (§0.3) instead of a field that didn't exist.
- Everything else unchanged: per-user OAuth scopes for Google/Outlook, one-way sync shipped before
  two-way, timestamp-based loop prevention reusing the Phase 4 `mutationId` dedup lesson, browser
  extension quick-capture through the existing NL-parse endpoint, PWA-vs-React-Native decision scoped
  to read/complete/quick-add if native is chosen.

**Definition of Done:** unchanged.
**Non-Goals:** unchanged (SSO/SAML deferred, no full native app parity, no full extension UI, no
usage-based billing).
**Carry-forward risks:** risk #1 (mutation-wrapper consolidation) is resolved as of Phase 4 Week 10
rather than flagged a third time. Risks #2 (calendar sync as the phase's highest-risk item, ship
one-way first) and #3 (billing bugs need real test coverage) are unchanged and still apply.

---

## 6. Summary of Every Change

| # | Issue | Where introduced | Where it would have bitten | Fix location |
|---|---|---|---|---|
| 1 | tRPC recommended but REST endpoints specified; Phase 5 public API needs REST | Phase 1 | Phase 5 API design | Phase 1 Week 2 — commit to REST |
| 2 | Same mutation routes hand-touched 3–4 times across phases despite the plan flagging it twice | Phase 4 | Phase 4 Wk 11/12, Phase 5 Wk 14 | Phase 4 Week 10 — build `mutate()` wrapper |
| 3 | Phase 5 assumes a `pushSubscription` field Phase 3 never created | Phase 3 → referenced Phase 5 | Phase 5 Week 15 implementation | Phase 3 Week 9 — add `PushSubscription` model |
| 4 | Full-text search index built, never queried | Phase 1 | Never — silently unused forever | Phase 2 Week 6 — wire up `/api/tasks/search` |
| 5 | `WorkspaceMember.role` defaults to `OWNER` | Phase 1 | Any future code path that omits `role` | Phase 1 — default to `EDITOR`, explicit `OWNER` on signup |
| 6 | Two sources of truth for workspace ownership (`ownerId` vs `role===OWNER`) | Phase 1 | Phase 4+ ownership transfer (undefined) | Phase 1 — rename to `createdById`, `role` is authoritative |
| 7 | `@dnd-kit` used in Phase 2 without being declared a dependency | Phase 2 | Nowhere functionally, but inaccurate plan | Phase 1 Week 1 — add to tooling list |
| 9 | Phase 1 stack changed to Python + JSON + vanilla JS; React/Postgres-specific choices in §0.1/0.4/0.7 and Phases 2–5 no longer match | Requested change | Phases 2–5 as currently written | Phase 1 rewritten; Phases 2–5 need their own follow-up pass (flagged, not yet done) |

Everything else in the original five plans — schema design, week-by-week sequencing, the
offline/realtime/calendar-sync "ship the simple version first" discipline, and the risk notes not
listed above — was internally consistent and is carried forward unchanged.
