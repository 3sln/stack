# Shape: the service

A backend that answers `Request → Response`. No DOM, no dodo — but the same
provider graph, the same declared dependencies, and increasingly the same
Actions.

This shape is the newer half of the stack. Treat what follows as a direction of
travel with one route fully migrated, not as forty routes already written this
way.

## Speak the platform's own types

A handler takes a `Request` and returns a `Response`. Nothing else. That single
choice is what lets the same server run on Node, Bun and Workers with a thin
adapter per runtime, and it costs nothing to adopt on day one.

```
adapters/node.js     ── http.createServer, Readable.toWeb ──┐
adapters/bun.js      ── Bun.serve ───────────────────────────┼──▶ createServer()
adapters/worker.js   ── export default { fetch } ───────────┘
```

Adapters are also where runtime-specific capabilities are *registered*, and the
registration is what keeps the bundle honest:

```javascript
// This runtime HAS a filesystem, so it registers the filesystem driver.
// Imported from the deep path rather than the barrel: that import is what
// pulls in node:fs, and the Workers adapter deliberately never makes it — so
// there, Filesystem is absent from the config form and absent from the bundle.
import { filesystemDriver } from '@3sln/trove/core/storage/filesystem.js';
```

## Two lifetimes

A service has resources that live as long as the process and resources that live
as long as one request. Do not conflate them.

| Lifetime | What | How |
| --- | --- | --- |
| Process | storage, metadata, search, KV, identity | lazy singletons in the container |
| Request | authorized handles, claims, transactions | `container.lease()`, released with the request |

The backbone is obtained up front and shared. Everything scoped is leased:

```javascript
export function leaseScope(container, principal, grant = null) {
  const held = [];
  const obtain = async (name, request) => {
    const lease = await container.lease({ [name]: { principal, grant, ...request } });
    held.push(lease);
    return lease.resources[name];
  };
  return {
    access: {
      node: (id, capability, opts) => obtain('node', { id, capability, ...opts }),
      collection: (id, capability) => obtain('collection', { id, capability }),
    },
    // Every lease gets released, including the ones after a release that threw.
    // A sequential `for (…) await l.release()` leaks the rest of the list on
    // the first failure — precisely the moment leaking hurts most.
    release: async () => {
      const all = held.splice(0);
      await Promise.allSettled(all.map(l => l.release()));
    },
  };
}
```

Write that once and share it across **every** request surface. An HTTP router
needs it; so does an MCP endpoint, because an agent holding Alice's token is
Alice, and there must be no second, MCP-shaped way around the ACL.

## Authorization is a provider

The most valuable thing this shape gets from ngin. Rather than *stat the node,
look up its collection, assert the capability, then operate* — four steps at
twenty-six call sites — the grant is the object you operate through, obtained by
the container before `execute` runs.

Read [Ethos §3](/?c=%2Fethos.md) and
[Providers → Providers as policy](/?c=%2Fcore%2Fproviders.md). The short version:
a `read` handle has no `remove`, and a route never holds a raw id it could take
somewhere else.

## Routes are the surface

A route parses, calls, and serialises. It holds no logic and no invariants.

Two consequences worth stating, because both were violated before they were
written down:

- **A route must not reach past a façade** into the thing behind it. When a rule
  spans two stores — write the tag, then mirror it for querying — that rule lives
  in one method on the façade, not as two writes at each call site with the
  second one `.catch(() => {})`-swallowed.
- **Errors funnel through one place.** A domain error becomes the right status
  and JSON body; anything unexpected becomes a clean 500 that leaks nothing.

Guards that *are* route business — a key must live under this publication's
prefix; a published artifact is frozen — read well as small named functions at
the top of the route module.

## Actions, and where Queries stop making sense

Work that a route must **answer about immediately and finish later** is an
Action with a [feed](/?c=%2Fcore%2Ffeeds.md). That covers scans, imports,
transcodes, anything resumable.

Queries mostly do not fit here. A one-shot HTTP read is not a subscription, and
running it through a boot/kill store buys nothing. The exception is a long-lived
server object clients genuinely watch — a Durable Object, an SSE stream — where
boot/kill is exactly right.

## Background work is a provider too

"Run this later" differs per platform: a queue, `waitUntil`, a Durable Object, a
timer. Make it an interface with an implementation per runtime, and let the
absent case be a null implementation rather than a branch in the domain.

```javascript
const faststartQueue = env.FASTSTART_QUEUE
  ? new CloudflareQueueProvider(env.FASTSTART_QUEUE)
  : new NullQueueProvider();
```

## Migrating an existing server

The container earns its place fastest on a server that has grown a long
`createServer` — because that function is three hand-maintained lists wearing a
trench coat: build order, teardown order, and what is eager.

Do it one route at a time, and keep `createServer` as a facade so the existing
tests drive the new code unchanged. See [Testing](/?c=%2Fcore%2Ftesting.md).

Know the two costs before you start:

- **A dispatch costs a turn of the event loop.** Anything that used to be
  decided before a function returned is now decided one turn later.
- **Wrapping instances is not the same as owning them.** Half a graph of
  pre-built singletons wrapped in providers is an honest first step, but it does
  not yet demonstrate teardown ordering or laziness — which is where the
  remaining value actually is.
