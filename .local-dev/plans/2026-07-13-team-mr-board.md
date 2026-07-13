# Team MR Board Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn mr-board from a single-user board into a config-driven team review hub with a sidebar, a merged "All" view, and selectable grouping/sorting.

**Architecture:** The server fetches all open MRs once (unchanged) and produces a flat, member-tagged snapshot. `/data.json` returns `{ title, members, mrs, ... }`. The React client reads view state (member + grouping + sort) from the URL / localStorage and computes the displayed groups entirely in-browser via pure functions in `src/view.ts`, so switching views is instant with no round-trips.

**Tech Stack:** Bun, TypeScript, React 19, `@forge-glance/sdk` (GitLab provider + `getMRDashboardProps`), `bun test`.

## Global Constraints

- Runtime is **Bun**; run tests with `bun test`, serve with `bun run serve`.
- Import local modules with explicit `.ts` / `.tsx` extensions (existing convention).
- No new dependencies.
- Token scope stays `read_api` — the board never writes to GitLab.
- The GitLab fetch stays `provider.fetchPullRequests({ state: "opened" })` — do not add per-project or per-user fetch calls.
- Members roster and view state must degrade gracefully: one member's failed profile lookup never blocks others; invalid URL/localStorage values fall back to defaults.

---

## File Structure

- `src/config.ts` — **modify.** `BoardConfig` gains `members: Member[]`, drops `username`. Extract a pure `parseConfig(raw)` from `loadConfig` for testability.
- `src/data.ts` — **modify.** `BoardMR` gains `createdAt` and `pipelineState`. `Snapshot` becomes `{ mrs, fetchedAt, fetchError }`. `buildGroups` → `buildBoard` returning a flat `BoardMR[]`.
- `src/cache.ts` — **modify.** Mechanical rename `groups` → `mrs` to match the new `Snapshot`.
- `src/view.ts` — **create.** Pure view logic: `filterByMember`, `sortMRs`, `groupMRs`, `parseViewState`, `serializeViewState`, plus the group/sort key enums and types.
- `src/server.ts` — **modify.** Wire `buildBoard`, build the members roster (with cached per-member name lookups), change `/data.json` shape.
- `src/client.tsx` — **modify.** Sidebar + main-pane layout, URL/localStorage view state, render via `src/view.ts`, per-member header identity, copy current view.
- `src/style.css` — **modify.** Sidebar layout styles.
- `src/__tests__/board.test.ts` — **modify.** Update fixture to `members`; retarget tests to `buildBoard` and the new `Snapshot` shape.
- `src/__tests__/view.test.ts` — **create.** Unit tests for every `src/view.ts` function.
- `config.example.json` — **modify.** New `members` shape.
- `README.md` — **modify.** Document the team config and views.

---

## Task 1: Config with members roster

**Files:**
- Modify: `src/config.ts`
- Modify: `config.example.json`
- Test: `src/__tests__/config.test.ts` (create)

**Interfaces:**
- Produces: `interface Member { username: string; name?: string }`; `BoardConfig` with `members: Member[]` (no `username`); `parseConfig(raw: string): BoardConfig`; `loadConfig(): BoardConfig`; `loadGitLabToken(): string` (unchanged).

- [ ] **Step 1: Write the failing test**

Create `src/__tests__/config.test.ts`:

```ts
import { describe, expect, test } from "bun:test";
import { parseConfig } from "../config.ts";

const base = {
  gitlabHost: "https://gitlab.com",
  projects: ["org/repo"],
  members: [{ username: "alice", name: "Alice Ng" }, { username: "bob" }],
};

describe("parseConfig", () => {
  test("parses members and applies defaults", () => {
    const cfg = parseConfig(JSON.stringify(base));
    expect(cfg.members).toEqual([{ username: "alice", name: "Alice Ng" }, { username: "bob" }]);
    expect(cfg.title).toBe("MRs ready for review");
    expect(cfg.port).toBe(7930);
  });

  test("throws when members is missing or empty", () => {
    expect(() => parseConfig(JSON.stringify({ ...base, members: [] }))).toThrow(/members/);
    const { members, ...noMembers } = base;
    expect(() => parseConfig(JSON.stringify(noMembers))).toThrow(/members/);
  });

  test("throws when a member has no username", () => {
    expect(() => parseConfig(JSON.stringify({ ...base, members: [{ name: "No User" }] }))).toThrow(/username/);
  });

  test("throws when gitlabHost or projects missing", () => {
    const { gitlabHost, ...noHost } = base;
    expect(() => parseConfig(JSON.stringify(noHost))).toThrow(/gitlabHost/);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bun test src/__tests__/config.test.ts`
Expected: FAIL — `parseConfig` is not exported.

- [ ] **Step 3: Implement**

Replace the contents of `src/config.ts` with:

```ts
import { readFileSync } from "fs";
import { join } from "path";
import { homedir } from "os";

export interface Member {
  username: string;
  /** Optional display name; falls back to the GitLab profile lookup, then username. */
  name?: string;
}

export interface BoardConfig {
  gitlabHost: string;
  /** GitLab project paths whose MRs are eligible, e.g. "assured/assured-dev". */
  projects: string[];
  /** Team members whose authored MRs the board shows, in sidebar order. */
  members: Member[];
  title: string;
  port: number;
}

const CONFIG_PATH = join(import.meta.dir, "..", "config.json");
const RT_SECRETS_PATH = join(homedir(), ".rt", "secrets.json");

/** Parse and validate raw config JSON. Separated from file IO for testing. */
export function parseConfig(raw: string): BoardConfig {
  const cfg = JSON.parse(raw) as Partial<BoardConfig>;
  for (const key of ["gitlabHost", "projects", "members"] as const) {
    const value = cfg[key];
    if (!value || (Array.isArray(value) && value.length === 0)) {
      throw new Error(`config.json is missing required field "${key}"`);
    }
  }
  for (const member of cfg.members!) {
    if (!member || !member.username) {
      throw new Error(`config.json has a member with no "username"`);
    }
  }
  return {
    gitlabHost: cfg.gitlabHost!,
    projects: cfg.projects!,
    members: cfg.members!,
    title: cfg.title ?? "MRs ready for review",
    port: cfg.port ?? 7930,
  };
}

export function loadConfig(): BoardConfig {
  let raw: string;
  try {
    raw = readFileSync(CONFIG_PATH, "utf8");
  } catch {
    throw new Error(`config.json not found at ${CONFIG_PATH} — copy config.example.json and fill it in`);
  }
  return parseConfig(raw);
}

export function loadGitLabToken(): string {
  if (process.env.GITLAB_TOKEN) return process.env.GITLAB_TOKEN;
  try {
    const secrets = JSON.parse(readFileSync(RT_SECRETS_PATH, "utf8"));
    if (secrets.gitlabToken) return secrets.gitlabToken;
  } catch {
    // fall through to the error below
  }
  throw new Error(`no GitLab token: set GITLAB_TOKEN or add "gitlabToken" to ${RT_SECRETS_PATH}`);
}
```

- [ ] **Step 4: Update `config.example.json`**

Replace its contents with:

```json
{
  "gitlabHost": "https://gitlab.com",
  "projects": ["group/project"],
  "members": [
    { "username": "your-gitlab-username", "name": "Your Name" },
    { "username": "teammate-username" }
  ],
  "title": "MR Board",
  "port": 7930
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `bun test src/__tests__/config.test.ts`
Expected: PASS (4 tests).

- [ ] **Step 6: Commit**

```bash
git add src/config.ts src/__tests__/config.test.ts config.example.json
git commit -m "feat: config members roster (drop single username)"
```

---

## Task 2: Flat member-tagged snapshot (data + cache)

**Files:**
- Modify: `src/data.ts`
- Modify: `src/cache.ts`
- Test: `src/__tests__/board.test.ts`

**Interfaces:**
- Consumes: `BoardConfig` with `members` (Task 1).
- Produces:
  - `type PipelineState = "passed" | "running" | "failed" | "none"`
  - `type BoardMR = MRDashboardProps & { updatedAt: string | null; createdAt: string | null; unresolvedThreads: number; pipelineState: PipelineState }` (note: `author: UserRef` comes from `MRDashboardProps`).
  - `interface Snapshot { mrs: BoardMR[]; fetchedAt: number; fetchError: string | null }`
  - `buildBoard(prs: PullRequest[], config: BoardConfig): BoardMR[]`
  - `projectPathFromWebUrl(webUrl: string, gitlabHost: string): string | null` (unchanged)
  - `SnapshotCache` constructed with `fetchMRs: () => Promise<BoardMR[]>`.

- [ ] **Step 1: Write the failing tests**

Rewrite `src/__tests__/board.test.ts`. Update the `config` fixture and the grouping tests, and keep the ticket / `projectPathFromWebUrl` / cache tests (adapted). Full file:

```ts
import { describe, expect, test } from "bun:test";
import type { PullRequest } from "@forge-glance/sdk";
import { buildBoard, projectPathFromWebUrl } from "../data.ts";
import { SnapshotCache } from "../cache.ts";
import type { BoardConfig } from "../config.ts";
import { extractTicketId } from "../ticket.ts";

const config: BoardConfig = {
  gitlabHost: "https://gitlab.com",
  projects: ["org/repo-a", "org/repo-b"],
  members: [{ username: "alice" }, { username: "bob" }],
  title: "Test board",
  port: 0,
};

function pr(overrides: Partial<PullRequest>): PullRequest {
  return {
    id: "gitlab:1",
    iid: 1,
    repositoryId: "gitlab:42",
    title: "An MR",
    description: null,
    state: "opened",
    draft: false,
    conflicts: false,
    webUrl: "https://gitlab.com/org/repo-a/-/merge_requests/1",
    sourceBranch: "feat/x",
    targetBranch: "main",
    createdAt: "2026-07-09T12:00:00Z",
    updatedAt: "2026-07-10T12:00:00Z",
    sha: null,
    author: { id: "gitlab:7", username: "alice", name: "Alice", avatarUrl: null },
    assignees: [],
    reviewers: [],
    roles: ["author"],
    pipeline: null,
    unresolvedThreadCount: 0,
    approvalsLeft: 1,
    approved: false,
    approvedBy: [],
    diffStats: null,
    detailedMergeStatus: null,
    autoMergeEnabled: false,
    autoMergeStrategy: null,
    mergeUser: null,
    mergeAfter: null,
    divergedCommitsCount: null,
    rebaseInProgress: false,
    mergeOngoing: false,
    inProgressMergeCommitSha: null,
    mergeError: null,
    shouldBeRebased: false,
    mergeabilityChecks: [],
    blockingMergeRequestsCount: 0,
    approvalsRequired: 1,
    squash: false,
    squashOnMerge: false,
    mergeTrainIndex: null,
    ...overrides,
  } as PullRequest;
}

describe("extractTicketId", () => {
  test("exact branch segment", () => {
    expect(extractTicketId("feature/cv-1287", "whatever")).toBe("CV-1287");
  });
  test("prefixed branch segment", () => {
    expect(extractTicketId("feature/cv-1287-add-photos", "whatever")).toBe("CV-1287");
  });
  test("falls back to title prefix", () => {
    expect(extractTicketId("some-branch", "CV-2388: simplify things")).toBe("CV-2388");
  });
  test("null when nothing matches", () => {
    expect(extractTicketId("main", "fix stuff")).toBeNull();
  });
});

describe("projectPathFromWebUrl", () => {
  test("extracts group/project", () => {
    expect(projectPathFromWebUrl("https://gitlab.com/org/sub/repo/-/merge_requests/7", "https://gitlab.com")).toBe(
      "org/sub/repo",
    );
  });
  test("null for foreign host", () => {
    expect(projectPathFromWebUrl("https://other.com/org/repo/-/merge_requests/7", "https://gitlab.com")).toBeNull();
  });
});

describe("buildBoard", () => {
  test("keeps only member-authored, open, non-draft MRs in configured projects", () => {
    const mrs = buildBoard(
      [
        pr({ iid: 1, author: { id: "a", username: "alice", name: "Alice", avatarUrl: null } }),
        pr({ iid: 2, author: { id: "b", username: "bob", name: "Bob", avatarUrl: null } }),
        pr({ iid: 3, author: { id: "c", username: "carol", name: "Carol", avatarUrl: null } }), // not a member
        pr({ iid: 4, draft: true }),
        pr({ iid: 5, state: "merged" }),
        pr({ iid: 6, webUrl: "https://gitlab.com/other/repo/-/merge_requests/6" }), // wrong project
      ],
      config,
    );
    expect(mrs.map((m) => m.iid).sort()).toEqual([1, 2]);
  });

  test("tags each MR with author, createdAt, and derived pipelineState", () => {
    const [mr] = buildBoard([pr({ createdAt: "2026-07-01T00:00:00Z" })], config);
    expect(mr!.author.username).toBe("alice");
    expect(mr!.createdAt).toBe("2026-07-01T00:00:00Z");
    expect(mr!.pipelineState).toBe("none"); // pipeline: null
  });
});

describe("SnapshotCache", () => {
  test("caches within TTL and revalidates after", async () => {
    let calls = 0;
    let clock = 1_000;
    const cache = new SnapshotCache(
      async () => {
        calls++;
        return [pr({ iid: calls })].map(() => ({ iid: calls } as any));
      },
      () => clock,
      60_000,
    );
    const first = await cache.get();
    expect(first.mrs).toHaveLength(1);
    expect(calls).toBe(1);
    await cache.get();
    expect(calls).toBe(1); // within TTL
    clock += 61_000;
    await cache.get(); // serves stale, kicks background refresh
    await new Promise((r) => setTimeout(r, 0));
    expect(calls).toBe(2);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `bun test src/__tests__/board.test.ts`
Expected: FAIL — `buildBoard` not exported, `Snapshot.mrs` / cache signature mismatch.

- [ ] **Step 3: Rewrite `src/data.ts`**

```ts
import { getMRDashboardProps, type MRDashboardProps, type PullRequest } from "@forge-glance/sdk";
import type { BoardConfig } from "./config.ts";

export type PipelineState = "passed" | "running" | "failed" | "none";

/** Dashboard props plus the raw fields the board renders that props omit. */
export type BoardMR = MRDashboardProps & {
  updatedAt: string | null;
  createdAt: string | null;
  unresolvedThreads: number;
  pipelineState: PipelineState;
};

export interface Snapshot {
  mrs: BoardMR[];
  fetchedAt: number;
  /** Set when the latest refresh failed and this data is older than it should be. */
  fetchError: string | null;
}

/** Parse "group/project" out of a GitLab MR web URL. */
export function projectPathFromWebUrl(webUrl: string, gitlabHost: string): string | null {
  if (!webUrl.startsWith(gitlabHost)) return null;
  const rest = webUrl.slice(gitlabHost.length).replace(/^\//, "");
  const idx = rest.indexOf("/-/");
  return idx === -1 ? null : rest.slice(0, idx);
}

/** Collapse the SDK's pipeline signals into one grouping/sorting key. */
function derivePipelineState(props: MRDashboardProps): PipelineState {
  if (!props.pipeline) return "none";
  if (props.blockers.pipelineFailing) return "failed";
  if (props.blockers.pipelineRunning) return "running";
  return "passed";
}

/**
 * Shape raw MRs into a flat board list: authored by a configured member, open,
 * not draft, in a configured project. Each MR is tagged with its author, created
 * / updated timestamps, unresolved-thread count, and derived pipeline state. The
 * client owns all grouping and sorting, so this list is unsorted.
 */
export function buildBoard(prs: PullRequest[], config: BoardConfig): BoardMR[] {
  const members = new Set(config.members.map((m) => m.username));
  const projects = new Set(config.projects);
  const out: BoardMR[] = [];
  for (const pr of prs) {
    if (pr.draft || pr.state !== "opened") continue;
    if (!pr.author || !members.has(pr.author.username)) continue;
    const path = pr.webUrl ? projectPathFromWebUrl(pr.webUrl, config.gitlabHost) : null;
    if (!path || !projects.has(path)) continue;
    const props = getMRDashboardProps(pr);
    out.push({
      ...props,
      updatedAt: pr.updatedAt,
      createdAt: pr.createdAt,
      unresolvedThreads: pr.unresolvedThreadCount,
      pipelineState: derivePipelineState(props),
    });
  }
  return out;
}
```

- [ ] **Step 4: Update `src/cache.ts`**

Rename `groups` → `mrs` and the constructor param. Full file:

```ts
import type { BoardMR, Snapshot } from "./data.ts";

const TTL_MS = 60_000;

/**
 * Single-snapshot stale-while-revalidate cache.
 *
 * Within TTL: serve the snapshot. Past TTL: serve the stale snapshot
 * immediately and kick exactly one background refresh (concurrent visitors
 * share it). A failed refresh keeps the last good data and stamps the error.
 */
export class SnapshotCache {
  private snapshot: Snapshot | null = null;
  private inflight: Promise<Snapshot> | null = null;

  constructor(
    private readonly fetchMRs: () => Promise<BoardMR[]>,
    private readonly now: () => number = Date.now,
    private readonly ttlMs: number = TTL_MS,
  ) {}

  async get(): Promise<Snapshot> {
    if (this.snapshot && this.now() - this.snapshot.fetchedAt < this.ttlMs) {
      return this.snapshot;
    }
    const refresh = this.refresh();
    if (this.snapshot) {
      // Stale data beats waiting; refresh continues in the background.
      refresh.catch(() => {});
      return this.snapshot;
    }
    return refresh;
  }

  private refresh(): Promise<Snapshot> {
    if (this.inflight) return this.inflight;
    this.inflight = this.fetchMRs()
      .then((mrs) => {
        this.snapshot = { mrs, fetchedAt: this.now(), fetchError: null };
        return this.snapshot;
      })
      .catch((err) => {
        const message = err instanceof Error ? err.message : String(err);
        console.error(`refresh failed: ${message}`);
        if (this.snapshot) {
          this.snapshot = { ...this.snapshot, fetchError: message };
          return this.snapshot;
        }
        this.snapshot = { mrs: [], fetchedAt: this.now(), fetchError: message };
        return this.snapshot;
      })
      .finally(() => {
        this.inflight = null;
      });
    return this.inflight;
  }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `bun test src/__tests__/board.test.ts`
Expected: PASS (all describe blocks).

- [ ] **Step 6: Commit**

```bash
git add src/data.ts src/cache.ts src/__tests__/board.test.ts
git commit -m "feat: flat member-tagged snapshot (buildBoard)"
```

---

## Task 3: view.ts — filter and sort

**Files:**
- Create: `src/view.ts`
- Test: `src/__tests__/view.test.ts` (create)

**Interfaces:**
- Consumes: `BoardMR`, `PipelineState` (Task 2).
- Produces:
  - `type GroupKey = "age" | "author" | "status" | "pipeline"`; `type SortKey = "oldest" | "pipeline" | "progress"`; `const GROUP_KEYS`, `const SORT_KEYS`.
  - `filterByMember(mrs: BoardMR[], member: string): BoardMR[]`
  - `sortMRs(mrs: BoardMR[], sort: SortKey): BoardMR[]` (returns a new array).

- [ ] **Step 1: Write the failing test**

Create `src/__tests__/view.test.ts`:

```ts
import { describe, expect, test } from "bun:test";
import type { BoardMR } from "../data.ts";
import { filterByMember, sortMRs } from "../view.ts";

function mr(overrides: Partial<BoardMR>): BoardMR {
  return {
    iid: 1,
    title: "MR",
    author: { id: "x", username: "alice", name: "Alice", avatarUrl: null },
    createdAt: "2026-07-01T00:00:00Z",
    updatedAt: "2026-07-01T00:00:00Z",
    pipelineState: "none",
    reviews: { required: 2, given: 0, isApproved: false },
    blockers: {},
    ...overrides,
  } as unknown as BoardMR;
}

describe("filterByMember", () => {
  const list = [mr({ iid: 1, author: { username: "alice" } as any }), mr({ iid: 2, author: { username: "bob" } as any })];
  test("all returns everything", () => {
    expect(filterByMember(list, "all")).toHaveLength(2);
  });
  test("filters to one member", () => {
    expect(filterByMember(list, "bob").map((m) => m.iid)).toEqual([2]);
  });
});

describe("sortMRs", () => {
  test("oldest: oldest createdAt first, nulls last", () => {
    const list = [
      mr({ iid: 1, createdAt: "2026-07-05T00:00:00Z" }),
      mr({ iid: 2, createdAt: null }),
      mr({ iid: 3, createdAt: "2026-07-01T00:00:00Z" }),
    ];
    expect(sortMRs(list, "oldest").map((m) => m.iid)).toEqual([3, 1, 2]);
  });

  test("pipeline: failed, running, none, passed", () => {
    const list = [
      mr({ iid: 1, pipelineState: "passed" }),
      mr({ iid: 2, pipelineState: "failed" }),
      mr({ iid: 3, pipelineState: "none" }),
      mr({ iid: 4, pipelineState: "running" }),
    ];
    expect(sortMRs(list, "pipeline").map((m) => m.iid)).toEqual([2, 4, 3, 1]);
  });

  test("progress: highest approval ratio first", () => {
    const list = [
      mr({ iid: 1, reviews: { required: 2, given: 0, isApproved: false } as any }),
      mr({ iid: 2, reviews: { required: 2, given: 2, isApproved: true } as any }),
      mr({ iid: 3, reviews: { required: 2, given: 1, isApproved: false } as any }),
    ];
    expect(sortMRs(list, "progress").map((m) => m.iid)).toEqual([2, 3, 1]);
  });

  test("does not mutate input", () => {
    const list = [mr({ iid: 1, createdAt: "2026-07-05T00:00:00Z" }), mr({ iid: 2, createdAt: "2026-07-01T00:00:00Z" })];
    sortMRs(list, "oldest");
    expect(list.map((m) => m.iid)).toEqual([1, 2]);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bun test src/__tests__/view.test.ts`
Expected: FAIL — `src/view.ts` does not exist.

- [ ] **Step 3: Implement `src/view.ts`**

```ts
import type { BoardMR, PipelineState } from "./data.ts";

export type GroupKey = "age" | "author" | "status" | "pipeline";
export type SortKey = "oldest" | "pipeline" | "progress";

export const GROUP_KEYS: readonly GroupKey[] = ["age", "author", "status", "pipeline"];
export const SORT_KEYS: readonly SortKey[] = ["oldest", "pipeline", "progress"];

/** Sentinel that sorts after any ISO date, so null timestamps land last. */
const LATEST = "9999";

const PIPELINE_RANK: Record<PipelineState, number> = { failed: 0, running: 1, none: 2, passed: 3 };

/** Approval ratio in [0,1]; used by the "progress" sort. */
function progress(mr: BoardMR): number {
  const req = mr.reviews.required;
  if (req > 0) return mr.reviews.given / req;
  return mr.reviews.given > 0 ? 1 : 0;
}

export function filterByMember(mrs: BoardMR[], member: string): BoardMR[] {
  return member === "all" ? mrs : mrs.filter((m) => m.author.username === member);
}

/** Return a new array ordered by the chosen sort. Never mutates the input. */
export function sortMRs(mrs: BoardMR[], sort: SortKey): BoardMR[] {
  const byOldest = (a: BoardMR, b: BoardMR) => (a.createdAt ?? LATEST).localeCompare(b.createdAt ?? LATEST);
  const copy = [...mrs];
  switch (sort) {
    case "oldest":
      copy.sort(byOldest);
      break;
    case "pipeline":
      copy.sort((a, b) => PIPELINE_RANK[a.pipelineState] - PIPELINE_RANK[b.pipelineState] || byOldest(a, b));
      break;
    case "progress":
      copy.sort((a, b) => progress(b) - progress(a) || byOldest(a, b));
      break;
  }
  return copy;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `bun test src/__tests__/view.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/view.ts src/__tests__/view.test.ts
git commit -m "feat: view.ts filter + sort"
```

---

## Task 4: view.ts — grouping

**Files:**
- Modify: `src/view.ts`
- Test: `src/__tests__/view.test.ts`

**Interfaces:**
- Consumes: `BoardMR`, `GroupKey` (Tasks 2-3).
- Produces:
  - `interface Group { label: string; mrs: BoardMR[] }`
  - `groupMRs(mrs: BoardMR[], group: GroupKey, memberOrder: string[], now: number): Group[]` — returns groups in natural order for the dimension. Does NOT sort within groups (the caller applies `sortMRs` per group).

- [ ] **Step 1: Add failing tests**

Append to `src/__tests__/view.test.ts`:

```ts
import { groupMRs } from "../view.ts";

const NOW = Date.parse("2026-07-13T12:00:00Z");
const daysAgo = (n: number) => new Date(NOW - n * 86_400_000).toISOString();

describe("groupMRs age", () => {
  test("buckets by day then week, ordered", () => {
    const list = [
      mr({ iid: 1, createdAt: daysAgo(0) }),
      mr({ iid: 2, createdAt: daysAgo(1) }),
      mr({ iid: 3, createdAt: daysAgo(3) }),
      mr({ iid: 4, createdAt: daysAgo(9) }),
      mr({ iid: 5, createdAt: daysAgo(30) }),
    ];
    const groups = groupMRs(list, "age", [], NOW);
    expect(groups.map((g) => g.label)).toEqual(["Today", "Yesterday", "3 days ago", "Last week", "Older"]);
  });
});

describe("groupMRs author", () => {
  test("one group per member in config order, skips empty", () => {
    const list = [
      mr({ iid: 1, author: { username: "bob", name: "Bob" } as any }),
      mr({ iid: 2, author: { username: "alice", name: "Alice" } as any }),
    ];
    const groups = groupMRs(list, "author", ["alice", "bob", "carol"], NOW);
    expect(groups.map((g) => g.label)).toEqual(["Alice", "Bob"]);
  });
});

describe("groupMRs status", () => {
  test("severity order: conflicts, ci failing, needs review, in review, approved", () => {
    const list = [
      mr({ iid: 1, blockers: { hasConflicts: false, pipelineFailing: false } as any, reviews: { required: 2, given: 0, isApproved: false } as any }),
      mr({ iid: 2, blockers: { hasConflicts: true } as any }),
      mr({ iid: 3, blockers: {} as any, reviews: { required: 2, given: 2, isApproved: true } as any }),
    ];
    const groups = groupMRs(list, "status", [], NOW);
    expect(groups.map((g) => g.label)).toEqual(["conflicts", "needs review", "approved"]);
  });
});

describe("groupMRs pipeline", () => {
  test("order failed, running, none, passed", () => {
    const list = [
      mr({ iid: 1, pipelineState: "passed" }),
      mr({ iid: 2, pipelineState: "failed" }),
      mr({ iid: 3, pipelineState: "none" }),
    ];
    const groups = groupMRs(list, "pipeline", [], NOW);
    expect(groups.map((g) => g.label)).toEqual(["pipeline failed", "no pipeline", "pipeline passed"]);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bun test src/__tests__/view.test.ts`
Expected: FAIL — `groupMRs` not exported.

- [ ] **Step 3: Implement — append to `src/view.ts`**

```ts
export interface Group {
  label: string;
  mrs: BoardMR[];
}

/** Age band for an MR by creation date: by day for the first week, then weekly. */
function ageBucket(createdAt: string | null, now: number): { label: string; order: number } {
  if (!createdAt) return { label: "Unknown", order: 1000 };
  const days = Math.floor((now - Date.parse(createdAt)) / 86_400_000);
  if (days <= 0) return { label: "Today", order: 0 };
  if (days === 1) return { label: "Yesterday", order: 1 };
  if (days <= 6) return { label: `${days} days ago`, order: days };
  if (days <= 13) return { label: "Last week", order: 7 };
  if (days <= 20) return { label: "2 weeks ago", order: 8 };
  return { label: "Older", order: 9 };
}

/** Coarse review-readiness bucket, most-blocking first. */
function statusBucket(mr: BoardMR): { label: string; order: number } {
  const b = mr.blockers;
  if (b.hasConflicts) return { label: "conflicts", order: 0 };
  if (b.pipelineFailing) return { label: "ci failing", order: 1 };
  if (mr.reviews.isApproved) return { label: "approved", order: 4 };
  if (mr.reviews.given > 0) return { label: "in review", order: 3 };
  return { label: "needs review", order: 2 };
}

const PIPELINE_LABEL: Record<PipelineState, string> = {
  failed: "pipeline failed",
  running: "pipeline running",
  none: "no pipeline",
  passed: "pipeline passed",
};

/** Group by a keyed bucket, ordering groups by the bucket's `order`. */
function groupBy(mrs: BoardMR[], bucket: (mr: BoardMR) => { label: string; order: number }): Group[] {
  const map = new Map<string, { order: number; mrs: BoardMR[] }>();
  for (const mr of mrs) {
    const b = bucket(mr);
    const entry = map.get(b.label) ?? { order: b.order, mrs: [] };
    entry.mrs.push(mr);
    map.set(b.label, entry);
  }
  return [...map.entries()]
    .sort((a, b) => a[1].order - b[1].order)
    .map(([label, entry]) => ({ label, mrs: entry.mrs }));
}

/** One group per member, in config order; members with no MRs are skipped. */
function groupByAuthor(mrs: BoardMR[], memberOrder: string[]): Group[] {
  const rank = new Map(memberOrder.map((u, i) => [u, i]));
  const byUser = new Map<string, BoardMR[]>();
  for (const mr of mrs) {
    const u = mr.author.username;
    const list = byUser.get(u) ?? [];
    list.push(mr);
    byUser.set(u, list);
  }
  return [...byUser.entries()]
    .sort((a, b) => (rank.get(a[0]) ?? 999) - (rank.get(b[0]) ?? 999))
    .map(([username, list]) => ({ label: list[0]!.author.name || username, mrs: list }));
}

/**
 * Partition MRs into ordered display groups. Groups are ordered naturally for
 * the dimension; ordering WITHIN each group is the caller's job (apply sortMRs
 * to each group's `mrs`).
 */
export function groupMRs(mrs: BoardMR[], group: GroupKey, memberOrder: string[], now: number): Group[] {
  switch (group) {
    case "age":
      return groupBy(mrs, (mr) => ageBucket(mr.createdAt, now));
    case "author":
      return groupByAuthor(mrs, memberOrder);
    case "status":
      return groupBy(mrs, statusBucket);
    case "pipeline":
      return groupBy(mrs, (mr) => ({
        label: PIPELINE_LABEL[mr.pipelineState],
        order: PIPELINE_RANK[mr.pipelineState],
      }));
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `bun test src/__tests__/view.test.ts`
Expected: PASS (Task 3 + Task 4 blocks).

- [ ] **Step 5: Commit**

```bash
git add src/view.ts src/__tests__/view.test.ts
git commit -m "feat: view.ts grouping (age/author/status/pipeline)"
```

---

## Task 5: view.ts — view state parse/serialize

**Files:**
- Modify: `src/view.ts`
- Test: `src/__tests__/view.test.ts`

**Interfaces:**
- Consumes: `GroupKey`, `SortKey`, `GROUP_KEYS`, `SORT_KEYS` (Task 3).
- Produces:
  - `interface ViewState { member: string; group: GroupKey; sort: SortKey }`
  - `const DEFAULT_VIEW: ViewState`
  - `parseViewState(search: string, stored: Partial<ViewState> | null, validMembers: string[]): ViewState` — URL wins, then localStorage, then default; invalid values are ignored.
  - `serializeViewState(v: ViewState): string` — query string (leading `?`) omitting defaults; `""` when all default.

- [ ] **Step 1: Add failing tests**

Append to `src/__tests__/view.test.ts`:

```ts
import { parseViewState, serializeViewState, DEFAULT_VIEW } from "../view.ts";

describe("parseViewState", () => {
  const members = ["alice", "bob"];
  test("defaults when nothing provided", () => {
    expect(parseViewState("", null, members)).toEqual(DEFAULT_VIEW);
  });
  test("URL wins over localStorage", () => {
    expect(parseViewState("?member=bob&group=status", { member: "alice", sort: "pipeline" }, members)).toEqual({
      member: "bob",
      group: "status",
      sort: "pipeline",
    });
  });
  test("ignores unknown member and invalid group/sort", () => {
    expect(parseViewState("?member=ghost&group=bogus&sort=bogus", null, members)).toEqual(DEFAULT_VIEW);
  });
});

describe("serializeViewState", () => {
  test("omits defaults", () => {
    expect(serializeViewState(DEFAULT_VIEW)).toBe("");
  });
  test("includes non-defaults", () => {
    expect(serializeViewState({ member: "bob", group: "status", sort: "oldest" })).toBe("?member=bob&group=status");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `bun test src/__tests__/view.test.ts`
Expected: FAIL — `parseViewState` not exported.

- [ ] **Step 3: Implement — append to `src/view.ts`**

```ts
export interface ViewState {
  member: string;
  group: GroupKey;
  sort: SortKey;
}

export const DEFAULT_VIEW: ViewState = { member: "all", group: "age", sort: "oldest" };

/** URL query params win, then stored localStorage values, then defaults. Invalid values are dropped. */
export function parseViewState(search: string, stored: Partial<ViewState> | null, validMembers: string[]): ViewState {
  const params = new URLSearchParams(search);
  const members = ["all", ...validMembers];

  const resolve = <T extends string>(key: keyof ViewState, valid: readonly T[], fallback: T): T => {
    const fromUrl = params.get(key);
    if (fromUrl && valid.includes(fromUrl as T)) return fromUrl as T;
    const fromStore = stored?.[key];
    if (typeof fromStore === "string" && valid.includes(fromStore as T)) return fromStore as T;
    return fallback;
  };

  return {
    member: resolve("member", members, "all"),
    group: resolve("group", GROUP_KEYS, "age"),
    sort: resolve("sort", SORT_KEYS, "oldest"),
  };
}

/** Query string (with leading "?") carrying only non-default values; "" when all default. */
export function serializeViewState(v: ViewState): string {
  const params = new URLSearchParams();
  if (v.member !== DEFAULT_VIEW.member) params.set("member", v.member);
  if (v.group !== DEFAULT_VIEW.group) params.set("group", v.group);
  if (v.sort !== DEFAULT_VIEW.sort) params.set("sort", v.sort);
  const s = params.toString();
  return s ? `?${s}` : "";
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `bun test src/__tests__/view.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/view.ts src/__tests__/view.test.ts
git commit -m "feat: view.ts view-state parse/serialize"
```

---

## Task 6: Server — roster and /data.json shape

**Files:**
- Modify: `src/server.ts`

**Interfaces:**
- Consumes: `buildBoard` (Task 2), `SnapshotCache` (Task 2), `loadConfig` (Task 1).
- Produces: `/data.json` → `{ title: string, members: RosterMember[], mrs: BoardMR[], fetchedAt: number, fetchError: string | null }` where `RosterMember = { username: string; name: string | null; count: number }`.

- [ ] **Step 1: Rewrite the owner/roster section and the `cache` construction**

In `src/server.ts`, replace the owner block (current lines ~11-41: from `const config = loadConfig();` through `await refreshOwner();`) with:

```ts
const config = loadConfig();
const provider = new GitLabProvider(config.gitlabHost, loadGitLabToken());

const cache = new SnapshotCache(async () => {
  const prs = await provider.fetchPullRequests({ state: "opened" });
  return buildBoard(prs, config);
});

/** Display names resolved from GitLab profiles, keyed by username. Long TTL. */
const memberNames = new Map<string, string | null>();
let namesFetchedAt = 0;
const NAMES_TTL_MS = 60 * 60_000;

async function refreshMemberNames(): Promise<void> {
  if (namesFetchedAt && Date.now() - namesFetchedAt < NAMES_TTL_MS) return;
  namesFetchedAt = Date.now();
  await Promise.all(
    config.members.map(async (member) => {
      try {
        const res = await provider.restRequest(
          "GET",
          `/api/v4/users?username=${encodeURIComponent(member.username)}`,
        );
        const users = (await res.json()) as Array<{ username: string; name?: string }>;
        memberNames.set(member.username, users[0]?.name ?? member.name ?? null);
      } catch (err) {
        console.error(`name lookup failed for ${member.username}: ${err instanceof Error ? err.message : err}`);
        memberNames.set(member.username, member.name ?? null);
      }
    }),
  );
}
await refreshMemberNames();

interface RosterMember {
  username: string;
  name: string | null;
  count: number;
}

/** Members in config order, each with a resolved name and open-MR count. */
function buildRoster(mrs: { author: { username: string } }[]): RosterMember[] {
  return config.members.map((member) => ({
    username: member.username,
    name: memberNames.get(member.username) ?? member.name ?? null,
    count: mrs.filter((mr) => mr.author.username === member.username).length,
  }));
}
```

- [ ] **Step 2: Update the imports at the top of `src/server.ts`**

Change the data import line to import `buildBoard`:

```ts
import { buildBoard } from "./data.ts";
```

(Leave the other imports — `loadConfig`, `loadGitLabToken`, `SnapshotCache`, `GitLabProvider`, `join`, `readFileSync` — as they are.)

- [ ] **Step 3: Update the `/data.json` handler**

Replace the `case "/data.json":` block with:

```ts
      case "/data.json": {
        void refreshMemberNames();
        const snapshot = await cache.get();
        return new Response(
          JSON.stringify({
            title: config.title,
            members: buildRoster(snapshot.mrs),
            mrs: snapshot.mrs,
            fetchedAt: snapshot.fetchedAt,
            fetchError: snapshot.fetchError,
          }),
          { headers: { "content-type": "application/json" } },
        );
      }
```

- [ ] **Step 4: Verify the server serves the new shape**

Requires a real `config.json` with `members` and a valid token. Run:

```bash
bun run serve &
sleep 2
curl -s http://localhost:7930/data.json | bun -e 'const d=JSON.parse(await Bun.stdin.text()); console.log("keys:", Object.keys(d)); console.log("members:", d.members?.map(m=>`${m.username}:${m.count}`)); console.log("mrs:", d.mrs?.length)'
kill %1
```

Expected: `keys: [ "title", "members", "mrs", "fetchedAt", "fetchError" ]`; `members:` lists each configured username with a count; `mrs:` is a number. (If you lack a live token, confirm `bun run serve` starts without throwing and `curl -s http://localhost:7930/healthz` returns `ok`.)

- [ ] **Step 5: Commit**

```bash
git add src/server.ts
git commit -m "feat: server roster + flat /data.json shape"
```

---

## Task 7: Client — sidebar, view state, grouping controls

**Files:**
- Modify: `src/client.tsx`
- Modify: `src/style.css`

**Interfaces:**
- Consumes: `BoardMR` (Task 2); `view.ts` exports (Tasks 3-5): `filterByMember`, `sortMRs`, `groupMRs`, `parseViewState`, `serializeViewState`, `GROUP_KEYS`, `SORT_KEYS`, and types `GroupKey`, `SortKey`, `ViewState`, `Group`. `/data.json` shape (Task 6).

- [ ] **Step 1: Update the `BoardData` interface and imports**

At the top of `src/client.tsx`, replace the `BoardData` interface and add view imports:

```tsx
import { filterByMember, sortMRs, groupMRs, parseViewState, serializeViewState, GROUP_KEYS, SORT_KEYS } from "./view.ts";
import type { GroupKey, SortKey, ViewState } from "./view.ts";

interface RosterMember {
  username: string;
  name: string | null;
  count: number;
}

interface BoardData {
  title: string;
  members: RosterMember[];
  mrs: BoardMR[];
  fetchedAt: number;
  fetchError: string | null;
}
```

Remove the old `owner` field from `BoardData` and delete the `ViewMode` grid/rows note only if unused — keep `ViewMode`, `VIEW_KEY`, rows/grid as they are (still used).

- [ ] **Step 2: Add a localStorage key and a text-label control for group/sort**

Below the existing `VIEW_KEY` / `THEME_KEY` constants add:

```tsx
const STATE_KEY = "mrs-view-state";

const GROUP_LABEL: Record<GroupKey, string> = {
  age: "age",
  author: "author",
  status: "status",
  pipeline: "pipeline",
};
const SORT_LABEL: Record<SortKey, string> = {
  oldest: "oldest",
  pipeline: "pipeline",
  progress: "progress",
};

/** A labelled segmented control (text labels, unlike the icon-only Segmented). */
function LabeledSeg<T extends string>({
  legend,
  options,
  labels,
  value,
  onChange,
}: {
  legend: string;
  options: readonly T[];
  labels: Record<T, string>;
  value: T;
  onChange: (v: T) => void;
}) {
  return (
    <span className="tui-seg tui-seg-text" role="group" aria-label={legend}>
      {options.map((o) => (
        <button key={o} className={o === value ? "active" : ""} onClick={() => onChange(o)}>
          {labels[o]}
        </button>
      ))}
    </span>
  );
}
```

- [ ] **Step 3: Add the sidebar component**

Add before the `Board` component:

```tsx
function Sidebar({
  members,
  total,
  active,
  onPick,
}: {
  members: RosterMember[];
  total: number;
  active: string;
  onPick: (member: string) => void;
}) {
  return (
    <nav className="tui-sidebar" aria-label="team members">
      <button className={active === "all" ? "tui-side-item active" : "tui-side-item"} onClick={() => onPick("all")}>
        <span className="tui-side-name">◉ All</span>
        <span className="tui-side-count">{total}</span>
      </button>
      {members.map((m) => (
        <button
          key={m.username}
          className={
            (active === m.username ? "tui-side-item active" : "tui-side-item") + (m.count === 0 ? " tui-side-empty" : "")
          }
          onClick={() => onPick(m.username)}
          title={m.name ?? m.username}
        >
          <span className="tui-side-name">
            <OwnerSprite username={m.username} /> {m.name ?? m.username}
          </span>
          <span className="tui-side-count">{m.count}</span>
        </button>
      ))}
    </nav>
  );
}
```

- [ ] **Step 4: Rewrite the `Board` component body**

Replace the entire `Board` function with:

```tsx
function Board() {
  const [data, setData] = useState<BoardData | null>(null);
  const [loadError, setLoadError] = useState(false);
  const [view, setView] = useState<ViewMode>(() => (localStorage.getItem(VIEW_KEY) as ViewMode) ?? "rows");
  const [theme, setTheme] = useState<ThemeMode>(() => (localStorage.getItem(THEME_KEY) as ThemeMode) ?? "system");

  // View state (member/group/sort). Members are validated once data arrives.
  const [state, setState] = useState<ViewState>(() => {
    let stored: Partial<ViewState> | null = null;
    try {
      stored = JSON.parse(localStorage.getItem(STATE_KEY) ?? "null");
    } catch {
      stored = null;
    }
    return parseViewState(location.search, stored, []);
  });

  const pickView = (v: ViewMode) => {
    localStorage.setItem(VIEW_KEY, v);
    setView(v);
  };
  const pickTheme = (m: ThemeMode) => {
    localStorage.setItem(THEME_KEY, m);
    window.__applyTheme();
    setTheme(m);
  };
  const update = (patch: Partial<ViewState>) => {
    setState((prev) => {
      const next = { ...prev, ...patch };
      localStorage.setItem(STATE_KEY, JSON.stringify(next));
      history.replaceState(null, "", serializeViewState(next) || location.pathname);
      return next;
    });
  };

  useEffect(() => {
    let timer: ReturnType<typeof setInterval> | undefined;
    const load = () =>
      fetch("/data.json")
        .then((r) => r.json())
        .then((d: BoardData) => {
          setData(d);
          setLoadError(false);
          // Re-validate the member against the real roster (drops stale ?member=).
          setState((prev) => parseViewState(location.search, prev, d.members.map((m) => m.username)));
        })
        .catch(() => setLoadError(true));
    const onVisible = () => {
      if (!document.hidden) load();
    };
    document.addEventListener("visibilitychange", onVisible);
    load();
    timer = setInterval(() => {
      if (!document.hidden) load();
    }, 60_000);
    return () => {
      clearInterval(timer);
      document.removeEventListener("visibilitychange", onVisible);
    };
  }, []);

  if (!data) {
    return <p className="tui-loading">{loadError ? "✗ failed to load board data" : "fetching…"}</p>;
  }

  const total = data.members.reduce((n, m) => n + m.count, 0);
  const staleMins = Math.round((Date.now() - data.fetchedAt) / 60_000);
  const now = Date.now();

  const filtered = filterByMember(data.mrs, state.member);
  const groups = groupMRs(filtered, state.group, data.members.map((m) => m.username), now).map((g) => ({
    label: g.label,
    mrs: sortMRs(g.mrs, state.sort),
  }));
  const activeMember = state.member === "all" ? null : data.members.find((m) => m.username === state.member) ?? null;
  const summaryText = boardSummary(filtered);

  return (
    <div className={view === "grid" ? "tui tui-wide tui-app" : "tui tui-app"}>
      <Sidebar members={data.members} total={total} active={state.member} onPick={(member) => update({ member })} />

      <div className="tui-main">
        <header className="tui-header">
          <div>
            <h1>
              <span className="tui-prompt">❯</span> {data.title.toLowerCase()}{" "}
              {activeMember && <span className="tui-author">--author @{activeMember.username}</span>}
            </h1>
            <p className="tui-sub">
              <span className="tui-comment"># {filtered.length} awaiting review · pick one, it opens in gitlab</span>
            </p>
          </div>
          <div className="tui-controls">
            {filtered.length > 0 && (
              <CopyButton text={summaryText} className="tui-copy" title="copy summary for Slack" />
            )}
            <LabeledSeg legend="group" options={GROUP_KEYS} labels={GROUP_LABEL} value={state.group} onChange={(group) => update({ group })} />
            <LabeledSeg legend="sort" options={SORT_KEYS} labels={SORT_LABEL} value={state.sort} onChange={(sort) => update({ sort })} />
            <Segmented options={["rows", "grid"] as const} value={view} onChange={pickView} label="view" />
            <Segmented options={["light", "dark", "system"] as const} value={theme} onChange={pickTheme} label="theme" />
          </div>
        </header>

        {data.fetchError && <div className="tui-banner">⚠ data from {staleMins}m ago — gitlab fetch failing</div>}

        {filtered.length === 0 && !data.fetchError ? (
          <p className="tui-empty">nothing waiting on review ✓</p>
        ) : (
          groups.map((g) => (
            <Panel key={g.label} title={g.label} count={g.mrs.length}>
              {view === "rows" ? <RowView mrs={g.mrs} now={now} /> : <GridView mrs={g.mrs} now={now} />}
            </Panel>
          ))
        )}

        <footer className="tui-footer">updated {staleMins < 1 ? "just now" : `${staleMins}m ago`}</footer>
      </div>
    </div>
  );
}
```

- [ ] **Step 5: Update `boardSummary` to take a flat MR list**

Replace the `boardSummary` function with:

```tsx
/** The current view as text: a count heading, then each MR as a "- title: url" bullet. */
function boardSummary(mrs: BoardMR[]): string {
  return [`${mrs.length} MR's ready for review :pray:`, ...mrs.map((mr) => `- ${mrLine(mr)}`)].join("\n");
}
```

- [ ] **Step 6: Add sidebar styles**

Append to `src/style.css`:

```css
/* team sidebar layout */
.tui-app { display: grid; grid-template-columns: 220px 1fr; gap: 20px; align-items: start; }
.tui-app.tui-wide { max-width: 1400px; }
.tui-main { min-width: 0; }
.tui-sidebar { display: flex; flex-direction: column; gap: 2px; position: sticky; top: 20px; }
.tui-side-item {
  display: flex; justify-content: space-between; align-items: center; gap: 8px;
  width: 100%; padding: 6px 10px; border: 1px solid transparent; border-radius: 6px;
  background: none; color: var(--fg); font: inherit; cursor: pointer; text-align: left;
}
.tui-side-item:hover { background: var(--panel); }
.tui-side-item.active { border-color: var(--border); background: var(--panel); }
.tui-side-name { display: inline-flex; align-items: center; gap: 6px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.tui-side-count { color: var(--muted); font-family: var(--mono, monospace); font-size: 0.85em; }
.tui-side-empty { opacity: 0.45; }
.tui-seg-text button { padding: 2px 8px; font-size: 0.8em; }
@media (max-width: 720px) {
  .tui-app { grid-template-columns: 1fr; }
  .tui-sidebar { position: static; flex-direction: row; flex-wrap: wrap; }
}
```

Note: reuse existing CSS custom properties. If `--panel`, `--border`, `--muted`, `--fg` aren't the exact names in the file, open `src/style.css`, find the actual variable names in `:root`, and substitute them before saving.

- [ ] **Step 7: Verify in the browser**

```bash
bun run serve
```

Open http://localhost:7930 and confirm:
- Sidebar lists "All" + every configured member with sprite avatars and counts; zero-MR members are greyed.
- Clicking a member filters the main pane and adds `?member=<user>` to the URL; the header shows `--author @user`.
- Changing group/sort updates the panels and the URL (`?group=status&sort=pipeline`), and reloading preserves the view.
- Copying "for Slack" copies only the current view's MRs.
- Rows/grid and theme toggles still work.

Restart the server after edits (the client bundles at startup).

- [ ] **Step 8: Commit**

```bash
git add src/client.tsx src/style.css
git commit -m "feat: sidebar + merged view with grouping/sort controls"
```

---

## Task 8: Docs and full verification

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Update the config section of `README.md`**

Replace the `## config` table rows and the "the board lists MRs" paragraph to describe `members` instead of `username`, and document the views. Concretely, change the config field table to:

```markdown
| field | meaning |
|---|---|
| `gitlabHost` | your gitlab instance, e.g. `https://gitlab.com` |
| `projects` | project paths whose MRs are eligible |
| `members` | array of `{ "username", "name"? }` — the teammates whose authored MRs the board shows, in sidebar order |
| `title` | page heading and tab title |
| `port` | listen port (default 7930) |

the board lists open, non-draft MRs authored by any configured member in one of `projects`. a left sidebar switches between **All** (the whole team) and a single member; the **All** view (and each member view) can be grouped by age / author / status / pipeline and sorted by oldest / pipeline / review progress. the current member, grouping, and sort live in the URL (shareable) and are remembered across visits.
```

Also update the intro paragraph's "your open gitlab merge requests" to "your team's open gitlab merge requests" and adjust the "whose board it is" paragraph to mention per-member headers rather than a single owner.

- [ ] **Step 2: Run the full test suite**

Run: `bun test`
Expected: PASS — config, board (build/cache/ticket/url), and view (filter/sort/group/state) suites all green.

- [ ] **Step 3: Typecheck the whole thing builds**

Run: `bun run serve` and confirm it starts and prints `mr-board serving on http://localhost:7930` with no bundle errors, then stop it.

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "docs: team board config and views"
```

---

## Self-Review Notes

- **Spec coverage:** config `members` (T1); flat member-tagged snapshot + `createdAt`/`pipelineState` (T2); filter/sort incl. oldest/pipeline/progress (T3); grouping incl. by-day-then-weekly age buckets, author, status, pipeline (T4); URL + localStorage view state with validation/fallbacks (T5); server roster with per-member name lookups + new `/data.json` (T6); sidebar nav, per-member header identity, controls everywhere, copy current view, edge cases — zero-MR greying, empty state (T7); docs (T8). All spec sections mapped.
- **Type consistency:** `buildBoard`, `Snapshot.mrs`, `BoardMR.pipelineState`/`createdAt`, `RosterMember`, `ViewState`, `GroupKey`/`SortKey` names are used identically across tasks.
- **Pipeline severity order** locked to failed → running → none → passed for both sort and grouping.
