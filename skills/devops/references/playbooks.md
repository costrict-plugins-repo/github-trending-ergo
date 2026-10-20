# Diagnostic Playbooks

Eleven playbooks for the symptoms that actually come up. Each follows observe -> hypothesize ->
test -> confirm -> recommend. Names and arguments come from `tools.md`, counter meanings from
`counters.md`, the semantics behind them from `framework-internals.md`, and anything that needs more
than one reading from `watching.md`.

Two habits apply to all of them. **Read `ergo://cluster` first**: every URI and every tool argument
takes its node name from there. **Put one question to many nodes with `cluster_query`**, not with a
loop of tool calls: it answers at once with `ergo://job/<key>` and the nodes report into it.

## 1. Cluster Health

**Symptoms:** "something feels off", a periodic check, verification before or after a deploy.

**Observe**

```
resources/read ergo://cluster
cluster_query key=health tool=node nodes=[...]
   -> ergo://job/health
resources/read ergo://job/health          # and again with ?since= until every node has reported
```

`ergo://cluster` already carries uptime, creation and why an absent node went. The run adds each
node's own reading: memory, processes, applications, error counters, log volume.

**Present as a table**, one row per node: node, uptime, processes total and running, memory used
against the limit, send and call errors, error and panic log counts, applications running against
loaded.

**Flag**

- an uptime outlier: that node restarted, and the creation in `ergo://cluster` confirms it
- processes climbing on one node only
- a memory outlier, or memory used approaching `MemoryLimit` where one is set
- send or call error counters moving at all
- the error and panic cells of `LogMessages` growing: the legend names the axis
- applications running fewer than loaded: something is loaded and not started
- a node in `ergo://cluster` marked unreachable, with the reason it gives

**Confirm** by reading the outlier node directly: `ergo://<node>` for the numbers, then
`ergo://<node>/log?levels=error,panic` for what it says about itself.

There is no round-trip probe on this surface. Liveness comes from `ergo://cluster` - a node absent
or marked unreachable, and how old its last reading is - and from the counters of the connection to
it.

## 2. Performance Bottleneck

**Symptoms:** rising latency, one endpoint slow, complaints.

**Observe**

```
resources/read ergo://<node>/processes?minMailbox=100
processes node=<node> minMailbox=100 limit=20
```

The lens and the tool answer the same rows. Sort in the reader: this surface has no `sort_by`.

**Hypothesize by pattern**

| Mailbox | What else | Hypothesis | Next |
|---------|-----------|-----------|------|
| Deep, and draining | High `MessagesIn`, healthy `RunningTime` | Producer outruns consumer | Scale the consumer, or put a Pool behind it |
| Deep, and not draining | `State` stuck in `running` for tens of seconds | A callback that blocks | Confirm with `goroutines` below |
| Deep, and not draining | Modest `RunningTime`, high mailbox latency | Waiting on something else, not on CPU | Read its `process_state` for what it waits on |
| Shallow but growing everywhere | Every process affected | Node-wide: GC pressure, CPU starvation, a shared dependency | The memory and CPU counters of `ergo://<node>` |

**Test: what the process says about itself**

```
process_state node=<node> pid=<pid>
process_state node=<node> pid=<pid> items=["help"]   # which sections it offers
```

That is the process's own `HandleInspect` output. An actor exposing nothing useful there cannot be
diagnosed from the outside.

**Test: is the callback blocked**

```
goroutines node=<node> stack="<behavior or package>"
```

Look for `chan receive`, `select`, `sync.(*Mutex).Lock` or a blocking syscall inside a handler. This
pauses the node while it walks: **once, filtered, and read it carefully.**

**On CPU.** This surface has no CPU profiler. What it has: `RunningTime` per process, the cumulative
time spent inside callbacks, which identifies a hot process; and the node's `CPUTimeGC` against
`CPUTimeTotal`, which says whether the cost is collection rather than work. For a real CPU profile
build the node with `-tags=pprof`, which starts an HTTP pprof server on `localhost:9009`, and use
`go tool pprof` against it.

**Recommend**

- producer outruns consumer: a Pool, or bounded sending with backpressure
- blocked callback: move the wait out of the handler. That is a bug, not a capacity problem
- hot process: the work itself, or spread it

## 3. Memory Growth

**Symptoms:** RSS trending up, GC pauses, an eventual OOM.

**Observe**

```
resources/read ergo://<node>            # MemoryUsed, MemoryAlloc, HeapLive, HeapGoal, GCCycles,
                                        # HeapAllocObjects, HeapFreeObjects, CPUTimeGC, CPUTimeTotal
heap_profile node=<node> minBytes=1048576 limit=25
```

`heap_profile` answers the largest entries first, and `TotalInuse`, `TotalAlloc` and `TotalObjects`
describe the whole profile rather than the page you were handed. `minBytes` drops the noise at the
source. It pauses the node: once.

**Trend it**, because one reading proves nothing: read `ergo://<node>`, wait, read it again, compare.

**Read the two memory numbers correctly.** Neither is a `runtime.MemStats` field, and neither is
cumulative: the node samples `runtime/metrics`, which costs it nothing. `MemoryUsed` is all memory
obtained from the OS (`/memory/classes/total:bytes`), `MemoryAlloc` is the bytes held by live heap
objects (`/memory/classes/heap/objects:bytes`), and `MemoryAlloc` is therefore always the smaller of
the two. Their difference is not "freed bytes" and their ratio is not churn.

**Interpret**

- `MemoryAlloc` rising across readings while `GCCycles` climbs: the collector ran and could not
  reclaim. Something is live. A leak.
- `MemoryUsed` rising while `MemoryAlloc` stays flat: the heap is not growing, so look at what else
  takes memory from the OS - stacks, and the number of goroutines.
- churn is the growth of `HeapAllocObjects` between two readings; the objects currently alive are
  `HeapAllocObjects - HeapFreeObjects`.
- GC cost is `CPUTimeGC` against `CPUTimeTotal`, plus the growth of `GCCycles`. Both are cumulative
  seconds, so compare two readings rather than one.
- `MemoryLimit` is `MaxInt64` when no limit is set, which the legend states as a sentinel.
- the top entries of `heap_profile` in your own packages: the leak suspect.
- deep mailboxes: messages are memory. A backlog of large messages is a memory symptom whose cause
  is a slow consumer, so this becomes playbook 2.

**Recommend** the allocation site, pooling, or bounding the queue that holds the memory.

## 4. Process Leak

**Symptoms:** the process count climbing, file descriptors growing, memory following.

**Observe**

```
resources/read ergo://<node>                              # ProcessesTotal against ProcessesRunning
resources/read ergo://<node>/processes?pidStart=0&pidLimit=200
```

The id space is ordered, so `pidStart` with `pidLimit` walks it repeatably: two walks a minute apart
are comparable, which is what makes growth visible. `pidLimit=-1` asks instead for whatever is alive
now, in no order, at the cost of the living rather than of every id the node ever used.

**Hypothesize**

- many rows with tiny uptime: churn, go to playbook 5
- rows with long uptime and no messages at all: spawned and forgotten
- one behavior dominating the count: its spawner
- `ProcessesZombee` above zero: something was killed and its remains are still counted

**Test** who spawns them: `app_tree node=<node> app=<name>` for the shape of the application, or
`process_state` on the supervisor above them, whose restart bookkeeping is in its inspect output.

**Recommend** the missing termination path: a supervisor where there is none, a link so the parent's
death takes the child, or an explicit stop.

## 5. Restart Loop

**Symptoms:** the log repeating the same start line, work not progressing, a supervisor that dies.

**Observe**

```
resources/read ergo://<node>/processes                    # many rows with tiny uptime
resources/read ergo://<node>/log?levels=error,panic
   ... and again with ?since=<NextSeq> to catch the loop turning
```

The log lens starts collecting at the first read, so that answer is near-empty by design. The second
one, with `since=`, is the data.

**Test** the supervisor: `process_state` on it exposes its restart count and its children. A
supervisor that terminated with `gen.ErrExceeded` ("limit exceeded") hit its intensity window, whose
semantics are in `framework-internals.md`.

**Interpret** the pattern rather than the last error. The same reason every time is a broken child;
different reasons every time is an environment broken underneath them all.

**Recommend** fixing the child, or the strategy: intensity, period, or a different restart mode.

## 6. Stuck and Blocked Processes

**Symptoms:** work stops without anything crashing, one caller times out.

**Observe**

```
resources/read ergo://<node>/processes?state=running
```

A process in `running` for tens of seconds is inside a callback. That is either heavy work or a
wait, and from here the two look identical.

**Test**

```
goroutines node=<node> state=<state> minWait=60
```

`minWait` is in seconds, but the node reports whole minutes, so any value below 60 selects what 60
selects. `state` is an exact match on the goroutine state.

**Interpret** a parked goroutine inside a handler as a blocking wait in a callback, which is the one
thing the actor model does not forgive. Without `-tags=pprof` the dump cannot be attributed to a PID
at all, only to a stack, so build the tag into the node you expect to debug.

**Recommend** taking the wait out of the handler: a bounded timeout at worst, a message at best.

## 7. Network Issues

**Symptoms:** cross-node calls failing or slow, a peer flapping.

**Observe both ends.** A one-sided reading cannot see loss:

```
resources/read ergo://<node-a>/connection/<node-b>
resources/read ergo://<node-b>/connection/<node-a>
```

**Interpret**

| Reading | Hypothesis |
|---------|-----------|
| A's messages and bytes out far exceed B's in | Something in the pool is broken, or messages are being dropped |
| Reconnections climbing | An unstable link: the network, or a peer restarting |
| Connection uptime far below both node uptimes | It flapped, and recently |
| Handshake errors above zero on an acceptor | Cookie mismatch, or incompatible versions |
| `Disconnected` present in the reading | It is gone now, and this is the last reading taken |

Acceptor counters live in `ergo://<node>/network` under `Acceptors`, one entry per listening
interface, each with its own handshake error count.

There is no connect, disconnect or ping here: connecting is the framework's business, and what you
observe is whether a connection exists and what crossed it.

**Recommend** by cause: the network, a cookie, a version, or a peer that keeps restarting - which is
playbook 5 on that node.

## 8. Event System Issues

**Symptoms:** a subscriber receives nothing, or a producer publishes into the void.

**Observe**

```
resources/read ergo://<node>/events
resources/read ergo://<node>/event/<name>
```

The `event` lens is the registration: producer, buffering, how many are subscribed. It is not the
traffic.

**Test the traffic** with the stream lens, which is the messages themselves:

```
resources/read ergo://<node>/stream/<name>
   ... and again with ?since=<NextSeq>
```

**Interpret**

- subscribers zero while the producer publishes: fanout into nothing. Nobody linked, or they linked
  on another node and the link did not survive.
- subscribers present and the stream empty: the producer is not publishing. Read the producer.
- the stream carrying messages while the subscriber does nothing: the subscriber's own mailbox and
  state, playbook 2.
- an event that no longer exists: the lens refuses with the reason instead of answering empty.

The fanout mechanics, one network message per node rather than per subscriber, are in
`framework-internals.md`.

## 9. Error Identity Lost Across Nodes

**Symptoms:** `errors.Is` matches locally and stops matching after a hop.

**Observe**

```
errors node=<node-a> text=<part of the message>
errors node=<node-b> text=<part of the message>
```

An error registered for the wire crosses as its identity. One that is not crosses as plain text, and
the far side gets an error that reads the same and matches nothing.

**Interpret** by text, never by the `ID` in the answer: the id is a local cache position, and two
nodes negotiate their own mapping at handshake, so the same sentinel legitimately has different ids
on either side.

**Recommend** registering the sentinel on both sides, which is a code change and not a configuration
one. Registration happens in `init()`; the framework skill's `edf.md` has the rules.

## 10. Wire Size and the Type Registry

**Symptoms:** more bytes on the wire than expected, or a type that will not cross.

**Observe**

```
types node=<node> name=<part of the type name>
types node=<node> kind=struct limit=50
atoms node=<node>
```

**Interpret**

- a type absent from `types` cannot cross a node boundary at all
- an atom absent from `atoms` still crosses, in full, on every hop: that is frame size, never
  correctness
- `MinSize` is the wire size of the zero value including its type tag, and `SizeVariable` says the
  real size can exceed it
- the encode and decode counters read zero unless the node was built with `-tags=typestats`, and the
  answer says which

**Recommend** registering the type, registering the hot atoms, or compression on the producing
process - which `process_tune` changes on the running node.

## 11. Whole-Cluster Sweeps

**Symptoms:** a question only meaningful across nodes: who holds the deep mailboxes, who is on the
old build, who lost a peer.

**One question, many nodes**

```
cluster_query key=mailboxes tool=processes args={"minMailbox":100} nodes=[...]
   -> ergo://job/mailboxes
resources/read ergo://job/mailboxes                  # what has landed, and what is still pending
resources/read ergo://job/mailboxes?since=<NextSeq>  # only what arrived since
```

**Different questions, different nodes**

```
cluster_batch key=deploy-check \
  steps=[{"id":"orders","node":"orders-1@host","tool":"processes","args":{"minMailbox":100}},
         {"id":"billing","node":"billing-1@host","tool":"applications"}]
```

There is no `nodes` argument here: every step names its own `node`, alongside a `tool` and its
`args`. Steps run in parallel rather than as a sequence each node walks through, and each answer
comes back labelled with its `id` - which is why the ids are worth choosing instead of leaving to
the index.

Use this when you already know which node runs which application and want one business flow in one
go. When it is the **same** question over several nodes, `cluster_query` with a `nodes` list is the
one you want.

**Rules**

- `key` is yours, and required. The same key with the same question joins the run in flight; with a
  different question it is refused, because joining would hand you answers you did not ask for.
- `nodes` omitted on `cluster_query` asks every node the observer knows. Name them when the question
  concerns a subset: a sweep costs every node in it.
- `retain` asks how long the finished result is kept, bounded by the deployment's `JobMaxRetention`
  (five minutes by default). Every read pushes the expiry out.
- `job_list` names your runs; `job_cancel key=<key>` stops one, and `force=true` stops it without
  waiting.
- **Never fan out `goroutines` or `heap_profile`.** Each pauses the node it runs on, and a sweep
  would pause the cluster.

## Cross-References

- `tools.md` - the surface: URI grammar, every lens and its parameters, every tool and its arguments
- `counters.md` - what each counter in each reading means
- `watching.md` - accumulating lenses, cursors, keyed watches, subscriptions, trace samplers
- `process-model.md` - states, queues, and the metrics these playbooks read
- `framework-internals.md` - restart intensity, pool backpressure, event fanout, important delivery
- `build-tags.md` - which numbers do not exist without which tag
- the framework skill's `errors.md` and `edf.md` - sentinel identity and wire registration
