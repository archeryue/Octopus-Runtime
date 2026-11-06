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

**k3s for orchestration, on bare metal. The default isolation technology is
KubeVirt — a full VM per account. It is a default, not a final decision: the
bake-off (§3.2) validates it, with Kata Containers as the fallback.**

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

### 2.2 What a tenant's machine is: a cloud computer

A tenant's Runtime is **not a sandbox** where an agent runs a few commands and
leaves. It is the user's **cloud computer**:

- always on;
- where their agents build software (file-heavy work: `npm install`, `git`,
  compilers, test suites);
- where they install tools and run services;
- expected to be exactly as they left it tomorrow.

Five requirements follow, and every option below is judged against them:

| # | Requirement |
|---|---|
| R1 | **Long-lived and always on**: months of uptime, not minutes |
| R2 | **The whole machine persists**: what the user or their agent installs and configures survives a restart, not only a data volume |
| R3 | **Full compatibility**: a real OS with systemd, so services, Docker and any tool run as on an ordinary server |
| R4 | **Fast file and process work**: builds and test suites close to native speed |
| R5 | **Reliability over long runs**: recovers by itself from crashes and host restarts, with the disk intact; management-plane failures do not take machines down |

Most agent-sandbox technology is optimized for the opposite: **creating
short-lived sandboxes fast** — sub-second boots, warm pools, snapshots. Boot
time barely matters here, because a tenant's machine does not stop (§8).

### 2.3 The options

**KubeVirt — a full VM per account (the default).** A real VM, e.g. Ubuntu with
systemd, managed as a Kubernetes `VirtualMachine`. Its root disk is a persistent
volume, cloned per account from a base image by CDI (Containerized Data
Importer). It runs on libvirt + QEMU/KVM.

- ✅ **R2:** the whole disk persists. A restart is a real computer rebooting:
  installed packages, services and configuration are still there.
- ✅ **R3:** a full OS; Docker, systemd services, cron, anything.
- ✅ **R4:** root, temp files and caches all sit on a real block disk with a
  normal filesystem. Nothing goes through a shared-directory path.
- ✅ **R5:** the most proven shape for long-lived VMs. It is productized as Red
  Hat OpenShift Virtualization, where enterprises run VMs for months.
  - A VM runs in its own QEMU process on the node, so KubeVirt's controller,
    API or node agent restarting — or k3s restarting — does not stop running
    VMs.
  - Run strategies restart a VM after a crash or a host reboot.
  - A guest agent lets a hung VM be detected and restarted.
- ✅ Features we will want later come with it: disk snapshots; hot-plugging
  disks, NICs, CPU and memory; GPU passthrough; live migration (once storage is
  shared).
- ❌ Heaviest per account: a full OS, roughly 0.5–1 GB more memory than a
  container, plus one launcher pod per VM.
- ❌ Slowest to boot by default: firmware, bootloader, a generic cloud image.
  Fixable the way WSL2 does it — direct kernel boot and a minimal image, so the
  guest boots in seconds (§3.2).
- ❌ Most to operate:
  - VM disk images;
  - an OS inside every machine that needs its own updates (§13);
  - KubeVirt and CDI themselves, with care on upgrades;
  - more tested on OpenShift than on k3s.

**Kata Containers — a container inside a small VM (the fallback).** A pod runs
in its own lightweight VM through a `RuntimeClass` (QEMU or Cloud Hypervisor
underneath).

- ✅ A real VM boundary with a container's workflow: OCI images, ordinary pods.
- ✅ Lighter than a full VM; fast to start (≈ 6 s for a full pod in a 2026
  third-party test; ≈ 0.2 s for the hypervisor alone).
- ✅ R4 can be met, but only if both the volume (hot-plugged as `virtio-blk` when
  its source is a block device) and the container's root filesystem
  (containerd's devmapper snapshotter) are on block devices. The default puts
  the root filesystem on `virtio-fs`, which is slow on small-file work —
  `node_modules`, `git` metadata, `~/.cache`.
- ❌ **R2 fails:** only the mounted volume persists. The container's root
  filesystem is rebuilt from the image on every restart, so whatever the user
  installed is gone.
- ❌ **R3 is weak:** the guest is a minimal kernel plus a container, not a full
  OS, so systemd, Docker inside and system services are awkward.
- ❌ **R5 is less proven:** the VM's life is tied to the pod and the shim, more
  parts sit inside the guest boundary (agent, shim, hot-plugged devices), and
  its production use is mostly short-to-medium-lived pods. Compatibility
  pressure is higher too: Kata, containerd, Kubernetes, the hypervisor and the
  guest kernel must all match.

**gVisor — a user-space kernel (the baseline).** A pod's system calls are served
by a kernel written in user space; no VM.

- ✅ Lightest and simplest: plugs straight into Kubernetes, fast to start, little
  memory, no KVM needed. Native in Kubernetes' `agent-sandbox`.
- ❌ **R4:** file- and process-heavy work is slower, and some system calls
  behave differently — the riskiest choice for an agent whose job is building
  software.
- ❌ **R2, R3:** a container, with the same limits as Kata.
- ❌ Not a hypervisor boundary; isolation is good but weaker. Modal, which made
  gVisor its default, shipped real-VM sandboxes in October 2026.

**Considered, not candidates:**

| Option | Why not, for an always-on cloud computer |
|---|---|
| **Firecracker** | Built for AWS Lambda: very fast boots, tiny, very few devices. No hot-plug, no GPU, no live migration, no shared filesystem, and its own orchestration to build. Its strength — creating many short-lived VMs fast — is exactly what we do not need |
| **Cloud Hypervisor** (directly, or via **Virtink**, a lightweight KubeVirt-like add-on built on it) | Modern, lean, Rust; supports hot-plug, `virtio-fs`, passthrough, even live migration. But libvirt's support for it is new, mainline KubeVirt does not use it, and Virtink's production use is far smaller. An optional entry in the bake-off, not the default |
| **libkrun** (Microsandbox) | Young; aimed at local and ephemeral sandboxes |
| **Plain containers (`runc`)** | Not a boundary between tenants: an agent with a shell is arbitrary code |
| **Hosted sandboxes** (E2B, Daytona, Modal, Northflank) | Priced per second for short sandboxes; always-on machines for every account are expensive, and not self-hosted |

**Why QEMU, which sounds old.** QEMU began as an emulator, but here it does not
emulate anything that matters for speed:

- **KVM**, in the Linux kernel, virtualizes CPU and memory. The guest's
  instructions run directly on the real CPU through VT-x / AMD-V.
- **QEMU** is only the VM manager: it creates the VM and provides devices, and
  those are paravirtual `virtio` disks and NICs, not emulated hardware.
- This QEMU/KVM pair is the standard behind OpenStack, Proxmox and many
  clouds.
- Compared with Cloud Hypervisor:
  - **CPU:** the same speed, because KVM does the work in both.
  - **I/O:** close, both through `virtio` (QEMU can add `vhost`).
  - **Boot:** Cloud Hypervisor is faster, but boot is rarely on our path.
- What KubeVirt gets from QEMU and libvirt — snapshots, hot-plug, passthrough,
  migration, guest-agent integration — matters more for an always-on machine
  than a leaner VMM.
- The cost is a larger attack surface. It is mitigated by enabling only
  `virtio` devices; KubeVirt also runs QEMU in an unprivileged pod under
  seccomp.

**k3s runs KubeVirt.** KubeVirt is an operator on any conformant Kubernetes:
an API, a controller, an agent on every node (`virt-handler`), plus CDI. Three
things to get right:

- **Storage:** k3s's built-in `local-path` is not enough. KubeVirt and CDI want
  block-mode volumes, cloning and snapshots: TopoLVM, or Longhorn.
- **Overhead:** KubeVirt's components cost roughly a few hundred MB to 1 GB for
  the cluster, plus a little per VM. It counts in the capacity math (§8).
- **Networking:** a VM's network goes through its launcher pod, so k3s's flannel
  and NetworkPolicy apply as for any pod.
- **Every node that runs VMs needs `/dev/kvm`.** It is native on bare metal and
  nested on WSL2 (§3.3).

**Harvester** (SUSE) — Kubernetes (RKE2) + KubeVirt + Longhorn + a VM web UI,
on its own OS — is the same stack already assembled. It is a reference design
and a fallback if operating VMs ourselves proves the expensive part, but it is
heavier and less flexible than plain k3s + KubeVirt.

### 2.4 Start and resume, for reference

Published numbers, on other hardware; ours come from the bake-off.

| | Number |
|---|---|
| Kata, hypervisor alone | ≈ 0.2 s (Cloud Hypervisor); a few seconds (QEMU) |
| Kata, full pod start (2026 third-party test) | ≈ 6.1 s (Cloud Hypervisor), ≈ 7.1 s (QEMU) |
| KubeVirt, QEMU domain start | ≈ 0.24–0.5 s p50 |
| KubeVirt, density | 100 VMs ≈ 3.3 min; 200 ≈ 10 min |
| KubeVirt, guest ready | No good published number. Tens of seconds with a stock cloud image; a few seconds expected with direct kernel boot and a minimal image, as WSL2 shows |
| Pause / resume | Near-instant for both: memory stays in place |
| Snapshot restore | Disk-level; restoring means a boot |

Start time is on a user's path only after a host reboot or an image rollout,
never in normal operation (§8). The measure that matters is **time from host
reboot until every tenant answers**.

### 2.5 Performance: configuring KubeVirt for near-native

A tenant's machine must be **as fast as possible** — CPU, disk and network —
because the v1 priority is experience parity (§3.1) and its main work is
building software.

**Why the pod around the VM costs almost nothing.** A KubeVirt VM is a QEMU
process inside an ordinary pod. A container is not a virtualization layer: it
is namespaces and cgroups around a normal process on the host kernel. So the
pod is not on the hot path:

- the guest's CPU instructions run on the real CPU through KVM;
- its memory is ordinary host memory;
- inside, the guest OS sees an ordinary VM, with every capability.

What the pod *does* touch is the host-side plumbing and accounting: the
network path, how the disk is attached, and the cgroup limits. That is where
performance is won or lost, and every setting below is a deliberate choice.
Field names follow KubeVirt's API and are checked against the version
installed.

**CPU**

| Setting | Choice | Why |
|---|---|---|
| CPU model | `host-passthrough` | The guest sees the host CPU's full feature set (AVX and the rest) — no lowest-common-denominator model. Live migration then needs nodes with the same CPU, which v1 does not do anyway |
| CPU limits | **No hard CPU limit.** A CPU request sizes scheduling | A hard limit makes the kernel's CFS throttle the VM when its quota runs out mid-build, causing latency spikes. Agents mostly wait on a model, so vCPUs share well |
| Dedicated CPUs | Off by default; an option for a premium tier (`dedicatedCpuPlacement`, Guaranteed QoS) | Pinning gives the most stable latency, but caps tenants at physical cores / vCPUs per tenant. Measured in the bake-off before being offered |
| NUMA | One socket per host (§12) | On a dual-socket machine, a VM whose vCPUs and memory sit on different sockets pays for every remote access. If a dual-socket host is used, KubeVirt's NUMA-aware placement keeps each VM on one socket |
| Host | CPU governor `performance`; deep C-states limited | Removes frequency ramp-up latency on interactive work |

**Memory**

| Setting | Choice | Why |
|---|---|---|
| Guest memory | Request = limit, sized per tier; no swap, no overcommit (§8) | The VM's memory is always there; no eviction under pressure |
| QEMU overhead | KubeVirt's computed overhead **plus a margin** above the guest's memory | The pod's memory limit covers the whole QEMU process, not only the guest. Too tight a limit lets the host's OOM killer **kill the whole VM** — the one real risk the pod adds |
| Huge pages | Measured in the bake-off (2 MiB pages, reserved on the host) | Fewer TLB misses for memory-heavy builds; costs flexibility, since reserved huge pages are not available to anything else |
| Host | Transparent huge pages on | A cheap gain for the QEMU processes |

**Disk**

| Setting | Choice | Why |
|---|---|---|
| Volume mode | **Block** (`volumeMode: Block`) on local NVMe through TopoLVM | The volume reaches QEMU as a raw block device. **Not** filesystem mode, which puts the VM's disk image as a file on the host's filesystem — an extra layer |
| Thin vs. thick LVM | Thin (snapshots need it), with its overhead measured; thick for a tier that wants the last few percent | Thin provisioning adds metadata work on first writes; snapshots are the price of "restore my computer" |
| Bus | `virtio` (`virtio-blk`) | Paravirtual; no emulated SATA or SCSI controller |
| Cache and I/O | `cache: none` with native AIO (`io: native`) | O_DIRECT: no double caching in the host's page cache, and writes are on the device when the guest is told they are. This is the fast *and* safe pair for block devices |
| I/O threads | A dedicated I/O thread per disk (`dedicatedIOThread`) | Disk I/O does not contend with QEMU's main loop |
| Queues | Multi-queue block (`blockMultiQueue`), one queue per vCPU | Parallel I/O from parallel build jobs |
| Guest filesystem | ext4 (or xfs), with discard / periodic `fstrim` | Freed blocks return to the thin pool |
| Never | `virtio-fs`, or disk images on network storage, for a tenant's root | The slow paths for small-file work (`node_modules`, `git`) |

**Network**

| Setting | Choice | Why |
|---|---|---|
| Binding | `masquerade` on the pod network | KubeVirt's recommended binding for the pod network: works with NetworkPolicy and future migration. Bridge binding on the pod network has caveats; a secondary network (Multus bridge or SR-IOV) is the route if a tier ever needs line rate |
| Acceleration | `vhost-net` in use (requires `/dev/vhost-net` on the node, checked at node setup) | Packet processing in the host kernel instead of in QEMU's userspace |
| Queues | `networkInterfaceMultiqueue` | Parallel network queues, one per vCPU |
| MTU | Set end to end for the overlay in use: none within one node; WireGuard's lower MTU across nodes | Avoids fragmentation, which silently halves throughput |
| Data path | The Octopus site reaches each tenant's agent over the pod network on the same node | No hop through the k3s API server (§3) |

**The guest**

- **Direct kernel boot**, a minimal image, no cloud-init: seconds to boot (§2.4).
- `virtio` drivers throughout.
- `qemu-guest-agent`, for health checks and for freezing the filesystem during a
  snapshot.
- `kvm-clock` for time.
- Only the services the machine needs.

**Verified, not assumed.** The bake-off (§3.2) measures every one of these
against the host itself, not only against `LocalRuntime`:

- disk: `fio` random and sequential, IOPS and latency, inside the VM vs. on the
  host NVMe;
- network: `iperf3` and request latency to the model APIs;
- the application benchmarks.

Each knob above that is marked "measured" is an explicit row in the bake-off
matrix. Targets are set before the run, not after: disk within a few percent of
the host device, network within a few percent of the host, builds within the
§3.1 margin.

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

KubeVirt is the default (§2.3); the bake-off validates it on the real box
against `LocalRuntime`, before the decision is final.

**Entries:**

- **KubeVirt**, with an optimized guest: direct kernel boot, a minimal Ubuntu
  image without cloud-init, systemd with only what we need, the disk already on
  a local block volume.
- **Kata**, the fallback, with both the volume and the root filesystem on block
  devices — not the defaults.
- **gVisor**, as the lightweight baseline.
- Optionally **Virtink** (Cloud Hypervisor).

**Measures**, beyond the gate of §3.1, with the configuration of §2.5 and its
"measured" knobs as rows of the matrix:

- `fio` and `iperf3` inside the VM against the bare host;
- a large `git clone` and a full build of Octopus, with cold and warm caches;
- idle memory per tenant;
- **time from host reboot until every tenant answers**, and one machine's
  restart time;
- **fault injection** during a multi-day soak:
  - kill a VM's hypervisor process;
  - restart k3s;
  - reboot the host;
  - fill a tenant's disk;
  - exhaust a guest's memory.

  For each, record the time to recover and confirm the disk is intact.

KubeVirt becomes final if it passes the gate. Otherwise the first entry that
passes takes its place, best reliability first, ties to the simplest to operate.

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
| The tenant's machine | By default a KubeVirt `VirtualMachine` with run strategy *Always* (a `StatefulSet` with `runtimeClassName: kata` if the fallback wins, §3.2). It stays running (§8); it is stopped only when an account is disabled or a node is drained. |
| Root disk (`DataVolume` / PVC) | The machine's whole disk — OS, installed software, `/home/tenant` — cloned per account from the base image by CDI (§6). It outlives any VM; a restarted VM boots from the same disk. |
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

With KubeVirt, the tenant's **whole machine** lives on its disk: the OS and
everything installed on it, and under `/home/tenant` the workspace, agent
memory, Application code and data, uploads, **and the CLIs' own state** —
Claude's and Codex's logins and transcripts. (With the Kata fallback, only the
mounted volume persists, §2.3.) Losing it, or
starting a machine without it, signs the tenant out of everything
(`runtime-providers.md` §5). The conformance case `state_survives_release`
exists for this.

- **Shared pool (Linux box):** an LVM-backed CSI driver (TopoLVM or OpenEBS
  LocalPV-LVM) on the box's NVMe — one logical volume per tenant. Quotas are
  the volume size; snapshots are cheap and atomic, which gives backups and
  "restore my computer to yesterday". It supports the block-mode volumes and
  cloning KubeVirt's CDI needs. k3s's bundled `local-path` provisioner has
  neither, so it is not the serving answer.
- **Session history** lives here too. Octopus writes it, as it does today, but
  through a storage operation of the Runtime interface; Octopus's own site keeps
  only metadata, the Ledger and rebuildable indexes (Octopus `docs/roadmap.md`
  item 2).
- **Storage/compute separation is the recovery model.** The VM process is
  disposable; the disk is the product. A crashed VM restarts and boots from
  the same disk.
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

- **CPU:** 8–16 cores with VT-x or AMD-V enabled in the BIOS, **on a single
  socket**. A dual-socket machine adds NUMA effects that need explicit placement
  (§2.5); a single socket avoids them.
- **Memory:** 128 GB preferred — v1 holds every tenant's memory with no
  overcommit (§8), so memory sets capacity. Used workstations take cheap DDR4
  ECC.
- **Disk:** 1–2 TB NVMe for tenant volumes, plus somewhere to send snapshots.
- **Form factor:** a tower workstation (Dell Precision, Lenovo ThinkStation, HP
  Z class) — rack servers are too loud for a house. A UPS for clean shutdowns.

## 13. Open questions — for discussion

1. **The isolation technology, final** — KubeVirt by default, confirmed or
   replaced by the bake-off (§3.2).
2. **Virtualization and the persistent disk, in detail** — the next design to
   write, once (1) is final:
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
4. **Storage driver** — TopoLVM vs. OpenEBS LocalPV-LVM vs. Longhorn, under
   CDI.
5. **Updates on a machine that diverges.** Each account's OS changes as the user
   uses it, so "update by rebuilding the image" no longer works. Expected
   policy, to settle:
   - OS security updates inside each VM (unattended upgrades, with reboots
     scheduled out of hours);
   - our agent and the CLIs (Claude, Codex) updating themselves in place;
   - the base image affecting only new accounts.
6. **Root inside the tenant's machine** — it is the tenant's own machine, and
   package installs want root; the boundary is the VM.
7. **Direct access for the user** — a terminal or SSH to their own cloud
   computer, not only through their agents. Natural for a cloud computer; a
   product and security decision.

## 14. Order of work

1. **Agent + provider + conformance, no cluster.** `octopus-runtime-agent` and
   `KubeRuntime`'s execution path, with tenant machines replaced by a local
   process per tenant — pass the Octopus conformance suite first.
2. **The box and the bake-off.**
   - k3s, KubeVirt (the default) and the other entries on the real box.
   - The parity benchmark of §3.1 against `LocalRuntime`.
   - Confirm KubeVirt, or take the fallback (§3.2).
   - If the operator's PC is to be a node, the nested-virtualization run of
     §3.3 is part of this step.
3. **Virtualization and disks in detail** (§13 item 2), for the confirmed
   technology; the controller with its per-tenant objects.
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
  (`VirtualMachine`), with disks on persistent volumes; built on libvirt and
  QEMU/KVM.
- **CDI / DataVolume** — KubeVirt's Containerized Data Importer; a DataVolume is
  a VM disk it imports or clones onto a PVC.
- **QEMU** — the VM manager that creates VMs and provides their `virtio`
  devices; with KVM it emulates no CPU.
- **Cloud Hypervisor, Firecracker** — leaner VM managers on KVM; Firecracker is
  built for short-lived serverless VMs.
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
