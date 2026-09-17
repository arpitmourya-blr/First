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
- The dock is never designed separately — it renders `Σ workspace counts` from the same polling state as the switcher. Dock shows 32 → switcher rows sum to 32. A workspace shows 12 → its DM rail + bell + calls badges sum to 12.
- Bell *feed behavior* stays as-is (rows visible in the feed unchanged). Only *counting* changes.

### Transport split (decided)

| Surface | Transport | Source |
|---|---|---|
| Dock badge | **HTTP poll** (30s + refetch triggers) | Σ workspace counts via shared hook |
| Workplace switcher | **HTTP poll** (same hook, same state) | per-workspace `count` |
| DM rail badge | **Zero sync (live)** | Σ per-channel DM unreads, client-side (`useDmUnreadCount`, new) |
| Bell badge | **Zero sync (live)** | `useUnreadActivitiesCount` (existing, filters updated) |
| Calls rail badge | **Zero sync (live)** | `useMissedCallCount` (existing, untouched) |

The server computes the same three parts for the polled total; the client derives the same three numbers from Zero for the rails. Same predicate, same rows, two transports. Rails are live; dock/switcher lag ≤30s (accepted).

**Why this dissolves the dmCount-subtraction precision risk:** the DM rail and DM list badges derive from the same Zero state through the same hook — the rail *is* the sum of the list, so drill-down consistency is automatic. Edge cases (multi-mention, keyword overlap, mark-unread timing) affect client and server computations identically, so they cannot cause user-visible divergence — consistency is guaranteed even if a rare edge case makes both sides "off" in the same way. The remaining requirement is **implementation parity** (§3.1), enforced by tests.

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

¹ dmCount subtracts unread top-level mention activities in GROUP_DM channels (mention wins the bucket). The subtraction is applied **per-channel on the client** (rail + list badges, same hook) and **in aggregate on the server** (same predicate) — see §3.1.
² Reaction rows create a **dot** (no number) on the DM channel row and the DM rail (only when the numeric badge is 0), and on the bell. Cleared on view. Never a number anywhere.
³ Not counted anywhere; visible only when opening the channel itself.
⁴ SKIP = not counted anywhere; current bell filter logic kept verbatim; classification rework deferred to a later PR (Q1 decided → (b), see §5).
⁵ ERROR rows counted (decided): LLM failure must not lose potentially important counts; revisit in the classification PR.
⁶ Excluded from all counts; reappear if the channel is reopened (accepted).

**Key behavioral changes vs today:**
- Missed calls: workplace/dock start counting them (via callCount).
- Channel removals: workplace stops counting them (today's unfiltered server count includes them).
- DM thread replies: dock starts counting them (bell shelf — today's dock excluded threads).
- Tickets/channel-less: dock starts counting them.
- Closed-channel rows: bell stops counting them (new filter).
- GROUP_DM top-level mentions: dm shelf subtracts them (mention wins).
- Bell: `removed` and `missed_call` filters stay; **new** closed-channel filter and full `direct_message` (legacy) exclusion.

## 3. Backend changes

### 3.1 Shared predicate constant (new, in `packages/shared`)

The single source of truth for "what counts as a bell-shelf unread activity" and "what the dm shelf subtracts," imported by both the dashboard hooks and the backend endpoint:

```ts
// packages/shared/src/unread/bellCountRules.ts
export const BELL_COUNT_RULES = {
  excludedActorActions: ['added_v2', 'removed'],
  excludedCalls: { actionSource: 'call', actorAction: 'missed_call' },
  excludedClassifications: ['SKIP'],            // ERROR + PENDING counted
  excludedLegacyDirectMessages: true,           // actorAction = 'direct_message' → dm shelf's domain
  excludeClosedChannels: true,
  dmShelf: {
    channelScopes: ['DM', 'GROUP_DM'],
    subtractUnreadTopLevelMentions: true,       // GROUP_DM only; isThreadActivity = false
  },
} as const;
```

Client (zql filter in the hooks) and server (Prisma `where` in the endpoint) each adapt these rules to their query builder. **Parity tests** (§7) assert both adapters return the same set on golden fixtures — this replaces the earlier "subtraction precision" risk with a mechanical guarantee.

### 3.2 Extend `activityService.getWorkspaceActivityCounts` ([activityService.ts:716](apps/backend/src/services/activity/activityService.ts#L716))

**Response shape — minimal** (no rail consumes the poll anymore):

```ts
// today:  { workspaceId, userId, count }              // count = raw unread activities (unfiltered)
// target: { workspaceId, count }                      // count = dmCount + bellCount + callCount
```

**Why `userId` exists today:** an org member holds one `user` identity per workspace ([activityService.ts:725-735](apps/backend/src/services/activity/activityService.ts#L725-L735)), and the response is a direct dump of a Prisma `groupBy userId` — one row per identity, with `userId` as the group key (and the disambiguator if a member holds two identities in one workspace). The client then merges rows per workspace itself, last-write-wins ([WorkspaceSwitcher.tsx:106-110](apps/dashboard/src/components/AppSidebar/WorkspaceSwitcher.tsx#L106-L110)).

**Why it can be dropped:** the merge moves server-side (`groupBy workspaceId, SUM`), so `workspaceId` becomes the unique row key and a row may aggregate several identities — `userId` no longer identifies anything. The only consumer (WorkspaceSwitcher) never reads it; it reads `workspaceId` + `count` only. Dropping it removes the client-side last-write-wins race entirely. (Verify no out-of-repo consumer — e.g. a mobile app — reads `userId` before deploy, same check as the `count` semantics change below.)

Computed internally, not exposed:

- **`dmCount`** = `SUM(channel_user_status.unreadCount)` over the user's open (`isClosed = false`, `isDeleted = false`) DM/GROUP_DM channels **minus** unread top-level mention activities in GROUP_DM channels (per §3.1 rules; floored at 0 per channel).
- **`bellCount`** = `COUNT(activities)` where `isRead = false` and the §3.1 bell rules (excludes `added_v2`, `removed`, `missed_call`, SKIP, legacy `direct_message`; excludes closed channels; counts ERROR/PENDING).
- **`callCount`** = `COUNT(activities)` where `actionSource = 'call' AND actorAction = 'missed_call' AND isRead = false` — identical semantics to the Calls rail's `userMissedCalls` Zero query ([queries.ts:2463](packages/shared/src/zero/queries.ts#L2463)).

Also merge identities server-side: `groupBy workspaceId, SUM` — removes the switcher's last-write-wins merge risk ([WorkspaceSwitcher.tsx:107-110](apps/dashboard/src/components/AppSidebar/WorkspaceSwitcher.tsx#L107-L110)) if a member ever holds two identities in one workspace.

⚠️ **Rollout note:** the `count` field's *meaning* changes (raw unfiltered → dm+bell+call). Only known consumer is the switcher (migrating in the same release); verify no mobile/other consumers before deploy.

### 3.3 Fix the frozen DM counter (Phase 0 — prerequisite)

`handleUnreadCount` skips recompute when `channel_stats.lastActivityAt <= lastViewedAt` ([unreadCountUtlis.ts:37](apps/backend/src/zero/utils/unreadCountUtlis.ts#L37)), but ordinary messages never update `channel_stats.lastActivityAt` — only channel creation, membership changes, and calls do. Once you've viewed a DM channel, every future recompute for it is silently skipped → `unreadCount` frozen at 0.

**Fix:** in `conversations-handler.ts` `onInsert` ([conversations-handler.ts:39-45](apps/backend/src/zero/side-effects/tables/conversations-handler.ts#L39-L45)), bump `channel_stats.lastActivityAt = conversation.createdAt` before calling `handleUnreadCount`. ~3 lines in one file.

Now doubly critical: the DM rail renders `channel_user_status.unreadCount` **directly** via Zero — without this fix the new rail badge is frozen for every viewed channel.

### 3.4 What does NOT change on the backend

- Zero mutators (`markChannelAsViewed`, `markChannelUnreadFrom`, `markMissedCallsAsRead`, activity read mutators) — untouched; they remain the write path Zero syncs.
- Activity creation side-effects — untouched. No row is created or suppressed differently.
- Classification pipeline — untouched this phase (Q1 → (b): leave the pipeline as-is; SKIP handling per §2 note 4).
- `userMissedCalls` query, Calls rail — untouched.

## 4. Frontend changes

### 4.1 Shared polling hook `useWorkspaceUnreadCounts` — dock + switcher only

`apps/dashboard/src/hooks/useWorkspaceUnreadCounts.ts`:

```ts
// exposes:
//   byWorkspace: Record<workspaceId, number>
//   totalAllWorkspaces: number     // dock value
//   refetch: () => void
```

- Polls `GET /activity/workspace-counts` every **30s**
- Refetch triggers (perceived-instant dock updates): window focus, `visibilitychange → visible`, custom `unread:refetch` event emitted after read-mutations
- Single instance mounted next to `ElectronBadgeSync` ([AppRoot.tsx:968](apps/dashboard/src/routes/AppRoot.tsx#L968)); consumers via context
- No rail consumes this hook — rails are Zero-synced

Read-mutation refetch triggers: emit `unread:refetch` from the state-machine observers that already track these mutations (cleaner than touching every call site): `markChannelAsViewed` / `markChannelUnreadFrom` / `closeDm` / `reopenDm`, activity `markAsRead` / `markAsReadByFilter`, `markMissedCallsAsRead` ([CallHistoryScreen.tsx:585](apps/dashboard/src/routes/CallHistoryScreen/CallHistoryScreen.tsx#L585)).

### 4.2 `useElectronBadge` — source switch ([useElectronBadge.ts](apps/dashboard/src/hooks/useElectronBadge.ts))

```ts
// before: total = Σ useAllUnreadCount() values
// after:  const { totalAllWorkspaces } = useWorkspaceUnreadCounts();
```

Update the doc comment (the "active workspace only" limitation is resolved). Behind flag `DOCK_BADGE_POLL_SOURCE` — follow the existing env-flag pattern in [config.ts](apps/dashboard/src/config.ts) (like `VITE_ENABLE_SUMMARY_ACTION_BUTTON`).

### 4.3 `WorkspaceSwitcher` — migrate to shared hook

Replace private `fetchActivityCounts` + `setInterval` + state ([WorkspaceSwitcher.tsx:100-170](apps/dashboard/src/components/AppSidebar/WorkspaceSwitcher.tsx#L100-L170)) with `byWorkspace` from the shared hook. Rendered value: plain number (decided — decomposition lives on the rail badges).

### 4.4 New `useDmUnreadCount` — Zero-synced DM rail badge

`apps/dashboard/src/hooks/useDmUnreadCount.ts`:

```ts
// Σ over visible DM/GROUP_DM channels of:
//   max(0, channelUserStatus.unreadCount − unreadTopLevelMentionRows(channelId))
// where mention rows = unread activities, actorAction IN (mentioned_user, group_mention),
//   actionSource = 'message', isThreadActivity = false, channel scopeType = GROUP_DM
```

- Zero-synced (live) — derived from the same state as the DM list badges (see 4.6), so rail = Σ list by construction
- Numeric badge on the `/chat/dm` rail item (missed-call styling, `99+` cap)
- **Reaction dot** when the number is 0 and unread `added_v2` rows exist in DM/GROUP_DM channels — extends `hasPendingDirectMessages` ([AppSidebar.tsx:328](apps/dashboard/src/components/AppSidebar/AppSidebar.tsx#L328)); cleared on view
- Bell rail item: numeric badge unchanged, **plus reaction dot** when its count is 0 and unread `added_v2` rows exist in non-DM channels

### 4.5 `useUnreadActivitiesCount` — filter updates ([useUnreadActivitiesCount.ts](apps/dashboard/src/hooks/useUnreadActivitiesCount.ts))

Match the server's bellCount exactly, derived from the shared rules (§3.1):
- **Add** closed-channel exclusion (query already relates `channel`)
- **Add** full `direct_message` exclusion (replaces the ACTIONABLE/FYI gate — legacy rows are dm-shelf content)
- Keep: `added_v2`, `removed`, `missed_call`, `SKIP` exclusions
- ERROR and PENDING: counted (current behavior for ERROR; no change needed — only SKIP is filtered today)

### 4.6 `useAllUnreadCount` — per-channel GROUP_DM mention subtraction

The DM list badges (and 4.4's rail) must subtract that channel's unread top-level mention rows for GROUP_DM channels, floored at 0. `useAllUnreadCount` already has `unreadActivities` and `userChannelStatuses` in scope ([useUnreadCount.ts:12,14](apps/dashboard/src/hooks/useUnreadCount.ts#L12)) — extend it once; all consumers fixed (DmsPage, UnreadsInbox, MobileChatDirectory, GlobalCommandMenu). The activity-half (non-DM rows) also gets the closed-channel exclusion to match bellCount.

### 4.7 Bell count vs feed display (unchanged feed)

The bell *feed* continues to render rows the *count* excludes (e.g., missed-call cards). Deliberate: "keep activities as they were." The bell number may be less than visible feed items — accepted.

## 5. Resolved decisions

**Q1 — "do not classify anything to skip" → decided (b):** leave the classification pipeline completely untouched this phase (current behavior: non-DM SKIP already coerces; DM SKIP rows deleted; new SKIP rows nearly impossible since XYNE-17185) — implement the "nothing is SKIP" state properly in the later classification PR. Zero classification code touched. Counts behave identically either way (SKIP never counts).

## 6. Rollout

1. **Phase 0:** frozen-counter fix (§3.3) — independently shippable; fixes the live DM badge bug today.
2. **Phase 1a:** backend — shared rules constant + endpoint change (`count` = dm+bell+call, identity merge). ⚠️ semantic change of `count`; verify external consumers first.
3. **Phase 1b:** `useWorkspaceUnreadCounts` + WorkspaceSwitcher migration. Corrected numbers; no UI change.
4. **Phase 1c:** bell filter updates (§4.5) + `useDmUnreadCount` rail badge + per-channel subtraction (§4.6) + reaction dots (§4.4). Rail numbers change visibly.
5. **Phase 1d:** dock source switch (§4.2) behind `DOCK_BADGE_POLL_SOURCE`, default off; compare `|zeroSum − polledTotal|` in dev; flip on. (Classification handling per Q1 → (b): no changes this phase.)
6. **Monitoring:** endpoint latency + DB load (alert if p95 > 200ms); log invariant violations — `count ≠ dm+bell+call` (structurally impossible server-side) and `switcher(activeWs) ≠ dmRail + bell + calls` beyond the 30s freshness window.

## 7. Test matrix

**Parity (new, replaces the precision risk):** golden fixture activity sets → client predicate (zql/hook) and server predicate (Prisma) must return identical counts. Fixtures cover: plain GROUP_DM message; mention; mention+keyword overlap (one row — [handler dedupes](apps/backend/src/zero/side-effects/tables/messages-handler.ts#L743)); mention+thread-reply rows for the same user; multi-user mention (3 rows, per-user 1:1); mark-unread re-marking (rows and counter re-align); read-clearing both together; closed-channel rows; ERROR/PENDING/SKIP rows.
**Backend unit:** dmCount (open/closed, DM/GROUP_DM, subtraction cases); bellCount rules; callCount; identity merge; `count` sum invariant.
**Frontend unit:** `useWorkspaceUnreadCounts` polling/focus/refetch; `useDmUnreadCount` subtraction + flooring; `useUnreadActivitiesCount` new filters.
**E2E (Electron):** the §2 matrix as scenarios — DM arrives → dm rail +1 live, dock/switcher +1 on next poll; GROUP_DM mention → bell +1, dm rail 0; missed call → calls rail +1, workplace +1; reaction → dot only; read all → rails clear instantly (Zero), dock clears on refetch trigger; closed channel → nothing counts; dock 32 → switcher sums 32 → workspace A's rails sum to 12.
**Soak:** frozen-counter regression (DM to a recently-viewed channel must increment the rail); poll endpoint load.

## 8. Files touched (summary)

| File | Change |
|---|---|
| `packages/shared/src/unread/bellCountRules.ts` | **new** — shared count rules constant |
| `apps/backend/src/services/activity/activityService.ts` | `getWorkspaceActivityCounts` → `{workspaceId, count}` with dm+bell+call |
| `apps/backend/src/zero/side-effects/tables/conversations-handler.ts` | bump `channel_stats.lastActivityAt` on insert (Phase 0) |
| `apps/dashboard/src/hooks/useWorkspaceUnreadCounts.ts` | **new** — shared polling hook (dock + switcher only) |
| `apps/dashboard/src/hooks/useDmUnreadCount.ts` | **new** — Zero-synced DM rail badge with mention subtraction |
| `apps/dashboard/src/hooks/useElectronBadge.ts` | source switch to `totalAllWorkspaces` |
| `apps/dashboard/src/components/AppSidebar/WorkspaceSwitcher.tsx` | migrate to shared hook |
| `apps/dashboard/src/components/AppSidebar/AppSidebar.tsx` | DM rail numeric badge + reaction dots (dm + bell rails) |
| `apps/dashboard/src/hooks/useUnreadActivitiesCount.ts` | closed-channel + direct_message filters from shared rules |
| `apps/dashboard/src/hooks/useUnreadCount.ts` | GROUP_DM mention subtraction + closed-channel on activity half |
| state machine / read-mutation observers | emit `unread:refetch` after read mutations |
| `apps/dashboard/src/routes/AppRoot.tsx` | mount shared hook provider |

Not touched: Zero mutators, activity creation side-effects, classification pipeline (Q1 → (b), left as-is this phase), Calls rail query, bell feed rendering.
