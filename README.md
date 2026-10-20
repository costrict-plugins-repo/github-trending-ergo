# Ergo Framework Claude Code Integration

Claude Code agents and skills for building and operating distributed actor-based systems with Ergo Framework (v3.3+).

Two complementary pairs:

- **framework** - designing and implementing actor systems (build-time).
- **devops** - diagnosing running clusters through the observer's MCP surface (runtime).

Each pair is split into an **agent** (behaviour / policy / decision logic) and a **skill** (compact reference data, API surface, playbooks). The agent is slim and decides *what to do*; the skill is loaded progressively and answers *what the code actually says*.

## Contents

```
claude/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── agents/
│   ├── framework-architect.md
│   └── devops.md
└── skills/
    ├── framework/
    │   ├── SKILL.md                       # navigation + critical rules
    │   └── references/
    │       ├── actors.md                  # lifecycle, mailbox, Link/Monitor, Alias, Events, CoreEvent
    │       ├── supervision.md             # types, strategies, intensity, per-child restart, dynamic
    │       ├── messages.md                # Send/Call, important delivery, priority, compression
    │       ├── application.md             # embed app.Application, Load(args), lifecycle, helper API
    │       ├── pool.md                    # Pool, act.Router, act.WebWorker
    │       ├── meta.md                    # TCP/UDP/Web/Port, ProcessPool routing
    │       ├── node.md                    # NodeOptions, NetworkFlags, Security, CertManager/TLS
    │       ├── edf.md                     # Network().RegisterType, schema evolution, visibility
    │       ├── cluster.md                 # etcd/Saturn, Registrar, ResolveApplication, proxy
    │       ├── tracing.md                 # samplers, business spans, exporters, TracingFlags
    │       ├── cron.md                    # gen.Cron, CronJob, cron actions, MessageCron
    │       ├── logging.md                 # gen.Log, LogLevel, structured fields, LoggerBehavior
    │       ├── errors.md                  # gen sentinels, TerminateReason*, errors.Is
    │       ├── testing.md                 # unit/stage/check/mock: Spawn, Should*, injection
    │       ├── actor-lib.md               # actor/health, leader, metrics
    │       ├── applications-lib.md        # application/observer, pulse, radar
    │       ├── meta-lib.md                # meta/websocket, meta/sse
    │       ├── integrations.md            # registrar/etcd, saturn, logger/colored, rotate, sentry
    │       └── erlang-protocol.md         # proto/erlang23 (EPMD, ETF, DIST)
    └── devops/
        ├── SKILL.md                       # navigation + critical rules
        └── references/
            ├── tools.md                   # the surface: URI grammar, lenses, tools, arguments
            ├── process-model.md           # states, mailbox, liveness formula
            ├── counters.md                # every counter of every reading, with meanings
            ├── framework-internals.md     # Important errors, restart intensity, pool, fanout
            ├── playbooks.md               # diagnostic playbooks, symptom by symptom
            ├── watching.md                # accumulating lenses, keyed watches, runs, samplers
            └── build-tags.md              # pprof, latency, verbose, norecover, typestats
```

## How the Split Works

**Agents** (~6 KB each) carry only behavioural rules: trigger cues, diagnostic approach, decision tables, anti-patterns, permission rules. They contain no API signatures.

**Skills** use progressive disclosure - the top-level `SKILL.md` is a small (~3 KB) index with a navigation table plus the five non-negotiable rules. Concrete API details, tool schemas, formulas, and playbooks live in `references/*.md`, each ~150-500 lines and loaded on demand.

A typical task loads:
1. `SKILL.md` (auto) - ~3 KB.
2. One or two relevant `references/*.md` - ~5-10 KB.

Total working-set ≈ 8-13 KB vs ~17 KB monolithic; detail per topic is higher rather than lower.

## Pair Comparison

| | framework | devops |
|--|---|---|
| **Purpose** | Design and implement actor systems | Diagnose running clusters |
| **Invoked when** | Designing an actor architecture, writing a test, choosing a supervisor strategy | Investigating a production issue, reading counters, running `pprof` |
| **Style** | Design doc format + concrete code patterns | Hypothesis-driven investigation + MCP commands |
| **Requires** | Ergo Framework v3.3+ source or module cache | `application/observer` on one node of the cluster |

## Installation

### Easiest: from the Claude Code plugin marketplace

The plugin is already published in the Claude Code marketplace. In Claude Code, open the plugins section (run `/plugin`), search for `ergo`, and install it - no manual setup required.

### From the source repo

Add this repository as a marketplace and install; updates are handled by Claude Code:

```bash
/plugin marketplace add ergo-services/claude
/plugin install ergo@ergo-services
```

After install, agents and skills are namespaced under the plugin - invoke skills as `/ergo:framework` / `/ergo:devops`; agents pick themselves up from trigger phrases.

Update later with `/plugin update`, remove with `/plugin uninstall ergo@ergo-services`.

### Manual (development / fork / offline)

If you are editing the agents and skills locally, or running without network access, link or copy the files directly.

#### Symlink into `~/.claude`

```bash
cd path/to/ergo.services/claude

mkdir -p ~/.claude/agents ~/.claude/skills

# Agents
ln -sf $(pwd)/agents/framework-architect.md ~/.claude/agents/
ln -sf $(pwd)/agents/devops.md              ~/.claude/agents/

# Skills (directories - references/ is picked up transitively)
ln -sf $(pwd)/skills/framework ~/.claude/skills/
ln -sf $(pwd)/skills/devops    ~/.claude/skills/
```

#### Copy

```bash
cp -r agents/* ~/.claude/agents/
cp -r skills/* ~/.claude/skills/
```

## Usage

### framework-architect (Agent)

Designs Ergo Framework applications with DDD bounded contexts, supervision trees, and cluster topology. Outputs an implementable design document.

**Trigger phrases**
- "design ergo application"
- "ergo architecture"
- "create ergo design document"
- "actor system design"

**Output** - a design document with bounded context, cluster topology, supervision tree, data structures, message flow, load analysis, and implementation phases.

### devops (Agent)

Connects to a running cluster through the observer's MCP surface and runs hypothesis-driven investigations. Never mutates state without explicit user permission.

**Trigger phrases**
- "why is it slow"
- "find process leak"
- "check cluster health"
- "debug this node"
- "monitor for anomalies"

**Capabilities**
- Performance bottleneck investigation (mailbox depth, latency, drain ratio, liveness score).
- Process leak and zombie detection.
- Restart loop analysis.
- Memory growth investigation (heap profiling, GC pressure).
- Network connectivity and traffic analysis (both sides of every connection).
- Event / pub-sub diagnostics (fanout, publishing-to-void, starved subscribers).
- Goroutine debugging (deadlocks, leaks, per-process stack traces).
- Watching over time: accumulating lenses read with a cursor, keyed watches, subscriptions.
- Cluster-wide queries as runs: one question put to many nodes at once.

**Requires** - `ergo.services/application/observer` on one node. The nodes it inspects need nothing: they already run the built-in `system` application it asks.

### framework (Skill)

Reference for implementing Ergo Framework applications. Load via `/ergo:framework` or automatically when the topic matches. Follow the navigation table in `SKILL.md` to pull in only the needed `references/*.md`.

### devops (Skill)

Reference for diagnosing live Ergo nodes. Load via `/ergo:devops`. Navigation table lists 7 topic files - the surface catalog, process model, counters, watching over time, playbooks, framework internals, build tags.

## MCP Surface Setup

The `devops` pair works through the MCP surface of the Observer application:

```go
import "ergo.services/application/observer"

node, _ := ergo.StartNode("mynode@host", gen.NodeOptions{
    Applications: []gen.ApplicationBehavior{
        observer.CreateApp(observer.Options{
            Port:    9911,                              // default; serves UI, API and MCP
            Ceiling: observer.Ceiling{ReadOnly: true},  // production: refuse the manage. plane
        }),
    },
})
```

Connect from Claude Code:

```bash
claude mcp add --transport http ergo http://localhost:9911/mcp
# or through a proxy that authenticates the caller:
claude mcp add --transport http ergo http://localhost:9911/mcp \
    --header "Authorization: Bearer ${TOKEN}"
```

Readings are resources addressed as `ergo://<node>/<lens>`, and every tool takes a `node` argument. Both take the node name from `ergo://cluster`, which is where an investigation starts. The node serving MCP holds no privileged position: name the target explicitly, including that one.

## Build Tags for Better Diagnostics

| Tag | Effect |
|-----|--------|
| `-tags=pprof` | Per-process goroutine labels + pprof HTTP server at `localhost:9009` |
| `-tags=latency` | Enables `MailboxLatency` measurement (required for liveness score) |
| `-tags=verbose` | Verbose framework-internal logging |
| `-tags=norecover` | **Disables** panic recovery (debug only) |
| `-tags=typestats` | Per-type EDF encode/decode counters (`gen.RegisteredTypeStats`) |

See `skills/devops/references/build-tags.md` for detail.

## Reference

When a signature or constant must be exact, verify against the framework source:

```bash
ls $(go env GOMODCACHE)/ergo.services/ergo@*/version.go
ls $(go env GOMODCACHE)/ergo.services/ergo@*/docs/
```

## Requirements

- Claude Code CLI
- Ergo Framework v3.3+
- For `devops`: `ergo.services/application/observer` on one node, which reaches the rest of the cluster
