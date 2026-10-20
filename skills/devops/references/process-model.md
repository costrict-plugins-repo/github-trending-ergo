# Process Model

The operational reference for interpreting a `processes` listing, the `ergo://<node>/process/<pid>`
resource behind one of its rows, and the metrics inside both. Covers states, mailbox queues, metric
fields, the liveness formula, and how to read backpressure from telemetry.

Three answers are easy to confuse. `processes` is the listing, one row per process.
`ergo://<node>/process/<id>.<creation>` is the full record of one process, as the framework sees it.
`process_state` is the process describing **itself** - the sections a behavior chooses to expose
(`ergo:restarts_count` on a supervisor, `ergo:messages_unhandled` on a pool) - and carries none of
the counters below.

## Process States

Returned as strings by `process_state` and inside `ProcessShortInfo.State`.

| State | Constant | Meaning | Allowed ops |
|-------|----------|---------|-------------|
| `init` | `ProcessStateInit` | Running `ProcessInit`. Not yet registered, no PID lookup. | Spawn, Send async, setters. No Call, Link, Monitor, RegisterName. |
| `sleep` | `ProcessStateSleep` | Idle, mailbox empty. Goroutine is parked. | All operations. |
| `running` | `ProcessStateRunning` | Dispatching a message. | All operations. |
| `"wait response"` | `ProcessStateWaitResponse` | Blocked in `Call()` awaiting reply. | Limited. Abnormal if > 5s (default Call timeout). |
| `terminated` | `ProcessStateTerminated` | Shutting down. | Send-only. |
| `zombee` | `ProcessStateZombee` | Killed forcibly. Goroutine orphaned. | Nothing. |

Zombies persist in the process table - they are a leak indicator. Investigate with
`goroutines node=<node> stack=<behavior>` to find the stuck goroutine.

## Mailbox Queues (Priority)

Every process has four mailbox queues dispatched in priority order:

| Priority | Constant | Queue | Typical use |
|----------|----------|-------|-------------|
| Max | `MessagePriorityMax` | Urgent | Kill, admin controls |
| High | `MessagePriorityHigh` | System | Framework messages |
| Normal | `MessagePriorityNormal` | Main | Application messages (default) |
| - | - | Log | Log messages (if process is a logger) |

Senders pick a priority via `SendWithPriority` or the process's default `SendPriority`. Every pop checks Urgent -> System -> Main -> Log and returns the first available message.

## MailboxSize

- `ProcessOptions.MailboxSize = 0` (default): **unlimited**. The mailbox can grow without bound -> a
  source of silent memory growth. Find the deep ones with `processes node=<node> minMailbox=1000`.
- `MailboxSize > 0`: senders receive `gen.ErrProcessMailboxFull` on overflow, unless
  `ProcessFallback` is configured.

An unlimited mailbox is reported as `Info.MailboxSize: -1`, not as zero - the queue answers `-1` for
"no limit". The framework's own comment on `gen.ProcessInfo.MailboxSize` saying zero means unlimited
describes the option, not the reading.

## Listing row fields (`Processes[]` of `processes`, `app_tree`, `subtree`)

| Field | Meaning | How to use |
|-------|---------|------------|
| `PID` | Unique identifier | Always cite as `<ABC.0.1005>` |
| `Name` | Registered name | Empty if not registered |
| `Behavior` | Behavior type (e.g. `myapp.Worker`) | Also the `stack=` filter of a goroutine dump |
| `Kind` | Process kind classifier (`ProcessKind`) | See Process Kind below. Groups rows by role. |
| `Application` | Owning application (empty for root processes) | |
| `State` | See table above | |
| `StateTime` | Time in current state (ns) | Pair with `State`: how long a process has sat in `wait response` or `running`. Rising = stuck. |
| `Parent` | Parent PID | |
| `Leader` | Application leader PID | |
| `Uptime` | Seconds since spawn | |
| `MessagesIn` | Cumulative messages received | |
| `MessagesOut` | Cumulative messages sent | |
| `MessagesMailbox` | **Current** queue depth (all 4 queues combined) | Backpressure indicator |
| `MailboxLatency` | Age of oldest unhandled message (ns) | Requires `-tags=latency`; `-1` if disabled |
| `RunningTime` | Cumulative callback time (ns) | `RunningTime / Uptime` = utilization |
| `InitTime` | `ProcessInit` duration (ns) | Slow init detection |
| `Wakeups` | Sleep->Running transitions | Denominator for Drain |
| `LogLevel` | Current per-process log level | Set at spawn or via `process.Log().SetLevel()` |

`StateTime` complements the Call Timeout section below: a process whose `State` is `wait response` with a `StateTime` above `gen.DefaultRequestTimeout` (5s) has been blocked on a single `Call` longer than the default budget.

## Process Kind

Every process carries a `Kind` (`gen.ProcessKind`, a string). It is resolved once at spawn from the base behavior and emitted in every listing row, so you can group and filter a large node by role at a glance.

The base behaviors report their own kind: `act.Actor` -> `actor`, `act.Supervisor` -> `supervisor`, `act.Pool` -> `pool`, `act.Router` -> `router`, `act.WebWorker` -> `web`. A process built directly on `gen.ProcessBehavior` reports `ProcessKindCustom` (`"custom"`) - the default when a behavior does not report its kind.

`ProcessKind` is a free string. Beyond the topology kinds above, the framework defines a shared classifier vocabulary an actor can adopt to self-describe (via `SetProcessKind` at Init/Running), for example:

| Group | Kinds |
|-------|-------|
| Behavior / flow | `fsm`, `saga`, `worker`, `scheduler`, `queue`, `producer`, `consumer`, `coordinator` |
| Boundary / IO | `web`, `gateway`, `proxy`, `stream`, `broker`, `client` |
| Data / state | `store`, `cache`, `session` |
| Operations | `metrics`, `leader`, `follower`, `health`, `monitor`, `logger` |

Any value not in the vocabulary is still valid and tooling treats it as a custom kind. When triaging an unfamiliar node, `Kind` tells you which rows are supervisors and pools (topology) versus application actors, and which self-declared roles (a `cache`, a `gateway`) are present. See godoc `gen.ProcessKind` for the exhaustive constant list.

## Drain

```
Drain = MessagesIn / Wakeups
```

Nothing on the surface sorts or computes this: the tools filter, and the ranking is yours. Take a
listing narrowed to what could matter (`minMailbox`, a `behavior`, an `app`) and compute Drain over
the rows you got back.

| Drain | Interpretation |
|-------|----------------|
| `≈ 1` | One message per wake. Idle-reactive; no backpressure. |
| `> 1` | Multiple messages per wake. Queue was building between wake-ups -> overloaded but keeping up. |
| `>> 1` | Severe build-up; likely bottleneck. |

## Utilization

```
Utilization = RunningTime_ns / (Uptime_sec * 1e9)
```

Interpretation:
- Close to 1.0: actor spends nearly all time in callbacks; bottleneck candidate.
- Close to 0: idle or very sparse load.

Utilization is **not** CPU. Two actors each with utilization 1.0 on a multi-core machine do not saturate the CPU - utilization is a per-actor busy ratio.

## MailboxLatency (requires -tags=latency)

`MailboxLatency_ns` is the age of the oldest message still in the queue (from enqueue to the current time). High latency + low utilization = the process is not processing its mailbox (stuck).

A transient non-zero latency (ms range) during active work is normal - it just means we caught the process mid-callback.

## Liveness Score

Detects stalled callbacks (actor holding a mutex, blocked on a channel, waiting on I/O with no deadline in a handler). Combines utilization with mailbox latency.

```
Liveness = RunningTime_ns / (Uptime_sec * MailboxLatency_ns)
```

Equivalent to: "how much callback work happens per unit of waiting message age".

### Collection

```
processes node=<node> minMailbox=1
```

`minMailbox` is the only filter that matters here: a process with an empty mailbox has nothing to be
late about. The rest of the narrowing is yours, over the rows that come back - drop anything with
`Uptime` under 60 seconds (warmup), anything with `MessagesIn == 0` (never worked), and anything in
`zombee` state, then rank what is left by `MailboxLatency`. There is no uptime filter and no sorting
argument on the tool.

### Interpretation

| MailboxLatency | Liveness | Diagnosis | Action |
|----------------|----------|-----------|--------|
| High (seconds) | `> 0.01` | Overloaded but alive - can't drain fast enough | Scale: pool, more workers, or shed load |
| High (seconds) | `< 0.001` | Callback blocked - holding mailbox without processing | Find blocking call: `goroutines node=<node> stack=<behavior>` |
| 0 | n/a | Healthy, mailbox empty | No action |
| Transient (ms) | any | Caught mid-callback | Normal - re-sample |

**Worked example:** same 18s MailboxLatency, two different processes.
- `RunningTime = 5000s, Uptime = 600s` -> `Liveness = 5000e9 / (600 * 18e9) ≈ 0.46` - heavily busy but alive.
- `RunningTime = 82µs, Uptime = 600s` -> `Liveness = 82e3 / (600 * 18e9) ≈ 7.6e-12` - stuck.

## Call Timeout

`gen.DefaultRequestTimeout` = 5 seconds. A process stuck in `"wait response"` for > 5s is abnormal - either the remote never returned or the response was lost. Find them with `processes node=<node> state="wait response"` and confirm with `goroutines node=<node> stack=waitResponse`.

## Restart Detection

- There is no uptime filter: read `Processes[].Uptime` from a listing and count the rows younger
  than ten seconds yourself. Many of them during steady state = a restart loop somewhere.
- Inspect the parent supervisor with `process_state node=<node> pid=<supervisor>` to see
  `ergo:restarts_count`, strategy, intensity, period. `subtree node=<node> pid=<supervisor>` shows
  the children themselves, read twice to see which ones are being replaced.
- `gen.ErrExceeded` ("limit exceeded") on a supervisor means its restart-intensity sliding window filled up. The supervisor terminates and escalation propagates to the parent supervisor. See `framework-internals.md`.

## Termination Reasons

At termination the framework tags every process with a reason. Two are graceful, two are abnormal. The distinction drives whether the node logs the exit and whether you should be alarmed - and it is the reason value you see in down and exit messages during a restart cascade.

| Reason | Constant | Graceful | Logging at termination |
|--------|----------|----------|------------------------|
| `normal` | `gen.TerminateReasonNormal` | Yes | None. Process returned this from a handler to stop itself. |
| `shutdown` | `gen.TerminateReasonShutdown` | Yes | None. Node is stopping (`node.Stop()`), children terminate in order. |
| `kill` | `gen.TerminateReasonKill` | No | Error logged. `node.Kill(pid)` or a kill signal - immediate, no graceful shutdown. |
| `panic` | `gen.TerminateReasonPanic` | No | Panic logged with stack trace. A handler panicked; the framework recovered and terminated the process. |

A `kill` delivered while the process is dispatching a message is exactly what leaves a process in `zombee` state (see Process States) - the goroutine is abandoned mid-callback. During a restart-loop investigation, an exit reason of `normal` or `shutdown` is expected traffic; `kill` and `panic` are the ones to trace back to a cause. Correlate with the log, read as `ergo://<node>/log?levels=error,panic` and read again with `since=`.

## Meta Processes

Meta processes are identified by `gen.Alias`, not `gen.PID`. The alias comes from `Info.Metas` of the
owning process and nowhere else. Their record is `ergo://<node>/meta/<alias>`, a narrower shape than
a process: two mailbox depths (`Main`, `System`), no latency fields at all. `meta_state` is the
meta describing itself, the counterpart of `process_state`.

## Metrics to Watch per Symptom

| Symptom | Fields |
|---------|--------|
| Slow messages on one actor | `MessagesMailbox`, `MailboxLatency`, `RunningTime/Uptime`, Drain |
| Memory growth | `MessagesMailbox` across all (default unlimited mailbox), `heap_profile`, `Info.MemoryAlloc` from `node` |
| Stuck callbacks | Liveness score - low value with non-zero latency |
| Process leak | `Info.ProcessesSpawned` vs `Info.ProcessesTerminated` from `node` |
| Restart loop | `Processes[].Uptime` from `processes`, `process_state` on the supervisor |
| Zombie accumulation | `processes state=zombee` |
| Pool saturation | `process_state` on the Pool PID - `ergo:messages_unhandled` counter |
