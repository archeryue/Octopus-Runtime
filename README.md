# Octopus-Runtime

An independent platform that runs **one isolated machine per tenant** on bare
metal — its own controller, admin console and release cadence — and implements
[Octopus](https://github.com/archeryue/Octopus)'s **Runtime standard**, so
Octopus can run each account's sessions, background tasks and Applications on
a machine of that account's own. Octopus defines the standard and runs fully
without this; this platform follows the standard.

> **Status:** design. Nothing here runs yet. The interface it implements is
> specified in Octopus under
> [`docs/plans/runtime-providers.md`](https://github.com/archeryue/Octopus/blob/main/docs/plans/runtime-providers.md);
> how this repository implements it is [`docs/design.md`](docs/design.md) —
> k3s with Kata Containers on a bare-metal box, one lightweight VM per account,
> an owner's own machine joinable as a node, a controller and an admin console.

## Why this exists

Octopus runs on your own machine, as you, against your own subscription — and
that stays true. `LocalRuntime`, the default provider, is a plain subprocess and
is nobody's second-class citizen.

But the same product on a shared host has no trust boundary: every account's
agent runs as the same OS user, so any tenant can read any other tenant's files
and credentials. Octopus records that as a deliberate choice rather than an
oversight, and names what replaces it — *a container or VM per tenant*. This is
that.

## Shape

One runtime per **account**, not per session. A session is cheap to open
because the machine is already there; an Application backend has somewhere to
live that outlives the session that built it; and the cold start nobody wants
on the critical path of opening a chat simply is not on it.

What it has to provide is small, because Octopus's local-execution assumption
turns out to be concentrated rather than spread:

- **`exec`** — one primitive covering every shape Octopus spawns: a streaming
  turn with bidirectional stdin, a one-shot that must see EOF, a detached shell
  command that outlives its turn, and a long-running service.
- **`expose`** — a port inside the runtime plus an address the server can dial,
  because Application backends are reverse-proxied and "ask the OS for a free
  port" stops being true the moment the server and the child are not the same
  machine.
- **`callback_base`** — the other direction: the address work *inside* the
  runtime uses to reach the server back. Every MCP tool call a CLI makes, and
  every Application backend that talks to an agent, goes there — and on a
  separate machine `127.0.0.1` is the runtime itself, not Octopus.
- **`confine`** — the working-directory boundary, which in a per-tenant runtime
  is structural rather than a check: the machine only has that tenant's disk.
- **`ensure` / `release`** — start the account's machine before any work and
  put it away when nothing is left running. Octopus decides when, from facts it
  already has; the provider makes it fast.

One rule runs through all of it: **an execution's environment is exactly what
Octopus passes in.** A provider adds nothing from the server's own
environment — on a shared host that environment is the operator's keys.

The conformance suite in Octopus is the contract. A provider conforms when it
passes it, not when it matches this README.

Deliberately **not** in the first version: file primitives. Under a local
provider they are overhead; under a remote one, every caller that reaches for
`Path` breaks loudly and immediately, which is the right failure.

## Non-goals

- **Replacing `LocalRuntime`.** Running Octopus on a laptop must never require
  any of this.
- **Steering Octopus.** Octopus defines the Runtime standard and is developed
  without regard to this repository; this platform follows the standard's
  versions and proves conformance with Octopus's own suite.

## License

MIT — see [LICENSE](LICENSE).
