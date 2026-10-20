# Counters

Every counter the observer surface carries, grouped by the answer that carries it. A field is
addressed by its JSON path inside that answer - `Info.MemoryUsed`, `Connections[].BytesIn`,
`Nodes[].Info.LogMessages` - because the same counter appears under different paths in a single
reading and in a listing.

Counters are either **cumulative** (monotonically increasing since node start; the delta between two
readings is the activity) or **instantaneous** (the value now; one reading is the state). Getting
this wrong is the most common misreading: a cumulative number that looks alarming is usually just
old.

**Units and sentinels are published with the answer.** An answer that carries a legend puts it under
the JSON key `services.ergo/legend`, holding `units`, `sentinels` and `axes` keyed by field path
relative to the object that carries the legend, with `[]` for a collection step
(`Connections[].BytesIn`, `MailboxQueues.LatencyMain`, `Nodes[].Info.MemoryLimit`, `LogMessages`).
Read the legend at runtime rather than trusting a remembered unit. These carry one:

| Carries a legend | At |
|------------------|----|
| `node`, `network` | `Info` |
| `connection`, `connections` | top level (`Info.*`, `Connections[].*`) |
| `processes`, `app_tree`, `subtree` | top level (`Processes[].*`) |
| `ergo://<node>/process/<pid>`, `ergo://<node>/meta/<alias>` | `Info` |
| `applications`, `heap_profile`, `types`, `ergo://cluster` | top level |

`event` and `events` carry none, so `CreatedAt` and `LastPublishedAt` (unix ns) and `BufferSize`
(messages) are documented here and nowhere else. So do `goroutines`, `process_state`, `meta_state`,
`process_lookup`, `errors`, `atoms`, `cron`, `cron_schedule`, the four `registrar_*`, `capabilities`,
and the `log`, `stream` and `tracing` lenses. One gap inside the legend itself:
`Info.DecompressedOrigRecv` on a connection is bytes and has no `unit` entry, while its three
siblings do.

**Nothing here is `runtime.MemStats`.** The node samples `runtime/metrics` and never stops the world
to answer a reading.

## node - `ergo://<node>`

The `node` tool and the resource `ergo://<node>` (equivalently `ergo://<node>/node`) answer the same
`Info`. `ergo://cluster` carries a shorter version of it for every node at once, under
`Nodes[].Info`, which makes two readings of that one resource the cheapest trend in the surface.

### Process counters

`Info.*` of `node`, and `Nodes[].Info.*` of `ergo://cluster`.

| Counter | Kind | Meaning / Diagnostic use |
|---------|------|--------------------------|
| `ProcessesTotal` | instant | All processes on the node |
| `ProcessesRunning` | instant | Currently in Running state (dispatching) |
| `ProcessesWaitResponse` | instant | Blocked in sync `Call()`. Persistently high = stuck Calls. |
| `ProcessesZombee` | instant | Killed processes, goroutines orphaned. Should stay 0. Non-zero = leak. |
| `ProcessesSpawned` | cumulative | Total successful spawns |
| `ProcessesSpawnFailed` | cumulative | Failed spawns (factory errors, name taken, init errors) |
| `ProcessesTerminated` | cumulative | Total terminations |

**Process leak check:** `ProcessesSpawned - ProcessesTerminated ≈ ProcessesTotal`. To trend it, read
`ergo://<node>` twice with a gap of ten seconds, or read `ergo://cluster` once per interval and get
the same divergence for every node in one answer.

### Error counters

`Info.*` of `node`, and `Nodes[].Info.*` of `ergo://cluster`.

| Counter | Meaning | Root cause to investigate |
|---------|---------|---------------------------|
| `SendErrorsLocal` | Local `Send` failures | Target unknown / mailbox full / terminated |
| `SendErrorsRemote` | Remote `Send` failures | Connection problems, target missing on remote |
| `CallErrorsLocal` | Local `Call` failures | Same + timeouts |
| `CallErrorsRemote` | Remote `Call` failures | Same + network latency |

All four are cumulative. Non-zero is worth investigating, but the question is always the growth
between two readings: a node up for a week has non-zero counters from the week.

### Registration counters

`Info.*` of `node` only - not in the cluster answer.

| Counter | Kind | Meaning |
|---------|------|---------|
| `RegisteredNames` | instant | Currently registered process names |
| `RegisteredAliases` | instant | Live aliases |
| `RegisteredEvents` | instant | Registered events |

### Event pub/sub counters

`Info.*` of `node` only.

| Counter | Kind | Meaning |
|---------|------|---------|
| `EventsPublished` | cumulative | Local producers' publishes |
| `EventsReceived` | cumulative | Events arriving from remote nodes |
| `EventsLocalSent` | cumulative | Delivered to local subscribers |
| `EventsRemoteSent` | cumulative | Forwarded to remote subscriber nodes |

Fanout check: `EventsRemoteSent / EventsPublished` tells you how many remote deliveries each publish
generates. High ratio + small cluster = wide fanout.

### Application counters

`Info.*` of `node`, and `Nodes[].Info.*` of `ergo://cluster`.

| Counter | Kind | Meaning |
|---------|------|---------|
| `ApplicationsTotal` | instant | Loaded applications |
| `ApplicationsRunning` | instant | In Running state |

### Log and tracing counters

| Counter | Layout | Meaning |
|---------|--------|---------|
| `Info.LogMessages` | `[6]uint64` | Cumulative per level. Also `Nodes[].Info.LogMessages` in the cluster answer. |
| `Info.TracingSpans` | `[5]uint64` | Cumulative per observation point. Node answer only. |

Both are arrays whose index meaning arrives with the answer as the legend's `axes` entry
(`trace,debug,info,warning,error,panic` and `send,request,response,spawn,terminate`), so read it
there rather than counting positions from memory.

Watch the error and panic slots - growth indicates real problems. `ergo://cluster` gives every
node's array in one answer, which is how an outlier is found: one node logging far more errors than
its replicas.

`Info.TracingExporters[].DroppedSpans` is cumulative and is the counter that matters after tracing is
switched on: the node paid to make those spans and then threw them away, which is an exporter
problem rather than a sampler one.

### Runtime counters

All instantaneous unless marked, all under `Info` of `node`. The first two are the ones most often
misread.

| Counter | Kind | Meaning |
|---------|------|---------|
| `MemoryUsed` | instant | `/memory/classes/total:bytes` - all memory obtained from the OS |
| `MemoryAlloc` | instant | `/memory/classes/heap/objects:bytes` - bytes held by live heap objects, always the smaller of the two |
| `MemoryLimit` | config | bytes; `MaxInt64` when no limit is set, which the legend states as a sentinel |
| `HeapLive` | instant | bytes live as of the last GC |
| `HeapGoal` | instant | the heap size that triggers the next GC |
| `Goroutines` | instant | goroutine count |
| `GCCycles` | cumulative | completed GC cycles |
| `HeapAllocObjects` | cumulative | objects allocated since start |
| `HeapFreeObjects` | cumulative | objects freed since start |
| `CPUTimeGC` | cumulative (sec) | CPU seconds spent in GC |
| `CPUTimeTotal` | cumulative (sec) | available CPU seconds, `GOMAXPROCS`-defined, idle included |
| `UserTime` | cumulative (ns) | user CPU time |
| `SystemTime` | cumulative (ns) | system CPU time |
| `ServerTime` | timestamp | RFC3339 with offset, not a number |
| `Uptime` | instant (sec) | Seconds since the node started - the denominator of every rate you compute from a cumulative counter |

Derivations that hold:

```
objects alive  = HeapAllocObjects - HeapFreeObjects
allocation churn = growth of HeapAllocObjects between two readings
GC cost        = CPUTimeGC / CPUTimeTotal, plus the growth of GCCycles
headroom       = MemoryLimit - MemoryUsed        (only when a limit is set)
```

`MemoryAlloc - MemoryUsed` is not freed bytes: it is negative, and there is no reading in the surface
that gives bytes ever allocated. Use the object counters for churn.

`ServerTime` helps correlate events across nodes in different timezones.

The rest of the node answer is not counters but is read alongside them: `Name`, `Version`,
`Framework`, `Commercial[]`, `LogLevel`, `Loggers[]{Name, Behavior, Levels[]}`, `Tracing`, `Cron`,
and `Env`, which is empty unless `NodeOptions.Security.ExposeEnvInfo` is on.

## network - `ergo://<node>/network`

| Counter | Kind | Meaning |
|---------|------|---------|
| `Info.ConnectionsEstablished` | cumulative | Total connections ever opened |
| `Info.ConnectionsLost` | cumulative | Total connections lost |

**Current active connections** = `ConnectionsEstablished - ConnectionsLost`. Growing
`ConnectionsLost` = instability; pair it with the per-connection `Reconnections` counter from
`connection` to find the flaky peer.

`Info.MaxMessageSize` is bytes with `0` = unlimited. The rest of the answer is configuration that
explains counters elsewhere: `Mode`, `Registrar`, `HandshakeVersion`, `ProtoVersion`, `Nodes[]`,
`Routes[]`, `ProxyRoutes[]`, `EnabledSpawn[]`, `EnabledApplicationStart[]` and `Flags{...}`. In
`Flags`, `EnableSoftwareKeepAlive` is seconds with `0` = disabled; the other twelve are booleans.
The resource carries `Stopped` when the whole network stack is down.

### Acceptors

There is no separate acceptor tool: acceptors are `Info.Acceptors[]` of the same network answer.

| Counter | Kind | Meaning |
|---------|------|---------|
| `Info.Acceptors[].HandshakeErrors` | cumulative | Failed handshakes on this acceptor |

Non-zero = cookie mismatches, version incompatibility, or scanning. It is per-acceptor, so it
isolates which interface is being probed. Each acceptor also carries `Interface`, `MaxMessageSize`
(bytes), `Flags`, `TLS`, `CustomRegistrar`, `RegistrarServer`, `RegistrarVersion`,
`HandshakeVersion` and `ProtoVersion` - the last two being what a peer must match to get in.

## connection - `ergo://<node>/connection/<peer>` and `connections`

One connection is `connection node=A peer=B`, or the resource `ergo://A/connection/B`, with the
counters under `Info`. Every connection of a node is `connections`, with the same counters under
`Connections[]`.

**Always check both sides** - `connection node=A peer=B` and `connection node=B peer=A`, or one
`cluster_batch` with two steps under a key, read back at its `ergo://job/<key>` URI. Divergence is
silent data loss.

A `connection` to a peer this node is not talking to is a tool error (`node <node>: <reason>`), not a
row; only the resource carries `Disconnected: true`.

| Counter | Kind | Meaning |
|---------|------|---------|
| `Uptime` | instant (sec) | Remote node uptime (reported during handshake) |
| `ConnectionUptime` | instant (sec) | Age of this connection |
| `MessagesIn` | cumulative | Messages received from this peer |
| `MessagesOut` | cumulative | Messages sent to this peer |
| `BytesIn` / `BytesOut` | cumulative | Byte counters |
| `TransitBytesIn` / `TransitBytesOut` | cumulative | Proxy-transit traffic if this connection is a proxy |
| `Reconnections` | cumulative | Pool-item reconnections. Non-zero = at least one TCP link broke and was re-established; growing = ongoing flap. |

Cross-check rules:
- `A.MessagesOut` toward B should match `B.MessagesIn` from A. Significant divergence = data loss.
- `ConnectionUptime << Uptime` = the connection was re-established (flap); combine with
  `ConnectionsLost` from `network`.

Alongside the counters, `Info` carries what the two sides actually negotiated - `Node`, `Version`,
`HandshakeVersion`, `ProtoVersion`, `MaxMessageSize` (bytes), `TLS`, `NetworkFlags{...}` - which is
what makes a missing `FragmentsSent` or a flat compression counter explainable rather than
mysterious.

### Fragmentation counters

Messages above the peer's `MaxMessageSize` are split into fragments and reassembled on the far side.

| Counter | Kind | Meaning |
|---------|------|---------|
| `FragmentsSent` | cumulative | Individual fragments sent |
| `FragmentMessagesSent` | cumulative | Messages that were fragmented for sending |
| `FragmentsReceived` | cumulative | Individual fragments received |
| `FragmentMessagesRecv` | cumulative | Fragmented messages successfully reassembled |
| `FragmentTimeouts` | cumulative | Reassemblies that timed out. **Non-zero is a fault** - a fragmented message never completed = dropped payload. |

`FragmentTimeouts > 0` is a direct fault signal: pair it with an `A.MessagesOut` / `B.MessagesIn`
divergence to confirm dropped messages.

### Compression counters

Compression applies only to cross-node messages above the process compression threshold (see
`framework-internals.md` §Compression).

| Counter | Kind | Meaning |
|---------|------|---------|
| `CompressedSent` | cumulative | Messages compressed on send |
| `CompressedBytesSent` | cumulative | Bytes after compression (wire size) |
| `CompressedOrigBytesSent` | cumulative | Bytes before compression (original size) |
| `DecompressedRecv` | cumulative | Messages decompressed on receive |
| `DecompressedBytesRecv` | cumulative | Bytes before decompression (wire size) |
| `DecompressedOrigRecv` | cumulative | Bytes after decompression (original size); bytes, though the legend omits the unit here |

Actual send-side compression ratio = `CompressedBytesSent / CompressedOrigBytesSent`. A ratio near
1.0 means compression is buying little - the threshold may be set too low for this traffic. A
counter flat at zero is more often `EnableCompression` missing from the negotiated
`Info.NetworkFlags` than an absence of large messages.

### Tracing and clock counters

| Counter | Kind | Meaning |
|---------|------|---------|
| `TracedSent` | cumulative | Messages sent carrying a tracing wrapper |
| `TracedReceived` | cumulative | Messages received carrying a tracing wrapper |
| `ClockSkew` | instant (ns) | Estimated clock offset relative to the peer. Positive = remote clock ahead; `0` = not yet measured. |

`TracedSent` on one side against `TracedReceived` on the other is what confirms cross-node trace
propagation, which also needs `EnableTracing` in `NetworkFlags` on both sides.

`ClockSkew` complements the node's `Info.ServerTime` when correlating timestamped events across
nodes - subtract it before comparing remote timestamps to local ones.

### Pool fields

Instantaneous, not counters, but essential for spotting a degraded connection pool:

| Field | Kind | Meaning |
|-------|------|---------|
| `PoolSize` | config | Configured **target** number of TCP connections (set by the acceptor during handshake). Deliberately unitless in the legend: connections, not bytes. |
| `PoolLen` | instant | Current **live** number of TCP connections; reaches `PoolSize` once fully filled |
| `PoolDSN` | instant | Connection strings (`host:port`) for each pooled connection; non-nil means this side dialed out |

`PoolLen < PoolSize` means the pool has not fully filled - it is still filling, or has been degraded
by lost links. Cross-check with a growing `Reconnections` to confirm churn.

## process - `ergo://<node>/process/<id>.<creation>`

The full per-process record is a resource, reached from a `processes` row. It is not
`process_state`: that tool asks the process for its own sections and answers
`{State: map[string]string, Error}` with none of these counters in it.

### Mailbox depth and latency

The process resource carries all eight fields under `Info.MailboxQueues`:

| Queue | Depth field | Latency field (ns) |
|-------|-------------|--------------------|
| Urgent | `Info.MailboxQueues.Urgent` | `Info.MailboxQueues.LatencyUrgent` |
| System | `Info.MailboxQueues.System` | `Info.MailboxQueues.LatencySystem` |
| Main | `Info.MailboxQueues.Main` | `Info.MailboxQueues.LatencyMain` |
| Log | `Info.MailboxQueues.Log` | `Info.MailboxQueues.LatencyLog` |

Latency is the age of the oldest message in that queue: `-1` means the node was built without
`-tags=latency`, `0` means the queue is empty. The legend states both sentinels.

A **listing** carries one of each instead: `Processes[].MessagesMailbox` is the total depth and
`Processes[].MailboxLatency` the worst age across queues, with `-1` = no `latency` tag and `0` = all
queues empty. That is the pair to scan a node with; the eight-way split is for the one process the
scan pointed at.

`Info.MailboxSize` is the configured limit, `-1` = unlimited. The framework's own comment saying zero
means unlimited is stale.

### Message and time counters

`Info.*` of the process resource, and `Processes[].*` of `processes`, `app_tree` and `subtree`.

| Counter | Kind | Meaning |
|---------|------|---------|
| `MessagesIn` | cumulative | Messages the process received |
| `MessagesOut` | cumulative | Messages the process sent |
| `RunningTime` | cumulative (ns) | Time spent executing callbacks |
| `InitTime` | one-shot (ns) | Duration of `ProcessInit()` |
| `Wakeups` | cumulative | Sleep->Running transitions |
| `Uptime` | instant (sec) | Seconds since spawn |
| `StateTime` | instant (ns) | Time in the current state |

Derivations used in diagnostics:

```
Utilization   = RunningTime / (Uptime_sec * 1e9)
Drain         = MessagesIn / Wakeups
Liveness      = RunningTime / (Uptime_sec * MailboxLatency_ns)
```

`MailboxLatency` for the Liveness input comes from a listing row. In the process resource the
equivalent is the worst of the four `Info.MailboxQueues.Latency*` values.

See `process-model.md` for how to read these values.

The resource also carries what shapes the numbers: `State`, `Kind`, `Parent`, `Leader`, `Application`,
`Behavior`, `LogLevel`, `Compression{Enable, Type, Level, Threshold bytes}`, `MessagePriority`,
`Tracing`, `Fallback`, `Env`, `Aliases[]`, `Events[]`, `Metas[]`, the five `Monitors*` and five
`Links*` lists, `KeepNetworkOrder` and `ImportantDelivery`. `Info.Metas` is the only place a meta
alias comes from, which is what the `meta` resource needs.

`ergo://<node>/meta/<alias>` has a narrower shape: `Info.MailboxQueues` there has two depths
(`Main`, `System`) and no latency fields at all, plus `ID`, `MessagesIn`, `MessagesOut`,
`MailboxSize`, `MessagePriority`, `Uptime`, `State`, `Behavior`, `Parent`, `Application` and
`LogLevel`.

## event - `ergo://<node>/event/<name>` and `events`

One event is the `event` tool or resource, with fields under `Info`. Every event is `events`, same
fields under `Events[]`. Neither carries a legend, so the units below are the only source.

| Field | Kind | Meaning |
|-------|------|---------|
| `Event` | identity | The qualified event name |
| `Producer` | instant | PID of the registering process |
| `Subscribers` | instant | Current subscriber count (local + remote) |
| `Notify` | flag | Producer requested `MessageEventStart/Stop` notifications |
| `Open` | flag | Event accepts publishes from any local process (token check disabled) |
| `BufferSize` | config | Configured ring buffer size, in messages (from `RegisterEvent`) |
| `CurrentBuffer` | instant | Entries currently in the buffer |
| `CreatedAt` | timestamp | Unix nanosecond of registration |
| `MessagesPublished` | cumulative | Calls to `SendEvent` |
| `MessagesLocalSent` | cumulative | Delivered to local subscribers |
| `MessagesRemoteSent` | cumulative | Forwarded to remote subscribers |
| `LastPublishedAt` | timestamp | Unix nanosecond of the most recent `SendEvent`; `0` if nothing was ever published |

Three states worth deriving, since no field names them:
- `MessagesPublished > 0` and `Subscribers == 0` and `Notify == false` - publishing into the void.
- `Subscribers > 0` and `MessagesPublished == 0` and `Notify == false` - subscribers starved.
- `MessagesPublished > 0` with a **stale** `LastPublishedAt` - a silent producer: it used to publish
  and has gone quiet. The two rules above only catch producers that *never* published;
  `LastPublishedAt` is what catches one that *stopped*. Compare it against the node's
  `Info.ServerTime`, or a peer's `ClockSkew`-adjusted clock, to gauge how long the silence is.

The `event` **resource** additionally carries a sampled window of the event itself:
`Entries[]{Timestamp unix ns, Type, Message, Verbose}`, `Suppressed`, `Closed`, `Reason`, `Watching`
and `WatchReason`. For the messages as they flow rather than a window,
`ergo://<node>/stream/<name>` is the accumulating subscription - see `watching.md`.

## applications - `ergo://<node>/applications`

Per application under `Applications[]`: `ProcessesTotal` with the sentinel `0 = the application is
fully stopped`, plus `Name`, `Mode`, `State`, `Weight`, `Parent`, `Uptime` (sec), `Version`,
`Description`, `Tags`, `Map`, `Depends`, `Group[]` and `Env`.

This `ProcessesTotal` is not the node's `Info.ApplicationsTotal`: it counts the processes of one
application, and a zero is how a loaded-but-stopped application is told apart from a running one
whose processes happen to be idle. `app_tree` and `subtree` are the way from here down to the
processes themselves.

## heap_profile, types, goroutines

`heap_profile`: `Records[]{InuseBytes, InuseObjects, AllocBytes, AllocObjects, FreeObjects, Stack[]}`
plus `TotalInuse`, `TotalAlloc`, `TotalObjects` and `Truncated` (a count). Records come largest
first, 25 by default, and the totals are of the whole profile rather than of the page. Taking one
pauses the node: ask once and read it carefully.

`types`: per registered type, `Stats{Enabled, Encoded, Decoded, EncodedBytes, DecodedBytes}` with
`Enabled: false` meaning the node was built without `-tags=typestats` and every counter stays zero,
plus `ID`, `Name`, `Kind`, `Schema`, `Proto`, `MinSize` (bytes) and `SizeVariable`.

`goroutines`: `Groups[]{Count, State, WaitSec, Origin, Current, Stack, IDs[]}`, `Total`, `Filtered`.
`WaitSec` comes from the runtime's whole-minute annotation, so a `minWait` below 60 selects the same
set as 60. This also pauses the node.

## ergo://cluster

`Nodes[]{Info, Age ms, Status, Reason}` where `Info` is the short node info: most of the `node`
counters (processes, errors, applications, `LogMessages`, `MemoryUsed`, `MemoryAlloc`,
`MemoryLimit`, `HeapLive`, `HeapGoal`, `Goroutines`, `GCCycles`, `CPUTimeGC`, `CPUTimeTotal`,
`UserTime`, `SystemTime`) plus `Uptime`, `Creation`, `Mode`, `LogLevel`, `Applications[]`, `ServerTime` and
`Peers[]{Node, ConnectionUptime, MessagesIn, MessagesOut, BytesIn, BytesOut, Reconnections, TLS}`.
Not present: `HeapAllocObjects`, `HeapFreeObjects`, `TracingSpans`, and the registration and event
counters.

Alongside: `Offline[]{Node, Status, Reason, Since unix sec, LastSeen unix sec (0 = never reached),
Peers[]}`, `Complete`, `Note`, `Watched`, `Limit` and `WatchPeriod` (ns).

Two readings of this one resource give the process, error, log and memory trend of every node at
once, which is where a cluster-wide question should start before any per-node tool.

## Patterns for Investigation

### Silent message loss

Compare both sides of a connection:

```
connection node=A peer=B   -> Info.MessagesOut = 2363
connection node=B peer=A   -> Info.MessagesIn  = 1
```

B never received what A sent. Inspect `Reconnections` and `FragmentTimeouts` on both sides - a
reconnect mid-transfer and a failed fragment reassembly both drop payload - and
`Info.SendErrorsRemote` on A's `node` answer.

### Growing heap

Read `ergo://<node>` twice and compare: `Info.MemoryAlloc` rising while `GCCycles` climbs means the
collector ran and could not reclaim. Put `Info.HeapLive` against `Info.HeapGoal` for where the next
cycle triggers, and `Info.MemoryUsed` against `Info.MemoryLimit` for how much room is left. Churn is
the growth of `HeapAllocObjects`; GC cost is `CPUTimeGC` against `CPUTimeTotal`. For the call paths,
`heap_profile node=<node> minBytes=1048576` - once, because it pauses the node.

### Restart loop

Growing `Info.ProcessesSpawned` without matching `Info.ProcessesTerminated`. Find the short-lived
processes by reading `Processes[].Uptime` from `processes`, or by watching the supervisor with
`subtree`; the reason each child died is in `ergo://<node>/log?levels=error`.

### Handshake problems

`Info.Acceptors[].HandshakeErrors > 0` from `network`. Typical causes: wrong cookie,
Erlang-vs-EDF protocol mismatch, TLS misconfiguration. Capture the detail with
`ergo://<node>/log?levels=error,panic`, read again with `since=`.

### Event waste

`events minSubscribers=0` lists the events nobody is listening to; the derivations above say which
of those are still being published into. Confirm with `ergo://<node>/stream/<name>`, which shows
messages still arriving with no subscriber to take them. Publishing cost is mostly wasted -
producers should either gate on subscriber presence (`Notify: true`) or stop publishing.
