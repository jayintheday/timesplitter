---
name: time-splitters
version: 1.0.0
description: Forensically reconstruct a daily work log from git, coding-agent sessions, GitHub PRs and Linear, then write a markdown worklog plus a shadcn/ui-styled interactive HTML hours dashboard (and a CSV). Triggers on "/time-splitters", "reconstruct my hours", "build my timesheet/worklog".
effort: medium
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
---

# /time-splitters

Reconstruct how much the user actually worked over a date range by mining **this machine** for
evidence, then produce three internally-consistent files:

1. a **markdown worklog** (`.md`) — the human-readable source of truth (written **first**),
2. a flat **`.csv`** (one row per day),
3. an interactive **`.html`** dashboard styled with **shadcn/ui** (React + Tailwind + Recharts via
   CDN; KPI cards, a shadcn-style gradient **area chart** for hours/day, weekly progress bars, a
   sortable detail table with ticket badges, and a **light/dark theme toggle**) — generated **last,
   after the `.md` exists**.

Evidence sources: **git commits + reflog** across all clones/worktrees, **Claude Code + Codex
session logs**, **GitHub PRs** (`gh`), and **Linear issue lifecycle** (MCP). Hours are
*activity-based estimates*, not stopwatch time. **Only git is required** — PRs, agent sessions, and
Linear are each optional and skipped cleanly when a project doesn't use them (Linear is auto-detected;
see Step 2f). The outputs list only the sources that actually contributed.

**One evolving timesheet, brought up to date on every run.** The first run is a full reconstruction
over the range; it also writes a hidden **state file** (the durable per-day dataset). Every later run
**catches up to *now***: it reads the state, re-fetches evidence only for the trailing edge (the last
covered day, which a mid-run leaves partial, through the current moment — spanning any skipped days),
merges, and re-renders the same files in place. A full 6-week reconstruction takes minutes; a daily
top-up takes seconds, because the slow sources (Linear, session-log parsing) are scoped to the small
window. `--full` forces a clean rebuild. This is **correct** because every signal is bucketed by its
own event timestamp, so days already computed never change — only the trailing edge can.

This skill is **pure instructions** — there is no pre-bundled helper script. For the deterministic
number-crunching (parsing git output, converting timezones, computing work blocks, templating the
HTML) you SHOULD write a short throwaway script into the scratchpad at runtime and run it; that
keeps the math reliable and stays fully transparent (the script is fresh and visible each run). Use
`python3` (verified available, stdlib `zoneinfo`/`json`/`csv` — no pip needed).

---

## Step 0 — Resolve config, then print the plan

Parse optional flags. On a **bare `/time-splitters`** resolve smart defaults, print the resolved
plan (range / repos / output paths), then **generate without pausing**.

| Flag | Default |
|---|---|
| `--from YYYY-MM-DD` | first run: `--to` minus 6 weeks. Later runs: reuse the saved `from`. |
| `--to YYYY-MM-DD` | **now** (today) — the timesheet always catches up to the current moment |
| `--name <prefix>` | basename of the **anchor repo** (the cwd project — see Step 1) |
| `--out <dir>` | `~/Desktop/<name>-timesheets/` |
| `--repo <path>` (repeatable) | autodiscover from the anchor repo — see Step 1 |
| `--author <email>` | git `user.email`; falls back to `me@vijay-patel.co.uk` |
| `--tz` | `Europe/London` |
| `--full` | ignore any saved state and rebuild the whole range from scratch |
| `--since YYYY-MM-DD` | widen the incremental recompute window back to this date (refresh older days) |
| `--no-linear` / `--linear` | force Linear off / on (default: auto-detect from the project's ticket refs — see Step 2f) |

A `2026-05-15..2026-06-26` positional form is also accepted as `--from..--to`.

**Output files (stable — one evolving timesheet, updated in place):**
`<out>/<name>-worklog.{md,csv,html}` plus the hidden state file `<out>/.<name>-worklog.state.json`.
The covered date range is shown *inside* the documents, not in the filenames, so each run overwrites
the same three files. (`--from`/`--to` adjust the range *inside* the one timesheet; they do not change
the filenames.)

Print a short block before generating — state the mode (see Step 0.5), e.g.:
```
Resolved:  project acme  ·  author you@example.com  ·  tz Europe/London  ·  out ~/Desktop/acme-timesheets/
Mode:      UPDATE (state found, covered through 2026-06-26 14:30)
           → recompute 2026-06-26 → now (2026-06-27 16:40); reuse 41 earlier days
→ updating md, csv, html…
```
(First run instead prints `Mode: FULL (no state) → 2026-05-16 .. now`.) Create `<out>` with
`mkdir -p` if missing.

---

## Step 0.5 — Load state & choose mode (FULL vs UPDATE)

Look for the state file `<out>/.<name>-worklog.state.json`.

- **No state file, or `--full`, or `--from` earlier than the saved `from`** → **FULL** run.
  Range = `[from .. now]` (`from` = `--from` or `--to` − 6 weeks). Gather everything (Steps 1–4), then
  write the state file. (`--from` earlier than the saved start forces FULL because that older span was
  never gathered.)
- **State file present** → **UPDATE** run. Read it. Reuse `from`, `author`, `tz`, repo labels, and the
  prior `records[]`. Compute the **recompute window**:
  - `window_start` = `--since` if given, else the **local date of the saved `covered_through`** (the
    last run's last day — re-finalize it, since a mid-run left it partial);
  - `window_end` = **now**.
  Only days in `[window_start .. today]` get re-gathered and recomputed (Steps 1–3, scoped); every day
  before `window_start` is reused verbatim from state.

### State file schema (`<out>/.<name>-worklog.state.json`)
```json
{
  "schema": 1,
  "name": "acme",
  "from": "2026-05-15",
  "covered_through": "2026-06-26T14:30:00+01:00",
  "generated_at": "2026-06-26T14:30:11+01:00",
  "author": "you@example.com",
  "tz": "Europe/London",
  "repos": [{"path": "/Users/you/code/acme", "label": "clone A"}],
  "records": [ { …one enriched per-day record (Step 3 shape)… } ]
}
```
`records[]` is the durable source of truth; the `.md/.csv/.html` are rendered from it. Write it
**after** the three files succeed (so a crash never leaves state ahead of the outputs).

---

## Step 1 — Discover repos & worktrees

**The current project is the default.** Anchor on the repo the user is sitting in, then scan other
locations to find *that same project's* other clones/worktrees — never mix in unrelated projects.

### 1a. Establish the anchor repo
- If `--repo` was given, the first one is the anchor.
- Else if the cwd is inside a git repo (`git rev-parse --show-toplevel`), that toplevel is the anchor.
- Else (not in a repo, no `--repo`) → **fallback**, see 1c.

From the anchor, capture:
- its toplevel path and basename → default `--name` (e.g. cwd `~/code/acme` → name `acme`);
- its **full remote set** — `git -C <anchor> remote -v` → normalize each URL (lowercase, strip
  `.git`, strip protocol/host so `git@github.com:Org/x.git` and `https://github.com/Org/x` match on
  `org/x`). A repo can have several remotes (e.g. a personal fork **and** the shared origin); keep
  them all.

### 1b. Find this project's other clones & worktrees (remote-matched)
Scan likely clone locations for git repos — `~/Downloads/*`, `~/conductor/workspaces/*/*`, and the
anchor's sibling directories (`<parent-of-anchor>/*`). For each candidate, read its normalized remote
set and **keep it only if it shares at least one remote with the anchor** (transitively — a fork that
links two upstreams still clusters together). This is what collapses `clone A`/`clone B`-style
duplicate checkouts of one product while excluding other projects living in the same folders.

For every kept clone, enumerate worktrees: `git -C <clone> worktree list` (a worktree shares its
clone's object DB, so `git log --all` in the clone already covers all of them — you only need to
enumerate worktrees for **labels**, not for extra log passes). Dedup commits by hash across the
distinct object DBs (one per clone).

Build a concise **path → label** map for the `repos` column: anchor checkout first, then sibling
clones and worktrees by a short basename/branch-derived label. Keep labels short. Attribute a commit
to a clone by which clone's object DB contains its hash. **Missing/deleted paths → warn and skip,
never abort.**

### 1c. Fallback when not in a git repo
With no anchor and no `--repo`: sweep `~/Downloads/*` and `~/conductor/workspaces/*/*`, cluster repos
by shared remote, and pick the cluster with the **most commits by `--author` in range** as the
project (its basename → `--name`). Print which project was chosen so the user can re-run with
`--repo`/`--name` if it guessed wrong.

> Reference run (the Musta project) for sanity only: anchor remote `org/qorum-meeting-flow` matched
> two clones — `~/Downloads/qorum-meeting-flow` (also forks `…/qorum-local`) and
> `~/Downloads/qorum-meeting-flow-dev` — plus their worktrees under `~/conductor/...` and
> `.claude/worktrees/`. Hash-dedup across the two object DBs = 261 commits.

---

## Step 2 — Gather evidence

**Convert every timestamp from UTC (or its stored offset) to `--tz` before bucketing into a calendar
day.** All four sources below stamp in UTC except git, which gives an offset-aware time via `%aI`.

**Incremental scoping (UPDATE mode).** In UPDATE mode the gather window is
`[window_start 00:00 .. now]` (from Step 0.5), **not** the full range — this is the entire speed win,
so scope each source to it:
- **git** — `--since=<window_start>T00:00:00<tzoffset>` (the `--until` is now). Fast either way.
- **Claude sessions** — only read `.jsonl` files whose **mtime ≥ `window_start`**
  (`find ~/.claude/projects -name '*.jsonl' -newermt <window_start>`); skip the rest. This avoids
  re-parsing thousands of old transcripts and is the biggest saving.
- **Codex sessions** — only the date-partitioned dirs `~/.codex/sessions/YYYY/MM/DD/` with date
  `≥ window_start`.
- **GitHub PRs** — query the window twice and union: `--created=<window_start>..<today>` **and**
  `--merged-at=<window_start>..<today>` (so a PR opened earlier but merged inside the window is caught).
- **Linear (MCP)** — `list_issues` with `updatedAt >= <window_start>` (+ comments) only.

(FULL mode uses the full range as `window_start = from`.)

### 2a. Git commits (the firmest signal)
Per clone:
```bash
git -C <clonepath> log --all \
  --since="<from>T00:00:00+01:00" --until="<to+1day>T00:00:00+01:00" \
  --author="<email>" \
  --pretty=format:'%H|%aI|%an|%ae|%s'
```
- `--all` is **mandatory** — the dated work usually isn't on the checked-out branch.
- **Dedup by full hash `%H`** across all clones (clone A and clone B are separate object DBs that
  contain the same logical commits). A `set()` of hashes is the dedup.
- Each commit is a **point event** at `%aI`. Record its repo label, subject `%s`, and any tickets.
- The deduped commit total is your correctness check.

### 2b. Reflog (best-effort point events)
```bash
git -C <clonepath> reflog --all --date=iso
```
Use entries in range as extra presence points. Reflog prunes after ~90 days, so treat as optional.

### 2c. Claude Code sessions (real time spans)
Glob `~/.claude/projects/*/*.jsonl`. Each line is JSON; lines carry a top-level `timestamp`
(ISO-8601 UTC) and a top-level `cwd`. Per file: **session span = min/max `timestamp`** over lines
that have one (skip header lines without a timestamp). Attribute the session to a repo by matching
`cwd` against a discovered repo/worktree path. Count one session per file overlapping the day; the
span is a real interval `[start,end]`.

### 2d. Codex sessions (real time spans)
Glob `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl`. First line is `type:session_meta` with
`payload.cwd` and `payload.git`. Span = min/max top-level `timestamp`. Attribute via `payload.cwd`.

### 2e. GitHub PRs (optional — skip cleanly if absent)
```bash
gh search prs --author=@me --created=<from>..<to> \
  --json number,title,createdAt,closedAt,state,repository --limit 1000
```
- **Scope to this project's repos:** keep only rows whose `repository.nameWithOwner` matches one of the
  resolved clones' remotes (Step 1). `gh search` returns PRs across *all* your repos, so this is what
  keeps another project's PRs out.
- `prs_opened` per day = bucket `createdAt`; `prs_merged` = rows with `state=="merged"` bucketed by
  `closedAt`. Wide range → split the window near the 1000-row cap.
- If `gh` is not installed/authed, or the project has no PRs → all PR counts are 0. **Not an error** —
  treat GitHub as an absent source (see "Optional sources" below).

### 2f. Linear (MCP) — optional, auto-detected, project-scoped
Linear is **workspace-wide**, so for a project that doesn't use Linear it would inject unrelated
noise. Decide whether Linear applies, per run:

1. **`--no-linear`** → skip entirely.
2. **`--linear`** → force-enable.
3. **Default = auto-detect.** Compute the project's **ticket prefixes** = the set of `[A-Z]{2,}`
   prefixes from `[A-Z]{2,}-\d+` matches in commit subjects + branch names within the range (Step 2g).
   - **No ticket refs at all → treat the project as non-Linear: skip the Linear query** (also a real
     speed win — no MCP round-trips).
   - **Ticket refs found → query Linear, scoped to those prefixes:** `mcp__linear__list_users` to
     resolve the `--author` user, then `mcp__linear__list_issues` (assignee = user, `updatedAt` within
     window, paginate) and **keep only issues whose identifier prefix is in the project's prefix set**.
     `linear` (per day) = count of those issues' lifecycle stamps (created/started/completed/canceled)
     that fall on the day; collect their identifiers to augment tickets.

When Linear is skipped or returns nothing, every day's `linear` = 0 and Linear is recorded as an
**absent source** (don't flip off-days to active on Linear, and don't list it as evidence — see below).

### 2g. Tickets
Per day, tickets = sorted-unique union of:
- regex `[A-Z]{2,}-\d+` (e.g. `LIM-123`, `TWT-9`) found in commit subjects and branch names — this is
  the **primary** source and works with any project's prefix, **plus**
- Linear identifiers touched that day (only if Linear is active).

### Optional sources — degrade, never fail
Every source except git is **optional**. git commits (+ reflog) are the floor; a usable timesheet can
be built from git + agent sessions alone. Track which sources actually contributed (`commits>0`,
`claude+codex sessions>0`, `prs_opened+prs_merged>0`, `linear>0` across the range) and adapt the
outputs to only what was present:
- the `.md` **methodology line** lists only the evidence sources that contributed;
- the `.md` **day headlines** omit a segment for any globally-absent source (e.g. drop `· N Linear`
  when there is no Linear, drop `· NPR↑/NPR↓` when there are no PRs);
- add a one-line **footnote** noting what was unavailable (e.g. "This project isn't tracked in Linear,
  so estimates rest on git commits, agent sessions and PRs.").
A missing/failed source is a warning, never an abort.

---

## Step 3 — Build the canonical per-day dataset

Produce **one enriched record per calendar day** in the range (include off days). This single
dataset is the source for all three files.

```json
{"date":"2026-06-22","dow":"Mon","start":9.53,"end":23.162,"span_hours":13.63,
 "hours":13.8,"session_blocks":1,"commits":44,"claude_sessions":16,"codex_sessions":0,
 "sessions":16,"reflog_actions":0,"prs_opened":22,"pr_m":22,"linear":36,
 "repos":["clone A","clone B"],"tickets":["LIM-609","..."],"note":"",
 "what":"LIM-636: …; LIM-633: …","off":false}
```
- `sessions` = `claude_sessions + codex_sessions` (HTML uses `sessions`; CSV keeps both split).
- `pr_m` = merged PRs (HTML key); CSV uses `prs_opened` / `prs_merged`.
- `what` = the day's commit subjects joined with `; ` (the HTML/CSV escape `<` themselves).
- `note` = short status when there are no commits, e.g. `agent session(s), no commit` /
  `Linear planning / ticket updates`. Off day → `off (no activity logged)`.

### Work-block / active-hours algorithm
`GAP = 90 min`, `PAD = 10 min`, timezone = `--tz`.

```
for each day in [from..to]:
    events = sessions (real intervals [start,end])  +  points (commits/reflog/linear at time t)
    if events empty:
        emit {off:true, start:null, end:null, hours:0.0, span_hours:0.0, session_blocks:0, …zeros}
        continue

    # SPAN — the "Worked" window, from UNPADDED bounds
    span_start = min(start of each session, t of each point)
    span_end   = max(end   of each session, t of each point)
    span_hours = round((span_end - span_start) in hours, 2)   # may be 0.0 for a lone point

    # ACTIVE — merge PADDED intervals
    intervals = [ [s,e] for sessions ] + [ [t-PAD, t+PAD] for points ]
    sort intervals by start
    blocks = []; cur = intervals[0]
    for iv in intervals[1:]:
        if iv.start - cur.end <= GAP: cur.end = max(cur.end, iv.end)   # same block
        else: blocks.push(cur); cur = iv                              # gap > 90m → new block
    blocks.push(cur)
    hours = round(sum(b.end-b.start for b in blocks) in hours, 1)
    session_blocks = len(blocks)

    start = decimal_hours(span_start)   # local H + minutes/60  → 18:26 = 18.434
    end   = decimal_hours(span_end)
    emit {off:false, start, end, span_hours, hours, session_blocks, …counts, repos, tickets, what}
```

**Validation anchors** (reference run, 2026-05-15 → 2026-06-26): total deduped commits = **261**;
2026-05-18 (lone Linear point) → `start==end==13.502`, span `0.0h`, active `0.33h`; 2026-05-15
(commit 18:26) → padded block `18:16–18:36`; busiest 2026-06-22 → `13.8h`, 44 commits, 16 sessions,
22 PRs merged. If your numbers diverge wildly, recheck `--all`, the author filter, and tz handling.

---

## Step 3.5 — Merge into prior state (UPDATE mode only)

FULL mode skips this — its recomputed records ARE the full dataset.

In UPDATE mode you only built records for `[window_start .. today]`. Merge them into the saved
`records[]`:

1. **Replace by date.** For every recomputed day, drop the old record with that `date` and insert the
   new one. Recompute is authoritative for those days (a commit/session belongs to exactly one day and
   that day was rebuilt from scratch), so there is **no double-counting** — replace, never add.
2. **Append new days.** Days after the previous `covered_through` are simply added.
3. **Keep older days untouched.** Everything before `window_start` is reused verbatim.
4. **Fill gaps.** Ensure one record per calendar day from `from` to today; any day with no events is an
   `off` record (so a skipped week shows as off days, not missing rows).
5. Sort by `date`. Recompute all **aggregates** (KPIs, weekly totals, busiest/longest/off lists) from
   the full merged set — never from the window alone.

The merged `records[]` is what Step 4 renders and what you write back to the state file (with
`covered_through = now`, `generated_at = now`).

---

## Step 4 — Write the outputs (strict order)

### 4a. Write the `.md` FIRST  → `<out>/<name>-worklog.md`
Render from the **full merged `records[]`** (not just the window). The covered range
`(<from-pretty> – <now-pretty>)` is shown in the title/subtitle. Mirror the reference structure:

- `# <Name> — daily work log (<from-pretty> – <to-pretty>)`
- An italic methodology line listing **only the sources that contributed** (see Step 2 "Optional
  sources"): _Forensic reconstruction from this machine. Evidence: git commits + HEAD reflog across
  <N clones + worktrees>[, Claude Code + Codex session logs][, GitHub PRs][, Linear issue lifecycle].
  All times local <tz abbrev>._ — drop the Linear clause for a non-Linear project, drop the PR clause
  if there were no PRs, etc.
- `## How to read this` — bullets defining **Span**, **Active** (90-min gap rule; sessions are real
  spans, point events padded ±10 min), and any data-quality caveat (e.g. periods with commits but no
  agent-session logs → commit count is the firmer signal, active hours undercount).
- `## Summary` — active days / commit days / off days; deduped commits; PRs opened / merged;
  Claude + Codex session counts; total span and total active hours; busiest-by-commits (top ~5);
  longest-active (top ~5); the explicit off-days list.
- `### Weekly totals` — table: `| Week (Mon) | Active days | Commits | Active h | Span h | PRs merged |`.
- `## Day by day` — grouped by `### Week of <Monday>`, each day:
  - headline: `**<date> <Dow>** — <HH:MM>–<HH:MM>  ·  **<active>h active** (span <span>h, <n> blocks)  ·  <commits> commits · <sessions> sessions · <opened>PR↑/<merged>PR↓ · <linear> Linear` (append a small tag like `⚙️ manual/web`, `🗂 planning`, `🧪 WIP` where apt). **Omit any segment whose source was globally absent** — e.g. no `· <linear> Linear` for a non-Linear project, no `· <opened>PR↑/<merged>PR↓` if there were no PRs. Off days: `**<date> <Dow>** — _off (no activity logged)_`.
  - `  - _Worked on:_ <commit subjects, truncated>`
  - `  - _Where:_ <repo labels>  ·  _Tickets:_ <ticket ids>` (omit empties)
  - padded block sub-lines: `    - <HH:MM>–<HH:MM> (<Hh MM>) — <n> commits[, <n> Claude][, <n> Codex]`

### 4b. Write the `.csv`  → `<out>/<name>-worklog.csv`
Exact 18-column header (one row per day, text fields quoted):
```
date,weekday,active_start,active_end,span_hours,active_hours,session_blocks,commits,claude_sessions,codex_sessions,prs_opened,prs_merged,linear_events,reflog_actions,repos_touched,tickets,note,what_worked_on
```
`active_start`/`active_end` = `HH:MM` of the unpadded span bounds (blank on off days);
`repos_touched` = labels joined by spaces; `tickets` = ids joined by spaces.

### 4c. Generate the `.html` LAST  → `<out>/<name>-worklog.html`
Only **after** the `.md` is on disk. The dashboard is a **single self-contained file styled with
shadcn/ui** — React + Tailwind + Recharts + the shadcn design tokens are pulled from CDNs, and
shadcn-style `Card` / `Badge` / `Table` / chart components are defined inline. It still opens by
double-click (no build/npm), but **needs internet** the first time to fetch the CDN scripts (say so
when you report the file).

Take the template below and substitute exactly five tokens:

| Token | Where | Fill with |
|---|---|---|
| `{{TITLE}}` | `<title>` | e.g. `Musta — hours worked` (HTML-escape) |
| `{{HEADING}}` | `#root data-title` | same as title (HTML-attr-escape `"` → `&quot;`) |
| `{{DATE_RANGE}}` | `#root data-range` | e.g. `15 May – 26 Jun 2026.` (HTML-attr-escape) |
| `{{FOOTNOTE}}` | `#root data-foot` | the caveat line, mirroring the `.md` note (HTML-attr-escape) |
| `__DATA_JSON__` | `<script>` | `JSON.stringify(records)` — a raw JS array literal, **no surrounding quotes** |

The text tokens live in HTML attributes (not JS string literals), so they only need HTML-attribute
escaping — no JS escaping. The `records` array must use the HTML record shape (keys: `date,dow,start,
end,hours,commits,sessions,pr_m,linear,repos,tickets,what,off`). Everything else in the template is
generic and data-driven — do not change the token config, the component definitions, or the app logic.

CDN-offline note: if the user needs a file that works without internet, that's the "real shadcn
project" path, not this one — keep this template's CDN model unless they ask otherwise.

#### Definitive shadcn/ui setup — DO NOT DEVIATE
The template below is the shadcn/ui setup that renders correctly as a single file. Every item is
load-bearing; the most common "looks almost-but-not shadcn / renders broken" failures map 1:1 to
skipping one of these. Emit the template **verbatim** (only the 5 tokens change) — do not "simplify"
the head.

1. **Tailwind = v3 Play CDN, pinned:** `https://cdn.tailwindcss.com/3.4.17`. The Play CDN serves
   Tailwind **v3**, where the global `tailwind.config` object wires the tokens. Do **NOT** swap in
   `@tailwindcss/browser` (v4) — v4 ignores `tailwind.config` (CSS-first `@theme`) and the tokens
   silently stop generating, so `bg-card`/`border-border` produce nothing.
2. **Tokens as HSL triplets** in a plain `<style>` `:root`/`.dark` (e.g. `--background:0 0% 100%`),
   and the config must map colors with the **`<alpha-value>` placeholder**:
   `border:'hsl(var(--border) / <alpha-value>)'` (not bare `hsl(var(--border))`). Without the
   placeholder, opacity utilities shadcn relies on — `border-border/50`, `hover:bg-muted/50`,
   `bg-primary/90` — silently don't apply. Tokens MUST be HSL triplets (not the newer oklch values
   from ui.shadcn.com) because the config wraps them in `hsl()`.
3. **The shadcn base layer is mandatory** — a `<style type="text/tailwindcss">` block with
   `@layer base{ *{ @apply border-border; } body{ @apply bg-background text-foreground; } }`.
   This is what shadcn ships in `globals.css`; without it every border falls back to Tailwind's
   default gray (not the theme token) and the body isn't token-driven. Omitting it is the #1 reason
   it looks "off."
4. **Dark mode is class-based (`darkMode:['class']`) with a user-facing toggle.** Drive theme by
   adding/removing `dark` on `<html>` — never via `@media (prefers-color-scheme:dark)` on the tokens
   (that takes control away from the user). Three pieces, all required:
   - an **early-init inline `<script>` in `<head>`** (before the React scripts) that sets the initial
     `dark` class from `localStorage.theme` if set, else the system `prefers-color-scheme` — this both
     respects the user's saved choice and avoids a flash of the wrong theme;
   - a **shadcn `Button` (`variant="outline" size="icon"`) theme toggle** (sun/moon inline SVG, no
     extra icon CDN) in the header that flips the `dark` class and **persists to `localStorage`**;
   - hold the theme in React state so toggling **re-renders the chart** (its grid/axis colors are
     `hsl(var(--token))` and must re-resolve against the new theme).
5. **Recharts load order + deps:** `react` → `react-dom` → `prop-types` → `react-is` → `recharts` →
   `@babel/standalone`, all **pinned, via jsdelivr** (unpkg 404s on the recharts UMD path). Recharts'
   UMD externalizes `prop-types` AND `react-is`; if either global is missing the UMD factory throws
   and `window.Recharts` is `undefined` (chart + whole app fail to mount). No `crossorigin` attribute
   (it triggers CORS failures on CDN redirects from `file://`).
6. **Font:** Inter via Google Fonts, `font-sans`+`antialiased` on `<body>`.
7. **Chart = the shadcn Chart pattern, not a bare Recharts chart.** Wrap the chart in a
   `ChartContainer`-style div that scopes a `--color-<key>` CSS var and carries the recharts overrides
   (`[&_.recharts-surface]:outline-none`, `[&_.recharts-layer]:outline-none`,
   `[&_.recharts-sector]:outline-none`, `[&_.recharts-dot]:stroke-transparent`). Use a gradient
   **`AreaChart`** (`<defs><linearGradient>` from `--color` at 0.8 → 0.1 opacity; `Area type="natural"
   stroke="var(--color-…)" fill="url(#…)" fillOpacity={0.4} dot={false}`), `CartesianGrid vertical={false}`
   + token-colored axes (`tickLine`/`axisLine` false, ticks filled `muted-foreground`), and a
   `ChartTooltipContent`-style tooltip card: `rounded-lg border border-border/50 bg-background … shadow-xl`
   with a colored `rounded-[2px]` indicator and a `font-mono … tabular-nums` value. A plain Recharts
   `<BarChart>` with the default tooltip is the tell that it's "not true shadcn."
7. **Verify in a browser before declaring done** (Step 5): no console errors, `window.Recharts` is an
   object, `getComputedStyle(body).backgroundColor` is white, and a `.bg-card` border-color equals the
   token (`rgb(228, 228, 231)`), not gray-200 (`rgb(229, 231, 235)`).

#### HTML template (verbatim except the 5 tokens)
````html
<!doctype html><html lang="en"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1"><title>{{TITLE}}</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
<script src="https://cdn.tailwindcss.com/3.4.17"></script>
<script>
tailwind.config={darkMode:['class'],theme:{extend:{
 fontFamily:{sans:['Inter','ui-sans-serif','system-ui','-apple-system','Segoe UI','Roboto','Helvetica','Arial','sans-serif']},
 colors:{border:'hsl(var(--border) / <alpha-value>)',input:'hsl(var(--input) / <alpha-value>)',ring:'hsl(var(--ring) / <alpha-value>)',
  background:'hsl(var(--background) / <alpha-value>)',foreground:'hsl(var(--foreground) / <alpha-value>)',
  primary:{DEFAULT:'hsl(var(--primary) / <alpha-value>)',foreground:'hsl(var(--primary-foreground) / <alpha-value>)'},
  secondary:{DEFAULT:'hsl(var(--secondary) / <alpha-value>)',foreground:'hsl(var(--secondary-foreground) / <alpha-value>)'},
  muted:{DEFAULT:'hsl(var(--muted) / <alpha-value>)',foreground:'hsl(var(--muted-foreground) / <alpha-value>)'},
  accent:{DEFAULT:'hsl(var(--accent) / <alpha-value>)',foreground:'hsl(var(--accent-foreground) / <alpha-value>)'},
  destructive:{DEFAULT:'hsl(var(--destructive) / <alpha-value>)',foreground:'hsl(var(--destructive-foreground) / <alpha-value>)'},
  popover:{DEFAULT:'hsl(var(--popover) / <alpha-value>)',foreground:'hsl(var(--popover-foreground) / <alpha-value>)'},
  card:{DEFAULT:'hsl(var(--card) / <alpha-value>)',foreground:'hsl(var(--card-foreground) / <alpha-value>)'},
  'chart-1':'hsl(var(--chart-1) / <alpha-value>)','chart-2':'hsl(var(--chart-2) / <alpha-value>)','chart-3':'hsl(var(--chart-3) / <alpha-value>)',
  'chart-4':'hsl(var(--chart-4) / <alpha-value>)','chart-5':'hsl(var(--chart-5) / <alpha-value>)'},
 borderRadius:{lg:'var(--radius)',md:'calc(var(--radius) - 2px)',sm:'calc(var(--radius) - 4px)'}}}}
</script>
<style>
/* shadcn/ui default theme tokens (HSL triplets, for Tailwind v3 hsl(var(--x) / <alpha-value>) mapping) */
:root{--background:0 0% 100%;--foreground:240 10% 3.9%;--card:0 0% 100%;--card-foreground:240 10% 3.9%;
 --popover:0 0% 100%;--popover-foreground:240 10% 3.9%;--primary:240 5.9% 10%;--primary-foreground:0 0% 98%;
 --secondary:240 4.8% 95.9%;--secondary-foreground:240 5.9% 10%;--muted:240 4.8% 95.9%;--muted-foreground:240 3.8% 46.1%;
 --accent:240 4.8% 95.9%;--accent-foreground:240 5.9% 10%;--destructive:0 84.2% 60.2%;--destructive-foreground:0 0% 98%;
 --border:240 5.9% 90%;--input:240 5.9% 90%;--ring:240 5.9% 10%;--radius:0.5rem;
 --chart-1:221.2 83.2% 53.3%;--chart-2:212 95% 68%;--chart-3:216 92% 60%;--chart-4:210 98% 78%;--chart-5:212 97% 87%;}
.dark{--background:240 10% 3.9%;--foreground:0 0% 98%;--card:240 10% 3.9%;--card-foreground:0 0% 98%;
 --popover:240 10% 3.9%;--popover-foreground:0 0% 98%;--primary:0 0% 98%;--primary-foreground:240 5.9% 10%;
 --secondary:240 3.7% 15.9%;--secondary-foreground:0 0% 98%;--muted:240 3.7% 15.9%;--muted-foreground:240 5% 64.9%;
 --accent:240 3.7% 15.9%;--accent-foreground:0 0% 98%;--destructive:0 62.8% 30.6%;--destructive-foreground:0 0% 98%;
 --border:240 3.7% 15.9%;--input:240 3.7% 15.9%;--ring:240 4.9% 83.9%;
 --chart-1:221.2 83.2% 53.3%;--chart-2:212 95% 68%;--chart-3:216 92% 60%;--chart-4:210 98% 78%;--chart-5:212 97% 87%;}
</style>
<style type="text/tailwindcss">
/* shadcn/ui base layer — every border + the body use the theme tokens */
@layer base{
  *{ @apply border-border; }
  body{ @apply bg-background text-foreground; }
}
</style>
<script>
/* set initial theme before paint (no FOUC): explicit stored choice, else system preference */
(function(){try{var t=localStorage.getItem('theme');var d=t?(t==='dark'):window.matchMedia('(prefers-color-scheme:dark)').matches;document.documentElement.classList.toggle('dark',d);}catch(e){}})();
</script>
<script src="https://cdn.jsdelivr.net/npm/react@18.3.1/umd/react.production.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/react-dom@18.3.1/umd/react-dom.production.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/prop-types@15.8.1/prop-types.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/react-is@18.3.1/umd/react-is.production.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/recharts@2.15.4/umd/Recharts.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@babel/standalone@7.26.4/babel.min.js"></script>
</head>
<body class="min-h-screen font-sans antialiased">
<div id="root" data-title="{{HEADING}}" data-range="{{DATE_RANGE}}" data-foot="{{FOOTNOTE}}"></div>
<script>const DATA=__DATA_JSON__;</script>
<script type="text/babel" data-presets="react">
const _root=document.getElementById('root');
const HEADING=_root.dataset.title, DATE_RANGE=_root.dataset.range, FOOTNOTE=_root.dataset.foot;
document.title=HEADING;
const {useState,useMemo}=React;
const R=window.Recharts;
const cn=(...c)=>c.filter(Boolean).join(' ');
// shadcn-style primitives
const Card=({className,children})=><div className={cn("rounded-xl border bg-card text-card-foreground shadow-sm",className)}>{children}</div>;
const CardHeader=({className,children})=><div className={cn("flex flex-col space-y-1.5 p-6",className)}>{children}</div>;
const CardTitle=({className,children})=><h3 className={cn("font-semibold leading-none tracking-tight",className)}>{children}</h3>;
const CardDescription=({className,children})=><p className={cn("text-sm text-muted-foreground",className)}>{children}</p>;
const CardContent=({className,children})=><div className={cn("p-6 pt-0",className)}>{children}</div>;
const Badge=({variant="secondary",className,children})=>{const v={default:"border-transparent bg-primary text-primary-foreground",secondary:"border-transparent bg-secondary text-secondary-foreground",outline:"text-foreground"}[variant];return <span className={cn("inline-flex items-center rounded-md border px-1.5 py-0.5 text-[11px] font-medium",v,className)}>{children}</span>;};
const Table=({children})=><div className="relative w-full overflow-auto"><table className="w-full caption-bottom text-sm">{children}</table></div>;
const THead=({children})=><thead className="[&_tr]:border-b">{children}</thead>;
const TBody=({children})=><tbody className="[&_tr:last-child]:border-0">{children}</tbody>;
const TR=({className,children,...p})=><tr className={cn("border-b transition-colors hover:bg-muted/50",className)} {...p}>{children}</tr>;
const TH=({className,children,...p})=><th className={cn("h-10 px-2 text-left align-middle font-medium text-muted-foreground select-none",className)} {...p}>{children}</th>;
const TD=({className,children,...p})=><td className={cn("p-2 align-middle",className)} {...p}>{children}</td>;
const Button=({variant="outline",size="icon",className,children,...p})=>{
 const base="inline-flex items-center justify-center gap-2 whitespace-nowrap rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:pointer-events-none disabled:opacity-50";
 const v={outline:"border border-input bg-background shadow-sm hover:bg-accent hover:text-accent-foreground",ghost:"hover:bg-accent hover:text-accent-foreground"}[variant];
 const s={icon:"h-9 w-9",sm:"h-8 rounded-md px-3"}[size];
 return <button className={cn(base,v,s,className)} {...p}>{children}</button>;};
const SunIcon=()=><svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><circle cx="12" cy="12" r="4"/><path d="M12 2v2M12 20v2M4.93 4.93l1.41 1.41M17.66 17.66l1.41 1.41M2 12h2M20 12h2M6.34 17.66l-1.41 1.41M19.07 4.93l-1.41 1.41"/></svg>;
const MoonIcon=()=><svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M12 3a6 6 0 0 0 9 9 9 9 0 1 1-9-9Z"/></svg>;
function ThemeToggle({dark,onToggle}){return <Button aria-label="Toggle light/dark theme" title="Toggle theme" onClick={onToggle}>{dark?<SunIcon/>:<MoonIcon/>}</Button>;}
// helpers
const toHM=d=>{if(d==null)return '–';let m=Math.round(d*60);return String(Math.floor(m/60)).padStart(2,'0')+':'+String(m%60).padStart(2,'0');};
const fmtUK=iso=>{const p=iso.split('-');return p[2]+'/'+p[1]+'/'+p[0];};
const mondayOf=iso=>{const[y,m,dd]=iso.split('-').map(Number);const u=new Date(Date.UTC(y,m-1,dd));const off=(u.getUTCDay()+6)%7;u.setUTCDate(u.getUTCDate()-off);return u.toISOString().slice(0,10);};
const win=d=>d.off?'—':toHM(d.start)+'–'+toHM(d.end);
const sum=k=>DATA.reduce((a,d)=>a+(d[k]||0),0);
const worked=DATA.filter(d=>!d.off);
// shadcn ChartContainer: scopes --color-* vars + applies the recharts overrides
const CHART_OVERRIDES="text-xs [&_.recharts-surface]:outline-none [&_.recharts-layer]:outline-none [&_.recharts-sector]:outline-none [&_.recharts-dot]:stroke-transparent";
function ChartContainer({className,style,children}){
 return <div className={cn("w-full",CHART_OVERRIDES,className)} style={style}>{children}</div>;
}
// shadcn ChartTooltipContent
function ChartTip({active,payload}){
 if(!active||!payload||!payload.length)return null;const d=payload[0].payload;
 const rows=d.off?[]:[['Hours',d.hours+'h'],['Commits',d.commits],['Sessions',d.sessions]];
 return <div className="grid min-w-[9rem] items-start gap-1.5 rounded-lg border border-border/50 bg-background px-2.5 py-1.5 text-xs shadow-xl">
  <div className="font-medium">{d.dow} {fmtUK(d.date)}{d.off?' · day off':''}</div>
  {d.off?null:<div className="grid gap-1.5">
   {rows.map(([k,v],i)=>(
    <div key={k} className="flex w-full items-center gap-2">
     {i===0?<div className="h-2.5 w-2.5 shrink-0 rounded-[2px]" style={{background:'var(--color-hours)'}}/>:<div className="w-2.5 shrink-0"/>}
     <div className="flex flex-1 items-center justify-between leading-none">
      <span className="text-muted-foreground">{k}</span>
      <span className="font-mono font-medium tabular-nums text-foreground">{v}</span>
     </div>
    </div>))}
   {d.what?<div className="mt-0.5 max-w-[220px] text-muted-foreground line-clamp-3">{d.what}</div>:null}
  </div>}
 </div>;
}
function HoursChart(){
 const data=DATA.map(d=>({...d,label:fmtUK(d.date).slice(0,5)}));
 return <ChartContainer className="h-[250px]" style={{['--color-hours']:'hsl(var(--chart-1))'}}>
  <R.ResponsiveContainer width="100%" height="100%">
   <R.AreaChart data={data} margin={{top:10,right:12,left:-12,bottom:0}}>
    <defs>
     <linearGradient id="fillHours" x1="0" y1="0" x2="0" y2="1">
      <stop offset="5%" stopColor="var(--color-hours)" stopOpacity={0.8}/>
      <stop offset="95%" stopColor="var(--color-hours)" stopOpacity={0.1}/>
     </linearGradient>
    </defs>
    <R.CartesianGrid vertical={false} stroke="hsl(var(--border))" strokeOpacity={0.7}/>
    <R.XAxis dataKey="label" tickLine={false} axisLine={false} tickMargin={8} minTickGap={24} interval={2} tick={{fontSize:10,fill:'hsl(var(--muted-foreground))'}}/>
    <R.YAxis tickLine={false} axisLine={false} width={30} tickMargin={4} tick={{fontSize:10,fill:'hsl(var(--muted-foreground))'}} tickFormatter={v=>v+'h'}/>
    <R.Tooltip cursor={false} content={<ChartTip/>}/>
    <R.Area dataKey="hours" type="natural" stroke="var(--color-hours)" strokeWidth={2} fill="url(#fillHours)" fillOpacity={0.4} dot={false} activeDot={{r:3,strokeWidth:0}}/>
   </R.AreaChart>
  </R.ResponsiveContainer>
 </ChartContainer>;
}
function Weeks(){
 const weeks=useMemo(()=>{const wk={};DATA.forEach(d=>{const k=mondayOf(d.date);(wk[k]=wk[k]||{key:k,h:0,c:0,days:0});wk[k].h+=d.hours;wk[k].c+=d.commits;if(!d.off)wk[k].days++;});return Object.values(wk).sort((a,b)=>a.key.localeCompare(b.key));},[]);
 const mx=Math.max(...weeks.map(w=>w.h),1);
 return <div className="space-y-3">{weeks.map((w,i)=>
  <div key={i} className="flex items-center gap-3">
   <div className="w-28 shrink-0 text-sm"><span className="font-medium">Week {i+1}</span> <span className="text-muted-foreground text-xs">{fmtUK(w.key)}</span></div>
   <div className="flex-1 h-2.5 rounded-full bg-muted overflow-hidden"><div className="h-full rounded-full bg-[hsl(var(--chart-1))]" style={{width:Math.round(w.h/mx*100)+'%'}}/></div>
   <div className="w-12 text-right text-sm font-medium tabular-nums">{Math.round(w.h)}h</div>
   <div className="w-32 text-right text-xs text-muted-foreground tabular-nums">{w.days} days · {w.c} commits</div>
  </div>)}</div>;
}
function DayTable(){
 const cols=[{k:'date',l:'Date'},{k:'dow',l:'Day'},{k:'win',l:'Worked'},{k:'hours',l:'Hours',num:1},{k:'commits',l:'Commits',num:1},{k:'sessions',l:'Sessions',num:1},{k:'what',l:'What you worked on'}];
 const [sort,setSort]=useState({k:'date',dir:1});
 const val=(d,k)=>k==='win'?(d.start??99):d[k];
 const rows=useMemo(()=>[...DATA].sort((a,b)=>{let x=val(a,sort.k),y=val(b,sort.k);if(typeof x==='string')return sort.dir*x.localeCompare(y);return sort.dir*((x||0)-(y||0));}),[sort]);
 const click=k=>setSort(s=>s.k===k?{k,dir:-s.dir}:{k,dir:1});
 return <Table><THead><TR className="hover:bg-transparent">{cols.map(c=>
   <TH key={c.k} className={cn("cursor-pointer",c.num&&"text-right")} onClick={()=>click(c.k)}>{c.l}{sort.k===c.k?(sort.dir>0?' ↑':' ↓'):''}</TH>)}</TR></THead>
  <TBody>{rows.map(d=>
   <TR key={d.date} className={d.off?"text-muted-foreground":""}>
    <TD>{fmtUK(d.date)}</TD><TD>{d.dow}</TD><TD className="tabular-nums">{win(d)}</TD>
    <TD className="text-right tabular-nums">{d.off?'—':d.hours.toFixed(1)}</TD>
    <TD className="text-right tabular-nums">{d.commits||''}</TD>
    <TD className="text-right tabular-nums">{d.sessions||''}</TD>
    <TD className="text-muted-foreground"><span>{d.what||(d.off?'day off':'')}</span>{d.tickets&&d.tickets.length?<span className="ml-1 inline-flex flex-wrap gap-1 align-middle">{d.tickets.map(t=><Badge key={t} variant="outline">{t}</Badge>)}</span>:null}</TD>
   </TR>)}</TBody></Table>;
}
function Kpi({label,value}){return <Card><CardHeader className="pb-2"><CardDescription>{label}</CardDescription><CardTitle className="text-3xl tabular-nums">{value}</CardTitle></CardHeader></Card>;}
function App(){
 const [dark,setDark]=useState(()=>document.documentElement.classList.contains('dark'));
 const toggle=()=>setDark(v=>{const nv=!v;document.documentElement.classList.toggle('dark',nv);try{localStorage.setItem('theme',nv?'dark':'light');}catch(e){}return nv;});
 const kpis=[['Days worked',worked.length+' of '+DATA.length],['Total hours','~'+Math.round(sum('hours'))+'h'],['Commits',sum('commits')],['PRs merged',sum('pr_m')]];
 return <div className="mx-auto max-w-5xl px-6 py-10 space-y-6">
  <div className="flex items-start justify-between gap-4">
   <div className="space-y-1">
    <h1 className="text-2xl font-semibold tracking-tight">{HEADING}</h1>
    <p className="text-sm text-muted-foreground max-w-3xl">{DATE_RANGE} <b>Estimated hours</b> = how long you were active each day, from your git commits, coding-agent sessions and local git activity. A break longer than 90 minutes ends a working block; the day's hours are the blocks added up. <b>Worked</b> shows the first→last activity time.</p>
   </div>
   <ThemeToggle dark={dark} onToggle={toggle}/>
  </div>
  <div className="grid gap-4 sm:grid-cols-2 lg:grid-cols-4">{kpis.map(k=><Kpi key={k[0]} label={k[0]} value={k[1]}/>)}</div>
  <Card><CardHeader className="pb-2"><CardTitle className="text-base">Hours per day</CardTitle><CardDescription>{DATE_RANGE} Hover the area for a day's detail.</CardDescription></CardHeader><CardContent><HoursChart/></CardContent></Card>
  <Card><CardHeader className="pb-2"><CardTitle className="text-base">By week</CardTitle></CardHeader><CardContent><Weeks/></CardContent></Card>
  <Card><CardHeader className="pb-2"><CardTitle className="text-base">Every day</CardTitle><CardDescription>Click a column header to sort.</CardDescription></CardHeader><CardContent><DayTable/></CardContent></Card>
  <p className="text-xs text-muted-foreground max-w-3xl">{FOOTNOTE}</p>
 </div>;
}
ReactDOM.createRoot(_root).render(<App/>);
</script></body></html>
````

### 4d. Write the state file LAST  → `<out>/.<name>-worklog.state.json`
Only **after** the `.md`/`.csv`/`.html` are written, persist the full merged dataset + metadata using
the Step 0.5 schema (`covered_through = now`, `generated_at = now`, the resolved `from`, `author`,
`tz`, `repos`, and the complete `records[]`). Writing it last guarantees the state never gets ahead of
the rendered files if a step fails.

---

## Step 5 — Sanity-check before finishing

- The deduped commit total in the `.md` Summary matches
  `git -C <clone> log --all --since --until --author=<email> --pretty=%H` unioned across clones and `sort -u | wc -l`.
- `.md`, `.csv`, `.html` agree on per-day commits/hours/sessions.
- `DATA` is valid JSON; no leftover `{{…}}`/`__DATA_JSON__` tokens remain in the `.html`.
- **Render-verify the dashboard in a real browser** (the file uses runtime React/Tailwind/Recharts,
  so static inspection is not enough). Load it headless (e.g. gstack `/browse`: `goto file://…`) and
  confirm: no console errors; `typeof window.Recharts === 'object'`; an `h1` exists; KPI numbers sum
  correctly; `getComputedStyle(document.body).backgroundColor` is white (light default); a `.bg-card`
  border-color is the token `rgb(228, 228, 231)` (proves the base layer applied, not gray-200); the
  area chart has a curve + gradient and the table has one row per day. Click the theme toggle and
  confirm `<html>` gains/loses `dark`, the body bg flips (light `rgb(255,255,255)` ↔ dark
  `rgb(9,9,11)`), and the choice persists to `localStorage.theme`. If the page is unstyled or the
  chart is missing, the CDN scripts were blocked — say so rather than shipping a broken file.
- **UPDATE-mode checks:** the state file exists and its `covered_through` is now; days before
  `window_start` are byte-identical to the prior run (only the trailing edge changed); the merged
  record count = one per day from `from` to today (no gaps, no dupes). **Parity:** an UPDATE result
  must match a `--full` rebuild of the same range — spot-check totals (commits/PRs/active hours) are
  identical. If they differ, the merge or windowing is wrong.
- Report the output paths, the mode (FULL/UPDATE), what the update changed (re-finalized day + new
  days, with deltas), and the headline KPIs. Note the file needs internet on first open (CDN scripts).

## Notes & gotchas
- **Order matters**: write `.md` → `.csv` → `.html` → state file. Never emit the HTML before the
  markdown, and never write state before the three files (so state never gets ahead of the outputs).
- **Incremental is replace-by-date, not add.** Recompute whole days in the window and replace them;
  reuse all earlier days from state. Re-finalize the last covered day (a mid-run left it partial).
- **`--all` everywhere** on git log/reflog; **filter by author**; **dedup by hash**.
- **UTC → tz** for Claude/Codex/GitHub/Linear stamps before day-bucketing; git `%aI` is already offset-aware.
- Be honest about data quality in the methodology + footnote (e.g. ranges with commits but no agent
  logs undercount active hours — say so).
- Skip missing repos/worktrees with a warning; never abort the whole run for one bad path.
