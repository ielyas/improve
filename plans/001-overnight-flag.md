# Plan 001: Add an `--overnight` / `-on` flag to `execute` and `orchestrate` for runs while the owner is away

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. When done, update the status row for this plan
> in `plans/README.md` — unless a reviewer dispatched you and told you they
> maintain the index.
>
> **Drift check (run first)**: `git diff --stat 1b1d00c..HEAD -- skills/improve/SKILL.md skills/improve/references/orchestrate.md skills/improve/references/closing-the-loop.md README.md`
> If any in-scope file changed since this plan was written, compare the
> "Current state" excerpts against the live code before proceeding; on a
> mismatch, treat it as a STOP condition.

## Status

- **Priority**: P1
- **Effort**: S
- **Risk**: LOW
- **Depends on**: none
- **Category**: dx
- **Type**: Build
- **Planned at**: commit `1b1d00c`, 2026-10-19

## Why this matters

The owner often starts `execute` / `orchestrate` runs and then leaves: overnight, asleep, or out. The skill has no way to know that. In the last overnight run (Seedbox plan 180, 2026-10-18), the owner had to set everything up by hand over three rounds: questions, approvals, a "caffeinate" reminder, then "go". They sent "do NOT check on my physical devices" only after the run had started. The run also changed real data: a simulator check moved a real movie's shared playback position and approved a device-pairing code in a live service (r2tor). The estimate was 8–12 h and the run took about 1 h, and nothing told the owner it had finished. This plan adds one flag that turns all of that into standing rules: ask everything up front, wait for "go", keep the Mac awake, keep away from devices, live data and anything irreversible, finish cleanly, send a notification, and leave a morning report.

## Current state

This repo is a single agent skill, written entirely in markdown. There is no build or test suite, so verification is done with `grep` and `test`.

- `skills/improve/SKILL.md` — the skill entry point. It lists invocation variants (lines 117–129) and model flags (lines 131–151).
- `skills/improve/references/orchestrate.md` — the parallel-run procedure (Preflight, Launch, Run, Wrap-up, Merge, briefs).
- `skills/improve/references/closing-the-loop.md` — the `execute` dispatch, review and verdict rules.
- `README.md` — the public usage block (lines 25–39).
- Note: `~/.claude/skills/improve` is this repo's `skills/improve` directory. Edit the files in the repo.

Excerpts as they are today:

`skills/improve/SKILL.md:127` (the `orchestrate` bullet, abridged):
```
- `orchestrate <plans…>` (short: `orc`, ...) → run several plans in parallel. ... A **Master** thread oversees them all, reviews each finished plan, and posts a progress digest every 2 minutes. ... Nobody asks the user anything mid-run; defaults are logged for one confirm-or-change pass at the end. ... **Read [references/orchestrate.md](references/orchestrate.md) before starting.**
```

`skills/improve/SKILL.md:129` (the last bullet in "Invocation variants") starts with:
```
- `--issues` (modifier on any planning invocation) → also publish each written plan as a GitHub issue via `gh`, ...
```

`skills/improve/SKILL.md:135-140` (the model flags table):
```
| Flag | Role | Default |
|---|---|---|
| `-a <model>` / `--advisor` | ...
| `-l <model>` / `--lead` | ...
| `-e <model>` / `--executor` | ...
| `-s <model>` / `--scout` | ...
```

`skills/improve/references/orchestrate.md:87`, in Phase 2:
```
- `schedule`: `{type:"interval", everyMs:120000}`.
```

`skills/improve/references/orchestrate.md:182`:
```
### Digest prompt (scheduled every 2 minutes, posts into Master)
```

`skills/improve/references/orchestrate.md:201`, in Phase 4:
```
3. Run the hooks' **final verification**: the full suite per branch, plus any device installs the project wants. Device installs always go through the hooks' device lock, and the report names whose build is going onto which device.
```

`skills/improve/references/orchestrate.md:224` (the end of the Master brief) and `:237` (the Lead brief) both contain `Never ask the user` sentences. These are where the overnight line gets added.

`skills/improve/references/closing-the-loop.md:19`: `### Dispatch`, the section where `execute` creates its worktree and spawns the executor.

Conventions to match:
- Prose style: short, plain sentences. Use bold for rule names, put commands and tool names in backticks, and cite real incidents briefly in parentheses, the way `orchestrate.md:98` cites `(orch-20261008-2134)`.
- New reference material goes in its own file under `skills/improve/references/`. SKILL.md links to it with `**Read [references/x.md](references/x.md) before starting.**`, as the `execute` and `orchestrate` bullets already do.
- Commit style: `feat: …` / `feat(orchestrate): …` / `docs: …`, all lowercase, as in `git log --oneline` (for example `feat: -s scout flag, Haiku scouts for gathering and checks`).

## Owner decisions (2026-10-19)

- **Start**: with `-on`, the run asks every question up front, one at a time with a recommended answer, then waits for the owner to say `go`. Nothing runs before `go`. After `go`, nothing asks the owner anything.
- **Finishing early**: stop and report. Only the named plans run, and no new plan starts without the owner.
- **Hard no's**: these cannot be pre-approved overnight: physical devices, merge, push, deploy, remote migrations, `sudo`, and deleting anything the run did not create. The owner does these in the morning.

## Commands you will need

| Purpose | Command | Expected on success |
|---|---|---|
| New file exists | `test -f skills/improve/references/overnight.md && echo ok` | `ok` |
| Flag is documented | `grep -c -- "--overnight" skills/improve/SKILL.md README.md` | each file ≥ 1 |
| Links resolve | `grep -o "references/overnight.md" skills/improve/SKILL.md skills/improve/references/*.md \| wc -l` | ≥ 3 |
| Scope | `git status --short` | only the in-scope files |

## Scope

**In scope** (the only files you may modify):
- `skills/improve/references/overnight.md` (create)
- `skills/improve/SKILL.md`
- `skills/improve/references/orchestrate.md`
- `skills/improve/references/closing-the-loop.md`
- `README.md`
- `plans/README.md` (status row only)

**Out of scope**:
- `skills/improve/references/audit-playbook.md` and `plan-template.md`: overnight changes how plans run, not how they are written.
- Any project's `rules/parallel-runs.md` (for example Seedbox's). The flag reads an optional `## Overnight` section from it but does not create one.
- `examples/`.
- Any other skill under `~/.claude/skills/`.

## Git workflow

- Branch: `advisor/001-overnight-flag`.
- Commit per step. Messages: `feat: --overnight / -on flag for runs while the owner is away` (step 1–2), `feat(orchestrate): overnight preflight, digest and wrap-up` (step 3), `feat: overnight in execute` (step 4), `docs: README lists --overnight` (step 5).
- Do NOT push.

## Steps

### Step 1: Create `skills/improve/references/overnight.md`

Create the file with exactly this content (keep the wording; you may fix typos only):

````markdown
# Overnight — runs while the owner is away

`--overnight` (short `-on`) tells `execute` and `orchestrate` that the owner is leaving: overnight, asleep or out. It takes no value. Any other variant ignores it and says so in one line.

The run has two parts. Before `go`, ask everything. After `go`, ask nothing. Everything else in `execute` and `orchestrate` stays the same; these rules add to them and take precedence where they conflict.

---

## 1. Preflight — before the owner leaves

Do all of this before any executor, Lead or Master starts:

1. **Estimate the run.** Give a range per phase. Base it on measured times when they exist, such as earlier ledgers, plan NOTES, and build times recorded in the plan. Plan 180 estimated 8–12 h for VLCKit rebuilds that took 34 min. Say which numbers are measured and which are guesses.
2. **List the approval points.** Read every plan in the run for steps that need the owner: replacing or swapping a shared resource (a framework, a database, a config other checkouts use), destructive steps, and anything the plan marks as needing approval. Ask about each one, **one at a time**, with a recommended answer. If the owner says yes, that step can run unattended. If they say no or don't answer, the run stops that plan just before the step and leaves its output built but not applied.
3. **Hard no's.** These can't be pre-approved and never happen overnight: installs or checks on physical devices, merge, push, deploy, remote migrations, `sudo`, and deleting anything this run didn't create. Steps that need them become owner checks in the morning report.
4. **Ask any open product questions now**, one at a time, each with a recommended answer. After `go` there are no more questions.
5. **Check power.** Run `pmset -g batt | head -1`. If the Mac isn't on AC power, tell the owner to plug it in. If it's a laptop, also tell them to leave the lid open: closing it makes the Mac sleep even with `caffeinate`.
6. **One summary message**: the estimate, the approvals given, the hard no's, and the owner checks waiting for the morning. It ends with: `Say go when you leave.`
7. **Wait for `go`.** Nothing runs before it.

## 2. On `go`

- **Keep the Mac awake** for the whole run, not just the builds:
  ```
  caffeinate -ims -t <seconds> &   # display may sleep; seconds from the rule below
  echo $! > <ledger or plans dir>/caffeinate.pid
  ```
  Record the PID in `state.json` (`orchestrate`) or the plan index notes (`execute`). Say in one line that it's running and for how long. Set `-t` to the larger of 2 × the high estimate and 8 h, plus 5 h for one usage-limit wait (§3). Example: high estimate 3 h → max(6 h, 8 h) + 5 h = 13 h = `-t 46800`.
- **Start the heartbeat** so the run survives a usage limit or a dropped connection. A model that's out of usage can't wake itself up, so the app's scheduler has to.
  - `orchestrate`: the 30-minute digest is the heartbeat. Add the resume check below to its prompt.
  - `execute`: create one with `schedule_task` (`{type:"interval", everyMs:1800000}`, `bindToCurrentThread:true`, title `<plan> overnight heartbeat`). Its prompt is the resume check. Store the `scheduledTaskId` in the plan index notes.
- Record `overnight: true` and the approvals list in `state.json` (`orchestrate`). For `execute`, put them in the plan index notes.

## 3. During the run

- **Never ask the owner anything.** Decide from the plan, the repo's docs and the project's history. Log each judgment call as a default, one line each, in `$LEDGER/NNN-defaults.md` (`orchestrate`) or the executor report's NOTES (`execute`).
- **Physical devices are off limits.** All checks run on simulators. Device steps in a plan or in the hooks become owner checks.
- **Leave live data alone.** Simulators and test runs must not change real state the owner relies on. That covers shared playback positions or progress, approving pairing or sign-in codes in a live service, purchases, messages, and anything that writes to a production account. Use a test account or profile when the project's hooks define one (an `## Overnight` section in `rules/parallel-runs.md`). Without one, skip that check and log it as an owner check. (Plan 180 moved a real film's playback position and approved a pairing code in live r2tor.)
- **Blocked means stop, not work around.** When a plan hits a STOP condition, a failed gate, exhausted revisions, or an unapproved approval point:
  - stop that plan at its last approved commit;
  - write down why in the index (BLOCKED, or `WAITING OWNER` for an unapproved approval point);
  - keep running the other plans that don't depend on it.
  
  Never skip a gate, weaken a check or swap in a different approach to keep going.
- **Finishing early is fine.** When the named plans are done, wrap up. Don't start plans the owner didn't name.
- **Digest every 30 minutes, not 2** (`orchestrate`). Nobody is watching, so set `everyMs:1800000`. Master still handles READY FOR REVIEW, BLOCKED and OWNER CHANGE messages as soon as they arrive.
- **Usage limits and dropped connections pause the run; they don't fail it.** On each heartbeat tick, run the **resume check**:
  1. Find the run's threads whose latest run is `failed` or `interrupted`. That's this thread for `execute`, and every plan thread plus Master for `orchestrate`. Use `t3_thread_list` / `t3_thread_read` (`view:"activity"`, last few items).
  2. Read the error text. It's a **usage limit** if it mentions a usage or rate limit or a reset time. It's a **connection drop** if it shows a network, TLS or overload error.
  3. **Usage limit with a reset time that hasn't passed yet:** append `PAUSED usage limit, resets <time>` to the events log once, and do nothing else this tick.
  4. **Usage limit whose reset time has passed, a limit with no time given, or a connection drop:** send that thread one message with `t3_thread_send` (`clientRequestId:<RUN_ID>-resume-<thread>-<n>`): `Resuming overnight run after <usage limit|connection drop>. Re-read the plan and the ledger, check git status in your worktree, and continue from the last approved step.` Then append `RESUMED`.
  5. A thread that fails again within 10 minutes of being resumed, for a reason that isn't a limit or a connection drop, is a real failure. Handle it with the normal stall and BLOCKED rules.
  
  Limit waits and connection drops never count toward a plan's revision rounds or Master's two-failure BLOCK rule. Ticks that fire while the account is still limited will fail too; that's expected. The first tick after the reset resumes the run. Record every pause, with its start and end times, for the morning report.
- **The owner may come back early.** A message from the owner mid-run is an OWNER CHANGE, as usual. If they say they're back, drop the overnight rules for the rest of the run and say so in one line.

## 4. Wrap-up

When every plan is APPROVED, BLOCKED, WAITING OWNER or REJECTED:

1. **Stop `caffeinate`**: `kill $(cat <…>/caffeinate.pid)`. Confirm it's gone with `pgrep caffeinate`. For `execute`, also delete the heartbeat with `delete_scheduled_task`.
2. **Shut down** every simulator this run booted. Leave worktrees, branches and simulators in place for the merge, as usual.
3. **Skip device installs**, including those listed in the hooks' final verification. List them as owner checks.
4. **Notify the owner once.** Use the `PushNotification` tool if the host has it (load it with ToolSearch `select:PushNotification`). Keep it to one line: `<run> done: N approved, M blocked`. If the tool isn't there, say so in the report.
5. **Morning report**, in the owner's report format (Did / Need from you / Next), plus:
   - the actual time each phase took, next to the estimate, and every usage-limit or connection pause with its start and end times;
   - **Changed outside the branch**: every change to shared or real state (a swapped framework, a regenerated checksum, a simulator account), each with its undo command. Write `none` if there were none;
   - the defaults to confirm or change, as one list;
   - owner checks: device listens and installs, skipped live-data checks, each as one short step;
   - the **first thing to do**, as one line (for example `say "launch on iphone", then listen with Boost Dialogue on and off`).
````

**Verify**: `test -f skills/improve/references/overnight.md && grep -c "^## " skills/improve/references/overnight.md` → `4`.

### Step 2: Wire the flag into `SKILL.md`

1. After the `--issues` bullet (line 129) in "Invocation variants", add this bullet:

```
- `--overnight` (short `-on`; modifier on `execute` and `orchestrate`) → the owner is leaving. Before `go`: estimate, ask every approval and open question one at a time, check power, then wait for `go`. After `go`: keep the Mac awake with `caffeinate -ims`, ask nothing, stay off physical devices and live data, never merge, push or deploy, stop blocked plans instead of working around them, digest every 30 minutes, notify once at the end and leave a morning report. Ignored by other variants. **Read [references/overnight.md](references/overnight.md) before preflight.**
```

2. Under the model flags table (after line 140), add one sentence so `-on` isn't parsed as a model flag:

```
`-on` / `--overnight` is not a model flag: it takes no value, and the token after it is never a model. See "Invocation variants".
```

**Verify**: `grep -n -- "--overnight" skills/improve/SKILL.md` → 2 lines, one in "Invocation variants" and one under the flags table.

### Step 3: Hook it into `orchestrate.md`

Make four small additions. Don't rewrite existing text.

1. In Phase 1 — Preflight, add this step between step 5 (the run ledger, so the ledger exists for the PID) and step 6 (Launch Master):

```
5b. **Overnight (`-on`).** Run the preflight in [overnight.md](overnight.md) §1 here, in the invoking thread, and wait for `go` before launching Master. Then start `caffeinate` (§2) and put `overnight: true`, the approvals and the caffeinate PID in `state.json`.
```

2. At line 87 (`everyMs:120000`), add to that bullet: ` With `-on`, use `everyMs:1800000` (30 minutes), and append the resume check from [overnight.md](overnight.md) §3 to the digest prompt.`
2b. In "Failure handling" (near line 243), add a bullet: `- **A usage limit or a dropped connection** is a pause, not a failure. Don't count it toward the two-failure BLOCK rule. With `-on`, the digest's resume check restarts the thread after the reset; see [overnight.md](overnight.md) §3.`

3. At Phase 4 step 3 (line 201), add at the end of the step: ` With `-on`, skip device installs and list them as owner checks; then follow [overnight.md](overnight.md) §4.`

4. In the Master brief (ends near line 224) and the Lead brief (near line 237), add this sentence after each brief's `Never ask the user …` sentence:

```
Overnight run: `<yes|no>`. If yes, follow `references/overnight.md` §3 (path: `<absolute path>`): no physical devices, leave live data alone, stop blocked work instead of working around it.
```

**Verify**: `grep -c "overnight" skills/improve/references/orchestrate.md` → ≥ 5.

### Step 4: Hook it into `closing-the-loop.md` (`execute`)

At the start of the `### Dispatch` section (line 19), add this paragraph:

```
**Overnight (`-on`).** Before creating the worktree, run the preflight in [overnight.md](overnight.md) §1 and wait for `go`; then §2, which includes creating the heartbeat that resumes the run after a usage limit. Add §3's rules to the executor prompt, after the preamble: no physical devices, leave live data alone, stop at a STOP condition rather than work around it. A BLOCK verdict overnight is reported in the morning, not asked about. Finish with §4.
```

**Verify**: `grep -n "overnight.md" skills/improve/references/closing-the-loop.md` → 1 line, inside the Dispatch section.

### Step 5: README usage line and version

1. In `README.md`'s usage block, after the `--issues` line (line 37), add:
```
/improve ex 019 --overnight     run while you're away (short -on): ask first, wait for go, morning report
```
2. In `skills/improve/SKILL.md` frontmatter, change `version: "1.1.0"` to `version: "1.2.0"`.

**Verify**: `grep -n -- "-on " README.md` → 1 line; `grep -n 'version: "1.2.0"' skills/improve/SKILL.md` → 1 line.

## Test plan

The repo has no test suite. Instead, do a cold read: give `skills/improve/references/overnight.md` and the SKILL.md bullet to a fresh agent with no other context. Ask it to list, for `orchestrate 101 102 -on`, every action in order up to the morning report. It passes if the list includes: the questions asked one at a time, the wait for `go`, starting `caffeinate`, the 30-minute digest, no device installs, stopping `caffeinate`, the push notification, and the "Changed outside the branch" section.

## Done criteria

- [ ] `test -f skills/improve/references/overnight.md` exits 0, and the file has 4 `## ` sections
- [ ] `grep -c -- "--overnight" skills/improve/SKILL.md` ≥ 2
- [ ] `grep -c "overnight" skills/improve/references/orchestrate.md` ≥ 5
- [ ] `grep -c "overnight.md" skills/improve/references/closing-the-loop.md` = 1
- [ ] `grep -c "1800000" skills/improve/references/orchestrate.md skills/improve/references/overnight.md` → ≥ 1 each
- [ ] `grep -n 'version: "1.2.0"' skills/improve/SKILL.md` matches
- [ ] `grep -c "resume check" skills/improve/references/overnight.md` ≥ 2, and `grep -c "usage limit" skills/improve/references/orchestrate.md` ≥ 1
- [ ] Cold read (test plan) lists every required action
- [ ] `git status --short` shows only in-scope files
- [ ] `plans/README.md` row for 001 updated

## STOP conditions

- An excerpt in "Current state" doesn't match the file, for example if the digest interval is no longer `everyMs:120000`.
- `-on` already means something else anywhere in the skill (`grep -rn -- " -on\b" skills/`).
- A change appears to need an edit to `audit-playbook.md`, `plan-template.md` or a project's `rules/` file.

## Maintenance notes

- If `orchestrate` gains new owner-facing steps (device installs, merges, deploys), check that each one respects the overnight hard no's in `overnight.md` §1.3.
- If the digest interval or wrap-up order changes in `orchestrate.md`, update `overnight.md` to match. The two files reference each other by section number (§1–§4), so renumbering breaks the links.
- Projects can define a test account or profile for overnight simulator checks in an `## Overnight` section of `rules/parallel-runs.md`. Seedbox should get one, as a separate change in that repo, so overnight checks there stop skipping playback-position and pairing tests.
- Not yet observed: how T3 shows a usage-limit failure in a thread's activity (the exact error text and whether it includes the reset time). The resume check matches loosely on purpose. After the first real overnight limit, put the exact text into §3.
- Possible later: run Master and the heartbeat on another provider (for example Codex) so they keep running while Claude is out of usage, and can resume the Claude threads right at the reset time.
- Deferred: letting the owner pre-approve merge or deploy overnight. The owner decided these stay hard no's (2026-10-19).
