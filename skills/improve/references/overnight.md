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
