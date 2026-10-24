# improve

> Fork of [shadcn/improve](https://github.com/shadcn/improve) adding `orchestrate` (parallel multi-plan runs in T3 Code), hand-made executor worktrees, `--overnight` unattended runs, `-a`/`-l`/`-e` flags to pick the advisor, lead and executor models, Haiku scouts (`-s`) for gathering and checks, and `-X-`/`-C-` file names for done and cancelled plans. Upstream: `git fetch upstream && git merge upstream/main`.

An agent skill that audits any codebase and writes implementation plans for other agents to execute.

The idea: use your most capable model for the part where intelligence compounds — understanding the codebase, judging what's worth doing, writing the spec — and hand execution to cheaper models. The skill never implements anything itself. The plan is the product.

```
you          →  /improve                    (expensive model, advises)
plans/       →  001-fix-n-plus-one.md       (self-contained specs)
other agent  →  implements, tests, ships    (cheap model, executes)
```

## Install

```bash
npx skills add ielyas/improve
```

Works in any agent that supports [Agent Skills](https://agentskills.io) format. The plans it writes are plain markdown, so any agent (or human) can pick them up.

## Usage

```
/improve                        full audit → prioritized findings → plans
/improve quick                  cheap pass: hotspots, top findings only
/improve deep                   exhaustive: every package, every category
/improve security               focused audit (also: perf, tests, bugs, ...)
/improve branch                 audit only what the current branch changes
/improve next                   feature suggestions — where to take the project
/improve plan <description>     skip the audit, spec one thing
/improve review-plan <file>     critique and tighten an existing plan
/improve execute <plan>         dispatch a cheaper executor, review its work
/improve orchestrate <plans…>   run several plans in parallel under a Master thread
/improve reconcile              refresh the backlog: verify, unblock, retire
/improve ... --issues           also publish plans as GitHub issues
/improve ex 019 --overnight     run while you're away (short -on): ask first, wait for go, morning report
/improve ... -a opus -l sonnet -e sonnet -s haiku   pick advisor, lead, executor and scout models
```

Scouts (`-s`, default Haiku 5.5) are read-only helpers that do the gathering and mechanical checks: medium-breadth audit categories, evidence for `plan`, the cold read in `review-plan`, running done criteria in reviews, the `orchestrate` digest and `reconcile` drift checks. They return evidence labelled OBSERVED or GUESS; every decision stays with the advisor, Lead or Master. `-s off` turns them off.

## How to use

A typical first run, start to finish:

1. Open your agent in the repo and run `/improve` (or `/improve quick` to keep it cheap).
2. It maps the repo, audits it, and comes back with a findings table. Reply with the ones you want planned — "plan 1, 3 and 5".
3. Plans land in `plans/` — one file each, plus an index with the recommended order. Read them; they're meant to be reviewed.
4. Hand a plan to any agent ("implement plans/001-*.md"), or let the skill run it: `/improve execute 001`. It dispatches a cheaper model in an isolated worktree, reviews the diff against the plan, and reports back with a verdict. Merging stays up to you.
5. Next session, run `/improve reconcile` to clean up the backlog: verify what landed, refresh what drifted, unblock what got stuck.

Before a PR, `/improve branch` does the same thing scoped to just what your branch changes.

## Example

A run against [shadcn/ui](https://github.com/shadcn-ui/ui) came back with findings like:

```
| # | Finding                                        | Category  | Effort | Confidence |
|---|------------------------------------------------|-----------|--------|------------|
| 1 | shadow-config duplicated in search.ts/view.ts, | tech-debt | M      | HIGH       |
|   | copies already drifted (TODO at search.ts:31)  |           |        |            |
| 2 | O(n²) icon migration (migrate-icons.ts:168)    | perf      | S      | HIGH       |
```

…and rejected a few, with reasons recorded so they don't come back next run:

```
- [SEC-01] https_proxy env var "SSRF": by-design — standard proxy convention,
  every CLI honors it. Not a finding.
```

Picking #1 produced [this plan](./examples/001-extract-shadow-config-resolution.md) — current code excerpted, exact steps, the repo's own test/lint commands as verification gates, and STOP conditions for when reality doesn't match.

## How it works

**Recon.** Maps the repo: stack, conventions, and the exact build/test/lint commands — these become verification gates in every plan. It also ingests intent and design docs when present — ADRs (`docs/adr/`), PRDs, `CONTEXT.md`, `DESIGN.md`, `PRODUCT.md` — so decided tradeoffs aren't re-flagged as findings, direction suggestions stay grounded in stated product intent, and plans speak the repo's own vocabulary. Composes with any repo that already maintains these docs.

**Audit.** Fans out parallel subagents across nine categories: correctness, security, performance, test coverage, tech debt, dependencies & migrations, DX, docs, and direction (feature suggestions — every one must cite evidence from the repo itself, no generic idea-slop). Every finding carries `file:line` evidence, impact, effort, and confidence.

**Vet.** Subagents over-report, so the advisor re-reads every cited location itself before showing you anything — false positives get dropped, wrong attributions get corrected, rejections get recorded.

**Prioritize.** Findings land in a table ordered by leverage (impact ÷ effort, weighted by confidence). You pick what becomes plans.

**Plan.** One file per selected finding, written into `plans/` with an index, priority order, and dependency graph.

**File names.** Plans are `plans/NNN-short-slug.md`, numbered in execution order. The number never changes, so "plan 007" always finds it. The status shows in the name:

```
plans/
  README.md                  index: order, dependencies, status
  007-X-watchlist-redesign.md   done and merged (-X-)
  012-C-sleep-timer.md          cancelled by you (-C-)
  014-quick-open.md             todo, parked, superseded or rejected
```

The rename happens in the same commit as the status change, with every link to the old name updated. A plan's companion files (`NNN-review.md`) take the same letter. Any run that reads an existing `plans/` backfills missing `-X-` and `-C-` names first.

## What makes the plans executable

Plans are written for the weakest plausible executor — a model that has never seen the advisor session and may be much smaller. Three properties carry that:

- **Self-contained.** All context is inlined: exact file paths, current-state code excerpts, repo conventions with an exemplar file, verified commands. No "as discussed above."
- **Verification gates.** Every step ends with a command and its expected output. Done criteria are machine-checkable. The executor never has to judge whether it succeeded.
- **Hard boundaries.** Explicit out-of-scope lists, and STOP conditions — "if X, stop and report" — instead of letting a small model improvise when reality doesn't match the plan.

Each plan also stamps the git commit it was written against, so executors run a mechanical drift check before touching anything.

## Closing the loop

Plans aren't fire-and-forget:

- **`execute <plan>`** spawns a cheaper executor subagent in an isolated git worktree, hands it the plan, then reviews the result like a tech lead — re-runs every done criterion, checks scope compliance, reads the diff against intent. Verdict: approve (merging stays your call), send back for revision (max 2 rounds), or block and refine the plan.
- **`orchestrate <plans…>`** (short `orc`) runs several plans in parallel in T3 Code. Each plan gets its own thread, worktree and devices: a **Lead** (advisor model, `-l`) has an executor (Sonnet 5.5, `-e`) write each step and reviews it before the next. A **Master** thread launches the plans, reviews each finished one, and posts a progress digest every 2 minutes. You watch Master only, and can still steer any plan thread directly. Nobody asks you anything mid-run; defaults are logged for one confirm-or-change pass at the end. Merging still waits for you. Without the T3 tools it falls back to `execute` on each plan in turn.
- **`--overnight`** (short `-on`, on `execute` or `orchestrate`) is for when you leave. Before you say `go`, it estimates the run, asks every approval and open question one at a time, and checks power. After `go` it asks nothing: it keeps the Mac awake, stays off physical devices and live data, never merges, pushes or deploys, stops a blocked plan instead of working around it, and leaves a morning report.
- **`reconcile`** processes what happened since: verifies DONE plans still hold, renames done and cancelled plans that lack `-X-` or `-C-`, investigates BLOCKED ones and rewrites around the obstacle, refreshes drifted plans, retires findings that got fixed independently.
- **`--issues`** publishes plans as GitHub issues — same self-contained body, so any agent or human can pick them up where work already lives.

## Hard rules

- Never modifies source code itself. The only writes go to `plans/`; executors edit only in disposable worktrees, and merging is always yours.
- Never runs commands that mutate your working tree — read, search, and read-only analysis only.
- Never reproduces secret values. Locations and credential types only, rotation always recommended.
- Asked to implement? It declines and points at the plan (or offers `execute`).

## License

MIT © shadcn
