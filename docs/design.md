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
  standard — file operations (§11), an offline `ensure`, session history kept
  in the runtime — is proposed to Octopus and lands there, with its
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

- **Heterogeneous nodes are the starting point, not a later scaling problem.**
  Most accounts get a Kata VM on the bare-metal box; the operator's own account
  runs on the operator's own PC. k3s places each tenant on the right node with
  labels, `nodeSelector` and taints, and runs it the right way with
  `RuntimeClass`. Writing that placement, the node agents and the cross-node
  networking ourselves would be the larger project.
- **Isolation where it is needed, none where it is not.** Other people's
  accounts get a VM boundary (`kata`). The operator's account, on the
  operator's machine, runs as a plain container (`runc`) — near-native, and no
  virtualization nested inside WSL's own VM.
- **Mature pieces we would otherwise build:** persistent volumes, network
  policy (k3s ships a policy controller), service discovery, health checks,
  and tooling an operator already knows (`kubectl`, k9s, Headlamp).

| Isolation option | Boundary | Verdict |
|---|---|---|
| Plain containers (runc) | shared kernel | Not a boundary between tenants — an agent with a shell is arbitrary code. Fine for the operator's own account on the operator's own machine. |
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
 │  k3s server ─────────────────────────────────────────────┐                     │
 │  k3s agent (this node: label octopus/pool=shared)       │                     │
 │   ┌── pod tenant B (kata) ──┐ ┌── pod tenant C (kata) ──┐ │  ...               │
 │   │ VM: octopus-runtime-    │ │ VM: octopus-runtime-    │ │                    │
 │   │ agent, claude, codex,   │ │ agent, claude, codex,   │ │                    │
 │   │ /home/tenant ◄─ PVC B   │ │ /home/tenant ◄─ PVC C   │ │                    │
 │   └─────────────────────────┘ └─────────────────────────┘ │                    │
 └─────────────────────────────────────────────────────────────┼────────────────────┘
                              home LAN                         │ k3s: 6443, 10250,
 ┌──────────────────────── Windows PC ──────────────────────────▼─ flannel ────────┐
 │  Windows: the operator's daily machine                                          │
 │  WSL2 distro "octopus-node": k3s agent (label octopus/owner=<operator id>)      │
 │   ┌── pod tenant A — the operator (runc) ──┐                                    │
 │   │ octopus-runtime-agent, claude, codex   │                                    │
 │   │ /home/tenant ◄─ local volume on WSL    │                                    │
 │   └────────────────────────────────────────┘                                    │
 └─────────────────────────────────────────────────────────────────────────────────┘
```

- **The Linux box** runs everything that must be always on: the Octopus site
  (and its public tunnel), the k3s server, the controller and admin console,
  and the Kata VMs of every account except the operator's.
- **The Windows PC** stays a Windows PC. A dedicated WSL2 distro joins the
  cluster as a k3s agent and runs exactly one tenant: the operator's own.

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
| `StatefulSet` (replicas 0 or 1) | The tenant's machine. 1 = running, 0 = released. `runtimeClassName: kata` for the shared pool, `runc` for an owner node. |
| `PersistentVolumeClaim` | The tenant's disk (§6). Survives replicas going to 0 — that is what `release` means. |
| `nodeSelector` + toleration | Pins the tenant to its pool: `octopus/pool=shared`, or `octopus/owner=<account id>` for a brought machine. |
| `NetworkPolicy` | What the pod may reach (§7). |
| `Secret` | The agent's credential for Octopus → agent calls. |
| `ResourceQuota` / limits | CPU, memory, ephemeral storage per tier (§8). |

A brought machine is **tainted** (`octopus/owner=<id>:NoSchedule`) so nothing
else is ever scheduled onto it, and its tenant has the matching toleration and
no other eligible node — so it can never be moved somewhere its disk is not.

## 5. The protocol, mapped

| Protocol call | What happens here |
|---|---|
| `ensure(tenant)` | Controller: scale the tenant's StatefulSet to 1 and wait until the agent answers. Idempotent; concurrent calls share one wait. Fails fast with a clear "runtime offline" if the tenant's node is `NotReady` (§9). |
| `release(tenant)` | Scale to 0. **The PVC stays.** Octopus calls this only when the tenant has nothing running, after its grace period. |
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
same conformance suite, works the same under `kata` and `runc`, and does not
route every turn's output through the k3s API server.

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
- **Brought machine (WSL):** a local volume on the WSL distro's own disk. The
  data never leaves the operator's PC.

Local volumes pin a tenant to its node; for the shared pool that is one box
today. Network storage (Ceph, Longhorn) removes the pin and adds a distributed
system to operate — decide when there is a second Kata node (§11).

## 7. Network

### Between the machines (k3s)

| Port | Direction | What |
|---|---|---|
| TCP 6443 | WSL node → Linux box | k3s agent to API server |
| TCP 10250 | Linux box → WSL node | API server to kubelet (logs, health) |
| UDP 8472 (VXLAN) or UDP 51820 (WireGuard) | both ways | the pod network (flannel) |

**The WSL node needs inbound rules.** The operator's WSL runs with
`networkingMode=mirrored` and `firewall=true`: the distro shares the PC's LAN
address, and the Hyper-V firewall drops unsolicited inbound by default. 10250
and the flannel port must be allowed **from the Linux box's address only**.
Flannel's WireGuard backend is preferable on a home LAN: one UDP port, and pod
traffic between the machines is encrypted.

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

- **Octopus decides when.** `ensure` before every execution, `release` after
  the tenant has had nothing running for `runtime_release_grace_seconds`
  (default 5 min). The controller does not run its own idle timer.
- **Cold start is on the critical path only for an idle tenant.** A scheduled
  fire for a released tenant pays it; an active conversation never does.
  Target: pod scheduled, VM booted and agent answering in a few seconds — to be
  measured on the real box, not assumed.
- **Sizing.** Tiers by CPU / memory limits — 2 / 4 GB default, 4 / 8 GB larger.
  vCPUs oversubscribe well (agents mostly wait on a model). Memory does not
  oversubscribe for free: a Kata VM's memory is held by the hypervisor unless
  returned via a balloon / free-page reporting. How far to overcommit is the
  main capacity question (§11).
- **Hard limits** the controller enforces regardless of Octopus: max running
  tenants per node, per-tenant disk quota, a ceiling on how long a pod may run
  with no exec (catches a lost `release`).

## 9. The owner's machine

The operator's own account runs on the operator's PC, in a WSL2 distro joined
to the cluster. What that needs, and what it changes:

- **A dedicated distro** (`octopus-node`), not the one used day to day. The
  k3s agent writes iptables rules and creates interfaces; a separate distro does
  not isolate the network stack (all WSL2 distros share one), but it keeps the
  agent's packages, services and disk out of the daily environment. The tenant
  pod does **not** mount `/mnt/c`, so the agent cannot reach Windows files.
- **`runc`, not `kata`.** It is the operator's own account on the operator's own
  machine; a VM boundary would only add nested virtualization.
- **When the PC is off, the runtime is offline.** k3s marks the node
  `NotReady`; because the tenant is pinned (§4), its pod stays Pending rather
  than being rescheduled onto a node without its disk. `ensure` fails fast with
  "runtime offline", and Octopus must surface that — in the chat, and for a
  scheduled fire as a failed run with that reason — instead of hanging. The
  operator's scheduled work (nightly checks, mail triage, daily refreshes)
  therefore needs a policy (§11): keep the PC on, Wake-on-LAN plus a Windows
  task that starts WSL at boot, or move scheduled work to a shared-pool pod.
- **Bring-your-own machine is the general case of this.** Nothing here is
  specific to the operator: any account could attach its own Linux machine as
  a pinned `runc` node. Whether that becomes a product feature is a separate
  decision.

## 10. Admin console

An operator-facing web UI served by the controller — separate from Octopus's
own Monitor page, which is about one install, not the fleet.

- **Tenants** — every account: state (running / released / starting /
  offline / failed), node, tier, volume usage, last active, running executions
  (turns, background tasks, Application backends).
- **Nodes** — each machine: Ready / NotReady, tenants on it, CPU / memory /
  disk committed vs. physical.
- **Events** — starts, releases, failed starts, OOM kills, quota hits, a
  brought machine going offline, with the agent's last output for a failure.
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

Plus one new behaviour: **`ensure` can fail** with "runtime offline" (§9), and
every caller — a turn, a scheduled fire, a delegation — must report it rather
than hang.

## 12. Hardware for the Linux box

No GPU; modest CPU; memory is what runs out first.

- **CPU:** 8–16 cores with VT-x or AMD-V enabled in the BIOS.
- **Memory:** 64 GB minimum, 128 GB preferred — Kata VMs hold their memory, and
  memory overcommit is limited (§8). Used workstations take cheap DDR4 ECC.
- **Disk:** 1–2 TB NVMe for tenant volumes, plus somewhere to send snapshots.
- **Form factor:** a tower workstation (Dell Precision, Lenovo ThinkStation, HP
  Z class) — rack servers are too loud for a house. A UPS for clean shutdowns.

## 13. Open questions — for discussion

1. **The operator's offline policy** (§9): keep the PC on, Wake-on-LAN, or
   scheduled work in the shared pool?
2. **k3s on WSL2 with mirrored networking** — expected to work, not yet proven.
   The first thing to test once the box exists (§14 step 3).
3. **Memory overcommit** — how much, with which mechanism (balloon, free page
   reporting, KSM — and KSM's cross-tenant side-channel concern)?
4. **Storage** — TopoLVM vs. OpenEBS LocalPV-LVM; and when local volumes stop
   being enough.
5. **The tenant image** — who updates the CLIs, how often, and does a running
   tenant pick up a new version on its next `ensure`?
6. **Root inside the tenant pod** — the tenant's own machine, and package
   installs want it; the boundary is the VM.
7. **Bring-your-own machine as a feature** for other accounts (§9).
8. **Where session history lives** — on the tenant's volume with metadata on
   the site, and how history is read while a tenant is released.

## 14. Order of work

1. **Agent + provider + conformance, no cluster.** `octopus-runtime-agent` and
   `KubeRuntime`'s execution path, with pods replaced by a local process per
   tenant — pass the Octopus conformance suite first.
2. **The Linux box.** k3s server + agent, Kata with Cloud Hypervisor, the LVM
   CSI driver, the controller (StatefulSet / PVC / NetworkPolicy per tenant).
   Conformance against real Kata pods; measure cold start and per-VM memory.
3. **The WSL node.** Dedicated distro, k3s agent, Hyper-V firewall rules,
   WireGuard flannel, taint + pinning, `runc`. Conformance against the
   operator's node; verify the offline behaviour (§9).
4. **Octopus file operations** (`runtime-1.1`) — designed with both node kinds
   in view, implemented in `LocalRuntime` and the agent.
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
