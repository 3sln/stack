# Shape: the headless process

A CLI, a daemon, a worker loop, a background service. No DOM and no request
cycle — just a process that starts, holds resources, does work, and shuts down.

This is the shape that shows most clearly that ngin is not a frontend library.

## The process is a Query

The insight that makes this shape click: a long-running process **is** a
subscription. It boots, it emits status over time, and it is torn down. That is
precisely a Query's lifecycle, so model it as one rather than as a `main()` with
a `while` loop and a pile of closure variables.

```javascript
// A single long-lived Query that boots the whole P2P tether and `notify`s a
// status object the CLI renders. All resources are pre-obtained by ngin and
// released when the query is killed. `this.kill` tears everything down.
export class Session extends Query {
  static deps = ['agent', 'signaling', 'frontend', 'registry'];

  async boot({ agent, signaling, frontend }, { notify }) {
    notify({ state: 'starting' });
    // …wire the pieces, notify on every state change…
    this.kill = async () => { /* close in one place */ };
  }
}
```

What you get for free is the part that is otherwise always slightly wrong:
teardown. Every resource the session needed was obtained by the container and is
released when the query is killed, in reverse order, whether the process is
exiting cleanly, crashing, or being restarted.

The rendering side is then trivial and completely dumb — subscribe, print:

```javascript
engine.query(new Session(name)).subscribe({ next: status => ui.render(status) });
```

## Providers wrap the operating system

Everything the process touches: a spawned child process, a socket, a WebRTC
peer, a JSON registry on disk, a service manager, an env file.

```javascript
// Business logic only ever sees these through `obtain()` — it never imports
// werift, the OpenAI SDK, or Bun.spawn directly. That's the whole point: swap
// any provider (local STT, a different agent runner, a different peer
// implementation) without touching logic.
export class FrontendProvider extends Provider {
  constructor() { super(); this.controller = new FrontendController(); }
  async obtain() { return this.controller; }
}
```

The payoff here is sharper than in a browser app, because a headless process is
otherwise the hardest thing to test: with the platform behind providers, the
session logic runs in a plain unit test with a fake child process and a fake
socket.

## The surface is still dumb

A terminal is a rendering target like any other. The rule does not change: the
thing that prints does not decide anything. Give it a status object and let it
format.

The corollary is that **the same domain drives more than one surface**. A status
object rendered as CLI output, a QR code, an HTTP status endpoint and an MCP
tool response is one query with four subscribers — not four code paths.

## Long-lived resources across reconnects

The distinction this shape forces you to be precise about: which resources
survive an interruption and which are minted fresh.

> A fresh WebRTC peer is minted per connection (peers are single-use), so a
> daemonized session survives unlimited reconnects. The agent process and the
> signalling socket persist across them.

That is a lifetime question, which means it is a provider question. A per-
connection resource is leased per connection; a per-session resource is a
singleton in the session's graph. Once those are separated properly, "survives
reconnects" stops being a feature you implement and becomes a consequence of
where each provider sits.

## Shutdown is a dependency

Do not close over a `closing` flag. Make it a resource, so an action or query
that must stop when the process is going down has to declare it:

```javascript
static deps = ['vfs', 'tasks', 'lifecycle'];
```

Then `lifecycle.closing` sits alongside `signal.aborted` and a user-initiated
cancel as [three lines meaning the same thing in one
place](/?c=%2Fcore%2Ffeeds.md) — and you can grep for everything that
participates in shutdown.
