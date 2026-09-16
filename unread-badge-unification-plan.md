# Unread Badge Unification Plan

**Status:** Plan only — no code changes yet
**Date:** Sep 2026 · **Author:** Arpit · **Repo:** xyne-spaces

---

## 1. Goal

```
dock badge            = Σ (workspace count) over ALL workspaces
workspace count (ws)  = DM count (ws) + activity count (ws)
```

- The dock badge and the workspace switcher must read the **same number from the same source** — zero divergence between them, by construction.
- The dock's data source moves from Zero sync (live, active-workspace-only) to the same HTTP polling the workspace switcher already uses.
- The DM rail entry in the left sidebar gets a numeric unread badge (today it only shows a dot).

## 2. Current vs target architecture

| | Today | Target |
|---|---|---|
| Dock badge source | `useAllUnreadCount()` — Zero sync, **active workspace only**, per-channel sum | Shared polling hook → `GET /activity/workspace-counts` — **all workspaces** |
| Workspace switcher source | Same endpoint, raw `activities` count where `isRead = false` (no filters) | Same endpoint, extended response: `{ dmCount, activityCount, count }` |
| DM rail | Presence dot only (`hasPendingDirectMessages`) | Numeric badge = active workspace's `dmCount` |
| Bell | Zero sync, client-filtered (unchanged) | Unchanged |
| Consistency | Dock ≠ switcher ≠ bell (different sources + filters) | Dock ≡ Σ switcher counts (same hook state). Bell matches switcher's activity half at recompute time (same filters, different freshness) |

## 3. Decisions needed (with recommendations)

### D1 — Does the activity half include channel thread replies?
Today the dock **excludes** thread activities (XYNE-14040 — threads have their own rail badge); the bell **includes** them.
**Recommendation: include them** in the activity half (mirror the bell). The workspace badge becomes "attention sum" (like the bell), and the thread rail badge keeps its own separate counter. This makes workspace activity half ≡ bell semantics.
**Alternative:** exclude, keeping current dock semantics — then workspace count < bell count, and we've moved the divergence rather than removed it.

### D2 — GROUP_DM mentions: avoid double counting
A top-level GROUP_DM message with @mention creates **both** a conversation row (dmCount +1) and an activity row (activityCount +1) → workspace count +2 for one message.
**Recommendation: the activity half excludes all activities whose channel is DM/GROUP_DM.** DMs are fully represented by dmCount; one event = one count. Consequence: DM thread replies count 0 in workspace/dock (dmCount doesn't move on replies) but 1 in the bell — same as today's dock behavior, so no regression.
**Alternative:** accept the double count (rare event), or extend dmCount to thread replies (bigger change to the conversation counter).

### D3 — Closed/archived channels
**Recommendation:** filter `isClosed = false, isDeleted = false` in **both** halves server-side (a closed channel shouldn't nag). The bell keeps counting them (feed semantics) — accepted, documented divergence.

### D4 — Polling cadence vs dock responsiveness
Reading a DM currently clears the dock instantly (live sync). Under 30s polling, the dock lags up to 30s after a read.
**Recommendation:** keep the 30s baseline (same as switcher) **plus** immediate refetch on: window focus / visibility change, and after read-marking mutations (mutator success callbacks trigger refetch). This restores perceived instant clearing without a second data path.
**Alternative:** shorter interval (10s) — more load, still laggy.

### D5 — Filters for the server-side activity half
Mirror the bell's client filters exactly, server-side:
- exclude `actorAction IN ('added_v2', 'removed')` — reactions, membership removal
- exclude `actionSource = 'call' AND actorAction = 'missed_call'` — calls badge owns these
- exclude `classification = 'SKIP'` (defensive — SKIP rows are deleted today)
- include `PENDING` (matches bell)
- exclude DM-channel activities (per D2)
- include channel-less activities (tickets etc.) — the new dock is a sum, not channel-grouped, so these now count (today's dock drops them)

## 4. Backend changes

**4.1 Extend `activityService.getWorkspaceActivityCounts`** ([activityService.ts:716](apps/backend/src/services/activity/activityService.ts#L716)) — additive response shape:

```ts
// today
{ workspaceId, userId, count }
// after
{ workspaceId, userId, dmCount, activityCount, count }  // count = dmCount + activityCount
```

- `dmCount`: `SUM(channel_user_status.unreadCount)` over the user's channels where channel `scopeType IN (DM, GROUP_DM)`, `isClosed = false`, `isDeleted = false`
- `activityCount`: filtered activity count per D5
- Response stays backward-compatible (`count` field retained); old consumers unaffected.

**4.2 Shared predicate** — extract the D5 filter list into one constant (e.g. `UNREAD_ACTIVITY_EXCLUSION`) importable by both the server count and (optionally later) the client hooks, so the filter can't drift again. Today the same rules live in two client hooks and nowhere server-side — that's the root cause of the original divergence.

**4.3 Performance** — one extra aggregate per user identity per request; typical members have few workspaces. Add a short server-side TTL cache (5s) if load warrants, since every dashboard client polls every 30s.

**4.4 Audit the sibling endpoint** — `GET /notifications/workspace-counts` ([notifications.ts:20](apps/backend/src/routes/notifications.ts#L20), notification rows not activities) has other consumers (Electron tray / mobile). Confirm none of them expect the old activity semantics; leave it untouched.

## 5. Data-correctness prerequisite (must ship first or with phase 1)

The DM half of the new number is only as good as `channel_user_status.unreadCount` — and that counter has the stale-guard defect we traced:

- `handleUnreadCount` skips recompute when `channel_stats.lastActivityAt <= lastViewedAt` ([unreadCountUtlis.ts:37](apps/backend/src/zero/utils/unreadCountUtlis.ts#L37)), but **ordinary messages never update `channel_stats.lastActivityAt`** — only channel creation, membership changes, and calls do.
- Result: once you've viewed a DM channel, its unread recompute can be permanently skipped → `dmCount` silently frozen at 0.

**Fix:** update `channel_stats.lastActivityAt` in the message/conversation insert side-effect path (or drop the guard and recompute unconditionally when a new conversation is inserted). Without this, the unified badge inherits the exact bug that started this investigation.

## 6. Frontend changes

**6.1 New shared hook `useWorkspaceUnreadCounts`** — single polling client for the extended endpoint:
- polls every 30s (baseline), refetch on focus/visibility/read-mutations (D4)
- exposes: `byWorkspace: Map<workspaceId, { dmCount, activityCount, count }>`, `totalAllWorkspaces: number`, `activeWorkspace: { dmCount, activityCount, count } | undefined`, `isStale` flag
- one interval for the whole app (mount near WorkspaceSwitcher / ElectronBadgeSync), not one per consumer

**6.2 `WorkspaceSwitcher`** — switch its local `fetchActivityCounts` to the shared hook. Displayed values identical (per-workspace `count`); removes its private axios call and interval.

**6.3 `useElectronBadge`** ([useElectronBadge.ts](apps/dashboard/src/hooks/useElectronBadge.ts)) — replace `useAllUnreadCount()` sum with `totalAllWorkspaces` from the shared hook. Update the doc comment (its "known limitation: active workspace only" note becomes obsolete — this fixes it). Dock and switcher now render from the same state object → divergence impossible.

**6.4 DM rail badge** ([AppSidebar.tsx:415-460](apps/dashboard/src/components/AppSidebar/AppSidebar.tsx#L415-L460)) — on the `/chat/dm` rail item, replace `hasPendingDirectMessages` dot with a numeric badge showing active workspace's `dmCount`, styled like the existing missed-call badge (`99+` cap). Keep the dot for `0 < dmCount` only if product prefers minimal chrome — default: number.

**6.5 Bell** — unchanged (Zero sync, live). Its filters now match the server's activity half, so at any recompute moment `bell(activeWs) == activityCount(activeWs)`; only freshness differs (live vs ≤30s).

**6.6 `useAllUnreadCount`** — **not deleted**: per-channel badges in the channel list / DM list still need it. Only the *aggregate sum* consumer (dock) moves to the poll.

## 7. Rollout

1. **Phase 0:** stale-`lastActivityAt` guard fix (§5) — independently shippable, fixes the frozen DM counter today.
2. **Phase 1:** backend endpoint extension (additive) — behind no flag; old clients unaffected.
3. **Phase 2:** shared hook + WorkspaceSwitcher migration — no visible change.
4. **Phase 3:** dock source switch + DM rail badge — behind feature flag `DOCK_BADGE_POLL_SOURCE` (default off) so we can compare and roll back.
5. **Phase 4 (optional cleanup):** remove the bell's dead `direct_message` ACTIONABLE gate; deprecate the second `/notifications/workspace-counts` if unused.
6. **Monitoring:** during flag rollout, log `|zeroDerivedSum − polledTotal|` per session; alert if >0 beyond the 30s window. Watch endpoint latency/DB load.

## 8. New matrix — what each badge counts after this change

● counted · ▲ conditional · ○ not counted

| Event | Dock (= Σ ws) | Switcher (per ws) | DM rail | Bell |
|---|:-:|:-:|:-:|:-:|
| **Direct messages** | | | | |
| 1:1 DM, top-level message | ● dmCount | ● dmCount | ● | ○ |
| DM thread reply | ○ ¹ | ○ ¹ | ○ ¹ | ● |
| GROUP_DM top-level message | ● dmCount | ● dmCount | ● | ○ |
| GROUP_DM message w/ @mention | ● dmCount ² | ● dmCount ² | ● ² | ● |
| GROUP_DM thread reply w/ @mention | ○ ¹ | ○ ¹ | ○ ¹ | ● |
| **Channels** | | | | |
| @mention of you | ● activity | ● activity | ○ | ● |
| @channel / @here | ● activity | ● activity | ○ | ● |
| Keyword match | ● activity | ● activity | ○ | ● |
| Canvas mention (has channelId) | ● activity | ● activity | ○ | ● |
| Thread reply in channel | ● activity ³ | ● activity ³ | ○ | ● |
| **Reactions & membership** | | | | |
| Reaction on your message | ○ ⁴ | ○ ⁴ | ○ | ○ ⁴ |
| You are added to a channel | ● activity | ● activity | ○ | ● |
| You are removed from a channel | ○ ⁴ | ○ ⁴ | ○ | ○ ⁴ |
| **Calls** | | | | |
| Missed call | ○ ⁴ | ○ ⁴ | ○ | ○ ⁴ |
| **Channel-less** | | | | |
| Ticket activity (no channelId) | ● activity ⁵ | ● activity ⁵ | ○ | ● |
| **Cross-cutting** | | | | |
| Classified SKIP (if persisted) | ○ | ○ | ○ | ○ |
| Still PENDING | ● | ● | ● | ● |
| Closed/archived channel | ○ ⁶ | ○ ⁶ | ○ | ● ⁶ |
| Another workspace | ● (in Σ) | ● (own row) | ○ | ○ |
| Daily recap | ○ | ○ | ○ | ○ |

¹ DM thread replies create no conversation row and DM-channel activities are excluded from the activity half (D2) — 0 in workspace/dock/dm-rail, 1 in bell. Same as today's dock.
² Counted once via dmCount; the mention activity row is excluded to avoid double counting (D2).
³ Per D1 (recommendation: include, matching the bell). This is a behavior change vs today's dock, which excluded threads.
⁴ Server-side count now applies the same filters as the client (D5) — fixes today's switcher over-count for reactions, removals, and missed calls.
⁵ Channel-less activities now count in the dock (it's a sum, not channel-grouped) — fixes today's dock under-count.
⁶ Per D3 (recommendation: filter closed channels server-side). Bell keeps counting them (feed semantics) — accepted divergence.

## 9. Intentionally remaining divergences (document, don't "fix")

1. **Bell vs dock/switcher freshness** — bell is live (Zero sync), dock/switcher poll ≤30s. Same value, different clocks.
2. **DM thread replies** — bell only (feed item vs message-count semantics, ¹ above).
3. **Closed-channel activities** — bell only (feed semantics).
4. **Recap, thread-rail, missed-calls badges** — separate counters by design, untouched.
5. **Workspace ≠ bell exactly** — workspace = dmCount + filtered activities; bell = filtered activities (incl. DM-channel rows). The workspace badge is a superset by design (it adds DMs).

## 10. Test plan

- **Unit:** dmCount aggregation (open/closed DM channels, GROUP_DM); activity-half filters (each D5 rule); `count = dmCount + activityCount` invariant.
- **Integration:** endpoint response for a member with identities in 2 workspaces; backward compat (old `count` field).
- **E2E scenarios (Electron):** 3 DMs → dock +3 (was: bell/ws 0, dock 3 → now all agree on dmCount); reaction → nothing anywhere (was: switcher +1); read DM → dock clears ≤30s or on focus-refetch; mention in channel → dock/bell/switcher all +1; cross-workspace DM → dock counts it, bell doesn't.
- **Soak:** monitor `zeroDerivedSum − polledTotal` during flag rollout (§7).
