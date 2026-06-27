# CLAUDE.md — notes for agents working on `time-splitters`

This is the build/maintenance guide. `SKILL.md` is the runtime spec (how to run the skill); this file
is the **why**, the design decisions, and the **landmines that already cost debugging time** — read it
before editing `SKILL.md` so you don't reintroduce a fixed bug.

## What the skill does
Reconstructs a personal work log from local forensic evidence (git commits+reflog across all
clones/worktrees, Claude + Codex session logs, GitHub PRs, Linear) and emits, in this order:
`.md` (source-of-truth worklog) → `.csv` (18 cols) → `.html` (shadcn/ui dashboard) → hidden
`.state.json`. It was reverse-engineered from a hand-made output the user liked
(`~/Desktop/musta-timesheets/musta-worklog-2026-05-15_to_2026-06-26.*` — the reference artifacts).

## Decisions made WITH the user (don't silently reverse these)
- **Pure instructions, no bundled helper script.** The skill tells the model to write a throwaway
  Python script into the scratchpad at runtime. Rationale: transparency + fresh each run. (So there is
  intentionally no `.py` in this repo.)
- **Reusable + smart defaults**, anchored on the **current project** (cwd repo); sibling clones found
  by shared remote. Not hard-coded to the original "musta"/qorum project.
- **One evolving timesheet**: stable filenames (`<name>-worklog.{md,csv,html}`) updated in place, with
  a hidden `.<name>-worklog.state.json` as durable state. Range lives *inside* the docs.
- **Catch up to now on every run** (incremental): re-fetch only the trailing edge; `--full` rebuilds.
- **shadcn/ui dashboard as a single self-contained CDN file** (not a real shadcn project). Needs
  internet on first open. The "real shadcn project" path was explicitly declined.
- **Light/dark toggle**, default = system preference, choice persisted.

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

## Why incremental is correct
Every signal is bucketed by its **own event timestamp**, so a day, once computed, never changes —
only the trailing edge can (and the last day of a prior run may be partial). So UPDATE recomputes
`[last covered day .. now]`, replaces those day-records, reuses everything earlier. An UPDATE result
must equal a `--full` rebuild of the same range (the parity check).

## How this was verified (do the same after changes)
- **Render-verify in a real browser** (gstack `/browse`, `goto file://…`): no console errors,
  `typeof window.Recharts==='object'`, body bg flips light/dark via the toggle, `.bg-card` border ==
  token, area curve + `#fillHours` gradient present, one table row per day.
- **Incremental parity**: full(A→C) must equal full(A→B)+update(B→C), byte-identical day records.
  (Verified: 0 mismatches; totals 261 commits / 64-62 PRs / 94 sessions / 187 Linear / ~120h.)
- **Multi-project**: ran live against `isleofculture` (non-Linear) — Linear auto-skipped, dashboard
  clean; this is where landmine #8 was found and fixed.

## Verified evidence-source locations (this machine)
- Claude sessions: `~/.claude/projects/<flattened-cwd>/<uuid>.jsonl`; per-line top-level `timestamp`
  (UTC) + `cwd`.
- Codex sessions: `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl`; first line `session_meta` has
  `payload.cwd`/`payload.git`; per-line `timestamp`.
- GitHub: `gh` (jsonfields incl. `createdAt`,`closedAt`,`state`,`repository`). Scope to the project's
  repo; use `--created` + `--merged-at` for the incremental window.
- Linear: MCP `mcp__linear__list_users` / `list_issues` (assignee=me, `updatedAt` window).
- Reference machine repos: musta = two clones of `…/qorum-meeting-flow` under `~/Downloads/` (+ a fork
  `…/qorum-local`) with worktrees in `~/conductor/...` and `.claude/worktrees/`.

## Repo / install status
- This skill lives in `~/Desktop/skill development/` alongside `dashkit`, `squeaky-bum-time`,
  `thrill-me` (the user's skill-dev folder; each is its own git repo). It is the source of truth.
- It is **not** symlinked into `~/.claude/skills/`, so `/time-splitters` won't autocomplete until it
  is. To enable: `ln -s "$HOME/Desktop/skill development/time-splitters" "$HOME/.claude/skills/time-splitters"`.
- No git remote yet (sibling `dashkit` is pushed to `github.com/jayintheday/dashkit`). The runtime
  test harness lived in the session scratchpad (ephemeral) — not committed here by design.

## Known limitations / possible next steps
- The HTML needs network at first open (CDN). The offline-capable option is the "real shadcn project"
  path (declined so far).
- Linear per-day count is a lifecycle-stamp proxy (created/started/completed/canceled), not full
  issue history — under-counts very busy Linear days.
- The original `~/Desktop/musta-timesheets-shadcn/` export predates landmine #8's fix, so its hours
  are slightly inflated vs the current model; regenerate if you want both projects on identical logic.
