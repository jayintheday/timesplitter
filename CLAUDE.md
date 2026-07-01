# CLAUDE.md — notes for agents working on `time-splitters`

This is the build/maintenance guide. `SKILL.md` is the runtime spec (how to run the skill); this file
is the **why**, the design decisions, and the **landmines that already cost debugging time** — read it
before editing `SKILL.md` so you don't reintroduce a fixed bug.

## What the skill does
Reconstructs a personal work log from local forensic evidence (git commits+reflog across all
clones/worktrees, Claude + Codex session logs, GitHub PRs, Linear) and emits, in this order:
`.md` (source-of-truth worklog) → `.csv` (18 cols) → `.html` (shadcn/ui dashboard) → hidden
`.state.json`. It was reverse-engineered from a hand-made output the user liked
(a personal reference timesheet from an earlier internal project — names anonymized in this doc).
Before gathering, a first-run-only **pre-flight checkpoint** (Step 1.5) confirms the guessed config and
**detects other connected MCP servers** (Calendar, Notion, …) it could use, nudging the user about
them — see the decisions and landmine #11 below.

## Decisions made WITH the user (don't silently reverse these)
- **Pure instructions, no bundled helper script.** The skill tells the model to write a throwaway
  Python script into the scratchpad at runtime. Rationale: transparency + fresh each run. (So there is
  intentionally no `.py` in this repo.)
- **Reusable + smart defaults**, anchored on the **current project** (cwd repo); sibling clones found
  by shared remote. Not hard-coded to the original reference project it was built against.
- **One evolving timesheet**: stable filenames (`<name>-worklog.{md,csv,html}`) updated in place, with
  a hidden `.<name>-worklog.state.json` as durable state. Range lives *inside* the docs.
- **Catch up to now on every run** (incremental): re-fetch only the trailing edge; `--full` rebuilds.
- **shadcn/ui dashboard as a single self-contained CDN file** (not a real shadcn project). Needs
  internet on first open. The "real shadcn project" path was explicitly declined.
- **Light/dark toggle**, default = system preference, choice persisted.
- **Period toggle (7d/14d/30d/All) on the HTML dashboard**, added because a long project's chart
  degenerates into a long flat line with a few unreadable spikes (real example: `isleofculture`,
  ~150 days in range but only 21 active). `DATA` is one record per calendar day in order, so "last N
  days" is `DATA.slice(-N)` — no date math. It's a *global* filter: KPIs, chart, weekly bars and the
  day table all recompute from the same filtered slice, not just the chart, so the numbers stay
  internally consistent as you switch periods. Default is `"all"` (matches pre-feature behavior; a
  first load never silently hides history) and the choice persists to `localStorage.period`, mirroring
  the theme toggle. See landmine #12 for the chart-scaling bug this required fixing. **Placement:** it
  sits in the "Hours per day" card's own header, directly above the chart it scopes — not in the top
  masthead next to the theme toggle. The user moved it there after the first pass put it up top;
  visually anchoring the control to the thing it visibly changes (the graph) reads better than a
  global-feeling toggle floating next to dark/light mode. It still scopes the KPIs/weekly/table too,
  only its position moved.
- **Pre-flight checkpoint (Step 1.5), first run only.** The skill used to guess everything and run
  without checking in. Now, on the first run (or `--reconfigure`, or a genuinely ambiguous guess) it
  pauses *once* with an interactive multiple-choice confirmation (`AskUserQuestion`) of the resolved
  config + sources, persists the answers to state, and stays silent on every later "catch up to now"
  run. `--yes`/`--no-confirm` and non-interactive/headless runs skip it (back to the old auto-proceed
  behavior). This is the **only** place the skill ever blocks for input — keep it first-run-only so
  daily top-ups stay fast and scriptable.
- **MCP detection = scan visible `mcp__*` tool namespaces.** Detection reads the in-context tool list
  (optionally cross-checks `claude mcp list`); no network call, no extra permission. Servers are
  classified by an evidence-source registry (Step 2 preamble) into **auto-tier** (project-scopable,
  used automatically — issue trackers via the prefix gate, GitHub via remote match), **nudge-tier**
  (fuzzy/personal scope — Calendar/Notion/Slack/Gmail/Drive, mentioned not used), and an **ignore
  deny-list** (Spotify/Canva/Vercel/Supabase/… — never mentioned, so the nudge stays gentle).
- **Detect-and-nudge only (this release).** Linear stays the only *wired* MCP source. Other detected
  servers are surfaced (plan block + checkpoint + persistent artifact footnote), not gathered. Wiring
  their data-pull is a deliberate follow-up.
- **First-run default `--from` = "when you started this project," not a fixed 6-week window.**
  Originally defaulted to `--to` minus 6 weeks — leftover from building the skill against a single
  reference project that was already recent, so a 6-week window and "since I started" looked identical
  and the gap never surfaced. Caught testing against `isleofculture` (real first commit 2026-01-29):
  the 6-week default silently dropped ~4 months and 33 commits with zero indication anything was
  missing. Now Step 1d runs a cheap unbounded `git log --author --reverse` pass per discovered repo
  (after repo discovery, before Step 2's bounded gather) and takes the earliest date across them as
  `--from`. Falls back to the old 6-week default **only** if `--author` has zero commits in every
  discovered repo — deliberately not "the repo's first commit by anyone," which would be wrong on a
  shared codebase joined long after it started. The Step 1.5 checkpoint still shows the resolved range
  before anything runs, so a huge computed range is visible and overridable on first run. Later runs
  are unaffected — `from` is persisted in state and reused verbatim regardless of how it was derived
  (landmine #11's parity contract).

## Decided, not yet built
- **Calendar = bounded interval → real hours** (the model for *when* Calendar is eventually wired).
  Each accepted, single-day, non-all-day meeting becomes a real `[start,end]` block, clipped to the
  day and capped per-event, merged into active hours alongside the padded points. This is NOT the
  landmine-#8 failure: #8 was an *unbounded* span (a session left open overnight = 45h); a meeting is
  a *short, trusted, bounded* interval still clipped to `[day0,day1]`, so the "no day > 24h" invariant
  holds. Required exclusions: all-day events, multi-day events, `declined` responses; cap any single
  event (e.g. ≤ 8h). Recorded in `state.sources.calendar_model = "bounded"`.

## Landmines (each of these was a real bug — keep them fixed)
1. **Tailwind = v3 Play CDN, pinned `cdn.tailwindcss.com/3.4.17`.** The Play CDN serves **v3**, where
   the global `tailwind.config` wires tokens. Do NOT switch to `@tailwindcss/browser` v4 — v4 ignores
   `tailwind.config` (CSS-first `@theme`) and `bg-card`/`border-border` silently generate nothing.
2. **Color tokens need the `<alpha-value>` placeholder:** `hsl(var(--border) / <alpha-value>)`, not
   bare `hsl(var(--border))`. Without it, opacity utilities (`border-border/50`, `hover:bg-muted/50`)
   are silent no-ops. Tokens are HSL triplets (NOT the oklch values on ui.shadcn.com — they'd be
   double-wrapped in `hsl()`).
3. **The shadcn base layer is mandatory** — `<style type="text/tailwindcss">` with
   `@layer base{ *{@apply border-border} body{@apply bg-background text-foreground} }`. Without it,
   borders fall back to Tailwind gray-200 (verify a `.bg-card` border == `rgb(228,228,231)`).
4. **Recharts UMD externalizes `prop-types` AND `react-is`** — load both before recharts or
   `window.Recharts` is `undefined` and the whole app fails to mount. Load order: react → react-dom →
   prop-types → react-is → recharts → @babel/standalone.
5. **Use jsDelivr, not unpkg, for the CDN scripts.** unpkg 404s on `recharts@2.15.4/umd/Recharts.min.js`
   and its version redirects + a `crossorigin` attr cause CORS failures from `file://`. No `crossorigin`.
6. **Dark mode is class-based with a toggle.** Drive it by `dark` on `<html>`; never put
   `@media (prefers-color-scheme:dark)` on the tokens (renders dark unexpectedly = looks broken). An
   early inline `<script>` sets the initial class (stored choice → else system) to avoid FOUC.
7. **Chart must be the shadcn Chart pattern**, not a bare Recharts chart: gradient `AreaChart`
   (`type="natural"`), token-colored grid/axes, and a `ChartTooltipContent`-style card. A plain
   `<BarChart>` with the default tooltip is the "not true shadcn" tell. (See the area example at
   ui.shadcn.com/charts/area.)
8. **Sessions are PER-MESSAGE timestamp points, clipped to the day — NOT a `min→max` span.** This is
   the subtle one: a session left open overnight / resumed across days, modelled as one solid span,
   produces impossible days (we saw 45.1h, and a 24h day with 0 commits/0 sessions). Every event
   (commit, session message, reflog/Linear stamp) is a point: bucket by local day, pad ±10 min, clip
   to `[day0, day1]`, merge with the 90-min gap. **No day may exceed 24h** — that invariant is the
   regression check.
9. **git: `--all` is mandatory** (work isn't on the checked-out branch), **filter by author email**,
   **dedup by full hash** across clones (separate object DBs hold the same commits).
10. **Linear is workspace-wide, so it's optional + auto-detected.** Derive the project's ticket
    prefixes from `[A-Z]{2,}-\d+` in commits/branches: none → skip Linear entirely; some → query but
    keep only matching-prefix issues. `--no-linear`/`--linear` override. Every source except git is
    optional; outputs list only sources that contributed.
11. **The persisted config + enabled-source set is part of the parity contract.** `state.sources.enabled`
    (and `from`/`author`/`tz`/`repos`) is reused verbatim on UPDATE — that's what keeps FULL == UPDATE.
    Changing it (via `--reconfigure`, `--full`, or an earlier `--from`) MUST force a FULL rebuild;
    otherwise historical days would have been computed under a different config than the trailing edge
    and parity breaks. Back-compat: a `schema:1` state file (no `sources`) loads by inferring `enabled`
    from non-zero columns and treating the checkpoint as already done — it must never re-prompt or
    spuriously force FULL.
12. **Chart tick interval / dot visibility must scale with the filtered length, not be hardcoded.**
    Found while adding the period toggle: the original `interval={2}` was tuned for the ~150-day
    `isleofculture` range. At 7 or 14 points that skips most day labels (2 kept out of every 3), making
    a short period unreadable — the opposite of what the toggle is for. Fixed by deriving
    `interval=Math.max(0,Math.ceil(data.length/15)-1)` and only showing per-point dots when
    `data.length<=31`. Any future change to the chart must re-check this against both a short (7d) and
    a long (150+ day) dataset, not just one.

## Why incremental is correct
Every signal is bucketed by its **own event timestamp**, so a day, once computed, never changes —
only the trailing edge can (and the last day of a prior run may be partial). So UPDATE recomputes
`[last covered day .. now]`, replaces those day-records, reuses everything earlier. An UPDATE result
must equal a `--full` rebuild of the same range (the parity check) — **given a fixed config**. The
config (range start + enabled-source set) lives in state and is reused verbatim; the only ways to
change it (`--reconfigure`/`--full`/`--from`) force FULL, so parity is never silently violated
(landmine #11).

## How this was verified (do the same after changes)
- **Render-verify in a real browser** (gstack `/browse`, `goto file://…`): no console errors,
  `typeof window.Recharts==='object'`, body bg flips light/dark via the toggle, `.bg-card` border ==
  token, area curve + `#fillHours` gradient present, one table row per day.
- **Incremental parity**: full(A→C) must equal full(A→B)+update(B→C), byte-identical day records.
  (Verified: 0 mismatches; totals 261 commits / 64-62 PRs / 94 sessions / 187 Linear / ~120h.)
- **Multi-project**: ran live against `isleofculture` (non-Linear) — Linear auto-skipped, dashboard
  clean; this is where landmine #8 was found and fixed.
- **Pre-flight checkpoint**: first run (no state) prompts once and persists `state.sources` with
  `checkpoint_done:true`; a second run does NOT prompt and reuses the persisted config; `--reconfigure`
  re-prompts and a changed range/source-set flips the run to FULL; `--yes`/headless never prompts and
  output is identical to the pre-checkpoint skill for the same config.
- **Back-compat**: an old `schema:1` state file updates silently (no prompt, no spurious FULL) and
  migrates to `schema:2` on write.
- **Detection + nudge**: with extra MCPs connected, nudge-tier servers (Calendar/Notion) appear in the
  `Detected:` plan line, the checkpoint, and the artifact footnote; deny-list servers
  (Spotify/Canva/Vercel/Supabase/…) are NOT mentioned; `--list-sources` prints the classification and
  exits without generating.
- **Period toggle**: clicking 7d/14d/30d/All changes KPIs, chart, weekly bars and the day table
  together (checked against `isleofculture`'s live dashboard); All reproduces the pre-toggle totals
  exactly; 7d shows per-day dots and one x-axis label per day (not the old fixed `interval={2}`, which
  would've skipped most of them — landmine #12); `localStorage.period` persists the choice.

## Verified evidence-source locations (this machine)
- Claude sessions: `~/.claude/projects/<flattened-cwd>/<uuid>.jsonl`; per-line top-level `timestamp`
  (UTC) + `cwd`.
- Codex sessions: `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl`; first line `session_meta` has
  `payload.cwd`/`payload.git`; per-line `timestamp`.
- GitHub: `gh` (jsonfields incl. `createdAt`,`closedAt`,`state`,`repository`). Scope to the project's
  repo; use `--created` + `--merged-at` for the incremental window.
- Linear: MCP `mcp__linear__list_users` / `list_issues` (assignee=me, `updatedAt` window).
- Reference machine repos: the original test project = two clones of an internal app repo under
  `~/Downloads/` (+ a personal fork) with worktrees in `~/conductor/...` and `.claude/worktrees/`.

## Repo / install status
- This skill lives in `~/Desktop/skill development/` alongside `dashkit`, `squeaky-bum-time`,
  `thrill-me` (the user's skill-dev folder; each is its own git repo). It is the source of truth.
- Invocable as `/timesplitter` (frontmatter `name:` in `SKILL.md`) — deliberately not `/time-splitters`,
  to match the repo's dev-folder name but not the invocation command. Symlinked into
  `~/.claude/skills/timesplitter` → `~/Desktop/skill development/time-splitters` (the dev folder itself
  keeps its original name; only the symlink's name is the trigger).
- No git remote yet (sibling `dashkit` is pushed to `github.com/jayintheday/dashkit`). The runtime
  test harness lived in the session scratchpad (ephemeral) — not committed here by design.

## Known limitations / possible next steps
- The HTML needs network at first open (CDN). The offline-capable option is the "real shadcn project"
  path (declined so far).
- Linear per-day count is a lifecycle-stamp proxy (created/started/completed/canceled), not full
  issue history — under-counts very busy Linear days.
- The very first hand-made export (that this skill was reverse-engineered from) predates landmine
  #8's fix, so its hours are slightly inflated vs the current model; regenerate if you want it on
  identical logic to a fresh run.
