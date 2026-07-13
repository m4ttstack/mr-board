# Team MR Board — Design

**Date:** 2026-07-13
**Status:** Approved, ready for implementation planning

## Summary

Turn mr-board from a single-person board into a team review hub. One config drives
boards for every teammate. You can view one member at a time (as today) or a merged
"All" view of the whole team's open MRs, with selectable grouping (age, author, status,
pipeline) and sorting (oldest-first, pipeline status, review progress). Navigation is a
sidebar; view state lives in the URL and localStorage.

The server already fetches every open MR in one call and holds a single cached snapshot;
this change reshapes that snapshot from project-grouped to member-tagged-and-flat, and
moves all grouping/sorting/filtering into the client so view switches are instant with
zero round-trips.

## Requirements

- Config-driven roster of teammates (single shared repo/project in the target use case).
- View one teammate at a time (current behavior).
- Merged "All" view of the whole team's open MRs.
- Grouping dimensions: age (default), author, status, pipeline.
- Sort orders: oldest-first (default), pipeline status, review progress.
- Grouping/sort controls available on both "All" and individual member views.
- Age buckets: by day then weekly, measured from MR creation date.
- Sidebar navigation with per-member sprite avatars and open-MR counts.
- View state (member + grouping + sort) shareable via URL and remembered via localStorage.

## Architecture Decision

**Server sends a flat, member-tagged list; the client does all grouping/sorting/filtering.**

The dataset is small and already fully in memory. A flat snapshot keeps the 60s
stale-while-revalidate cache view-agnostic (no per-view cache keys), puts all view logic
in one place, and makes URL-driven instant switching trivial. The rejected alternative —
server computes groups from query params — multiplies cache keys, adds a round-trip per
switch, and splits grouping logic across server and client for no benefit at this scale.

## Config

`config.json` gains an ordered `members` array and drops the single top-level `username`.

```json
{
  "gitlabHost": "https://gitlab.com",
  "projects": ["assured/assured-dev"],
  "members": [
    { "username": "alice", "name": "Alice Ng" },
    { "username": "bob" }
  ],
  "title": "MR Board",
  "port": 7930
}
```

- `members`: ordered array; `username` required, `name` optional. Sidebar order follows
  this array. Missing `name` falls back to the GitLab profile lookup, then to the username.
- `projects`: shared team-level list (one entry in the target use case). Unchanged meaning.
- `title`: page heading / tab title. No longer implies a single author.
- `username` (top-level): **removed.**

`loadConfig` validates that `members` is a non-empty array and every entry has a
`username`. `gitlabHost` and `projects` validation is unchanged.

## Data Model & Server

The snapshot becomes flat and member-tagged instead of project-grouped.

```ts
interface BoardMR {
  // ...existing fields...
  createdAt: string | null;                          // NEW — age bucketing + oldest-first sort
  author: { username: string; name: string | null }; // NEW — owning member
  pipelineState: "passed" | "running" | "failed" | "none"; // NEW — pipeline grouping/sort
}

interface Snapshot {
  mrs: BoardMR[];       // flat: all members, all projects, unsorted
  fetchedAt: number;
  fetchError: string | null;
}
```

- `buildGroups` → `buildBoard`. The fetch is identical
  (`fetchPullRequests({ state: "opened" })`). The filter changes from a single username to
  the **set** of configured member usernames; still requires open, not-draft, in a
  configured project. Each surviving MR is tagged with its author identity, `createdAt`,
  and a derived `pipelineState`. Output is a flat, unsorted array — the client owns all
  ordering.
- `pipelineState` is derived in the data layer from the existing pipeline signals
  (`blockers.pipelineFailing` / `blockers.pipelineRunning`, plus the raw PR pipeline data)
  into a single field so both grouping and sorting can key off it cleanly.
- **Roster:** the server computes a `members` array of `{ username, name, count }` where
  `count` is the member's open-MR count, so the sidebar renders every member (including
  those with zero MRs) instantly. `name` resolution moves from the single-owner lookup to a
  per-member profile fetch, cached with a long TTL (like the current `OWNER_TTL_MS`). One
  member's failed lookup never blocks the others.
- `/data.json` returns `{ title, members, mrs, fetchedAt, fetchError }`. No query params —
  view state is entirely client-side. The `SnapshotCache` (60s stale-while-revalidate) is
  unchanged.
- `projectPathFromWebUrl` is retained (still enforces the project filter).

## Client

### View state

Read from URL query params, falling back to localStorage, falling back to defaults:

- `member`: `"all"` (default) or a username
- `group`: `age` (default) · `author` · `status` · `pipeline`
- `sort`: `oldest` (default) · `pipeline` · `progress`

On any control change: update the URL via `history.replaceState` (no reload) **and**
localStorage. A bare visit with no query params restores from localStorage, else defaults.
Unknown `member` (removed from config) falls back to "All"; invalid `group`/`sort` fall
back to defaults.

### Layout — sidebar + main pane

```
┌──────────────┬─────────────────────────────────┐
│ ◉ All      7 │  [group: age ▾]  [sort: oldest ▾]│
│ ⚈ alice    2 │                                  │
│ ⚈ bob      3 │  ▸ Today (2)                      │
│ ⚈ cara     2 │      ● fix login retry  …         │
│              │  ▸ 2 days ago (1)                 │
│  (theme,     │      ● add board sort   …         │
│   view tgl)  │  ▸ Last week (4)   …              │
└──────────────┴─────────────────────────────────┘
```

- **Sidebar:** "All" (◉) then each member with their pixel-sprite avatar and open-MR
  count. Active item highlighted. Zero-MR members greyed with `0`. Theme + rows/grid
  toggles live in the sidebar (or a compact header — minor).
- **Header identity:** on a member view, shows `--author @alice` + that member's sprite; on
  "All", shows the team title.

### View logic — pure functions (`view.ts`, unit-tested, no React)

- `filterByMember(mrs, member)`
- `sortMRs(mrs, sort)` — `oldest` by `createdAt` ascending; `pipeline` by CI-state rank;
  `progress` by approvals `given / required`. Null-date handling defined and tested.
- `groupMRs(mrs, group)` → ordered `{ label, mrs }[]`.
  - **age:** buckets by day then weekly from `createdAt` — Today, Yesterday, "N days ago"
    (up to ~6 days), Last week, 2 weeks ago, Older. Chronological order.
  - **author:** one group per member, in config order.
  - **status:** grouped by the prioritized status phrase (needs review / n-of-m approved /
    approved / ci failing / conflicts), severity order.
  - **pipeline:** grouped by `pipelineState`, severity order (failed → running → none →
    passed, or as tuned).
- Groups are ordered naturally per dimension; MRs within each group follow the active sort.

### Reuse

`RowView`, `GridView`, `Panel`, `Watching`, `StatusDot`, `StatusPhrase`, `MetaTokens`,
`TicketLink`, `OwnerSprite`, and the copy pieces are reused as-is. A group is just a
`Panel` with a computed label and count.

### Copy-for-Slack

Copies the **current view's** MRs (respecting the member filter and grouping), so "copy
Bob's ready MRs" and "copy the whole team" both work from the one button.

## Error Handling & Edge Cases

- Member with zero open MRs: sidebar row greyed with `0`; member view shows the existing
  "nothing waiting on review" empty state.
- Unknown `member` in URL: fall back to "All".
- Invalid `group` / `sort` in URL: fall back to defaults.
- Per-member profile lookup fails: fall back to config `name`, then username; never blocks
  other members or the board.
- `fetchError` banner and stale-while-revalidate behavior unchanged.

## Testing (bun test)

- `buildBoard`: filters to the member set, tags author, excludes non-members / drafts /
  out-of-project MRs, derives `pipelineState`, computes roster counts.
- `view.ts` pure functions:
  - `sortMRs` — all three orders, including null-date handling.
  - `groupMRs` — age bucket boundaries (today / yesterday / week edges), author order,
    status and pipeline severity order.
  - `filterByMember`.
- Existing `SnapshotCache` and ticket-extraction tests stay green.
