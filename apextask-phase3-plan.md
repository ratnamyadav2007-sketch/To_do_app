# ApexTask — Phase 3 Detailed Plan: AI Capabilities & Smart Automation

**Duration:** Weeks 7–9
**Prerequisite:** Phase 1 + Phase 2 complete (multi-view engine, Pomodoro, command palette, offline read cache all working)
**Goal:** Use AI to remove friction from task entry and planning, and ship the recurring-task + notifications infrastructure that makes the app trustworthy enough to rely on daily.

---

## Week 7 — Natural Language Input & Auto-Categorization

### 1. Background job infra (do this first — everything else in Phase 3 depends on it)
AI calls are slow (1–3s) and shouldn't block the request/response cycle for quick-add. Set this up before writing any AI feature.

- [ ] BullMQ + Redis (Upstash for hosted Redis, matches target stack)
- [ ] `packages/jobs` (or `lib/jobs/` in a single-app layout): queue definitions for `parse-task-nlp`, `categorize-task`, `generate-daily-plan`, `send-notification-digest`
- [ ] Worker process — separate from the Next.js app (a small Node script run via `node worker.js`, deployed as its own process on Railway/Render, or a Vercel cron + queue-drain endpoint if staying serverless)
- [ ] Rate limiting + response caching on AI calls: hash the input string, cache the parsed result for 24h — avoids re-parsing "Submit report every Tuesday at 3pm" twice from the same user and controls OpenAI spend

### 2. Natural Language Input Parsing
- [ ] Extend `QuickAdd` (from Phase 1/2) to detect NL patterns client-side first with a cheap heuristic (contains "tomorrow", "every", "at 3pm", a weekday name, etc.) — only call the AI endpoint when heuristics suggest structured intent, to save cost on plain titles like "Buy milk"
- [ ] `POST /api/ai/parse-task` — sends the raw string to the model with a structured-output prompt (Vercel AI SDK's `generateObject` with a Zod schema is the natural fit here, avoids manual JSON-fence stripping)
  ```ts
  const parseResultSchema = z.object({
    title: z.string(),
    dueDate: z.string().datetime().nullable(),
    recurrenceRule: z.string().nullable(), // RFC 5545 RRULE string, see Recurring Task Engine below
    priority: z.enum(["P1", "P2", "P3", "P4"]).nullable(),
    suggestedTags: z.array(z.string()).max(3),
  });
  ```
- [ ] UI: as the user types in QuickAdd, show a live "preview chip" row under the input (parsed date badge, recurrence badge) so they can see what will be created before hitting Enter — debounce the AI call (600ms) so it's not firing on every keystroke
- [ ] Fallback: if parsing fails or the model is unavailable, just create the task with the raw string as the title — NL parsing must never block basic task creation

### 3. Auto-Categorization & Tag Suggestion
- [ ] On task creation (or edit of the description), suggest existing `Category`/`Tag` records by semantic similarity, not free-form new tags — prevents runaway tag sprawl. Two viable approaches, pick based on team appetite:
  - Cheap: send the task title + existing category/tag names as a list, ask the model to pick 0–3 matches (good enough for Phase 3, no new infra)
  - Better (defer to Phase 3.5 if time-constrained): embeddings (`text-embedding-3-small`) stored per Category/Tag, cosine similarity lookup — scales better but adds a vector index (pgvector) dependency
- [ ] Suggested tags/category appear as dismissible chips in the task detail panel, never auto-applied without a click — AI suggestions augment, they don't silently modify data
- [ ] Effort estimate suggestion (S/M/L or a time estimate) — same mechanism, new field: add `estimatedMinutes Int?` to `Task`

---

## Week 8 — Recurring Task Engine & Smart Daily Planner

### 4. Recurring Task Engine
This is infrastructure, not an AI feature — the NL parser (week 7) just produces the rule; a deterministic engine executes it.

- [ ] Store recurrence as an RRULE string (RFC 5545) on a new field, not a bespoke "daily/weekly/monthly" enum — RRULE already handles "every Tuesday," "last weekday of the month," "every 2 weeks," etc., and libraries exist (`rrule` npm package) so you're not hand-rolling recurrence math
  ```prisma
  model Task {
    // ...existing fields
    recurrenceRule String?   // RRULE string, e.g. "FREQ=WEEKLY;BYDAY=TU"
    recurrenceParentId String? // if this instance was generated from a recurring template
  }
  ```
- [ ] Design decision: **template + instances**, not a single mutating task. A task with `recurrenceRule` set is a template (never shown in Today/Upcoming itself); a scheduled job materializes concrete `Task` instances (with `recurrenceParentId` pointing back) some window ahead (e.g. next 30 days), so completing one Tuesday's instance doesn't affect next Tuesday's
- [ ] `generate-recurring-instances` cron job (daily, via BullMQ repeatable job or Vercel Cron): for every active template, use `rrule` to compute occurrences in the next 30 days, create any that don't yet exist, and stop generating for templates whose `dueDate` window has passed an end condition (RRULE supports `UNTIL`/`COUNT`)
- [ ] Editing a recurring task: prompt "This task repeats — apply to this occurrence only, or all future occurrences?" (the classic calendar-app pattern) — only-this-occurrence detaches the instance from the template, all-future edits the template and regenerates

### 5. Smart Daily Planner
- [ ] `POST /api/ai/daily-plan` — input: today's open tasks (title, priority, estimatedMinutes, dueDate) plus, if calendar sync exists, busy blocks (Phase 3 doesn't have external calendar sync yet — Phase 5 scope — so for now just use ApexTask's own Calendar view due-dated items as "already scheduled")
- [ ] Output: an ordered agenda — which tasks to tackle in what order and roughly when, respecting priority and any fixed due times, with the AI SDK's structured output again
- [ ] Present as a dismissible card at the top of the Today view ("Here's a suggested plan for today") rather than silently reordering the task list — the user's manual ordering from Phase 1 must remain the source of truth unless they explicitly accept the AI's plan
- [ ] "Accept plan" writes a `position` value to each task matching the suggested order; "Dismiss" just hides the card for today

---

## Week 9 — Notifications & Reminders

### 6. Notification data model
```prisma
model Notification {
  id          String   @id @default(cuid())
  userId      String
  workspaceId String
  taskId      String?
  type        NotificationType // DUE_SOON, OVERDUE, DAILY_DIGEST, MENTION (Phase 4)
  title       String
  body        String?
  readAt      DateTime?
  createdAt   DateTime @default(now())
}

model NotificationPreference {
  id            String  @id @default(cuid())
  userId        String  @unique
  inAppEnabled  Boolean @default(true)
  pushEnabled   Boolean @default(false)
  emailDigest   EmailDigestFrequency @default(DAILY)
  quietHoursStart Int?  // hour 0-23, skip push during this window
  quietHoursEnd   Int?
}
```

### 7. In-app notifications
- [ ] Bell icon in the dashboard header (new — Phase 1/2 didn't have a header slot for this, add one) with unread count badge
- [ ] Notification list: due-soon (configurable lead time, default 1h before), overdue (once per task, not repeated every hour), generated by a scheduled job scanning `dueDate` against `now()`
- [ ] Mark-as-read on click, "mark all read" action

### 8. Push notifications
- [ ] Web Push (VAPID keys + `web-push` npm package) — no native app yet (that's Phase 5), so this is browser push via a service worker
- [ ] Permission request UX: don't prompt on page load (browsers auto-deny repeat prompts and it trains users to dismiss); prompt contextually the first time a due-soon notification would have fired, with a one-line explanation
- [ ] Respect `quietHoursStart`/`quietHoursEnd` from `NotificationPreference` before sending push

### 9. Email digest
- [ ] Daily/weekly summary email (Resend or Postmark for delivery — pick one now, don't defer, since email infra is annoying to bolt on later): tasks due today, overdue count, yesterday's completion count
- [ ] `send-notification-digest` BullMQ repeatable job, respects `emailDigest` preference (`DAILY`, `WEEKLY`, `OFF`)
- [ ] Unsubscribe link in every email (legal requirement, not optional) — one-click sets `emailDigest = OFF`

---

## Definition of Done for Phase 3
- Typing a natural-language task string ("Submit report every Tuesday at 3pm") produces a correctly-dated, correctly-recurring task, with a visible preview before creation and a graceful fallback to plain-text title if parsing fails
- Existing categories/tags are suggested (never silently applied) based on task content
- Recurring tasks generate real, independently-completable instances up to 30 days ahead, with an occurrence-vs-template edit prompt
- A dismissible AI-suggested daily plan appears on the Today view without overriding manual task order unless explicitly accepted
- In-app notification bell shows due-soon/overdue items; push and daily email digest both work and both respect user-configured preferences and quiet hours

## Explicit Non-Goals for Phase 3 (deferred on purpose)
- External calendar sync (Google/Outlook) — Phase 5, so the daily planner only sees ApexTask's own data for now
- Real-time collaboration, @mentions, activity logs — Phase 4
- Embeddings-based semantic tag matching — noted as a "better" option above but the cheap list-based approach ships first; only upgrade if tag suggestion quality is visibly poor in practice
- Native mobile push (APNs/FCM) — web push only until a mobile app exists (Phase 5)

## Carry-forward risks to flag before starting
1. **AI cost control is a week-7 concern, not a week-9 afterthought.** Build the parse-result cache and the heuristic pre-filter (skip AI for obviously-plain titles) before shipping NL input to any real users, or a normal day of quick-adds turns into hundreds of API calls.
2. **Recurring task template/instance split is the highest-risk schema decision in this phase.** Get the "edit this occurrence vs. all future" UX and the generation-window job right in week 8 before building the Smart Daily Planner on top of it in the same week — the planner needs to read materialized instances, not templates.
3. **Push notification permission UX is easy to get wrong** and, once a user denies the browser permission, there's no programmatic way to re-prompt — they must fix it in browser settings. Ship the contextual (not on-load) prompt from day one; don't ship an on-load prompt "temporarily" and plan to fix it later.
