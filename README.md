# Octopus-Runtime

A **Runtime provider** for [Octopus](https://github.com/archeryue/Octopus): it
gives each account a durable machine of its own, and Octopus runs that
account's sessions, background tasks and Applications inside it.

> **Status:** design. Nothing here runs yet. The interface it implements is
> specified in Octopus under
> [`docs/plans/runtime-providers.md`](https://github.com/archeryue/Octopus/blob/main/docs/plans/runtime-providers.md).

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
- **`confine`** — the working-directory boundary, which in a per-tenant runtime
  is structural rather than a check: the machine only has that tenant's disk.

Deliberately **not** in the first version: file primitives. Under a local
provider they are overhead; under a remote one, every caller that reaches for
`Path` breaks loudly and immediately, which is the right failure.

## Non-goals

- **Replacing `LocalRuntime`.** Running Octopus on a laptop must never require
  any of this.
- **A general sandbox platform.** The target is Octopus's protocol, not an
  arbitrary workload API.
- **Deployment.** Host inventory, identity, network policy and egress belong to
  whoever runs it, not here.

## License

MIT — see [LICENSE](LICENSE).
