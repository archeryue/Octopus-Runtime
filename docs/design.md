# Octopus-Runtime — design

> **Status:** draft for discussion (revised 2026-11-08). Nothing here is built. The
> interface this implements is `server/runtime/base.py` in Octopus
> (`PROTOCOL_VERSION = "runtime-1.0"`), and the conformance suite there
> (`server/runtime/conformance.py`) is the contract.

## 1. What this is

**An independent platform for running isolated machines on bare metal** — it
turns a few servers into one always-on cloud computer per tenant, with its own
controller, its own admin console and its own release cadence. Implementing **Octopus's Runtime standard** is one of the
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
  standard — file operations (§12), an `ensure` that reports an outage, session
  history kept in the runtime — is proposed to Octopus and lands there, with its
  `LocalRuntime` implementation, before it is consumed here.

## 2. Technology choice

**Proxmox VE (PVE) on bare metal runs every tenant's VM. k3s runs inside one or
two PVE VMs, for everything that needs containers.** Each layer does the one
thing it is best at.

### 2.1 What a tenant's machine is: a cloud computer

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
time barely matters here, because a tenant's machine does not stop (§9).

### 2.2 The design: PVE for VMs, k3s inside for containers

| Layer | Runs | Driven by |
|---|---|---|
| **PVE**, installed on the bare-metal machine | Every account's cloud computer: a full VM (Ubuntu with systemd), its disk, snapshots, backups, firewall | The controller, through PVE's REST API |
| **k3s**, in one or two PVE VMs | Everything that needs containers: the Octopus site, the controller, the admin console, monitoring — and later, Octopus Application backends in containers | The Kubernetes API, when something needs a container |

Why this split:

- **Tenant VMs sit directly on the hypervisor.** A VM's NIC is on a host bridge
  and its disk is a block device on local NVMe. No pod network or cgroup layer
  wraps it, so R4 is met with the least configuration (§2.6).
- **PVE is built for exactly R1, R2 and R5:**
  - VM templates and per-account disk clones;
  - snapshots;
  - **Proxmox Backup Server (PBS)**: incremental, deduplicated, verifiable
    backups;
  - a host and per-VM firewall;
  - a web UI for the operator;
  - long production use in homelabs and small and medium businesses.
- **Containers are not lost.** k3s keeps the Kubernetes ecosystem —
  deployments, services, ingress, Helm — for the workloads that want it.
- **The added complexity is small.** Two systems, but each does only what it is
  best at, and both have mature Go clients for the controller (§3.4).

### 2.3 The options weighed

| Option | Verdict |
|---|---|
| **PVE for VMs + k3s in a VM for containers** | **Default.** Best fit for R1–R5 with the least machinery between a tenant and the hardware; backups and operations solved; containers kept |
| **k3s + KubeVirt** (VMs as Kubernetes objects) | The previous default. Meets R1–R5, and its management plane failing does not stop running VMs. But it puts a pod, the pod network and cgroup accounting around every VM; storage needs a block-capable CSI driver plus CDI; backups must be assembled. **Kept for one future use:** a provider for enterprises whose platform is already Kubernetes or OpenShift (OpenShift Virtualization is KubeVirt), e.g. Kraken private deployments |
| **Incus** (community fork of LXD) | API-first, projects as tenant boundaries, storage-pool snapshots, fewer layers. Close to PVE in fit; PVE wins on backups (PBS), operator UI and breadth of production use |
| **Harvester** (SUSE: RKE2 + KubeVirt + Longhorn) | The KubeVirt route already assembled. Heavier, and still pod-wrapped VMs |
| **Kata Containers** | A container inside a small VM. **Fails R2**: the container's root filesystem is rebuilt from the image on every restart. Weak on R3: a minimal guest, not a full OS. Builds are fast only if both the volume and the root filesystem are on block devices |
| **gVisor** | A user-space kernel, not a VM. Slower on file- and process-heavy work (R4), with system calls that sometimes differ; a container (R2, R3); weaker isolation |
| **Firecracker** | Built for short-lived serverless VMs: very fast boots, very few devices, no hot-plug or GPU, its own orchestration to build. Its strength is what we do not need |
| **Cloud Hypervisor** (or Virtink, built on it) | Modern and lean, but newer in libvirt, not used by mainline KubeVirt or PVE, and with far less production use |
| **libkrun**, plain containers, hosted sandboxes (E2B, Daytona, Modal, Northflank) | Young and ephemeral; not a boundary; or per-second pricing and not self-hosted |

### 2.4 Why QEMU, which sounds old

PVE, like KubeVirt, runs VMs with QEMU on KVM. QEMU began as an emulator, but
here it emulates nothing that matters for speed:

- **KVM**, in the Linux kernel, virtualizes CPU and memory. The guest's
  instructions run directly on the real CPU through VT-x / AMD-V.
- **QEMU** is only the VM manager: it creates the VM and provides its devices,
  which are paravirtual `virtio` disks and NICs, not emulated hardware.
- This pair is the standard behind PVE, OpenStack and many clouds.
- Compared with Cloud Hypervisor:
  - **CPU:** the same speed, because KVM does the work in both.
  - **I/O:** close, both through `virtio`, and QEMU can add `vhost`.
  - **Boot:** Cloud Hypervisor is faster, but boot is rarely on our path.
- What QEMU brings — snapshots, hot-plug, passthrough, migration,
  guest-agent integration — matters more for an always-on machine than a leaner
  VMM.
- The cost is a larger attack surface, kept down by giving tenant VMs only
  `virtio` devices.

### 2.5 Start and resume, for reference

- A minimal Ubuntu VM on PVE (standard firmware boot, no cloud-init, few
  services) is ready in **roughly ten seconds**. This is to be measured; WSL2
  shows the floor is lower with direct kernel boot.
- Pause and resume are near-instant: memory stays in place.
- Snapshot restore is disk-level and means a boot.

Start time is on a user's path only after a host reboot or an image rollout,
never in normal operation (§9). The measure that matters is **time from host
reboot until every tenant answers**.

### 2.6 Performance: configuring tenant VMs for near-native

A tenant's machine must be **as fast as possible** — CPU, disk and network —
because the v1 priority is experience parity (§3.1) and its main work is
building software. Every setting is a deliberate choice, set on the template and
checked by the controller. Option names follow PVE's and are checked against
the installed version.

**CPU**

| Setting | Choice | Why |
|---|---|---|
| CPU type | `host` | The guest sees the host CPU's full feature set (AVX and the rest). Live migration then needs nodes with the same CPU, which v1 does not do |
| CPU limit | None; vCPUs share the host's cores | A hard cap throttles a build mid-run. Agents mostly wait on a model, so vCPUs share well |
| Pinning | Off by default; an option for a premium tier (CPU affinity) | Most stable latency, but caps tenants at physical cores per vCPU count. Measured first |
| NUMA | One socket per host (§13). On a dual-socket host, PVE's NUMA option keeps each VM's vCPUs and memory on one socket | A cross-socket VM pays on every remote memory access |
| Host | CPU governor `performance`; deep C-states limited | No frequency ramp-up latency on interactive work |

**Memory**

| Setting | Choice | Why |
|---|---|---|
| Guest memory | Fixed per tier; **ballooning off**; no swap, no overcommit (§9) | The memory is always there |
| Huge pages | Measured (2 MiB, reserved on the host) | Fewer TLB misses for memory-heavy builds; costs flexibility |
| Host | Transparent huge pages on; memory kept for the host, PBS and the k3s VMs | The hypervisor and the container layer must never starve |

**Disk**

| Setting | Choice | Why |
|---|---|---|
| Storage | Local NVMe, as **LVM-thin** or **ZFS** (§6; chosen by measurement) | Block storage on the fastest device. Both snapshot |
| Controller and bus | **VirtIO SCSI single** with an **I/O thread** per disk (or `virtio-blk`, measured) | Paravirtual; disk I/O off QEMU's main loop |
| Cache and I/O | `cache=none`, with `io_uring` or `native` AIO (measured) | O_DIRECT: no double caching in the host page cache, and writes are on the device when the guest is told they are. Fast and safe together |
| Discard | `discard=on`, with SSD emulation; `fstrim` in the guest | Freed blocks return to the thin pool |
| Guest filesystem | ext4 or xfs | Ordinary, fast, well understood |
| Never | A tenant's root disk on a NAS or network share (§8) | Network latency on every small file: `node_modules`, `git` |

**Network**

| Setting | Choice | Why |
|---|---|---|
| NIC | `virtio`, with **multiqueue** (one queue per vCPU) | Paravirtual; parallel queues |
| Acceleration | `vhost-net` (PVE's default with KVM) | Packet processing in the host kernel |
| Bridge | Tenant VMs on a dedicated bridge or VNet (§7) | No container network in the path |
| MTU | Consistent end to end | Fragmentation silently halves throughput |

**The guest**

- A minimal Ubuntu template: few services, `virtio` drivers throughout.
- `qemu-guest-agent`: health checks, clean shutdowns, and freezing the
  filesystem during snapshots and backups, so SQLite files are consistent.
- `kvm-clock` for time.

**Verified, not assumed.** The validation run (§3.2) measures each setting
marked "measured" as a row of its matrix:

- `fio` inside the VM against the bare NVMe: random and sequential, IOPS and
  latency;
- `iperf3` and request latency to the model APIs;
- the application benchmarks.

Targets are set before the run: disk and network within a few percent of the
bare host, builds within the §3.1 margin.

## 3. v1: experience first, then topology

### 3.1 The v1 priority is experience parity

Early on, cost does not matter much. What matters is that a session on
Octopus-Runtime feels **the same as on today's `LocalRuntime`: the same latency,
the same stability.** That decides several things:

- **No resource reclaim in v1.** `release` is a no-op; a tenant's machine keeps
  all its memory all the time (§9).
- **No memory overcommit.** Each tier gets real memory, so there are no
  out-of-memory surprises from neighbours.
- **An acceptance gate.** v1 ships only if, on the same tasks, Octopus-Runtime is
  within a set margin of `LocalRuntime` on each of these:
  - time to first token;
  - turn latency;
  - a tool-heavy task's total time (`npm install` plus a test suite);
  - stability over a multi-day soak.

  The full Octopus e2e suite must also pass with zero failures.

### 3.2 The validation run

The design is validated on the real box before the decision is final.

- **Configurations measured:** the "measured" rows of §2.6 — LVM-thin vs. ZFS,
  `virtio-scsi-single` vs. `virtio-blk`, `io_uring` vs. `native`, huge pages,
  CPU pinning.
- **Measures:**
  - the gate of §3.1 against `LocalRuntime`;
  - `fio` and `iperf3` inside the VM against the bare host;
  - a large `git clone` and a full build of Octopus, with cold and warm caches;
  - **time from host reboot until every tenant answers**, and one VM's restart
    time;
  - **fault injection** during a multi-day soak. For each, record the time to
    recover and confirm the disk is intact:
    - kill a tenant VM's QEMU process;
    - restart the k3s VM;
    - reboot the host;
    - fill a tenant's disk;
    - exhaust a guest's memory;
    - restore a tenant from PBS.

PVE becomes final if it passes. If it does not, k3s + KubeVirt (§2.3) is the
fallback, measured the same way.

### 3.3 Topology

```
                          Internet
                             │  cloudflared (outbound tunnel)
 ┌───────────────────────────▼──────── bare-metal box: Proxmox VE ─────────────────┐
 │  ┌── k3s VM ─────────────────────────────┐                                       │
 │  │ Octopus site · controller · admin     │      PBS (on a second disk,          │
 │  │ console · monitoring · (later) app    │      or a second machine)            │
 │  │ containers                            │                                       │
 │  └───────────────────────────────────────┘                                       │
 │        │ tenant bridge / VNet (firewalled, §7)                                    │
 │   ┌── tenant A's VM ───────┐ ┌── tenant B's VM ───────┐                           │
 │   │ Ubuntu · octopus-      │ │ Ubuntu · octopus-      │   ...                     │
 │   │ runtime-agent · claude │ │ runtime-agent · claude │                           │
 │   │ codex · /home/tenant   │ │ codex · /home/tenant   │                           │
 │   └────────────────────────┘ └────────────────────────┘                           │
 └────────────────────────────────────────────────────────────────────────────────────┘
```

- **One PVE machine in v1.** It runs a k3s VM (the site, the controller, the
  admin console, monitoring), a VM for every account — the operator's included
  — and PBS on separate storage.
- **The operator's PC is a Device**, not a Runtime node. PVE is a whole operating
  system that owns its machine; it cannot run inside WSL2. Agents reach the PC
  through Octopus's Devices, allowed to go offline (Octopus
  `docs/plans/devices.md`).
- **More capacity** means more PVE machines joining a PVE cluster:
  - a cluster needs a quorum, so two machines need a small always-on QDevice
    as a third vote;
  - k3s can add a node VM on the new machine if containers need room.

### 3.4 Components in this repository

1. **`octopus-runtime-agent`** — a small static binary installed in the tenant
   template, the main service of every tenant VM. It implements `exec` (§5):
   spawn, stream, write stdin, signal a process group, report exit. Nothing
   else listens there.
2. **Controller** — the service Octopus talks to for lifecycle, running in the
   k3s VM.
   - It drives PVE through its REST API: clone, start, stop, firewall, backup
     jobs.
   - It runs Kubernetes workloads through the Kubernetes API when something
     needs a container.
   - It serves the admin console (§10).
3. **The provider** — `octopus_runtime.provider:PveRuntime`, a thin Python
   class implementing Octopus's `Runtime` protocol: lifecycle through the
   controller, execution straight to the tenant's agent. Octopus loads it with
   `OCTOPUS_RUNTIME_PROVIDER=octopus_runtime.provider:PveRuntime`.
4. **The tenant template** — a minimal Ubuntu VM template with the agent,
   `qemu-guest-agent`, `claude`, `codex`, Node, Python, `git` and common build
   tools.

**Languages: Go is this repository's main language.**

- **Go:** the agent, the controller and the admin console's backend.
  - The agent is one static binary.
  - The controller uses a Go PVE API client and `client-go` for Kubernetes.
- **The admin console's frontend** is a web app.
- **The provider is the one exception, and a temporary one.** Octopus loads a
  provider as a Python class in its own process, because its Runtime standard is
  a Python interface today. Until that changes, `PveRuntime` is a thin Python
  client of the controller and the agent, with no logic of its own.
  - When Octopus turns the standard into a wire protocol (Octopus's roadmap,
    item 2), Octopus ships a generic remote-Runtime client in its own
    repository.
  - This repository then only implements the protocol's server side, in Go,
    and contains no Python.

**Control path and data path are separate.** `ensure`, `release` and `expose`
go through the controller. An `exec` asks the controller once where the
tenant's agent is (its address on the tenant bridge, and a short-lived
credential) and then streams directly to the agent. A turn's stdout is the
hottest traffic in the system; routing it through the controller would make the
controller the bottleneck and a single point of failure for every running turn.

## 4. Per-tenant objects

For each account the controller maintains, in PVE:

| Object | Purpose |
|---|---|
| **VM** | The tenant's cloud computer, cloned from the template, with **start at boot** on so it returns after a host reboot. Stopped only when an account is disabled or a host is drained |
| **Disk** | The VM's whole disk — OS, installed software, `/home/tenant` — a full clone, so it does not depend on the template after creation (§6) |
| **Resource pool and tags** | Group the tenant's objects and record whose they are |
| **Firewall rules / security group** | What the VM may reach and who may reach it (§7) |
| **Backup job** | Its schedule and retention in PBS (§6) |
| **Agent credential** | For Octopus → agent calls, held by the controller |

## 5. The protocol, mapped

| Protocol call | What happens here |
|---|---|
| `ensure(tenant)` | Controller: confirm the tenant's VM is running and its agent answers. Normally immediate. A VM that is not running (after a host restart or maintenance) is started and waited for. Idempotent; concurrent calls share one wait. A host that is down is an outage, reported as one rather than hung on |
| `release(tenant)` | **A no-op in v1** (§3.1, §9). The VM keeps running with all its memory. Kept in the protocol so reclaim can be added later without a protocol change |
| `exec(spec)` | Provider → agent: start `argv` with exactly `spec.env` (nothing added), `cwd`, the stdin/stderr modes, its own process group if asked. Streams are multiplexed (stdin, stdout, stderr, signal, exit) over one WebSocket or gRPC stream |
| `Execution.signal(sig, group)` | A frame on that stream; the agent does `killpg` in the VM. Returns as soon as the frame is sent |
| `expose(tenant)` | Controller picks a free port on the tenant's VM, opens it in the VM's firewall for the site only, and returns `(port, "<VM address>:<port>")` |
| `callback_base(tenant)` | The Octopus site **as reachable from inside a tenant's VM**: its address on the tenant bridge, never `127.0.0.1` |
| `paths(tenant)` | The tenant's directories on its VM: `/home/tenant/{workspace,agents,applications,…}` |
| `confine(requested, tenant)` | Structural: anything that resolves inside `/home/tenant` is allowed; the VM has nothing else of the tenant's to protect. `extra_roots` is ignored |

Why our own agent instead of a VM's built-in guest agent: Octopus depends on two
behaviours a generic exec path does not give cleanly —

- **process-group signalling** — interrupting a turn must kill the CLI *and*
  every MCP child and sub-agent it started;
- **held executions** — a Claude process stays alive between turns, idle, with
  stdin open, and is written to again minutes later.

An agent we own implements exactly the protocol's semantics and is tested by the
same conformance suite.

## 6. Storage and backups: the tenant's disk is the product

A tenant's **whole machine** lives on its disk: the OS and everything installed
on it, and under `/home/tenant` the workspace, agent memory, Application code and
data, uploads, **and the CLIs' own state** — Claude's and Codex's logins and
transcripts. Losing it signs the tenant out of everything
(`runtime-providers.md` §5). The conformance case `state_survives_release`
exists for this.

- **Local NVMe, as LVM-thin or ZFS.** Both give fast block devices and
  snapshots. The choice is made by measurement (§3.2):
  - **LVM-thin** is lighter and faster on raw I/O;
  - **ZFS** adds checksums, compression and `send`/`receive` replication.
- **Full clones from the template**, so a tenant's disk stands alone.
- **Session history** lives on this disk too. Octopus writes it, as it does
  today, but through a storage operation of the Runtime interface; Octopus's
  own site keeps only metadata, the Ledger and rebuildable indexes (Octopus
  `docs/roadmap.md` item 2).
- **Backups with PBS.**
  - Incremental: changed blocks only, through QEMU dirty bitmaps.
  - Deduplicated, verifiable, optionally encrypted on the client.
  - `qemu-guest-agent` freezes the guest's filesystem for each backup, so
    SQLite and other files are consistent.
  - "Restore my computer to yesterday" is a PBS restore.
- **Backups must leave the machine.**
  - PBS runs on a second disk at least, better on a second machine.
  - A copy goes off site. An S3-compatible object store such as **Volcengine
    TOS** is the target: either PBS's S3-compatible datastore (added in PBS
    4.x; its maturity is checked in the installed version), or an `rclone` /
    `restic` copy of the PBS datastore.
- **NAS storage on the host** (NFS / SMB, native in PVE) is fine for ISO images,
  templates and backups. It is **never** used for a tenant's root disk: network
  latency on every small file defeats R4.
- **Storage/compute separation is the recovery model.** The VM process is
  disposable; the disk and its backups are the product. A crashed VM restarts
  from the same disk; a lost machine is restored from PBS onto another.

## 7. Network

The k3s VM and tenant VMs sit on a dedicated **tenant bridge or PVE SDN VNet**,
NATed to the internet. The PVE firewall enforces the rules per VM, set by the
controller through the API. Kubernetes NetworkPolicy does not reach PVE VMs, so
it is not used for tenants.

| Destination | Rule |
|---|---|
| Internet | Allowed — model APIs, packages, web pages, object storage |
| The Octopus site | Only the callback address (§5): MCP namespaces and the Application agent API, authenticated by Octopus's own per-session bearer |
| Another tenant's VM | **Denied** |
| The home LAN (router, NAS, the PC, …) | **Denied** — every RFC 1918 range except the callback address. A tenant's agent runs arbitrary code in your house. Per-tenant exceptions exist only for that tenant's own storage (§8) |
| PVE host, PBS, the k3s API | Denied |

- **Inbound to a tenant's VM:** only the agent's port and exposed Application
  ports, only from the site.
- **Anti-spoofing:** the firewall's IP and MAC filters are on, so a VM cannot
  claim another's address.
- **The public entry point** stays the one cloudflared tunnel to the site.
  Nothing else is reachable from the internet.

## 8. External storage for tenants: NAS and object storage

A user's cloud computer may need their own storage. This is the guest OS's
business, as on any Linux machine; PVE is not involved.

| Need | How | Note |
|---|---|---|
| The user's **NAS** | Mount NFS or SMB inside the VM | Blocked by default (§7). The controller adds a **per-tenant exception for that one address and port**, only for a NAS the user owns |
| **Object storage** (S3-compatible, Volcengine TOS) | `rclone mount` or `s3fs` for a directory, or the SDK / CLI directly | Over the internet, so §7 allows it; credentials stay in the tenant's own VM |

- **Neither is a place to work.** Network file systems and object-storage mounts
  are far slower than the local disk. They hold datasets, archives and large
  files; the workspace stays on the VM's own disk.
- **The alternative is an Octopus connector.** Octopus can expose a NAS or bucket
  to agents as a connector, through tools or `octo`, instead of a mount.
  Credentials are then managed centrally and access is recorded in the Ledger.
  Which suits depends on whether the user wants files that feel local or
  access on demand. That decision is Octopus's, not this platform's.

## 9. Lifecycle and capacity

- **A Runtime is always on.** Every tenant's VM keeps running: stopping it would
  end its held CLI processes and put a cold start in front of every scheduled
  fire and every first message. VMs start at boot.
- **No reclaim in v1.** `ensure` confirms the VM before every execution;
  `release` is a no-op (§3.1). Ballooning is off. Memory reclaim is deferred
  (§16).
- **No memory overcommit in v1.** Tiers by CPU / memory — 4 GB default, 8 GB
  larger. vCPUs may oversubscribe (agents mostly wait on a model); memory does
  not. On a 128 GB box, keeping roughly 16–24 GB for PVE, PBS and the k3s VM,
  that is about **12–25 accounts**. That is the honest early capacity, and it is
  enough.
- **A cold start happens only after a host restart, maintenance or a template
  rollout**, never in normal operation. Its path is still measured and kept
  correct, because recovery depends on it.
- **Hard limits** the controller enforces regardless of Octopus: max tenants per
  machine (from real memory), per-tenant disk quota.

## 10. Runtime nodes are always on

A Runtime is where sessions live, so a machine that hosts tenant VMs is **always
on**: its going down is an outage the platform recovers from, never a state the
design plans for.

- **A machine that should host sessions joins the PVE cluster** as a full PVE
  machine, held to "always on", and not treated as special because of who owns
  it.
- **A machine that comes and goes is a Device**, not a Runtime: the agent's
  hands, reached from its Runtime, allowed to be offline (Octopus
  `docs/plans/devices.md`).
- **No hybrid.** An offline-able home for sessions would need offline policies
  for turns, schedules and history. That is the complexity this rule exists to
  avoid.

## 11. Admin console

An operator-facing web UI served by the controller — separate from Octopus's own
Monitor page, which is about one install, not the fleet. PVE's own UI remains
for anything below this level, so this console can start thin.

- **Tenants** — every account: state (running / starting / failed), machine,
  tier, memory used vs. limit, disk usage, last backup, last active, running
  executions (turns, background tasks, Application backends).
- **Machines** — each PVE machine: tenants on it, CPU / memory / disk committed
  vs. physical, PBS health.
- **Events** — starts, failed starts, OOM kills, quota hits, failed backups, a
  machine going down, with the agent's last output for a failure.
- **Actions** — stop a tenant, resize, restore from backup, firewall
  exceptions (§8), drain a machine.
- **Metrics** — the parity measures of §3.1 over time, running tenants, memory
  committed vs. physical.

## 12. What Octopus must change before this runs

Steps 1–7 of `runtime-providers.md` §10 are done: every process goes through
`exec`, and Octopus never assumes a port or a callback address. Three gaps
remain, and each one breaks this provider immediately:

1. **Server-side file access** (`runtime-providers.md` §9). Octopus still reads
   and writes tenant files with `Path` on its own disk: `/showme`, uploads,
   large-prompt spills, agent memory provisioning, **Applications' static
   files**, and the full-copy fork. Here those files are on the tenant's VM. The
   protocol needs file operations (read / write / list / stat, streamed) — a
   `runtime-1.1` — implemented by `LocalRuntime` in Octopus first (on its own
   disk, so a standalone install keeps every feature) and by the agent here.
   This is the largest piece of work on the Octopus side.
2. **The environment allowlist** (step 8). Octopus still passes its own
   environment to the CLIs — correct on one box, but it would ship the
   operator's `OCTOPUS_AUTH_TOKEN` into a tenant's VM.
3. **Fork as a runtime step** (step 9) — the transcript copy runs on the
   tenant's VM.

Plus two more:

- **Session history through the Runtime's storage**, written by Octopus as today
  (§6).
- **An outage is reported, not hung on.** A Runtime is always on by assumption,
  but a machine can still fail; every caller — a turn, a scheduled fire, a
  delegation — reports it.

## 13. Hardware

No GPU; modest CPU; memory is what runs out first. The machine is **dedicated to
PVE**.

- **CPU:** 8–16 cores with VT-x or AMD-V enabled in the BIOS, **on a single
  socket**. A dual-socket machine adds NUMA effects (§2.6); a single socket
  avoids them.
- **Memory:** 128 GB preferred — v1 holds every tenant's memory with no
  overcommit (§9), so memory sets capacity. Used workstations take cheap DDR4
  ECC.
- **Disk:**
  - 1–2 TB NVMe for tenant disks;
  - **a second disk for PBS**, or PBS on a second machine;
  - an off-site copy to object storage (§6).
- **Form factor:** a tower workstation (Dell Precision, Lenovo ThinkStation, HP
  Z class) — rack servers are too loud for a house. A UPS for clean shutdowns.

## 14. Open questions — for discussion

1. **LVM-thin or ZFS** for tenant disks (§6, measured).
2. **PBS placement and off-site copy** — second disk vs. second machine; PBS's
   S3-compatible datastore vs. `rclone` / `restic` to TOS; retention.
3. **Updates on a machine that diverges.** Each account's OS changes as the user
   uses it, so "update by rebuilding the template" no longer works. Expected
   policy, to settle:
   - OS security updates inside each VM (unattended upgrades, with reboots
     scheduled out of hours);
   - our agent and the CLIs (Claude, Codex) updating themselves in place;
   - the template affecting only new accounts.
4. **Root inside the tenant's VM** — it is the tenant's own machine, and package
   installs want root; the boundary is the VM.
5. **Direct access for the user** — a terminal or SSH to their own cloud
   computer, not only through their agents. Natural for a cloud computer; a
   product and security decision.
6. **NAS exceptions** — how a user proves a NAS is theirs before the firewall
   opens to it (§8).

## 15. Order of work

1. **Agent + provider + conformance, no PVE.** `octopus-runtime-agent` and
   `PveRuntime`'s execution path, with tenant VMs replaced by a local process
   per tenant — pass the Octopus conformance suite first.
2. **The box.**
   - PVE installed; the tenant template; the tenant bridge and firewall rules.
   - The k3s VM, with the controller in it.
   - PBS on separate storage, and the off-site copy.
3. **The validation run** (§3.2): the configuration matrix, the gate against
   `LocalRuntime`, fault injection, a PBS restore.
4. **Octopus file operations and the history store** (`runtime-1.1`),
   implemented in `LocalRuntime` and the agent.
5. **Admin console.**
6. **Move the site** into the k3s VM (DB, master key, cloudflared tunnel), with a
   rollback path; migrate each account's data onto its VM.
7. **Full Octopus e2e and the gate of §3.1** with `OCTOPUS_RUNTIME_PROVIDER`
   pointing here. v1 ships on passing it.

## 16. What v1 defers

- **Resource reclaim and memory overcommit.** Cost is not the v1 priority.
  When it becomes one, two questions decide it:
  - does ballooning or free-page reporting keep idle VMs near their working
    set;
  - how far to overcommit, given KSM's cross-tenant side-channel concern.
- **High availability and live migration.** Both need shared or replicated
  storage (Ceph, or ZFS replication) and several machines.
- **A KubeVirt provider** for enterprises whose platform is Kubernetes or
  OpenShift (§2.3).
- **Fast ephemeral sandboxes** (Firecracker / libkrun) — not the shape of an
  always-on tenant machine.

## Glossary

- **KVM** — the Linux kernel's hardware virtualization. Native on bare metal;
  nested (slower) inside a VM such as WSL2.
- **QEMU** — the VM manager that creates VMs and provides their `virtio`
  devices; with KVM it emulates no CPU.
- **Proxmox VE (PVE)** — a Debian-based virtualization platform: KVM/QEMU VMs,
  LXC containers, clustering, firewall, SDN, a web UI and a REST API.
- **Proxmox Backup Server (PBS)** — incremental, deduplicated, verifiable
  backups for PVE VMs.
- **Template / full clone** — a VM image new tenant VMs are copied from; a full
  clone does not depend on its template afterwards.
- **LVM-thin / ZFS** — local block storage with snapshots; ZFS adds checksums,
  compression and replication.
- **Bridge / VNet** — the virtual network PVE attaches VMs to; a VNet is PVE
  SDN's managed version.
- **QDevice** — an external vote that keeps a two-machine PVE cluster in quorum.
- **qemu-guest-agent** — a service inside a VM that lets the host shut it down
  cleanly and freeze its filesystem for consistent snapshots.
- **k3s** — a small Kubernetes in one binary; here, inside a PVE VM, for
  containers.
- **KubeVirt** — runs VMs as Kubernetes objects; the alternative kept for
  enterprise Kubernetes platforms.
- **Kata Containers, gVisor, Firecracker, Cloud Hypervisor** — other isolation
  options weighed in §2.3.
- **Ballooning / free page reporting** — ways a VM hands unused memory back to
  the host, which is what makes memory overcommit possible (deferred, §16).
- **KSM** — the kernel merging identical memory pages across VMs; saves memory,
  but a known side-channel risk between tenants.
