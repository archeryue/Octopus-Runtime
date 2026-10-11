# Octopus-Runtime — design

> **Status:** draft for discussion (2026-10-11). Nothing here is built. The
> interface this implements is `server/runtime/base.py` in Octopus
> (`PROTOCOL_VERSION = "runtime-1.0"`), and the conformance suite there
> (`server/runtime/conformance.py`) is the contract.

## 1. What this is

A platform that turns bare-metal servers into one **lightweight VM per Octopus
account**, plus the Octopus provider that runs an account's work inside its VM.
Octopus decides *when* an account needs its machine; this decides *where* the
machine is, makes it start fast, keeps its disk, and shows an operator what is
going on.

It exists for one reason: on a shared host every Octopus account's agent runs as
the same OS user, so any tenant can read any other tenant's files and
credentials (`multi-tenancy.md` §2, isolation level A). A VM per tenant is the
boundary that fixes that — a separate kernel, not a separate directory.

What it is not:

- **Not required for Octopus.** `LocalRuntime` lives in Octopus, is the default,
  and needs nothing from this repository. Octopus never imports this code; this
  code depends on Octopus's protocol, never the reverse.
- **Not a general sandbox platform.** The workload is Octopus's, and the API is
  shaped by Octopus's protocol rather than by "run any container".

## 2. Technology choice

**Kata Containers on containerd, on bare metal with KVM.**

Kata runs each container sandbox inside its own small VM, with its own guest
kernel, while keeping the container workflow — images, containerd, a standard
runtime shim. That combination is the point: a VM's isolation with a
container's operational model.

| Option | Isolation | Why not (or why) |
|---|---|---|
| Plain containers (runc) | shared kernel | One kernel bug from another tenant's files. Not a boundary for hostile code, and an agent with a shell is arbitrary code. |
| gVisor | user-space kernel | Strong, but syscall compatibility and I/O performance matter for an agent running builds, `git`, `npm`, test suites. |
| Raw Firecracker / Cloud Hypervisor | VM | Same isolation as Kata, but we would rebuild image handling, networking and the guest agent ourselves. |
| **Kata Containers** | **VM** | VM boundary, reuses containerd, images and CNI; the hypervisor underneath is a config choice. |

**Hypervisor under Kata:** Cloud Hypervisor first. It supports `virtio-fs`
(sharing a host directory into the guest), which keeps the storage design open
(§5). Firecracker boots faster and is smaller, but cannot share a filesystem —
Kata on Firecracker needs a block-device snapshotter (devmapper) — so it is the
option to measure later, not the default.

**No Kubernetes in the first version.** On one or a handful of nodes,
containerd + Kata + our own control plane is less machinery than a cluster, and
every capability we need (§4) maps onto containerd directly. Kubernetes (k3s
with a Kata `RuntimeClass`) becomes worth it when scheduling across many nodes
is the hard problem; the Octopus-facing protocol does not change when it does.

**Bare metal, not cloud VMs.** Kata needs KVM. Cloud instances only offer it as
nested virtualization (slow, often unavailable) or as expensive `.metal`
instances; a dedicated or home server gives it for free.

## 3. Shape

```
             ┌────────────────────────── Octopus server ─────────────────────────┐
             │  sessions · schedules · Applications · MCP namespaces · DB        │
             │  runtime_provider = "octopus_runtime.provider:KataRuntime"        │
             └───────┬──────────────────────────────▲────────────────────────────┘
      lifecycle,     │ (1) control API              │ (3) callbacks: MCP tools,
      expose         │                              │     Application agent API
             ┌───────▼────────── control plane ─────┴──┐
             │  tenants · placement · volumes · ports  │◄── admin console (§8)
             │  quotas · events · metrics              │
             └───────┬─────────────────────────────────┘
                     │ node API
   ┌─────────────────▼──────────────── bare-metal node ────────────────────────┐
   │  containerd + kata-shim + Cloud Hypervisor           node daemon           │
   │  ┌─── tenant A VM ───────────┐  ┌─── tenant B VM ───────────┐              │
   │  │ octopus-runtime-agent     │  │ octopus-runtime-agent     │   ...        │
   │  │ claude / codex / git / sh │  │ claude / codex / git / sh │              │
   │  │ /home/tenant  ◄─ volume A │  │ /home/tenant  ◄─ volume B │              │
   │  └───────────▲───────────────┘  └───────────────────────────┘              │
   └──────────────┼─────────────────────────────────────────────────────────────┘
                  │ (2) exec data path: Octopus ⇄ agent directly
```

Four components, all in this repository:

1. **`octopus-runtime-agent`** — a small static binary that is the main process
   of every tenant VM. It implements `exec` (§4): spawn, stream, write stdin,
   signal a process group, report exit. Nothing else runs it and nothing else
   is listening in the VM by default.
2. **Node daemon** — one per bare-metal node. Drives containerd (create / start
   / stop a tenant's sandbox), attaches the tenant's volume, sets up the
   network, reports usage.
3. **Control plane** — the one service Octopus talks to. Knows every tenant,
   which node holds it, its volume, its exposed ports, its quota and its state;
   places new tenants on nodes; serves the admin console.
4. **The provider** — `octopus_runtime.provider:KataRuntime`, a Python class
   implementing Octopus's `Runtime` protocol by calling the control plane for
   lifecycle and talking to the tenant's agent for execution. Octopus loads it
   with `OCTOPUS_RUNTIME_PROVIDER=octopus_runtime.provider:KataRuntime`.

**Control path and data path are separate.** `ensure`, `release` and `expose`
go through the control plane. An `exec` asks the control plane once where the
tenant's agent is (address + short-lived credential) and then streams directly
to the agent. A turn's stdout is the hottest traffic in the system; routing it
through the control plane would make the control plane the bottleneck and a
single point of failure for every running turn.

## 4. The protocol, mapped

| Protocol call | What happens here |
|---|---|
| `ensure(tenant)` | Control plane: if the tenant's VM is running, return. Otherwise pick its node (the one with its volume, §5), create the sandbox, attach the volume, wait for the agent to answer. Idempotent; concurrent calls for one tenant share one start. |
| `release(tenant)` | Stop the VM. **Keep the volume.** Octopus calls this only when the tenant has nothing running, after its grace period. |
| `exec(spec)` | Provider → agent: start `argv` with exactly `spec.env` (nothing added), `cwd`, the stdin/stderr modes, its own process group if asked. Streams are a multiplexed connection (stdin, stdout, stderr, signal, exit) over one WebSocket or gRPC stream. |
| `Execution.signal(sig, group)` | A frame on that stream; the agent does `killpg` inside the VM. Returns as soon as the frame is sent. |
| `expose(tenant)` | Control plane allocates a port inside the VM and a host-side forward; returns `(port, "node-ip:host-port")`. |
| `callback_base(tenant)` | The address of the Octopus server **as reachable from the tenant's VM** — a fixed internal address routed to Octopus, never the operator's loopback. |
| `paths(tenant)` | The tenant's directories *inside its VM*: `/home/tenant/{workspace,agents,applications,…}`. |
| `confine(requested, tenant)` | Structural: anything that resolves inside `/home/tenant` is allowed; the VM has nothing else of the tenant's to protect. `extra_roots` is ignored. |

Why our own agent instead of `containerd exec` (which Kata supports through its
built-in guest agent): Octopus depends on two behaviours that the generic exec
path does not give cleanly —

- **process-group signalling** — interrupting a turn must kill the CLI *and*
  every MCP child and sub-agent it started;
- **held executions** — a Claude process stays alive between turns, idle, with
  stdin open, and is written to again minutes later.

An agent we own implements exactly the protocol's semantics, is tested by the
same conformance suite, and is not tied to Kata: if the hypervisor layer
changes, the agent does not.

## 5. Storage: the tenant's disk is the product

Everything a tenant has lives on one volume, mounted at `/home/tenant`:
workspace, agent memory, Application code and data, uploads, **and the CLIs'
own state** — Claude's and Codex's logins and transcripts. Losing it, or
starting a VM without it, signs the tenant out of everything (`runtime-providers.md`
§5). The conformance case `state_survives_release` exists for this.

Proposal: **one LVM thin volume (or ZFS zvol) per tenant on the node's local
NVMe**, attached to the VM as a block device.

- Quotas are the volume size — no per-directory accounting.
- Snapshots are cheap and atomic → backups and "restore my account".
- Works with every hypervisor, including Firecracker.

The cost: a tenant is **pinned to the node holding its volume**. Moving a
tenant means copying its volume. That is fine at a few nodes; network storage
(Ceph RBD) removes the pin and adds a distributed system to operate — decide
when one node is not enough (§10).

## 6. Network

| Direction | Rule |
|---|---|
| Tenant VM → internet | Allowed. The CLIs call model APIs; agents fetch packages and web pages. Optionally rate-limited per tenant. |
| Tenant VM → Octopus | Only the callback address (§4): MCP namespaces and the Application agent API. Every call is authenticated by Octopus's own per-session bearer, as today. |
| Tenant VM → tenant VM | **Denied.** |
| Tenant VM → node / control plane / metadata | Denied. |
| Octopus → tenant VM | The agent's port (exec) and exposed Application ports, only from Octopus's address. |

Each VM sits in its own network namespace (Kata wires the VM's interface into
it); rules are nftables on the node, generated by the node daemon from the
control plane's state.

## 7. Lifecycle and capacity

- **Octopus decides when.** `ensure` before every execution, `release` after
  the tenant has had nothing running for `runtime_release_grace_seconds`
  (default 5 min). The control plane does not run its own idle timer.
- **Cold start is on the critical path only for an idle tenant.** A scheduled
  fire for a released tenant pays it; an active conversation never does.
  Target: VM + agent ready in ≈1–2 s — to be measured, not assumed. VM
  snapshot/restore (Firecracker and Cloud Hypervisor both support it) is the
  lever if it is too slow.
- **Sizing.** Default 4 vCPU / 8 GB, a larger tier at 8 / 16. vCPUs oversubscribe
  well (agents are mostly waiting on a model). Memory does not oversubscribe
  for free: a VM's memory is reserved by the hypervisor unless returned via a
  balloon / free-page reporting. How far to overcommit memory is the main
  capacity question (§10).
- **Hard limits** the control plane enforces regardless of Octopus: max running
  VMs per node, per-tenant disk quota, a ceiling on how long a VM may run with
  no exec (catches a lost `release`).

## 8. Admin console

An operator-facing web UI served by the control plane — separate from Octopus's
own Monitor page, which is about one install, not the fleet.

- **Tenants** — every account: state (running / released / starting / failed),
  node, size tier, volume usage, last active, running executions (turns,
  background tasks, Application backends).
- **Nodes** — each bare-metal machine: VMs running, CPU / memory / disk
  committed vs. physical, health.
- **Events** — starts, releases, failed starts, OOM kills, quota hits, with the
  agent's last output for a failure.
- **Actions** — stop a tenant's VM, resize, snapshot / restore its volume, drain
  a node.
- **Metrics** — start latency, VM count, memory overcommit ratio over time.

Its own operator authentication; no tenant ever reaches it.

## 9. What Octopus must change before this runs

Steps 1–7 of `runtime-providers.md` §10 are done: every process goes through
`exec`, and Octopus never assumes a port or a callback address. Three gaps
remain, and each one breaks this provider immediately:

1. **Server-side file access** (`runtime-providers.md` §9). Octopus still reads
   and writes tenant files with `Path` on its own disk: `/showme`, uploads,
   large-prompt spills, agent memory provisioning, **Applications' static files**
   (served by Octopus from the app directory), and the full-copy fork. On this
   provider those files are on the tenant's volume, inside a VM. The protocol
   needs file operations (read / write / list / stat, streamed) — a
   `runtime-1.1` — and the agent implements them. This is the largest piece of
   work on the Octopus side and the first thing to design together.
2. **The environment allowlist** (step 8). Today Octopus still passes its own
   environment to the CLIs — correct on one box, but it would ship the
   operator's `OCTOPUS_AUTH_TOKEN` into a tenant's VM.
3. **Fork as a runtime step** (step 9) — the transcript copy runs inside the VM.

## 10. Open questions — for discussion

1. **Kata vs. driving Cloud Hypervisor / Firecracker directly.** Kata gives us
   images, CNI and containerd; it also adds a layer whose behaviour we must
   learn. Is the container workflow worth it for one long-lived VM per tenant?
2. **Memory overcommit.** How much, with which mechanism (balloon, free page
   reporting, KSM — and KSM's cross-tenant side-channel concern)?
3. **Local volumes vs. Ceph.** When does node pinning start to hurt?
4. **Kubernetes:** at how many nodes, if ever?
5. **Cold start budget.** Is ≈2 s acceptable for a scheduled fire? For the first
   message after lunch?
6. **The tenant image.** One image with `claude`, `codex`, Node, Python, `git`
   and common build tools — who updates it, how often, and does a running VM
   pick up a new CLI version on next `ensure`?
7. **Root inside the VM.** Does the tenant (their agent) get root in its own VM?
   It is their machine, and package installs want it; the boundary is the VM.
8. **Where Octopus itself runs** — on one of the nodes, or separately — and so
   what the callback address is.

## 11. Order of work

1. **Agent + conformance on a laptop.** `octopus-runtime-agent` and the provider
   class, with VMs replaced by a local process per tenant — pass the Octopus
   conformance suite before any virtualization exists.
2. **One node, Kata by hand.** containerd + Kata + Cloud Hypervisor on one
   bare-metal box; the node daemon; volumes; network rules. Conformance passes
   against real VMs. Measure cold start and per-VM memory overhead.
3. **Octopus file operations** (`runtime-1.1`, §9.1) — designed with both
   providers in view, implemented in `LocalRuntime` and the agent.
4. **Control plane + admin console.** Tenants, placement, events, metrics.
5. **Run a real Octopus on it** — the full Octopus e2e suite with
   `OCTOPUS_RUNTIME_PROVIDER` pointing here.
6. **Multi-node.** Placement, draining, volume moves.

## Glossary

- **KVM** — the Linux kernel's hardware virtualization; what makes a VM fast.
  Needs bare metal (or nested virtualization).
- **Hypervisor / VMM** — the program that runs a VM on top of KVM: QEMU, Cloud
  Hypervisor, Firecracker.
- **containerd** — the standard daemon that pulls images and runs containers;
  Docker and Kubernetes both use it underneath.
- **Kata Containers** — a containerd runtime that, instead of a namespaced
  process, starts a small VM and runs the container inside it.
- **virtio-fs** — shares a host directory into a VM efficiently.
- **Thin volume (LVM thin / ZFS zvol)** — a block device that only consumes the
  space actually written, with cheap snapshots.
- **CNI** — the plugin standard for container networking.
- **Ballooning / free page reporting** — ways a VM hands unused memory back to
  the host, which is what makes memory overcommit possible.
- **KSM** — the kernel merging identical memory pages across VMs; saves memory,
  but a known side-channel risk between tenants.
