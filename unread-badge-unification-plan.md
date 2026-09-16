# Unread Badge Consistency — Complete Implementation Plan

**Status:** Plan only — no code changes yet
**Date:** Sep 2026 · **Author:** Arpit · **Repo:** xyne-spaces

---

## 1. The invariant

```
dock badge              = Σ (workspace count) over ALL workspaces      [same state object]
workspace count (ws)    = dmCount (ws) + bellCount (ws) + callCount (ws)
```

- Every event lands in exactly one shelf: **DM shelf**, **bell shelf**, or **call shelf**.
- Dock is never designed separately — it renders `Σ workspace counts` from the same polling state as the switcher. If the dock shows 32, the switcher rows sum to 32; if a workspace shows 12, its DM rail + bell + calls badges sum to 12.
- Bell *feed behavior* stays as-is (rows visible in the feed unchanged). Only *counting* changes.

## 2. Final target matrix — every event, its shelf

| Event | dm | bell | call | workplace | dock |
|---|:-:|:-:|:-:|:-:|:-:|
| 1:1 DM, top-level message | ● | ○ | ○ | ● | ● |
| DM thread reply (`replied_v2`) | ○ | ● | ○ | ● | ● |
| GROUP_DM top-level message | ● | ○ | ○ | ● | ● |
| GROUP_DM top-level w/ @mention | ○ ¹ | ● | ○ | ● | ● |
| GROUP_DM thread reply w/ @mention | ○ | ● | ○ | ● | ● |
| @mention of you (channel) | ○ | ● | ○ | ● | ● |
| @channel / @here | ○ | ● | ○ | ● | ● |
| Keyword match | ○ | ● | ○ | ● | ● |
| Canvas mention | ○ | ● | ○ | ● | ● |
| Channel thread reply (`replied_v2`) | ○ | ● | ○ | ● | ● |
| Ticket / channel-less activity | ○ | ● | ○ | ● | ● |
| You are added to a channel | ○ | ● | ○ | ● | ● |
| Reaction on your message | ○ ² | ○ ² | ○ | ○ | ○ |
| You are removed from a channel | ○ | ○ | ○ | ○ | ○ ³ |
| Missed call (`missed_call`) | ○ | ○ | ● | ● | ● |
| Activity classified SKIP | ○ | ○ | ○ | ○ | ○ ⁴ |
| Activity ERROR / PENDING | ○ | ● ⁵ | ○ | ● | ● |
| Unread in closed/archived channel | ○ | ○ | ○ | ○ | ○ ⁶ |
| Daily recap | ○ | ○ | ○ | ○ | ○ |

¹ dmCount subtracts unread top-level mention activities in GROUP_DM channels (mention wins the bucket — your Q1 decision). Per-channel DM badges apply the same subtraction so the DM list sums to the DM rail.
² Reaction rows create a **dot** (no number) on the DM channel row and the DM rail (only when the numeric badge is 0), and on the bell. Cleared on view. Never a number anywhere.
³ Not counted anywhere; visible only when opening the channel itself (your decision).
⁴ SKIP = nothing, everywhere, this phase. Classification rework deferred to later PRs.
⁵ ERROR rows count like PENDING (LLM failure is not the user's problem to infer; counting them is the simpler consistent default until the classification rework).
⁶ Excluded from all counts; reappear if the channel is reopened (accepted).

**Key behavioral changes vs today** (rows that change):
- Missed calls: workplace **starts counting** them (via callCount) — today the raw server count includes them but under the new three-way split they're explicit.
- Channel removals: workplace **stops counting** them (today's unfiltered server count includes them).
- DM thread replies: dock **starts counting** them (bell shelf — today's dock excluded threads).
- Tickets/channel-less: dock **starts counting** them.
- Closed-channel rows: bell **stops counting** them (new filter).
- GROUP_DM top-level mentions: dmCount **subtracts** them (mention wins).
- Bell: `removed` filter stays (already excluded), `missed_call` filter stays (call shelf), **new** closed-channel filter.

## 3. Backend changes

### 3.1 Shared filter constant (new file)

`apps/backend/src/services/activity/unreadActivityFilters.ts` (or shared package so dashboard can import):

```ts
export const BELL_COUNT_FILTER = {
  excludedActorActions: ['added_v2', 'removed'],       // reactions, channel removals
  excludedCalls: { actionSource: 'call', actorAction: 'missed_call' },
  excludedClassifications: ['SKIP', 'ERROR'],          // ERROR excluded? — see open Q1
  requireDirectMessageGate: false,                     // legacy direct_message rows: exclude
  excludeClosedChannels: true,                         // new rule
} as const;
```

This is the single source of truth. The server count and the client hook both derive from it. Today the same rules live in two client hooks and nowhere server-side — the root cause of the entire divergence.

### 3.2 Extend `activityService.getWorkspaceActivityCounts` ([activityService.ts:716](apps/backend/src/services/activity/activityService.ts#L716))

Additive response (old `count` retained for compat during rollout):

```ts
// today:   { workspaceId, userId, count }
// target:  { workspaceId, userId, dmCount, bellCount, callCount, count }
//          count = dmCount + bellCount + callCount
```

**`bellCount`** — groupBy on activities with the shared filter:

```sql
activities WHERE userId IN (identities) AND isRead = false
  AND actorAction NOT IN ('added_v2', 'removed')
  AND NOT (actionSource = 'call' AND actorAction = 'missed_call')
  AND classification NOT IN ('SKIP', 'ERROR')            -- open Q1 on ERROR
  AND (actorAction != 'direct_message')                  -- legacy DM rows excluded; their content is dmCount's job
  AND channel-is-not-closed                              -- join or anti-join vs channel_user_status
```

Closed-channel check: `channelId IS NULL OR channel_user_status(userId, channelId).isClosed = false`. Needs the join to `channel_user_status` (or a two-step: fetch closed channelId set for the user, then `NOT IN`).

**`callCount`** — groupBy on activities:

```sql
activities WHERE userId IN (identities) AND isRead = false
  AND actionSource = 'call' AND actorAction = 'missed_call'
```

Identical semantics to the Calls rail's `userMissedCalls` Zero query ([queries.ts:2463](packages/shared/src/zero/queries.ts#L2463)) — same rows, different transport.

**`dmCount`** — the new aggregate:

```sql
SELECT SUM(cus.unreadCount) FROM channel_user_status cus
JOIN channels c ON c.id = cus.channelId
WHERE cus.userId IN (identities)
  AND c.scopeType IN ('DM', 'GROUP_DM')
  AND cus.isClosed = false AND cus.isDeleted = false
MINUS unread GROUP_DM top-level mention activities        -- mention-wins subtraction, open Q2
```

The subtraction (Q1 decision) — conceptually:

```sql
- (count of unread activities WHERE actorAction IN ('mentioned_user','group_mention')
    AND actionSource = 'message' AND isThreadActivity = false
    AND channel.scopeType = 'GROUP_DM' AND isRead = false)
```

⚠️ **Precision risk — open Q2:** this subtracts *activity rows*, but dmCount counts *conversation rows*. When one GROUP_DM message mentions 3 users, 3 activity rows exist but the sender's conversation row is 1. Per-user this aligns (each mentioned user has 1 row and the channel counter is per-user too), but **batched `replied_v2` rows** (one row per conversation, updated per reply) and messages with multiple mentions of the same user need verification. I'll write a unit test matrix for this before finalizing the SQL.

### 3.3 Fix the frozen DM counter (Phase 0 — prerequisite)

`handleUnreadCount` skips recompute when `channel_stats.lastActivityAt <= lastViewedAt` ([unreadCountUtlis.ts:37](apps/backend/src/zero/utils/unreadCountUtlis.ts#L37)), but ordinary messages never update `channel_stats.lastActivityAt`.

**Fix:** in `conversations-handler.ts` `onInsert` ([conversations-handler.ts:39-45](apps/backend/src/zero/side-effects/tables/conversations-handler.ts#L39-L45)), before calling `handleUnreadCount`, upsert-bump `channel_stats.lastActivityAt = conversation.createdAt`. One write per conversation insert (already a write path — negligible cost). Alternatively drop the guard entirely; the bump is safer (keeps the cheap-path optimization for quiet channels).

Without this, dmCount freezes at 0 for viewed channels and the invariant breaks at its foundation.

### 3.4 Endpoint exposure

No new routes. `GET /activity/workspace-counts` ([activityLog.ts:28](apps/backend/src/routes/activityLog.ts#L28)) response shape changes additively. Response contract:

```ts
{ counts: Array<{ workspaceId: string; dmCount: number; bellCount: number; callCount: number; count: number }> }
```

(Keep `userId` in the internal service return; strip or keep in response — check current consumers first: only [WorkspaceSwitcher.tsx:103](apps/dashboard/src/components/AppSidebar/WorkspaceSwitcher.tsx#L103) and [useElectronBadge.ts](apps/dashboard/src/hooks/useElectronBadge.ts) docs reference it.)

Also fix the identity-merge edge while here: the switcher's last-write-wins map ([WorkspaceSwitcher.tsx:107-110](apps/dashboard/src/components/AppSidebar/WorkspaceSwitcher.tsx#L107-L110)) should **sum** counts per workspaceId if a member ever holds two identities in one workspace. Backend can do this merge server-side (groupBy workspaceId, sum) — cleaner than client-side merging.

### 3.5 What does NOT change on the backend

- Zero mutators (`markChannelAsViewed`, `markChannelUnreadFrom`, `markMissedCallsAsRead`, activity read mutators) — untouched; they remain the write path that Zero syncs.
- Activity creation side-effects — untouched. No row is created or suppressed differently.
- Classification pipeline — untouched (SKIP = nothing this phase; rework deferred).
- `userMissedCalls` query, Calls rail — untouched.

## 4. Frontend changes

### 4.1 New shared hook `useWorkspaceUnreadCounts`

`apps/dashboard/src/hooks/useWorkspaceUnreadCounts.ts`:

```ts
type WorkspaceUnread = { dmCount: number; bellCount: number; callCount: number; count: number };
// exposes:
//   byWorkspace: Record<workspaceId, WorkspaceUnread>
//   totalAllWorkspaces: number          // Σ count — dock value
//   activeWorkspace: WorkspaceUnread | undefined
//   refetch: () => void
```

Behavior:
- Poll `GET /activity/workspace-counts` every **30s** (matches current switcher cadence)
- **Refetch triggers** (perceived-instant updates): window focus, `visibilitychange → visible`, and a custom app event `unread:refetch` dispatched after read-mutations
- Single instance mounted high in the tree (next to `ElectronBadgeSync` in [AppRoot.tsx:968](apps/dashboard/src/routes/AppRoot.tsx#L968)); consumers subscribe via context, not by each spawning their own poll

Read-mutation refetch triggers (dispatch `unread:refetch` on success):
- `markChannelAsViewed` / `markChannelUnreadFrom` / `closeDm` / `reopenDm` call sites (ChatList, ConversationPanel)
- Activity `markAsRead` / `markAsReadByFilter` call sites (ActivityListView)
- `markMissedCallsAsRead` call site ([CallHistoryScreen.tsx:585](apps/dashboard/src/routes/CallHistoryScreen/CallHistoryScreen.tsx#L585))

Implementation note: wrap the mutate calls or hook into the existing state-machine `SET_*` events — whichever is less invasive; the state machine already observes these mutations (its `unreadActivities`/`userChannelStatuses` context updates on them), so emitting `unread:refetch` from the same observers avoids touching every call site.

### 4.2 `useElectronBadge` — source switch ([useElectronBadge.ts](apps/dashboard/src/hooks/useElectronBadge.ts))

```ts
// before: const unreadCounts = useAllUnreadCount(); total = Σ values
// after:  const { totalAllWorkspaces } = useWorkspaceUnreadCounts();
//         api.setBadgeCount(totalAllWorkspaces);
```

Remove `useAllUnreadCount` import; update the doc comment (the "active workspace only / follow-up" note is resolved by this change). Behind flag `DOCK_BADGE_POLL_SOURCE` for rollout.

### 4.3 `WorkspaceSwitcher` — migrate to shared hook

Replace private `fetchActivityCounts` + `setInterval` + `activityCounts` state ([WorkspaceSwitcher.tsx:100-170](apps/dashboard/src/components/AppSidebar/WorkspaceSwitcher.tsx#L100-L170)) with `useWorkspaceUnreadCounts().byWorkspace`. Rendered value unchanged (`count` per row). The server-side identity merge (3.4) removes the last-write-wins risk.

### 4.4 DM rail badge + reaction dot ([AppSidebar.tsx:328, 415-460](apps/dashboard/src/components/AppSidebar/AppSidebar.tsx#L328))

On the `/chat/dm` rail item:
- **Numeric badge** = `activeWorkspace.dmCount` (styled like the missed-call badge: `99+` cap, same classes)
- **Reaction dot** when `dmCount === 0` and any unread `added_v2` activity exists in a DM/GROUP_DM channel — replaces/extends `hasPendingDirectMessages` (currently "any DM unread > 0"); new source: the existing `unreadActivities` state (filter: `added_v2` + channel is DM)
- Bell rail item: numeric badge unchanged (`useUnreadActivitiesCount`), **plus reaction dot** when its count is 0 and unread `added_v2` rows exist in non-DM channels
- Calls rail: unchanged (existing `useMissedCallCount`)

### 4.5 `useUnreadActivitiesCount` — filter updates ([useUnreadActivitiesCount.ts](apps/dashboard/src/hooks/useUnreadActivitiesCount.ts))

New rules to match the server's bellCount exactly:
- **Add** closed-channel exclusion (rows whose channel `isClosed` — the query already relates `channel`, so filter on the related field)
- **Add** `direct_message` full exclusion (currently the ACTIONABLE/FYI gate; since dmCount owns DM top-levels, all legacy `direct_message` rows go to zero)
- Keep: `added_v2`, `removed`, `missed_call`, `SKIP` exclusions
- Decide ERROR (open Q1): current hook counts ERROR (only SKIP filtered); server plan excludes — must match whichever you pick

Derive from the shared filter constant (3.1) so client and server can't drift.

### 4.6 Per-channel DM badges — mention subtraction (DmsPage, UnreadsInbox, MobileChatDirectory, GlobalCommandMenu)

All four consume `useAllUnreadCount()` for per-channel badges. For GROUP_DM channels, the displayed badge must subtract that channel's unread top-level mention activities (Q1 decision), so the DM list sums to the DM rail number.

Implementation: extend `useAllUnreadCount` — it already has both `unreadActivities` and `userChannelStatuses` in scope ([useUnreadCount.ts:12,14](apps/dashboard/src/hooks/useUnreadCount.ts#L12)); for GROUP_DM channels compute `status.unreadCount − unreadMentionRows(channelId)` and floor at 0. Single place, all consumers fixed.

### 4.7 `useAllUnreadCount` — closed-channel filter

It derives from `visibleChannels` (already `isClosed = false` via `userVisibleChannelsV3`) — the channel-half is already correct. The activity-half (non-DM rows) needs the closed-channel exclusion added to match bellCount. Fold into 4.5/4.6's shared-constant refactor.

### 4.8 Bell count vs feed display (unchanged feed)

The bell *feed* continues to show rows the *count* excludes (missed calls visible in feed? — today yes, as cards; reactions may render in feed too). We do not change feed rendering this phase. Note the deliberate consequence: bell number may be less than visible feed items (e.g., a missed call renders as a card but counts in the call shelf). Acceptable per "keep activities as they were."

## 5. The three open questions

**Q1 — ERROR rows:** count them (like PENDING) or exclude them (treat as classifier failure = no badge)? My lean: **exclude** — an infra failure shouldn't create badge noise; rows self-heal to PENDING on retry. But counting is the lower-drama default. Need your verdict.

**Q2 — dmCount mention-subtraction precision:** activity rows vs conversation rows are 1:1 for top-level GROUP_DM messages per recipient (verified in the handlers), but I want a unit-test matrix covering: multi-mention of the same user in one message, mention + keyword overlap (one row per user — [messages-handler dedupes](apps/backend/src/zero/side-effects/tables/messages-handler.ts#L707)), and mark-unread re-adding rows, before locking the SQL. Flagging as risk, not blocker.

**Q3 — switcher UI decomposition:** show `12` plain, or show/hover `7 DM · 4 bell · 1 call`? Data arrives either way; purely a UI choice. Default: plain `12` this phase.

## 6. Rollout

1. **Phase 0:** frozen-counter fix (3.3) — independently shippable, fixes live DM badge bug today.
2. **Phase 1a:** backend — shared filter constant + endpoint extension (additive) + server-side identity merge. Old clients unaffected.
3. **Phase 1b:** `useWorkspaceUnreadCounts` + WorkspaceSwitcher migration. Corrected numbers; no UI change.
4. **Phase 1c:** bell filter updates (4.5) + per-channel DM subtraction (4.6) + rail badges/dots (4.4). Bell numbers change visibly (closed-channel rows drop, legacy DM rows drop).
5. **Phase 1d:** dock source switch (4.2) behind `DOCK_BADGE_POLL_SOURCE` flag, default off; compare `|zeroSum − polledTotal|` in dev builds; flip on.
6. **Monitoring:** endpoint latency + DB load on the new aggregates; alert if p95 > 200ms. Log invariant violations (`count ≠ dm+bell+call`) — should be structurally impossible.

## 7. Test matrix

**Backend unit:** bellCount filter rules (each excluded type); callCount; dmCount (open/closed, DM/GROUP_DM, subtraction cases per Q2); identity merge; `count` invariant; backward-compat `count` field.
**Backend integration:** member with 2 workspaces; member with 2 identities in 1 workspace; empty state; all-read state.
**Frontend unit:** `useWorkspaceUnreadCounts` polling/focus/refetch logic; `useUnreadActivitiesCount` new filters; `useAllUnreadCount` GROUP_DM subtraction.
**E2E (Electron):** the 18-row matrix as scenarios — DM arrives → dock+1, switcher+1, DM rail+1, bell 0; GROUP_DM mention → bell+1, dm rail 0; missed call → calls rail+1, workplace+1; reaction → dot only; read everything → dock clears on focus-refetch; closed channel → nothing counts; dock 32 → switcher sums 32 → DM list + bell + calls sum to 12 for workspace A.
**Soak:** frozen-counter regression (DM to recently-viewed channel must increment); poll endpoint load.

## 8. Files touched (summary)

| File | Change |
|---|---|
| `apps/backend/src/services/activity/unreadActivityFilters.ts` | **new** — shared filter constant |
| `apps/backend/src/services/activity/activityService.ts` | extend `getWorkspaceActivityCounts` (dm/bell/call) |
| `apps/backend/src/zero/side-effects/tables/conversations-handler.ts` | bump `channel_stats.lastActivityAt` on insert |
| `apps/dashboard/src/hooks/useWorkspaceUnreadCounts.ts` | **new** — shared polling hook + context |
| `apps/dashboard/src/hooks/useElectronBadge.ts` | source switch to `totalAllWorkspaces` |
| `apps/dashboard/src/components/AppSidebar/WorkspaceSwitcher.tsx` | migrate to shared hook |
| `apps/dashboard/src/components/AppSidebar/AppSidebar.tsx` | DM rail numeric badge + reaction dots |
| `apps/dashboard/src/hooks/useUnreadActivitiesCount.ts` | closed-channel + direct_message filters, shared constant |
| `apps/dashboard/src/hooks/useUnreadCount.ts` | GROUP_DM mention subtraction for per-channel badges |
| Read-mutation call sites / state machine | emit `unread:refetch` after read mutations |
| `apps/dashboard/src/routes/AppRoot.tsx` | mount shared hook provider |

Not touched: Zero mutators, activity creation side-effects, classification pipeline, Calls rail query, bell feed rendering.
