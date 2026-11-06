# Octopus-Runtime — design

> **Status:** draft for discussion (revised 2026-11-06). Nothing here is built. The
> interface this implements is `server/runtime/base.py` in Octopus
> (`PROTOCOL_VERSION = "runtime-1.0"`), and the conformance suite there
> (`server/runtime/conformance.py`) is the contract.

## 1. What this is

**An independent platform for running isolated machines on bare metal** — it
turns a few servers into one always-on machine per tenant, with its own controller, its own admin console and its
own release cadence. Implementing **Octopus's Runtime standard** is one of the
things it does: stand it up, point Octopus's `runtime_provider` at it, and
Octopus runs every account's work on it.

It exists because on a shared host every Octopus account's agent runs as the
same OS user, so any tenant can read any other tenant's files and credentials
(`multi-tenancy.md` §2, isolation level A). A machine per tenant is the boundary
that fixes that — a separate kernel, not a separate directory.

### How it relates to Octopus

**Octopus defines the standard; this repository implements it.** The standard
is Octopus's: the `Runtime` protocol (`server/runtime/base.py`), its
`PROTOCOL_VERSION`, and the conformance suite (`server/runtime/conformance.py`)
that decides whether an implementation conforms. Octopus also ships the
reference implementation, `LocalRuntime`.

The dependency runs one way:

- **Octopus never looks here.** It is developed without regard to this
  repository. With `LocalRuntime` alone it runs every feature on one box's
  processes and filesystem — it gives up isolation between accounts, nothing
  else. When Octopus changes the standard it bumps `PROTOCOL_VERSION`, updates
  the conformance suite and implements the change in `LocalRuntime` in the same
  change; it does not coordinate with any provider.
- **This repository follows Octopus.** Each release states which
  `PROTOCOL_VERSION` it implements and is gated on passing that version's
  conformance suite. Catching up with a new standard is work on this side.
- **Never ahead of `LocalRuntime`.** Anything this platform needs from the
  standard — file operations (§11), an `ensure` that reports an outage, session
  history kept in the runtime — is proposed to Octopus and lands there, with its
  `LocalRuntime` implementation, before it is consumed here.

## 2. Technology choice

**k3s for orchestration, on bare metal. The isolation technology — KubeVirt,
Kata Containers or gVisor — is chosen by a bake-off (§3), not on paper.**

### 2.1 k3s

k3s is a small, single-binary Kubernetes. It does not create machines; it
manages machines that join it. A **server** holds the cluster state and decides
which machine runs what; an **agent** on each node (kubelet + containerd) runs
what it is given.

- **One way to run a tenant, on any node.** Every account — the operator's
  included — gets a machine of its own, made the same way on every node. A
  second node joins the same way and is not special (§9).
- **Placement and networking we would otherwise write:** k3s pins a tenant to
  the node that holds its volume and runs the cross-node network.
- **Mature pieces:** persistent volumes and their snapshots, network policy (k3s
  ships a policy controller), health checks, and tooling an operator already
  knows (`kubectl`, k9s, Headlamp).

v1 starts with one or two nodes (§3). Plain libvirt would do for one node
forever; k3s is chosen because the cluster is expected to grow.

### 2.2 The isolation technology: three candidates

Most agent-sandbox technology is optimized for **creating short-lived sandboxes
fast**: Firecracker, libkrun, warm pools, snapshots. Octopus-Runtime v1 wants the
opposite: **a few long-lived, always-on machines** with persistent disks, full
compatibility (builds, `npm`, `git`, perhaps Docker), good disk I/O and
stability. Boot time barely matters, because a tenant's machine does not stop
(§8). Three candidates fit that shape well enough to measure:

| Candidate | What a tenant gets | Expectation |
|---|---|---|
| **KubeVirt** | A full VM — e.g. Ubuntu with systemd — whose disk is a block device on a persistent volume, managed as a Kubernetes `VirtualMachine` | Closest to today's environment, which is an Ubuntu box. Most compatible and stable; heavier per tenant, more to operate |
| **Kata Containers** (Cloud Hypervisor) | A pod running inside its own small VM, through a `RuntimeClass` | A real VM boundary with a container's workflow. The guest is a minimal kernel plus a container, not a full OS; the volume's way in (block vs. `virtio-fs`) decides its I/O |
| **gVisor** | A pod whose system calls go to a user-space kernel | The lightweight baseline. Not a VM; some system calls differ, and I/O-heavy work is slower |

Considered and not candidates for v1:

- **Firecracker** — its strength, fast boots and snapshots, does not matter for
  always-on machines; no shared filesystem, and its own orchestration to build.
- **libkrun** — young, aimed at local and ephemeral sandboxes.
- **Plain containers (`runc`)** — not a boundary between tenants.
- **Hosted services** (E2B, Daytona, Modal, Northflank) — priced per second
  for short sandboxes, and not self-hosted.

`kubernetes-sigs/agent-sandbox` (a Kubernetes API for stateful agent
sandboxes, over gVisor or Kata) is worth reusing if Kata or gVisor wins; it is
an API layer, not a runtime.

**Bare metal.** KubeVirt and Kata need KVM. On a cloud VM, or inside WSL2, a
tenant VM is nested virtualization: it works — checked on the operator's WSL2:
`/dev/kvm`, `kvm_amd nested=1`, vhost-vsock present — but pays for every VM
exit and every I/O twice. A used tower server with plenty of RAM gives KVM
natively for little money (§12).

## 3. v1: experience first, then topology

### 3.1 The v1 priority is experience parity

Early on, cost does not matter much. What matters is that a session on
Octopus-Runtime feels **the same as on today's `LocalRuntime`: the same latency,
the same stability.** That decides several things:

- **No resource reclaim in v1.** `release` is a no-op; a tenant's machine keeps
  all its memory all the time (§8).
- **No memory overcommit.** Each tier gets real memory, so there are no
  out-of-memory surprises from neighbours.
- **An acceptance gate.** v1 ships only if, on the same tasks, Octopus-Runtime is
  within a set margin of `LocalRuntime` on each of these:
  - time to first token;
  - turn latency;
  - a tool-heavy task's total time (`npm install` plus a test suite);
  - stability over a multi-day soak.

  The full Octopus e2e suite must also pass with zero failures.

### 3.2 The bake-off

The same gate decides the technology. On the real box, measure **KubeVirt**,
**Kata** (block-device volume, then `virtio-fs`) and **gVisor** against
`LocalRuntime` and each other, adding idle memory per tenant. The winner is the
one that passes the gate with the best stability; ties go to the simplest to
operate.

### 3.3 Topology: one or two nodes

```
                          Internet
                             │  cloudflared (outbound tunnel)
 ┌───────────────────────────▼──────── Linux box (bare metal) ─────────────────────┐
 │  Octopus site: sessions metadata · schedules · Applications · MCP · DB         │
 │  runtime_provider = "octopus_runtime.provider:KubeRuntime"                     │
 │  octopus-runtime controller + admin console                                    │
 │  k3s server + agent                                                            │
 │   ┌── tenant A's machine ───┐ ┌── tenant B's machine ───┐                       │
 │   │ octopus-runtime-agent,  │ │ octopus-runtime-agent,  │   ...                 │
 │   │ claude, codex, …        │ │ claude, codex, …        │                       │
 │   │ /home/tenant ◄─ vol A   │ │ /home/tenant ◄─ vol B   │                       │
 │   └─────────────────────────┘ └─────────────────────────┘                       │
 └──────────────────────────────────────────────────────────────────────────────────┘
        (optional second node: the operator's PC, see below)
```

- **The Linux box** runs everything that must be always on: the Octopus site and
  its public tunnel, the k3s server, the controller and admin console, and a
  machine for every account.
- **The operator's PC is undecided.**
  - **By default** it stays a Windows PC: the operator's account lives on the
    box, and the PC can join Octopus as a **Device** — the agent's hands,
    allowed to go offline (Octopus's `docs/plans/devices.md`).
  - **It becomes a second Runtime node only if all of these hold:**
    - the operator wants their sessions to live on it;
    - nested virtualization inside WSL2 passes the gate of §3.1;
    - the PC is kept always on (§9);
    - WSL2 can spare the memory beside daily use.

    It would then join like any node, pinned to the operator's account only.
  - **Either way is reversible.** Adding the node later means joining the
    cluster and moving one volume.

### Components in this repository

1. **`octopus-runtime-agent`** — a small static binary, the main process of
   every tenant's machine. It implements `exec` (§5): spawn, stream, write
   stdin, signal a process group, report exit. Nothing else listens there.
2. **Controller** — the service Octopus talks to for lifecycle. It owns one
   set of Kubernetes objects per tenant (§4), records which node pool the
   tenant belongs to, and serves the admin console (§10). It talks to the k3s
   API; it does not touch nodes directly — k3s's agents do that.
3. **The provider** — `octopus_runtime.provider:KubeRuntime`, a Python class
   implementing Octopus's `Runtime` protocol: lifecycle through the controller,
   execution straight to the tenant's agent. Octopus loads it with
   `OCTOPUS_RUNTIME_PROVIDER=octopus_runtime.provider:KubeRuntime`.
4. **The tenant image** — Debian/Ubuntu-based, with the agent, `claude`,
   `codex`, Node, Python, `git` and common build tools: a VM disk image for
   KubeVirt, an OCI image for Kata or gVisor.

**Control path and data path are separate.** `ensure`, `release` and `expose`
go through the controller. An `exec` asks the controller once where the
tenant's agent is (its cluster IP + a short-lived credential) and then streams
directly to the agent. A turn's stdout is the hottest traffic in the system; routing it
through the controller would make the controller the bottleneck and a single
point of failure for every running turn. Octopus runs on a cluster node, so it
reaches a tenant's machine on any node through the cluster network.

## 4. Per-tenant objects

For each account the controller maintains:

| Object | Purpose |
|---|---|
| The tenant's machine | A KubeVirt `VirtualMachine`, or a `StatefulSet` (replicas 1) with `runtimeClassName: kata` / `gvisor` — whichever the bake-off picks (§3.2). It stays running (§8); it is stopped only when an account is disabled or a node is drained. |
| `PersistentVolumeClaim` | The tenant's disk (§6). It outlives any machine; a restarted machine mounts the same volume. |
| `nodeSelector` | Pins the tenant to the node that holds its volume. |
| `NetworkPolicy` | What the tenant's machine may reach (§7). |
| `Secret` | The agent's credential for Octopus → agent calls. |
| `ResourceQuota` / limits | CPU, memory, ephemeral storage per tier (§8). |

A tenant can never be scheduled somewhere its disk is not.

## 5. The protocol, mapped

| Protocol call | What happens here |
|---|---|
| `ensure(tenant)` | Controller: confirm the tenant's machine is up and its agent answers. Normally immediate. A machine that is not running (after a host restart or maintenance) is started and waited for. Idempotent; concurrent calls share one wait. A node that is down is an outage (§6), reported as one rather than hung on. |
| `release(tenant)` | **A no-op in v1** (§3.1, §8). The machine keeps running with all its memory; the volume is untouched. Kept in the protocol so reclaim can be added later without a protocol change. |
| `exec(spec)` | Provider → agent: start `argv` with exactly `spec.env` (nothing added), `cwd`, the stdin/stderr modes, its own process group if asked. Streams are multiplexed (stdin, stdout, stderr, signal, exit) over one WebSocket or gRPC stream. |
| `Execution.signal(sig, group)` | A frame on that stream; the agent does `killpg` in the tenant's machine. Returns as soon as the frame is sent. |
| `expose(tenant)` | Controller picks a free port on the tenant's machine; returns `(port, "<its cluster IP>:<port>")`, reachable from the Octopus host over the cluster network. |
| `callback_base(tenant)` | The Octopus site **as reachable from inside a tenant's machine**: its node IP on the cluster network, never `127.0.0.1`. |
| `paths(tenant)` | The tenant's directories on its machine: `/home/tenant/{workspace,agents,applications,…}`. |
| `confine(requested, tenant)` | Structural: anything that resolves inside `/home/tenant` is allowed; the machine has nothing else of the tenant's to protect. `extra_roots` is ignored. |

Why our own agent instead of `kubectl exec` (or a VM's built-in guest agent):
Octopus depends on two behaviours the generic exec path does not give cleanly —

- **process-group signalling** — interrupting a turn must kill the CLI *and*
  every MCP child and sub-agent it started;
- **held executions** — a Claude process stays alive between turns, idle, with
  stdin open, and is written to again minutes later.

An agent we own implements exactly the protocol's semantics, is tested by the
same conformance suite, and does not route every turn's output through the
k3s API server.

## 6. Storage: the tenant's disk is the product

Everything a tenant has lives on its volume, mounted at `/home/tenant`:
workspace, agent memory, Application code and data, uploads, **and the CLIs'
own state** — Claude's and Codex's logins and transcripts. Losing it, or
starting a machine without it, signs the tenant out of everything
(`runtime-providers.md` §5). The conformance case `state_survives_release`
exists for this.

- **Shared pool (Linux box):** an LVM-backed CSI driver (TopoLVM or OpenEBS
  LocalPV-LVM) on the box's NVMe — one logical volume per tenant. Quotas are
  the volume size; snapshots are cheap and atomic, which gives backups and
  "restore my account". k3s's bundled `local-path` provisioner is fine for the
  first prototype but has neither quotas nor snapshots, so it is not the
  serving answer.
- **Session history** lives here too. Octopus writes it, as it does today, but
  through a storage operation of the Runtime interface; Octopus's own site keeps
  only metadata, the Ledger and rebuildable indexes (Octopus `docs/roadmap.md`
  item 2).
- **Storage/compute separation is the recovery model.** The machine is
  disposable; the volume is the product. A crashed machine restarts and mounts
  the same volume.
  A lost node is restored from off-box backups onto another. How exactly — the
  volume's way into the VM, its layout, snapshots, backups and restore — is the
  open design of §13 item 2.

Local volumes pin a tenant to its node; for the shared pool that is one box
today. Network storage (Ceph, Longhorn) removes the pin and adds a distributed
system to operate — decide at around three nodes.

## 7. Network

### Between the machines (k3s)

With a second node, these open between the nodes only:

| Port | Direction | What |
|---|---|---|
| TCP 6443 | agent node → server | k3s agent to API server |
| TCP 10250 | server → agent node | API server to kubelet (logs, health) |
| UDP 51820 (WireGuard) | both ways | the pod network (flannel's WireGuard backend: one port, encrypted) |

### What a tenant's machine may reach (NetworkPolicy + node nftables)

| Destination | Rule |
|---|---|
| Internet | Allowed — model APIs, packages, web pages. |
| The Octopus site | Only the callback address (§5): MCP namespaces and the Application agent API, authenticated by Octopus's own per-session bearer. |
| Another tenant's machine | **Denied.** |
| The home LAN (router, NAS, the PC's Windows shares, …) | **Denied** — every RFC 1918 range except the callback address. A tenant's agent runs arbitrary code in your house. |
| k3s API, kubelets, controller | Denied. |

Inbound to a tenant's machine: only the agent's port and exposed Application ports, only
from the Octopus site. Nothing in the cluster is reachable from the internet;
the public entry point stays the one cloudflared tunnel to the site.

## 8. Lifecycle and capacity

- **A Runtime is always on.** Every tenant's machine keeps running: stopping it
  would end its held CLI processes and put a cold start in front of every
  scheduled fire and every first message.
- **No reclaim in v1.** `ensure` confirms the machine before every execution;
  `release` is a no-op (§3.1). Memory reclaim (balloon, free-page reporting) is
  deferred (§15).
- **No memory overcommit in v1.** Tiers by CPU / memory — 4 GB default, 8 GB
  larger. vCPUs may oversubscribe (agents mostly wait on a model); memory does
  not. On a 128 GB box with ~16 GB kept for the host, the site and k3s, that is
  roughly **12–25 accounts**. That is the honest early capacity, and it is
  enough.
- **A cold start happens only after a host restart, maintenance or an image
  update**, never in normal operation. Its path is still measured and kept
  correct, because recovery depends on it.
- **Hard limits** the controller enforces regardless of Octopus: max tenants
  per node (from real memory, not overcommit), per-tenant disk quota.

## 9. Runtime nodes are always on

A Runtime is where sessions live, so a Runtime node is **always on**: its going
down is an outage the platform recovers from, never a state the design plans
for. This decides what a machine outside the box can be.

- **A machine that should host sessions joins as a Runtime node** — virtualized
  like every other node, managed by k3s, held to "always on", and not treated as
  special because of who owns it. The operator's PC qualifies only on the
  criteria of §3.3: nested virtualization inside WSL2 must pass the gate, and
  the PC must stay on.
- **A machine that comes and goes is a Device**, not a Runtime: the agent's
  hands, reached from its Runtime, allowed to be offline (Octopus
  `docs/plans/devices.md`).
- **No hybrid.** An offline-able home for sessions would need offline policies
  for turns, schedules and history. That is the complexity this rule exists to
  avoid.

## 10. Admin console

An operator-facing web UI served by the controller — separate from Octopus's
own Monitor page, which is about one install, not the fleet.

- **Tenants** — every account: state (running / starting / failed), node,
  tier, memory used vs. limit, volume usage, last active, running executions
  (turns, background tasks, Application backends).
- **Nodes** — each machine: Ready / NotReady, tenants on it, CPU / memory /
  disk committed vs. physical.
- **Events** — starts, failed starts, OOM kills, quota hits, a node going down,
  with the agent's last output for a failure.
- **Actions** — stop a tenant, resize, snapshot / restore its volume, cordon or
  drain a node.
- **Metrics** — the parity measures of §3.1 over time, running tenants, memory
  committed vs. physical.

Its own operator authentication; no tenant ever reaches it. For anything below
this level, k9s or Headlamp on the cluster.

## 11. What Octopus must change before this runs

Steps 1–7 of `runtime-providers.md` §10 are done: every process goes through
`exec`, and Octopus never assumes a port or a callback address. Three gaps
remain, and each one breaks this provider immediately:

1. **Server-side file access** (`runtime-providers.md` §9). Octopus still reads
   and writes tenant files with `Path` on its own disk: `/showme`, uploads,
   large-prompt spills, agent memory provisioning, **Applications' static
   files**, and the full-copy fork. Here those files are on the tenant's volume,
   on a tenant's machine, possibly on another node. The protocol needs file operations
   (read / write / list / stat, streamed) — a `runtime-1.1` — implemented by
   `LocalRuntime` in Octopus first (on its own disk, so a standalone install
   keeps every feature) and by the agent here. This is the largest piece of
   work on the Octopus side.
2. **The environment allowlist** (step 8). Octopus still passes its own
   environment to the CLIs — correct on one box, but it would ship the
   operator's `OCTOPUS_AUTH_TOKEN` into a tenant's machine.
3. **Fork as a runtime step** (step 9) — the transcript copy runs on the
   tenant's machine.

Plus two more:

- **Session history through the Runtime's storage**, written by Octopus as today
  (§6).
- **An outage is reported, not hung on.** A Runtime is always on by assumption,
  but a node can still fail; every caller — a turn, a scheduled fire, a
  delegation — reports it.

## 12. Hardware for the Linux box

No GPU; modest CPU; memory is what runs out first.

- **CPU:** 8–16 cores with VT-x or AMD-V enabled in the BIOS.
- **Memory:** 128 GB preferred — v1 holds every tenant's memory with no
  overcommit (§8), so memory sets capacity. Used workstations take cheap DDR4
  ECC.
- **Disk:** 1–2 TB NVMe for tenant volumes, plus somewhere to send snapshots.
- **Form factor:** a tower workstation (Dell Precision, Lenovo ThinkStation, HP
  Z class) — rack servers are too loud for a house. A UPS for clean shutdowns.

## 13. Open questions — for discussion

1. **The isolation technology** — KubeVirt, Kata or gVisor, by the bake-off
   (§3.2).
2. **Virtualization and the persistent volume, in detail** — the next design to
   write, once (1) is known:
   - how the volume enters the tenant's machine (block device, `virtio-blk`, vs.
     shared directory, `virtio-fs`);
   - its layout and filesystem;
   - snapshot schedule and retention, and off-box backups, so a lost node is not
     lost data;
   - SQLite consistency in a snapshot (WAL plus a filesystem freeze, or the
     backup API);
   - restore, with recovery-point and recovery-time targets;
   - resize, encryption at rest, and moving a volume between nodes.
3. **The operator's PC** — Device or second Runtime node (§3.3).
4. **Storage driver** — TopoLVM vs. OpenEBS LocalPV-LVM (or KubeVirt's
   CDI-managed disks, if KubeVirt wins).
5. **The tenant image** — who updates the CLIs, how often, and how a running
   machine picks up a new version without a cold start in front of a user.
6. **Root inside the tenant's machine** — it is the tenant's own machine, and
   package installs want root; the boundary is the VM (or gVisor).

## 14. Order of work

1. **Agent + provider + conformance, no cluster.** `octopus-runtime-agent` and
   `KubeRuntime`'s execution path, with tenant machines replaced by a local
   process per tenant — pass the Octopus conformance suite first.
2. **The box and the bake-off.**
   - k3s and the three candidates on the real box.
   - The parity benchmark of §3.1 against `LocalRuntime`.
   - The choice (§3.2).
   - If the operator's PC is to be a node, the nested-virtualization run of
     §3.3 is part of this step.
3. **Virtualization and volumes in detail** (§13 item 2), for the winner; the
   controller with its per-tenant objects.
4. **Octopus file operations and the history store** (`runtime-1.1`),
   implemented in `LocalRuntime` and the agent.
5. **Admin console.**
6. **Move the site** to the Linux box (DB, master key, cloudflared tunnel), with
   a rollback path; migrate each account's data onto its volume.
7. **Full Octopus e2e and the gate of §3.1** with `OCTOPUS_RUNTIME_PROVIDER`
   pointing here. v1 ships on passing it.

## 15. What v1 defers

- **Resource reclaim and memory overcommit.** Cost is not the v1 priority.
  When it becomes one, two questions decide it:
  - does automatic free-page reporting alone keep idle machines near their
    working set, or is an explicit `release` hint needed;
  - how far to overcommit, given KSM's cross-tenant side-channel concern.
- **Network storage** (Ceph, Longhorn), at around three nodes.
- **Firecracker / libkrun-style fast ephemeral sandboxes** — not the shape
  of an always-on tenant machine.

## Glossary

- **KVM** — the Linux kernel's hardware virtualization. Native on bare metal;
  nested (slower) inside a VM such as WSL2.
- **Hypervisor / VMM** — the program that runs a VM on top of KVM: QEMU, Cloud
  Hypervisor, Firecracker.
- **containerd** — the daemon that pulls images and runs containers on a node.
- **k3s** — a small Kubernetes in one binary. **Server**: cluster state and
  scheduling. **Agent**: runs on each node — kubelet + containerd.
- **Node** — a machine in the cluster. **Pod** — the unit k3s schedules: one or
  more containers sharing a network address.
- **RuntimeClass** — per-pod choice of how containerd runs it: `runc` (ordinary
  container), `kata` (inside a VM) or `gvisor` (on a user-space kernel).
- **KubeVirt** — runs full virtual machines as Kubernetes objects
  (`VirtualMachine`), with disks on persistent volumes.
- **gVisor** — a user-space kernel that intercepts a container's system calls;
  stronger than a container, weaker than a VM.
- **StatefulSet** — keeps a fixed, named pod (here: one per tenant) with its
  own volume; scaling it to 0 stops the pod and keeps the volume.
- **PVC / CSI** — a persistent volume claim (a tenant's disk), provided by a
  storage driver (CSI) such as TopoLVM.
- **Taint / toleration, nodeSelector** — how pods are kept off, or pinned to,
  particular nodes.
- **flannel** — k3s's pod network between nodes (VXLAN or WireGuard).
- **NetworkPolicy** — firewall rules between pods and to the outside, enforced
  by k3s's policy controller.
- **virtio-fs** — shares a host directory into a VM.
- **Ballooning / free page reporting** — ways a VM hands unused memory back to
  the host, which is what makes memory overcommit possible (deferred, §15).
- **KSM** — the kernel merging identical memory pages across VMs; saves memory,
  but a known side-channel risk between tenants.
