# Orchestrate — run several plans in parallel

`orchestrate <plans…>` (e.g. `orchestrate 102 104 107`) runs several plans at once. Each plan gets its own top-level thread, worktree and devices. A **Master** thread oversees the run, and it is the only thread the user needs to watch.

The founding rule still holds: **no advisor edits source code.** Only executors write code; every other role dispatches, reviews and reports.

```
invoking thread ──launches──▶ Master (advisor model, read-only)
                                 │ launches, reviews, digests every 2 min
                                 ▼
                 Plan thread 102 · Plan thread 104 · Plan thread 107
                 Lead (advisor model, reviews every step)
                   └─ Executor subagent (Sonnet 5.5, writes the code)
```

Read [closing-the-loop.md](closing-the-loop.md) first. The Lead's dispatch, review and verdict rules come from its `execute` section, applied once per plan step.

---

## Requirements

- **T3 Code orchestration tools:** `t3_thread_launch`, `t3_thread_send`, `t3_thread_read`, `schedule_task`, `delete_scheduled_task`. Harnesses can attach them lazily (the names may carry an `mcp__t3-code__` prefix), so call one directly before deciding they're missing. If they really are missing, say so and fall back to running `execute` on each plan, one after another, in this thread.
- The repo is a git repository and every named plan exists in `plans/`.

## Roles

| Role | Model | Where | Writes |
|---|---|---|---|
| Invoking thread | the user's | wherever `orchestrate` was typed | the run ledger only |
| **Master** | inherited (advisor model) | the invoking thread's checkout | ledger; trackers outside git (Notion, Linear) |
| **Lead** (one per plan thread) | inherited (advisor model) | the plan's own worktree | nothing in code; commits the executor's work, the plan index and the project trackers on its branch |
| **Executor** (subagent of a Lead) | `sonnet` (Sonnet 5.5), unless the user named another | same worktree | code, tests |

Leave `modelSelection` unset on both launches so Master and the Leads inherit the advisor model. Never run a Lead on the executor model.

---

## Phase 1 — Preflight (invoking thread)

1. **Parse the plan list.** Read each plan file and `plans/README.md`, including any run-order table.
   - If one named plan depends on another named plan, it waits: it launches only after its parent is APPROVED, with `baseRef` set to the parent's branch.
   - If a dependency outside the set isn't DONE, drop that plan from the run and say why.
   - If the index groups the named plans with other plans that can run alongside them, mention those plans in one line, then continue with exactly the set the user named.
2. **Drift check** every plan, as in closing-the-loop. Note drifted plans; their Lead refreshes them before dispatching.
3. **Look for other sessions.** Use `t3_worktree_list` and ListAgents, plus `git worktree list` and `git branch --list 'advisor/*'`.
   - If a plan already has a live worktree or branch from another session, skip it and say so.
   - Never remove or reuse another session's worktree.
4. **Load the project hooks.** Use `rules/parallel-runs.md`, or else a `## Parallel runs` section in `AGENTS.md`/`CLAUDE.md`. Hooks define per-plan setup (simulators, devices, ports, databases), isolated build commands, shared-resource locks, teardown, and tracker rules. If there are none, use the generic defaults: a separate build output directory per worktree and no shared devices.
5. **Create the run ledger** outside the working tree, so it dirties no checkout and every worktree can see it:

```
RUN_ID=orch-$(date +%Y%m%d-%H%M)
LEDGER=$(git rev-parse --path-format=absolute --git-common-dir)/orchestrate/$RUN_ID
mkdir -p "$LEDGER/locks" && touch "$LEDGER/events.log"
```

`state.json` in the ledger holds the plan list, base branch, and the IDs of every thread and scheduled task, plus `digestedLines` (lines of `events.log` already reported). Master owns it after launch.

6. **Launch Master** with `t3_thread_launch`:
   - `title`: `Master — <plans>`.
   - `workspaceStrategy`: `{type:"existing_worktree", worktreePath:<this checkout>, branch:<current>}`, or `{type:"root"}` when the invoking thread is in the main checkout.
   - `message`: the **Master brief** (below), with the run ID, ledger path, plan list, dependency order, hooks text and base branch filled in.

   Record Master's `threadId` in `state.json`. Then tell the user, in one line, to switch to the Master thread. The invoking thread's job ends here.

## Phase 2 — Launch (Master)

For each plan that is ready to start:

1. **Launch the plan thread** with `t3_thread_launch`:
   - `title`: `NNN — <plan title>`.
   - `workspaceStrategy`: `{type:"worktree", baseRef:<base or parent branch>, branch:"advisor/NNN-<slug>", startFromOrigin:false}`.
   - `message`: the **Lead brief** (below).

   Uncommitted plan files don't reach a new worktree, so the brief always inlines the full plan text.
2. Record its `threadId` and worktree path in `state.json`, and append a `LAUNCHED` event.
3. Follow the project's tracker rule: set the plan's task to In progress, and its index row to IN PROGRESS.

Then **create the digest automation** with `schedule_task`:
- `schedule`: `{type:"interval", everyMs:120000}`.
- `bindToCurrentThread`: `true` (it must post into Master).
- `title`: `<RUN_ID> digest`.
- `prompt`: the **Digest prompt** (below).

Store the returned `scheduledTaskId` in `state.json`. Report the cadence and next run time in one line.

## Phase 3 — Run

### Events

Every thread appends one line per milestone to `$LEDGER/events.log`:

```
printf '%s | %s | %s | %s\n' "$(date -u +%H:%M:%S)" NNN STATE "one-line detail" >> "$LEDGER/events.log"
```

States:
- `LAUNCHED`
- `STEP k/n STARTED`
- `STEP k/n APPROVED`
- `REVISE` (the Lead sent fixes)
- `TESTS GREEN` / `TESTS RED`
- `WAITING LOCK <name>`
- `OWNER CHANGE` (the user steered the thread)
- `BLOCKED`
- `READY FOR REVIEW`
- `APPROVED`
- `REJECTED`

Events are the progress feed. A Lead *also* messages Master directly, using `t3_thread_send` with `mode:"queue"` and `clientRequestId:<RUN_ID>-NNN-<state>-<k>`, but only for **READY FOR REVIEW**, **BLOCKED** and **OWNER CHANGE**, the events that need Master to act.

### Lead loop (per plan thread)

1. Refresh the plan if it drifted. Run the hooks' **setup** (for example, create this plan's simulators) and record what was created in an event, so teardown can find it.
2. For each plan step:
   - Dispatch the executor with the closing-the-loop prompt: the full plan inlined, the current step named, the worktree's absolute path, the hooks' isolated build and test commands, and the locks it must take.
   - Use the Agent tool with `model:"sonnet"` and **no** `isolation`, because the thread is already bound to the worktree. If the Agent tool can't take a model, use `delegate_task` with the Sonnet model from `orchestrator_capabilities`.
   - Review the step exactly as closing-the-loop describes, and reject prototype or stub code standing in for the real feature unless the plan's Type is Spike: re-run its verification, check scope, read the diff and the tests. Revise at most 2 rounds, then BLOCK.
   - On approval, commit in the worktree, append `STEP k/n APPROVED`, and **start the next step immediately**. If the step overlaps another plan's files, that is a rebase note for the final report, not a reason to wait.
3. **Never ask the user anything mid-run.** Decide from the plan, the repo's docs and the project's history. Log every judgment call as a default in `$LEDGER/NNN-defaults.md`, one line each.
4. **Steering:** if the user writes in the plan thread, treat it as authoritative.
   - Apply it through the executor, even when it widens scope.
   - Append `OWNER CHANGE` with a one-line summary and message Master.
   - Record it in the plan's NOTES, so Master's review doesn't flag it as out of scope.
5. **Done:** run the full done criteria and the full test suite using the hooks' commands.
   - Update the project's trackers on the branch, as its agent docs require (for example TRACKER.md and a user-facing CHANGELOG).
   - Set the plan's index row to IN REVIEW, and commit.
   - Append `READY FOR REVIEW` and message Master. Then wait. Master answers with APPROVED or with specific fixes.
6. **After Master approves:** set the plan's index row to DONE, commit, and append `APPROVED`. Leave the worktree, branch and devices in place for the user's merge.

### Master on READY FOR REVIEW

Master reviews independently, as a second tech lead:
- Re-run the done criteria in the plan's worktree (`git -C <path>` and the hooks' commands, with the same locks).
- Check scope against the plan, counting OWNER CHANGE items as in scope.
- Read the whole diff.

Then either:
- send the Lead `APPROVED`, and update the tracker outside git (for example, set the Notion task to ready for review); or
- send specific fixes (REVISE). After two Master-level rounds, mark the plan BLOCKED.

Once a plan is APPROVED, launch any plan that was waiting on it, with that plan's branch as `baseRef`.

### Digest prompt (scheduled every 2 minutes, posts into Master)

> Orchestrate digest for `<RUN_ID>`, ledger `<LEDGER>`.
>
> Read `state.json` and the lines of `events.log` after `digestedLines`. Also run `t3_thread_read` on each plan thread that is still running, using `view:"activity"` and `limit:5`.
>
> - **Nothing new, and no thread stalled:** reply with exactly `·`.
> - **Otherwise:** post one compact update with one line per plan: `NNN · step k/n · state · detail`. Mark blockers and owner changes, and list `TESTS RED` first. Flag a plan as **stalled** when its thread shows no activity for 15 minutes and it isn't waiting on a lock or on review; send its Lead a one-line nudge.
>
> Then set `digestedLines` to the line count of `events.log`. Don't review code, launch threads, or change any plan state here.

## Phase 4 — Wrap-up (Master)

When every plan is APPROVED, BLOCKED or REJECTED:

1. **Delete the digest automation** (`delete_scheduled_task`) and remove its ID from `state.json`.
2. **Re-check the defaults.** Read every `NNN-defaults.md` against the project's documented decisions, and flip anything that contradicts one, through that plan's Lead.
3. Run the hooks' **final verification**: the full suite per branch, plus any device installs the project wants. Device installs always go through the hooks' device lock, and the report names whose build is going onto which device.
4. **Report**, in the project's or user's preferred report format:
   - per plan: verdict, branch, worktree path, a one-line diff summary, and rebase notes;
   - one **confirm or change** list of every default;
   - anything blocked, with its reason and the plan fix.

## Phase 5 — Merge (only when the user says so)

Never merge, push or deploy on your own. When the user says `merge` in Master, follow the project's merge routine for each APPROVED plan, in dependency order. Typically that is: merge into the base branch, resolve conflicts, update the index and trackers, and mark tasks done.

Then **tear down** what this run created, and nothing else:
- the hooks' teardown (for example, delete the plan's simulators);
- `git worktree remove` and `git branch -d` for this run's branches only;
- the ledger directory.

---

## Master brief (template)

> You are **Master** for orchestrate run `<RUN_ID>`. You oversee the parallel execution of plans `<list>` in `<repo>`. Follow `references/orchestrate.md` of the improve skill, Phases 2–5. Read it now: `<absolute path>`.
>
> Ledger: `<LEDGER>`. Base branch: `<base>`. Dependency order: `<…>`.
> Project hooks: `<inlined text or "none: generic defaults">`.
>
> You never edit source code. You launch one plan thread per plan, create the 2-minute digest automation, review each READY FOR REVIEW, and write the final report. Never ask the user questions mid-run. The user watches only this thread: keep every message short.

## Lead brief (template)

> You are the **Lead** for plan `NNN` in orchestrate run `<RUN_ID>`. Master's thread ID is `<id>`.
>
> - Your worktree: `<path>`, on branch `advisor/NNN-<slug>`. Ledger: `<LEDGER>`.
> - Follow the Lead loop in `references/orchestrate.md` and the review rules in `references/closing-the-loop.md`. Read both now: `<absolute paths>`.
> - Project hooks: `<inlined>`.
>
> You never write code yourself. A Sonnet 5.5 executor subagent does, and you review every step. The executor builds production code that ships, on every platform in scope — not a prototype or a handoff — unless the plan's Type is Spike. Never ask the user anything mid-run; log defaults instead. If the user writes in this thread, treat it as an owner change.
>
> The plan:
>
> `<full plan text>`

## Failure handling

- **A launch fails, or its response is lost:** check `t3_thread_list` before retrying, because the thread may already exist.
- **A plan thread dies or keeps stalling:** Master reads its last activity, sends one message to resume it, and on a second failure marks the plan BLOCKED with the evidence.
- **The tools disappear mid-run:** Master finishes reviewing what's in flight and reports. It never relaunches plans as subagents in its own thread.
