# time-splitters

A Claude Code skill that **forensically reconstructs a personal work log** from local evidence —
git commits + reflog across all your clones/worktrees, Claude Code + Codex session logs, GitHub
PRs, and Linear — and turns it into one evolving timesheet: a markdown worklog, a CSV, and an
interactive shadcn/ui-styled HTML dashboard.

## What it produces

- **`.md`** — a human-readable worklog: summary, weekly totals, and a day-by-day breakdown with
  commit subjects, session counts, and tickets.
- **`.csv`** — one row per day, 18 columns.
- **`.html`** — a self-contained shadcn/ui dashboard (React + Tailwind + Recharts via CDN): KPI
  cards, a gradient hours-per-day area chart with a **7d/14d/30d/All period toggle**, weekly
  progress bars, a sortable day table, and a light/dark theme toggle.

Every source except git is optional and auto-detected — a project with no GitHub PRs, no
coding-agent sessions, or no Linear tickets just gets a leaner (but still complete) timesheet.

## How it works

On first run, it discovers every clone/worktree of the current project (matched by shared git
remote), resolves your date range from your earliest commit, and gathers evidence from git,
Claude/Codex session logs, GitHub PRs, and — if the project uses ticket references — Linear. Every
event (a commit, a session message, a reflog action) is treated as a timestamped point, padded and
merged into work blocks per day; no day can ever exceed 24 active hours.

Every later run **catches up to now**: it re-fetches only the trailing edge since the last run and
merges it into the existing dataset, so a daily top-up takes seconds instead of minutes. `--full`
forces a clean rebuild.

Read [`SKILL.md`](SKILL.md) for the full runtime spec, and [`CLAUDE.md`](CLAUDE.md) for the design
decisions and landmines behind it — useful if you're extending the skill, not needed to just run it.

## Install

**Claude Code** — clone into your skills directory. Note: this skill is invoked as
`/timesplitter`, not `/time-splitters`, so the local folder name should match the invocation:

```bash
git clone https://github.com/jayintheday/time-splitters ~/.claude/skills/timesplitter
```

Then invoke it with `/timesplitter`, or just ask Claude to "reconstruct my hours" or "build my
timesheet."

## Requirements

- A local git repo to point it at — git is the only *required* evidence source.
- [`gh`](https://cli.github.com) (GitHub CLI), authenticated, if you want PR data.
- The `linear` MCP connector, if your project uses Linear (auto-detected, auto-skipped otherwise).
- Internet on first open of the HTML dashboard — it pulls Tailwind/React/Recharts from CDN.

## Repo contents

| File | Purpose |
|---|---|
| `SKILL.md` | The skill. Self-sufficient runtime spec — the full gather → build → render procedure. |
| `CLAUDE.md` | Maintenance guide: design decisions, landmines, and how to verify changes. Read it before editing `SKILL.md`. |

## Contributing / developing

Edit `SKILL.md` for behavior changes, and update `CLAUDE.md` alongside it — it's the record of
*why* things are built the way they are, and the bugs already fixed once so they don't come back.
Then commit and push. Issues and PRs welcome.

## License

[MIT](LICENSE) © 2026 Vijay Patel
