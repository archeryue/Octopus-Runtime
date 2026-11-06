# Octopus-Runtime — design

> **Status:** draft for discussion (2026-10-13). Nothing here is built. The
> interface this implements is `server/runtime/base.py` in Octopus
> (`PROTOCOL_VERSION = "runtime-1.0"`), and the conformance suite there
> (`server/runtime/conformance.py`) is the contract.

## 1. What this is

**An independent platform for running isolated machines on bare metal** — it
turns a few servers (and, where an owner brings one, their own machine) into
one machine per tenant, with its own controller, its own admin console and its
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

**k3s for orchestration, Kata Containers for isolation, on bare metal.**

**k3s** is a small, single-binary Kubernetes. It does not create machines; it
manages machines that join it. A **server** holds the cluster state and decides
which machine runs what; an **agent** on each machine (kubelet + containerd)
runs what it is given. Any Linux that can run the k3s agent and reach the
server can be a node — a bare-metal box, a cloud VM, or a WSL2 distro on a
Windows PC.

**Kata Containers** is how a node runs a workload *as a VM*. A pod names a
`RuntimeClass`; containerd on the node runs it either as an ordinary container
(`runc`) or, with `kata`, inside its own small VM with its own guest kernel.
The choice is per pod, so one cluster runs both.

Why this pair, for the deployment we actually have (§3):

- **One way to run a tenant, on any node.** Every account — the operator's
  included — gets a Kata VM. A second node, whether a second box or a machine at
  home, joins the same way and is not special (§9). k3s does the placement,
  pins a tenant to the node that holds its volume, and runs the cross-node
  networking. Writing that ourselves would be the larger project.
- **Mature pieces we would otherwise build:** persistent volumes, network
  policy (k3s ships a policy controller), service discovery, health checks,
  and tooling an operator already knows (`kubectl`, k9s, Headlamp).

| Isolation option | Boundary | Verdict |
|---|---|---|
| Plain containers (runc) | shared kernel | Not a boundary between tenants — an agent with a shell is arbitrary code. |
| gVisor | user-space kernel | Strong, but syscall compatibility and I/O performance matter for an agent running builds, `git`, `npm`, test suites. |
| Raw Firecracker / Cloud Hypervisor | VM | Same isolation as Kata, but we would rebuild images, networking and the guest agent ourselves. |
| **Kata Containers** | **VM** | VM boundary that plugs into containerd and therefore into k3s, per pod. |

**Hypervisor under Kata:** Cloud Hypervisor first. It supports `virtio-fs`
(sharing a host directory into the guest), which keeps the storage design open
(§6). Firecracker boots faster but cannot share a filesystem — it needs a
block-device snapshotter — so it is the option to measure later.

**Bare metal for the Kata node.** Kata needs KVM. On a cloud VM, or inside WSL2,
a Kata VM is nested virtualization: it works (this was checked on the operator's
WSL2 — `/dev/kvm`, `kvm_amd nested=1`, vhost-vsock all present) but pays for
every VM exit and every I/O twice. A used tower server with plenty of RAM gives
KVM natively for little money (§12).

## 3. Topology

```
                          Internet
                             │  cloudflared (outbound tunnel)
 ┌───────────────────────────▼──────── Linux box (bare metal) ─────────────────────┐
 │  Octopus site: sessions metadata · schedules · Applications · MCP · DB         │
 │  runtime_provider = "octopus_runtime.provider:KubeRuntime"                     │
 │  octopus-runtime controller + admin console                                    │
 │  k3s server + agent (this node: label octopus/pool=shared)                     │
 │   ┌── pod tenant A (kata) ──┐ ┌── pod tenant B (kata) ──┐                       │
 │   │ VM: octopus-runtime-    │ │ VM: octopus-runtime-    │   ...                 │
 │   │ agent, claude, codex,   │ │ agent, claude, codex,   │                       │
 │   │ /home/tenant ◄─ PVC A   │ │ /home/tenant ◄─ PVC B   │                       │
 │   └─────────────────────────┘ └─────────────────────────┘                       │
 └──────────────────────────────────────────────────────────────────────────────────┘
```

- **The Linux box** runs everything that must be always on: the Octopus site and
  its public tunnel, the k3s server, the controller and admin console, and a
  Kata VM for every account — the operator's included.
- **Every Runtime node is always on.** A node that may go offline is not a
  Runtime node (§9).
- **The operator's Windows PC stays a Windows PC.** It uses Octopus through the
  browser. If its agents should reach it, it joins Octopus as a **Device** (the
  agent's hands, allowed to go offline — Octopus's `docs/plans/devices.md`), not
  as a Runtime.

### Components in this repository

1. **`octopus-runtime-agent`** — a small static binary, the main process of
   every tenant pod. It implements `exec` (§5): spawn, stream, write stdin,
   signal a process group, report exit. Nothing else listens in the pod.
2. **Controller** — the service Octopus talks to for lifecycle. It owns one
   set of Kubernetes objects per tenant (§4), records which node pool the
   tenant belongs to, and serves the admin console (§10). It talks to the k3s
   API; it does not touch nodes directly — k3s's agents do that.
3. **The provider** — `octopus_runtime.provider:KubeRuntime`, a Python class
   implementing Octopus's `Runtime` protocol: lifecycle through the controller,
   execution straight to the tenant's agent. Octopus loads it with
   `OCTOPUS_RUNTIME_PROVIDER=octopus_runtime.provider:KubeRuntime`.
4. **The tenant image** — Debian-based, with the agent, `claude`, `codex`,
   Node, Python, `git` and common build tools.

**Control path and data path are separate.** `ensure`, `release` and `expose`
go through the controller. An `exec` asks the controller once where the
tenant's agent is (pod IP + short-lived credential) and then streams directly
to the agent. A turn's stdout is the hottest traffic in the system; routing it
through the controller would make the controller the bottleneck and a single
point of failure for every running turn. Octopus runs on a cluster node, so it
reaches pod IPs on either node through the cluster network.

## 4. Per-tenant objects

For each account the controller maintains:

| Object | Purpose |
|---|---|
| `StatefulSet` (replicas 1) | The tenant's machine, `runtimeClassName: kata`. It stays running: `release` reclaims its resources, it does not stop it (§8). Replicas 0 only when an account is disabled or a node is drained. |
| `PersistentVolumeClaim` | The tenant's disk (§6). It outlives any VM; a restarted VM mounts the same volume. |
| `nodeSelector` | Pins the tenant to the node that holds its volume. |
| `NetworkPolicy` | What the pod may reach (§7). |
| `Secret` | The agent's credential for Octopus → agent calls. |
| `ResourceQuota` / limits | CPU, memory, ephemeral storage per tier (§8). |

A tenant can never be scheduled somewhere its disk is not.

## 5. The protocol, mapped

| Protocol call | What happens here |
|---|---|
| `ensure(tenant)` | Controller: confirm the tenant's VM is up and its agent answers, restoring the resources `release` reclaimed. Normally immediate. A VM that is not running (after a host restart or maintenance) is started and waited for. Idempotent; concurrent calls share one wait. A node that is down is an outage (§6), reported as one rather than hung on. |
| `release(tenant)` | **Reclaim, don't stop.** The VM keeps running; its idle memory goes back to the host (balloon / free-page reporting, §8). The PVC is untouched. Octopus calls this only when the tenant has nothing running, after its grace period. |
| `exec(spec)` | Provider → agent: start `argv` with exactly `spec.env` (nothing added), `cwd`, the stdin/stderr modes, its own process group if asked. Streams are multiplexed (stdin, stdout, stderr, signal, exit) over one WebSocket or gRPC stream. |
| `Execution.signal(sig, group)` | A frame on that stream; the agent does `killpg` in the pod. Returns as soon as the frame is sent. |
| `expose(tenant)` | Controller picks a free port for the pod; returns `(port, "<pod IP>:<port>")`. The pod IP is reachable from the Octopus host over the cluster network. |
| `callback_base(tenant)` | The Octopus site **as reachable from inside a pod**: its node IP on the cluster network, never `127.0.0.1`. |
| `paths(tenant)` | The tenant's directories inside the pod: `/home/tenant/{workspace,agents,applications,…}`. |
| `confine(requested, tenant)` | Structural: anything that resolves inside `/home/tenant` is allowed; the pod has nothing else of the tenant's to protect. `extra_roots` is ignored. |

Why our own agent instead of `kubectl exec` (or Kata's built-in guest agent):
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
starting a pod without it, signs the tenant out of everything
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
- **Storage/compute separation is the recovery model.** The VM is disposable;
  the volume is the product. A crashed VM restarts and mounts the same volume.
  A lost node is restored from off-box backups onto another. How exactly — the
  volume's way into the VM, its layout, snapshots, backups and restore — is the
  open design of §13.

Local volumes pin a tenant to its node; for the shared pool that is one box
today. Network storage (Ceph, Longhorn) removes the pin and adds a distributed
system to operate — decide when there is a second Kata node (§11).

## 7. Network

### Between the machines (k3s)

One node today. A second node opens these, between the nodes only:

| Port | Direction | What |
|---|---|---|
| TCP 6443 | agent node → server | k3s agent to API server |
| TCP 10250 | server → agent node | API server to kubelet (logs, health) |
| UDP 51820 (WireGuard) | both ways | the pod network (flannel's WireGuard backend: one port, encrypted) |

### What a tenant pod may reach (NetworkPolicy + node nftables)

| Destination | Rule |
|---|---|
| Internet | Allowed — model APIs, packages, web pages. |
| The Octopus site | Only the callback address (§5): MCP namespaces and the Application agent API, authenticated by Octopus's own per-session bearer. |
| Another tenant's pod | **Denied.** |
| The home LAN (router, NAS, the PC's Windows shares, …) | **Denied** — every RFC 1918 range except the callback address. A tenant's agent runs arbitrary code in your house. |
| k3s API, kubelets, controller | Denied. |

Inbound to a pod: only the agent's port and exposed Application ports, only
from the Octopus site. Nothing in the cluster is reachable from the internet;
the public entry point stays the one cloudflared tunnel to the site.

## 8. Lifecycle and capacity

- **A Runtime is always on.** Every tenant's VM keeps running: stopping it would
  end its held CLI processes and put a cold start in front of every scheduled
  fire and every first message.
- **Octopus decides when to reclaim.** `ensure` before every execution;
  `release` after the tenant has had nothing running for
  `runtime_release_grace_seconds`. `release` hands the idle VM's memory back to
  the host — balloon / free-page reporting — and costs no stop and no cold start.
  The controller runs no idle timer of its own.
- **Capacity depends on reclaim working.** An idle VM must settle at its working
  set, not creep up to its limit through page cache. Measure it on the real box
  before selling a tier, and keep measuring (§13).
- **A cold start happens only after a host restart, maintenance or an image
  update**, never in normal operation. It is still measured, and its path kept
  correct, because recovery depends on it.
- **Sizing.** Tiers by CPU / memory limits — 2 / 4 GB default, 4 / 8 GB larger.
  vCPUs oversubscribe well (agents mostly wait on a model). Memory does not
  oversubscribe for free: a Kata VM's memory is held by the hypervisor unless
  returned via a balloon / free-page reporting. How far to overcommit is the
  main capacity question (§11).
- **Hard limits** the controller enforces regardless of Octopus: max running
  tenants per node, per-tenant disk quota, a ceiling on how long a pod may run
  with no exec (catches a lost `release`).

## 9. Runtime nodes are always on

A Runtime is where sessions live, so a Runtime node is **always on**: its going
down is an outage the platform recovers from, never a state the design plans
for. This decides what a machine outside the box can be.

- **A machine that should host sessions joins as a Runtime node** — virtualized
  with Kata like every other node, managed by k3s, held to "always on", and not
  treated as special because of who owns it. It needs bare-metal KVM. A
  Windows PC's WSL2 would mean nested virtualization, and it is not a machine
  that stays on.
- **A machine that comes and goes is a Device**, not a Runtime: the agent's
  hands, reached from its Runtime, allowed to be offline (Octopus
  `docs/plans/devices.md`).
- **No hybrid.** An offline-able home for sessions would need offline policies
  for turns, schedules and history. That is the complexity this rule exists to
  avoid.

## 10. Admin console

An operator-facing web UI served by the controller — separate from Octopus's
own Monitor page, which is about one install, not the fleet.

- **Tenants** — every account: state (running / reclaimed / starting /
  failed), node, tier, memory held vs. limit, volume usage, last active, running
  executions (turns, background tasks, Application backends).
- **Nodes** — each machine: Ready / NotReady, tenants on it, CPU / memory /
  disk committed vs. physical.
- **Events** — starts, reclaims, failed starts, OOM kills, quota hits, a node
  going down, with the agent's last output for a failure.
- **Actions** — stop a tenant, resize, snapshot / restore its volume, cordon or
  drain a node.
- **Metrics** — start latency, running tenants, memory overcommit over time.

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
   in a pod on possibly another machine. The protocol needs file operations
   (read / write / list / stat, streamed) — a `runtime-1.1` — implemented by
   `LocalRuntime` in Octopus first (on its own disk, so a standalone install
   keeps every feature) and by the agent here. This is the largest piece of
   work on the Octopus side.
2. **The environment allowlist** (step 8). Octopus still passes its own
   environment to the CLIs — correct on one box, but it would ship the
   operator's `OCTOPUS_AUTH_TOKEN` into a tenant's pod.
3. **Fork as a runtime step** (step 9) — the transcript copy runs in the pod.

Plus two more:

- **Session history through the Runtime's storage**, written by Octopus as today
  (§6).
- **An outage is reported, not hung on.** A Runtime is always on by assumption,
  but a node can still fail; every caller — a turn, a scheduled fire, a
  delegation — reports it.

## 12. Hardware for the Linux box

No GPU; modest CPU; memory is what runs out first.

- **CPU:** 8–16 cores with VT-x or AMD-V enabled in the BIOS.
- **Memory:** 64 GB minimum, 128 GB preferred — Kata VMs hold their memory, and
  memory overcommit is limited (§8). Used workstations take cheap DDR4 ECC.
- **Disk:** 1–2 TB NVMe for tenant volumes, plus somewhere to send snapshots.
- **Form factor:** a tower workstation (Dell Precision, Lenovo ThinkStation, HP
  Z class) — rack servers are too loud for a house. A UPS for clean shutdowns.

## 13. Open questions — for discussion

1. **Virtualization and the persistent volume, in detail** — the next design
   to write:
   - how the volume enters the Kata VM (block device, `virtio-blk`, vs. shared
     directory, `virtio-fs`);
   - its layout and filesystem;
   - snapshot schedule and retention, and off-box backups, so a lost node is
     not lost data;
   - SQLite consistency in a snapshot (WAL plus a filesystem freeze, or the
     backup API);
   - restore, with recovery-point and recovery-time targets;
   - resize, encryption at rest, and moving a volume between nodes.
2. **Memory reclaim** — balloon vs. free-page reporting, the guest kernel
   settings, and proof that an idle VM's memory really returns; how far to
   overcommit; KSM's cross-tenant side-channel concern.
3. **Storage** — TopoLVM vs. OpenEBS LocalPV-LVM; and when local volumes stop
   being enough.
4. **The tenant image** — who updates the CLIs, how often, and how a running VM
   picks up a new version without a cold start in front of a user.
5. **Root inside the tenant pod** — the tenant's own machine, and package
   installs want it; the boundary is the VM.

## 14. Order of work

1. **Agent + provider + conformance, no cluster.** `octopus-runtime-agent` and
   `KubeRuntime`'s execution path, with pods replaced by a local process per
   tenant — pass the Octopus conformance suite first.
2. **The Linux box.** k3s server + agent, Kata with Cloud Hypervisor, the LVM
   CSI driver, the controller (StatefulSet / PVC / NetworkPolicy per tenant).
   Conformance against real Kata pods; measure cold start and per-VM memory.
3. **Virtualization and volumes in detail** (§13 item 1), and memory reclaim
   proven on the box (§13 item 2).
4. **Octopus file operations and the history store** (`runtime-1.1`),
   implemented in `LocalRuntime` and the agent.
5. **Admin console.**
6. **Move the site** to the Linux box (DB, master key, cloudflared tunnel), with
   a rollback path; migrate each account's data onto its volume.
7. **Full Octopus e2e** with `OCTOPUS_RUNTIME_PROVIDER` pointing here.

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
  container) or `kata` (inside a VM).
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
  the host, which is what makes memory overcommit possible.
- **KSM** — the kernel merging identical memory pages across VMs; saves memory,
  but a known side-channel risk between tenants.
