# Watching Over Time

A single reading answers "what is it now". Most real symptoms are not visible in one reading: a
mailbox that grows and drains, a process that wakes every few seconds, an error that happens once a
minute. This file is how the observer surface covers time.

There is no server-side sampler process. Nothing on the node will call a tool for you on a timer and
keep a ring buffer. Four mechanisms cover what that used to do, and each covers a different case.

| Mechanism | Use it for |
|-----------|------------|
| Accumulating lens + `since=` | Everything the node emits: log lines, event messages, tracing spans |
| Keyed watch | Two investigations following the same thing without stealing each other's position |
| `subscriptions/listen` | Being told when a resource changes instead of asking again |
| A run read repeatedly | One question put to many nodes, answers arriving as nodes report |

For a counter that only exists as a snapshot (mailbox depth, memory, process count), the loop is
yours: read the resource, wait, read it again, compare. Keep the interval honest - a second is
plenty for a mailbox, ten for memory - and stop when the question is answered.

## Accumulating lenses

Three lenses gather rather than answer once:

| Lens | URI | Gathers |
|------|-----|---------|
| `log` | `ergo://<node>/log` | The lines the node logs |
| `stream` | `ergo://<node>/stream/{name}` | The messages flowing through one event |
| `tracing` | `ergo://<node>/tracing` | The spans the node emits while tracing is on |

They start collecting when the lens is **first read**, not when the node starts. The first read is
therefore almost always near-empty, and that is not a fault: it is the subscription being
established. Read again to get what arrived in between.

```
resources/read ergo://orders-1@host/log?levels=error,panic
   -> the lens starts, answers what it has (usually nothing)
   ... the symptom happens ...
resources/read ergo://orders-1@host/log?levels=error,panic&since=<cursor>
   -> only the lines that landed after the previous read
```

`since=` takes the `NextSeq` of the previous answer, a string of the form `<epoch>.<seq>` and not an
integer you can compute. If the answer comes back with `Dropped: true`, the cursor is stale: the
node moved on past what the buffer holds, so what fell in between is gone and the reading starts
from where it now is. Without `since=` a read answers the whole buffer again, which wastes context
and makes new lines indistinguishable from old.

They gather only while they are being watched, and they stop when nobody watches. A shared reading
idles out after **30 seconds**, a keyed one after **10 minutes**. After that the next read starts a
fresh collection, so a gap in the reading is a gap in the data, not a backlog waiting to be
collected.

Filters belong in the URI and are applied on the observed node, not in the agent: `levels`,
`messagePattern`, `messageExclude`, `limit` for `log`, and the equivalents per lens are announced in
the resource templates. Filtering at the source is the difference between reading a hundred lines
and reading a hundred thousand.

## Keyed watches

`ergo://<node>/watch/<key>/<lens>` is the same lens with its own cursor, held under a name you
choose:

```
ergo://orders-1@host/watch/deploy-check/log?levels=error
ergo://orders-1@host/watch/mailbox-hunt/stream/orders
```

The point is position, not data. Two investigations reading the shared lens consume one cursor
between them, so each read gives the other's leftovers. A keyed read gives each its own.

A key belongs to a caller, so a keyed reading needs a listener with an authorizer. On an open
listener the surface answers that it needs a caller instead of silently sharing a cursor with
everybody.

The key also buys patience: a keyed reading survives 10 minutes of not being read, against 30
seconds for a shared one. Reuse one key per investigation rather than one per read.

## Being told instead of asking

`subscriptions/listen` is a long-lived POST that follows a set of resources and delivers
`notifications/resources/updated` as they change.

Use it when the wait is open-ended: watching a deploy settle, waiting for a restart loop to fire
again, watching a mailbox drain. Use a read loop instead when the question has a horizon of a few
seconds.

The stream tells you what it accepted before anything else, with
`notifications/subscriptions/acknowledged` naming what it is following. If a resource stops being
available - the event unregistered, the node gone - the stream says so and re-acknowledges what is
left, and closes when nothing remains. Cancellation arrives as `notifications/cancelled`.

Each open stream costs one of the listener's `MaxStreams`, and each subscription inside it costs a
producer on the observed node, bounded by `MaxSubscriptions`. Close what you are done with.

## Runs, for many nodes over time

`cluster_query` and `cluster_batch` answer immediately with `ergo://job/<key>`, before the nodes have
reported. Reading that URI is therefore a time-based act too:

```
cluster_query  nodes=[...] tool=processes arguments={...}
   -> ergo://job/deploy-check
resources/read ergo://job/deploy-check              -> what has landed so far, and what is pending
resources/read ergo://job/deploy-check?since=<cur>  -> only what landed since
```

A slow node does not hold up the others, and an unreachable one is reported as refused rather than
waited for. `job_list` shows your runs, `job_cancel` stops one. Runs expire on their own
(`JobMaxRetention`, five minutes by default), and every read of a finished run pushes the expiry
out, so a run you keep reading stays readable.

The key is yours. Asking the same key the same question joins the run in progress; asking the same
key a different question is refused, because joining it would hand you answers to something you did
not ask.

## What has no equivalent, and what to do instead

| The old habit | Now |
|---------------|-----|
| An active sampler calling a tool every N ms | Your own read loop, or an accumulating lens where the data is a stream |
| A passive sampler subscribed to logs or events | `ergo://<node>/log`, `ergo://<node>/stream/{name}`, read with `since=` |
| A ring buffer read after the fact | The lens buffer, read with `since=`; it holds what arrived while watched |
| A proxy sampler running on a remote node | The lens already reads the remote node: the URI names it |
| `sample_list` / `sample_stop` | `job_list` / `job_cancel` for runs; a lens needs no stopping |

The one genuine loss is unattended collection over a long window: nothing keeps gathering while no
client is watching, except a run that is already in flight. For a symptom that happens once an hour,
hold a `subscriptions/listen` stream open rather than sampling.

## Trace samplers

Different concept, same word. A **trace sampler** decides, per outgoing `Send`/`Call`, whether that
message starts a new distributed trace. It is unrelated to watching a lens. The full model -
observation points, attributes, business spans, exporters, cross-node propagation - lives in the
framework skill's `tracing.md`. This section is the operator angle.

The key fact: **by default no process traces anything.** The default sampler is
`gen.TracingSamplerDisable`, so a node can have exporters wired and still emit zero spans. Traces
are born at the sampler, and the sampler is consulted only when no trace is already propagating, so
enabling it on one entry-point process lights up the whole chain that process kicks off.

| Sampler | Behavior |
|---------|----------|
| `disable` | Never starts a trace. The default for every process and node. |
| `always` | Starts a trace for every outgoing message. |
| `ratio` | A fixed-point accumulator, not a random draw: it adds the rate on every call and fires when the integer part crosses. `ratio 0.01` gives gaps of 100 messages with an occasional 101 where the rounding is corrected - deterministic and close to the rate, but not a strict every-Nth. |
| `rate_limit` | Token bucket capped per whole Unix second, refilled at each second boundary. Use it to protect the exporter under bursty load. |

From the MCP surface, `tracing_sampler_set` installs one:

```
tracing_sampler_set node=<node> type=always pid=<pid>   # one process
tracing_sampler_set node=<node> type=ratio rate=0.1     # the node's own Send/Call
tracing_sampler_set node=<node> type=disable pid=<pid>  # turn it back off
```

`type` is required, `rate` is read for `ratio`, `limit` for `rate_limit`, and `pid` narrows the
change from the node to one process. It is a mutating tool under `manage.set_node_tracing_sampler`
and `manage.set_process_tracing_sampler`, refused on a read-only surface. Sampling makes the node do
work it was not doing: turn it off when the reproduction is over.

Confirm spans are actually flowing before trusting an empty trace view:

- the `node` lens carries `TracingSpans`, a counter array whose axis the legend names
  (`send,request,response,spawn,terminate`); it increments as observations are recorded
- the `connection` lens counts messages carrying trace context across a link, which is what
  confirms cross-node propagation (that also needs `EnableTracing` in `NetworkFlags` on both sides)

If the counter stays flat after a non-`disable` sampler is installed, the process you pinned is not
the one originating the messages you care about, or no exporter is registered.
