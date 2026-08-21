# ApexTask — Phase 5 Detailed Plan: Platform & Scale

**Duration:** Weeks 13–15
**Prerequisite:** Phase 1 + 2 + 3 + 4 complete (multi-view engine, AI features, real-time collaboration, analytics all working)
**Goal:** Turn ApexTask from a working product into a business — billing, an API surface other tools can build on, external calendar sync, and the beginnings of a presence outside the web app itself.

This phase is the most heterogeneous one yet: five largely-independent workstreams (billing, API, calendar sync, browser extension, mobile/push) that each touch a different part of the stack. Unlike Phases 1–4, where each week built on the last, most of Phase 5's items can be parallelized across a small team — but if one person is doing this sequentially, order matters for the reasons noted in each section.

---

## Week 13 — Billing & Multi-Tenant Plan Limits

### 1. Stripe integration
- [ ] Stripe Customer created at workspace-creation time (not user-creation time — billing is per-workspace, consistent with every other Phase 1 decision to scope data by `workspaceId`)
  ```prisma
  model Workspace {
    // ...existing fields
    stripeCustomerId String?
    planTier         PlanTier @default(FREE)
    planRenewsAt     DateTime?
  }

  enum PlanTier {
    FREE
    PRO
    TEAM
  }
  ```
- [ ] Stripe Checkout for upgrade flow (redirect to Stripe-hosted checkout rather than building custom card forms — less PCI surface, less UI to build)
- [ ] Stripe Customer Portal for self-serve plan management/cancellation (also Stripe-hosted)
- [ ] Webhook handler (`POST /api/webhooks/stripe`) for `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted` — updates `Workspace.planTier`/`planRenewsAt`. Verify Stripe's webhook signature on every request; this endpoint is a common target if left unverified

### 2. Plan-tier limits
- [ ] Decide limits now, even if generous at first — retrofitting limits onto a workspace that's already over them is an awkward product moment to design under pressure later: e.g. FREE = 3 workspace members / 100 active tasks, PRO = unlimited members / unlimited tasks + AI features, TEAM = PRO + SSO (stretch, likely Phase 6+)
- [ ] Enforcement point: a `checkPlanLimit(workspaceId, limitType)` helper called from the relevant mutation routes (invite creation, task creation past the FREE cap) — same "touch existing routes" pattern flagged repeatedly in Phase 4's plan, so budget for it rather than treating it as new-code-only
- [ ] Soft warnings before hard blocks: "You're at 90/100 tasks on the Free plan" banner before actually rejecting the 101st task creation — respects the person's workflow instead of failing a request they didn't expect to fail
- [ ] Gate Phase 3's AI features behind PRO+ if the OpenAI cost model requires it — decide this explicitly rather than let free-tier AI usage become an unbounded cost surprise

### 3. Usage tracking & audit log
- [ ] Reuse Phase 4's `ActivityLogEntry` pattern for a workspace-level audit log (admin-only view): membership changes, plan changes, data exports, workspace deletion — distinct from the per-task activity log Phase 4 built, this is workspace-governance-focused
- [ ] Data export: `POST /api/workspaces/[id]/export` (OWNER only) — generates a JSON (and optionally CSV) dump of all workspace tasks/categories/tags/comments, emailed as a download link (reuse Phase 3's Resend) rather than held in-request, since a large workspace's export can take real time to generate
- [ ] Account/workspace deletion: soft-delete the workspace with a real grace period (recommend 30 days) before hard-deleting via a scheduled job, and send a confirmation email — GDPR-adjacent expectations apply even for non-EU users at this point

---

## Week 14 — Public API & Integrations

### 4. Public API
- [ ] API key model, distinct from session auth:
  ```prisma
  model ApiKey {
    id          String   @id @default(cuid())
    workspaceId String
    name        String   // user-facing label, e.g. "Zapier integration"
    keyHash     String   @unique // store a hash, never the raw key
    lastUsedAt  DateTime?
    createdAt   DateTime @default(now())
    revokedAt   DateTime?
  }
  ```
- [ ] `/api/v1/*` namespace, separate from the internal `/api/*` routes the web app uses — versioning from day one means the internal routes can keep evolving with the UI without breaking external integrations, and vice versa
- [ ] Rate limiting per API key (reuse Phase 3's Redis) — a public API without rate limits is an outage waiting to happen the first time an integration has a retry-loop bug
- [ ] API documentation — OpenAPI spec generated from the Zod schemas already defined in `lib/validation.ts` (several libraries do Zod → OpenAPI) rather than hand-written docs that drift from the actual validation

### 5. Webhook support (outbound)
- [ ] Distinct from Stripe's inbound webhooks (week 13) — this is ApexTask notifying *external* systems (Zapier, Make, a customer's own backend) when something happens in a workspace
  ```prisma
  model OutboundWebhook {
    id          String   @id @default(cuid())
    workspaceId String
    url         String
    events      String   // comma-separated: "task.created,task.completed"
    secret      String   // for HMAC signing outgoing payloads
    createdAt   DateTime @default(now())
  }
  ```
- [ ] Fire from the same mutation-route touch points as Phase 4's realtime emit and activity log — by now these routes have four concerns hand-added (permission check, DB write, realtime emit, activity log, webhook fire); this is the strongest signal yet that the mutation-wrapper consolidation flagged in Phase 4's risk #2 should happen before adding a fifth
- [ ] Delivery via the BullMQ queue infra from Phase 3 (retry with backoff on failure, don't block the original request on a slow customer endpoint)

### 6. Zapier/Make support
- [ ] Built on top of the public API (week 14, item 4) rather than bespoke — a Zapier app is mostly a thin wrapper describing the existing REST endpoints, so sequencing the public API first makes this straightforward
- [ ] Common triggers: "New task created," "Task completed"; common actions: "Create task," "Update task"

---

## Week 15 — Calendar Sync, Browser Extension, Mobile Presence

### 7. Google/Outlook two-way calendar sync
This is the highest-risk item in Phase 5 — two-way sync with an external calendar is a well-known source of subtle, hard-to-reproduce bugs (duplicate events, sync loops, timezone drift). Treat it with the same caution as Phase 2's CRDT-adjacent offline work.

- [ ] OAuth scopes for Google Calendar / Microsoft Graph, stored per-user (not per-workspace — a person's calendar is theirs, even inside a shared workspace)
- [ ] One-way first (ApexTask due dates → calendar events) before attempting two-way — validate the mapping and the user's mental model before adding the harder direction
- [ ] Two-way sync needs a clear source-of-truth rule per field to avoid loops: recommend "last writer wins by timestamp, but never sync a change back to its own origin within N seconds" (similar dedup shape to Phase 4's realtime `mutationId` pattern — reuse that lesson)
- [ ] This is also what finally lets Phase 3's Smart Daily Planner see real busy-blocks instead of just ApexTask's own due dates, closing the gap noted in that phase's explicit non-goals
- [ ] Scope cut if week 15 is tight: ship one-way sync only and mark two-way as a fast-follow — a working one-way sync beats a buggy two-way one, same lesson as Phase 2's offline-write timeboxing

### 8. Browser extension
- [ ] Quick-capture popup: highlight text on any page → "Add to ApexTask" → runs through the same NL parsing endpoint from Phase 3 (`/api/ai/parse-task`), so a highlighted "Follow up with Sarah next Tuesday" behaves identically to typing it into quick-add
- [ ] Manifest V3 (Chrome/Edge), stores the API key (week 14) or session token for auth
- [ ] Keep scope minimal for Phase 5: capture + view today's list. A full extension UI mirroring the web app is not worth building here

### 9. Mobile presence
- [ ] Decide React Native vs. PWA-with-push explicitly rather than defaulting — given Phase 2 already built an offline-capable, installable web app (IndexedDB cache, works without connection) and Phase 3 already has web push infrastructure, a PWA manifest + push subscription is a much smaller lift than a React Native app and may satisfy "mobile presence" for this phase
- [ ] If PWA: add a manifest.json, service worker for the push subscription (Phase 3's `NotificationPreference.pushSubscription` field already anticipates this), "Add to Home Screen" prompt
- [ ] If React Native: scope to read + complete + quick-add only for a first release, reusing the same `/api/v1/*` public API from week 14 rather than the internal API — this keeps mobile from becoming a second client that has to track every internal API change

---

## Definition of Done for Phase 5
- A workspace can upgrade from Free to Pro/Team via Stripe Checkout, self-manage billing via the Customer Portal, and see enforced plan limits with a soft-warning before any hard block
- Workspace owners can export all workspace data and can delete a workspace with a grace period, not an immediate hard delete
- An external developer can create an API key, read the OpenAPI docs, and call `/api/v1` to create/read/update tasks without touching the internal app
- Outbound webhooks fire reliably (with retry) when subscribed events occur, and a Zapier integration built on the public API works for the two common trigger/action pairs
- Due dates sync one-way to the user's Google or Outlook calendar at minimum; two-way sync ships if week 15's time allows without cutting corners on the loop-prevention logic
- A browser extension can quick-capture a highlighted selection into a parsed task
- Either a PWA with working push notifications or a scoped React Native app exists for mobile access

## Explicit Non-Goals for Phase 5 (deferred on purpose)
- SSO/SAML for Team-tier — real enterprise sales usually surface this need directly; build it when a deal requires it rather than speculatively
- A full-featured native mobile app matching every web view — scoped to read/complete/quick-add per week 15, item 9
- Full extension UI beyond quick-capture — a browser extension trying to be the whole app is a maintenance burden disproportionate to its use case
- Usage-based billing/metering — flat plan tiers only; usage-based pricing is a significant billing-system change better justified by actual customer demand than built speculatively

## Carry-forward risks to flag before starting
1. **This is the third phase in a row to flag the same unresolved issue**: mutation routes are now accumulating a fifth hand-added concern (webhook firing, after permission/DB-write/realtime-emit/activity-log from Phase 4). If the mutation-wrapper consolidation suggested in Phase 4's risk #2 hasn't happened yet, week 14 is the last reasonable point to do it before the pattern calcifies across the whole route list.
2. **Calendar sync is this phase's CRDT-equivalent risk** — the place most likely to eat unplanned time. Apply the same discipline Phase 2 used for offline writes: ship the safe, one-directional version first, and treat two-way sync as a scope cut candidate rather than a must-ship, even though "two-way sync" is what most users will assume "calendar sync" means.
3. **Billing logic bugs have a different failure mode than every other bug in this app so far** — a task-list bug loses someone's afternoon; a billing bug either charges someone incorrectly or lets a workspace use paid features for free. Write tests for the plan-limit enforcement and the Stripe webhook handler specifically, even if test coverage elsewhere in the app has been lighter — this is the one area of Phase 5 where "ship it and fix bugs as reported" is the wrong default.
