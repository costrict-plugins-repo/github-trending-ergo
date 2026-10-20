---
name: devops
description: Diagnose production issues on running Ergo Framework nodes through the MCP surface of the Observer application. Covers performance bottlenecks, process leaks, memory growth, network problems, event fanout, restart loops and stuck processes. Provides the resource and tool catalog, liveness analysis, watching over time, and the capability model that bounds what may be changed. Use when investigating issues on live Ergo nodes.
---

# Ergo DevOps

Diagnose production issues in running Ergo Framework systems through the MCP surface of the Observer
application (`ergo.services/application/observer`). This skill is the operational counterpart to the
`framework` skill: it assumes the system is running and something is suspect.

## Connecting

Observer serves MCP on `/mcp` of any listener that has the surface enabled, which by default is the
same listener serving its web UI on port 9911:

```bash
claude mcp add --transport http ergo http://localhost:9911/mcp
```

With an authenticating proxy in front, pass what it expects:

```bash
claude mcp add --transport http ergo http://localhost:9911/mcp \
  --header "Authorization: Bearer ${TOKEN}"
```

Only the node serving MCP needs the Observer application. Every Ergo node runs the built-in `system`
application that answers these questions, so one endpoint reaches the whole cluster.

## Start here, every time

1. **Read `ergo://cluster`.** It names every node this observer knows, and every other URI and every
   tool argument takes a node name from it. It is also the answer about the cluster as a whole: who
   is online, for how long, who fell out and why. A node name you invented is refused, not guessed
   at.
2. **Ask `capabilities` for the node you are about to work on**, before planning anything that
   changes it. It answers what this node allows this caller, already narrowed by every ceiling in
   between, so you decide what is possible instead of discovering it through refusals.

## The two halves

A **resource** is a reading with an address: `ergo://<node>/<lens>`, read again with `since=` to take
only what has landed since. A **tool** is an act that answers once.

The full URI grammar, every lens with its parameters, and every tool with its arguments are in
`references/tools.md`. Do not guess an argument name: a wrong one is refused, and the schema is one
read away.

## Reference files

| Topic | File | When to read |
|-------|------|--------------|
| The whole surface: URI grammar, every lens with its parameters, every tool with its arguments, which ones mutate, which ones pause the node | `references/tools.md` | Any investigation |
| Process model: states, mailbox queues, the metrics (`RunningTime`, mailbox latency, wakeups), the liveness reading | `references/process-model.md` | Interpreting `processes` / `process_state` |
| Counters, field by field, in what the `node`, `network`, `connection` and `event` readings answer | `references/counters.md` | Reading any of those |
| Watching over time: accumulating lenses and `since=`, keyed watches, `subscriptions/listen`, runs, and trace samplers | `references/watching.md` | Any symptom not visible in one reading |
| Diagnostic playbooks, symptom by symptom | `references/playbooks.md` | The symptom matches one |
| Framework internals: important-delivery errors, restart intensity, pool backpressure, connection pool, event fanout, shutdown | `references/framework-internals.md` | Explaining counter behavior or error semantics |
| Build tags: `pprof`, `latency`, `verbose`, `norecover`, `typestats` - what each enables and what reads as a sentinel without it | `references/build-tags.md` | A metric is missing or reads `-1` |

## Critical rules

1. **Every answer explains its own numbers.** A reading carrying `services.ergo/legend` states the
   unit of a field, the meaning of a sentinel value, and the axis of a counter array. Read it rather
   than assuming: `-1` in a latency field is a missing build tag, not a fast mailbox.
2. **An absent thing says why.** A refused reading names what could not be read and the reason. An
   empty list means empty; it never means "not permitted" or "gone".
3. **`goroutines` and `heap_profile` stop the world** on the node they run on, for as long as the
   walk takes. Ask once, filter at the source, read carefully. Never in a loop, never fanned out
   across a cluster.
4. **Mutating tools require explicit user permission.** `send`, `send_exit`, `kill`,
   `log_level_set`, `tracing_sampler_set`, `process_tune`, `app_start`, `app_stop`, `app_unload`
   change a running system. Each is a capability under `manage.` that a ceiling can refuse, and a
   read-only surface refuses all of them.
5. **Filter and page at the source.** Every listing takes filters and limits in the URI or in the
   arguments. A full dump is slow to transport and poor signal.
6. **Both sides of a connection.** When diagnosing the network, read the `connection` lens on each
   end. A one-sided view misses silent loss.
7. **A key belongs to a caller.** Keyed watches and runs are scoped to the authorized subject, so
   they need a listener with an authorizer, and another caller can neither read nor cancel them.
8. **One question to many nodes goes through `cluster_query` or `cluster_batch`**, which answer at
   once with the URI of a run. Read that URI, then read it again with `since=`. Do not loop a tool
   over thirty nodes by hand.

## Build tag shortcuts

| Tag | Effect | Default |
|-----|--------|---------|
| `-tags=pprof` | Labels actor and meta goroutines with their PID or alias, so a goroutine dump can be attributed to a process, and starts an HTTP pprof server on localhost:9009 (`PPROF_HOST`/`PPROF_PORT`) | off |
| `-tags=latency` | Measures mailbox latency per queue | off: latency fields read `-1` |
| `-tags=verbose` | Verbose internal logging through `lib.Verbose()` | off: internal traces silent |
| `-tags=norecover` | **Disables** panic recovery so a panic takes the node down (debug only) | off: recovery is enabled |
| `-tags=typestats` | Per-type EDF encode and decode counters, surfaced by the `types` tool | off: the counters read zero and the answer says so |

See `references/build-tags.md` for details on what breaks when each tag is missing.
