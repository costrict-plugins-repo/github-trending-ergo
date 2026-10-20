# The Surface

Everything the MCP surface of Observer offers is either a **resource** you read by URI or a **tool**
you call by name: 13 lenses and 38 tools.

A resource is a reading with an address. It can be read again, and read again with a cursor to take
only what has changed. A tool is an act: it takes arguments, does one thing, answers once.

Every tool takes `node`, naming the node to ask, and every URI carries that name too. Both take it
from `ergo://cluster`. The node serving MCP is not privileged: name it explicitly like any other.

## Resource URIs

```
ergo://cluster                                  every node this observer knows
ergo://<node>                                   the node lens (the default when no lens is given)
ergo://<node>/<lens>                            a lens with no target
ergo://<node>/<lens>/<target>                   process, meta, connection, event, stream
ergo://<node>/<lens>?<params>                   filters and paging
ergo://<node>/watch/<key>/<lens>                the same lens with a cursor of your own
ergo://job/<key>                                a run started by cluster_query or cluster_batch
```

`since=` is reserved on every URI. `cluster`, `job` and `watch` are reserved words and no lens may
be called any of them. A target is percent-escaped, and `<node>@host:ID.Creation` is accepted where
`ID.Creation` alone would do.

`resources/list` holds exactly one entry, `ergo://cluster`. Everything else is a template:
`resources/templates/list` announces one per lens, with its parameters.

## The lenses

| Lens | URI | Answers |
|------|-----|---------|
| `node` | `ergo://<node>` | Uptime, memory, processes, applications, connections, version, environment |
| `processes` | `ergo://<node>/processes` | Every process: identity, what it runs under, what it handled, its state |
| `process` | `ergo://<node>/process/{pid}` | One process: mailbox, links, monitors, aliases, events, env, metas, and its own inspect output |
| `meta` | `ergo://<node>/meta/{alias}` | One meta process: a socket, a listener, a stream |
| `network` | `ergo://<node>/network` | The network as configured and as running, including acceptors |
| `connections` | `ergo://<node>/connections` | Every peer, with what was negotiated |
| `connection` | `ergo://<node>/connection/{+peer}` | One peer: flags, pool, counters each way |
| `applications` | `ergo://<node>/applications` | Applications loaded, running or not, with mode and group |
| `events` | `ergo://<node>/events` | Every event registered: producer, subscribers, counters |
| `event` | `ergo://<node>/event/{name}` | One event's registration, not its traffic |
| `stream` | `ergo://<node>/stream/{name}` | The messages flowing through one event **(accumulates)** |
| `log` | `ergo://<node>/log` | The lines the node logs **(accumulates)** |
| `tracing` | `ergo://<node>/tracing` | The spans the node emits while tracing is on **(accumulates)** |

### Parameters

| Lens | Query parameters |
|------|------------------|
| `processes` | `namePattern`, `behavior`, `application`, `state`, `minMailbox`, `pidStart`, `pidLimit` |
| `connections` | `limit`, `namePattern` |
| `events` | `namePattern`, `limit`, `timestamp`, `notifyMode`, `bufferedMode`, `openMode`, `minSubscribers` |
| `stream` | `limit`, `typePattern`, `messagePattern`, `messageExclude`, `force`, `verbose`, `since` |
| `log` | `levels`, `limit`, `messagePattern`, `messageExclude`, `since` |
| `tracing` | `limit`, `kinds`, `points`, `messagePattern`, `messageExclude`, `since` |
| the rest | none |

`pidStart` with `pidLimit` walks the id space in order, which makes a page repeatable. `pidLimit=-1`
asks instead for whatever is alive now, in no order, at the cost of the living rather than of every
id the node ever used.

`levels` is a comma list drawn from `debug`, `info`, `warning`, `error`, `panic`.

### The three that accumulate

`stream`, `log` and `tracing` start collecting at the **first read**, so that answer is near-empty by
design. Read again with `since=<NextSeq>` for what landed in between. `NextSeq` is a string of the
form `<epoch>.<seq>`, taken from the previous answer, and `Dropped: true` means the cursor went
stale and what fell in between is gone.

They gather only while watched: a shared reading idles out after 30 seconds, a keyed one after 10
minutes. Details and the patterns built on them are in `watching.md`.

## Tools

`node`, `network`, `connections`, `connection`, `applications`, `events` and `event` answer the same
data as their lenses. Use whichever fits the step: a tool inside a `cluster_query`, a resource when
you intend to read it again.

### About the node

| Tool | Required | Arguments |
|------|----------|-----------|
| `node` | `node` | none |
| `network` | `node` | none |
| `connections` | `node` | `peer` (substring), `limit` |
| `connection` | `node`, `peer` | `peer` |
| `applications` | `node` | none |
| `events` | `node` | `name`, `limit`, `newestFirst`, `notify`, `buffered`, `open`, `minSubscribers` |
| `event` | `node`, `name` | `name` |
| `capabilities` | `node` | none |

`capabilities` answers what this node allows **this caller**, already narrowed by every ceiling in
between. Ask it before planning anything that writes.

### Processes

| Tool | Required | Arguments |
|------|----------|-----------|
| `processes` | `node` | `app`, `name` (substring), `behavior`, `state`, `minMailbox`, `limit`, `scan`, `start` |
| `process_state` | `node`, `pid` | `pid`, `items` |
| `process_lookup` | `node` | `name`, `pid` |
| `app_tree` | `node`, `app` | `app`, `limit` |
| `subtree` | `node`, `pid` | `pid`, `limit` |
| `meta_state` | `node`, `alias` | `alias`, `items` |

`state` is one of `init`, `sleep`, `running`, `wait response`, `terminated`, `zombee` (note the space
in "wait response"). There is no `sort_by` anywhere on this surface: sort the rows in the reader.

`process_state` is the process's own `HandleInspect` map; `items=["help"]` asks which sections it
offers. `meta_state` is the same for a meta process, and the alias comes from the `Metas` field of
`ergo://<node>/process/<pid>`.

`process_lookup` resolves a name to a PID or the reverse, and answers `PID`, `Name`, `State`.

### The wire

| Tool | Required | Arguments |
|------|----------|-----------|
| `types` | `node` | `name` (substring), `kind`, `limit` (default 50) |
| `errors` | `node` | `text` (substring), `limit` |
| `atoms` | `node` | `name` (substring), `limit` |

A type absent from `types` cannot cross a node boundary. An error absent from `errors` crosses as
plain text, so `errors.Is` stops matching on the far side. An atom absent from `atoms` still crosses,
in full, every hop: frame size, never correctness.

The `ID` in the answers of `errors` and `atoms` is a local cache position. Two nodes negotiate their
own mapping at handshake, so compare by text and by name, never by id.

The encode and decode counters in `types` read zero unless the node was built with `-tags=typestats`,
and the answer says which.

### Expensive

| Tool | Required | Arguments |
|------|----------|-----------|
| `goroutines` | `node` | `stack` (substring in the stack text), `state` (exact), `minWait` |
| `heap_profile` | `node` | `minBytes`, `limit` (default 25) |

**Both stop the world** on the node they run on, for as long as the walk takes. Once, filtered,
never in a loop and never fanned out across a cluster.

`minWait` is in seconds, but the node reports whole minutes, so any value below 60 selects what 60
selects. `heap_profile` answers largest first, and its `TotalInuse`, `TotalAlloc` and `TotalObjects`
describe the whole profile rather than the page.

Without `-tags=pprof` a goroutine cannot be attributed to a process at all, only to a stack.

### Scheduling and discovery

| Tool | Required | Arguments |
|------|----------|-----------|
| `cron` | `node` | `job` |
| `cron_schedule` | `node` | `job`, `since`, `hours`, `limit` |
| `registrar_nodes` | `node` | none |
| `registrar_routes` | `node`, `peer` | `peer` |
| `registrar_proxy_routes` | `node`, `peer` | `peer` |
| `registrar_application_routes` | `node`, `name` | `name` |

### Many nodes

| Tool | Required | Arguments |
|------|----------|-----------|
| `cluster_query` | `key`, `tool` | `key`, `tool`, `args`, `nodes`, `retain` |
| `cluster_batch` | `key`, `steps` | `key`, `steps`, `retain` |
| `job_list` | none | none |
| `job_cancel` | `key` | `key`, `force` |

Note what is **not** required: `node`. These two are about many of them, and `nodes` omitted means
every node the observer knows.

Both answer at once with `ergo://job/<key>`, before the nodes have reported. Read that URI, then read
it again with `since=`. `steps` is a list of `{"id": ..., "tool": ..., "args": {...}}`, and each node
walks the whole sequence.

The `key` is yours and required. The same key with the same question joins the run in flight; with a
different question it is refused. `retain` asks how long a finished result is kept, bounded by the
deployment's `JobMaxRetention`, and every read pushes the expiry out.

A run belongs to the caller that started it, so on an authorized listener nobody else can read or
cancel it. Runs are bounded by `JobLimit` over every caller.

### Changing the node

Every one of these is refused on a read-only surface, with the capability that refused it named in
the answer.

| Tool | Required | Arguments | Capability |
|------|----------|-----------|------------|
| `send` | `node`, `type`, `value` | `pid` or `alias`, `type`, `value`, `priority` | `manage.send`, `manage.send_meta` |
| `send_exit` | `node` | `pid` or `alias`, `reason` | `manage.send_exit`, `manage.send_exit_meta` |
| `kill` | `node`, `pid` | `pid` | `manage.kill` |
| `log_level_set` | `node`, `level` | `level`, `target`, `pid`, `alias` | `manage.set_log_level`, and the process or meta variants |
| `tracing_sampler_set` | `node`, `type` | `type`, `rate`, `limit`, `pid` | `manage.set_node_tracing_sampler`, `manage.set_process_tracing_sampler` |
| `process_tune` | `node`, `knob` | `knob`, `pid` or `alias`, and the field the knob reads | seven `manage.set_process_*` plus `manage.set_meta_send_priority` |
| `app_start` | `node`, `name` | `name`, `mode` | `manage.app_start` |
| `app_stop` | `node`, `name` | `name`, `force` | `manage.app_stop` |
| `app_unload` | `node`, `name` | `name` | `manage.app_unload` |

**`send`** carries a typed value, not JSON: `type` names it and `value` is always given as text. The
types are `bool`, `string`, `atom`, `binary`, `int`, `int8`, `int16`, `int32`, `int64`, `uint`,
`uint8`, `uint16`, `uint32`, `uint64`, `float32`, `float64`, `time`, `pid`, `process_id`, `alias`,
`ref`, `event`. A struct cannot be constructed this way: a receiver expecting one will not get it
from here. `priority` is `normal`, `high` or `max`.

**`log_level_set`** changes the node unless `target` says `process` or `meta`, in which case `pid` or
`alias` names it.

**`tracing_sampler_set`** takes `type` from `disable`, `always`, `ratio`, `rate_limit`; `rate` is
read for `ratio` (0.1 is one in ten, deterministically every tenth), `limit` for `rate_limit`
(spans per second), and `pid` narrows the change from the node to one process. Sampling makes the
node do work it was not doing: turn it off afterwards.

**`process_tune`** changes one setting at a time. Name the knob, then fill the field it reads:

| `knob` | Field it reads | Values |
|--------|----------------|--------|
| `send_priority` | `priority` | `normal`, `high`, `max` |
| `compression` | `enabled` | boolean |
| `compression_type` | `type` | `gzip`, `zlib`, `lzw` |
| `compression_level` | `level` | `best size`, `best speed`, `default` |
| `compression_threshold` | `threshold` | bytes |
| `keep_network_order` | `enabled` | boolean |
| `important_delivery` | `enabled` | boolean |

`alias` is accepted for `send_priority` only, which is the one knob a meta process has.

**`app_start`** takes `mode` from `temporary`, `transient`, `permanent`; `app_stop` takes `force`
when a graceful stop will not complete. System applications are protected from all three.

## Reading any answer

**The legend.** An answer carrying `services.ergo/legend` states the unit of a field, the meaning of
a sentinel value, and the axis of a counter array. Each key is the path from the object holding the
legend, with `[]` for a step into a list. Read it rather than assuming: `-1` in a latency field means
the node lacks `-tags=latency`.

**Refusals.** A reading that cannot be taken names what could not be read and why. An empty list
means empty; it never means "gone" or "not permitted".

**Identifiers are text.** A PID reaches you as `<node>@host:ID.Creation`, an alias as
`<node>@host:ID.ID.ID.Creation`, and either form is accepted back, as is the short form without the
node when the node is already an argument.

## Transport

POST-only, following the 2026-07-28 revision: every request carries the protocol version, states
what the client accepts, and names its method in a header. A compliant client does this. If a
hand-made request is refused with `-32020`, the message names the header that is missing.

`subscriptions/listen` is the long-lived POST that follows resources and delivers
`notifications/resources/updated`; see `watching.md`.

The tool and resource listings, and the discovery answer, are cacheable for as long as the
deployment's `CacheTTL` says (five minutes by default). Nothing in them changes inside one process,
but a client that outlived a restart may be calling tools with the arguments of the binary it first
met.
