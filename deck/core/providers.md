# Providers and the Container

A **Provider** owns one resource's lifetime. A **Container** owns the graph of
them. This is the only part of the stack that every application has, screen or
no screen.

## The one rule people get wrong

Providers and consumers are injected with **different things**.

| Injected into | Receives | You must |
| --- | --- | --- |
| A **Provider**'s constructor | the *dependency Providers* | call `.obtain()` yourself, and `.release()` when done |
| An **Action**, **Query** or **Interceptor** | the *obtained resources* | nothing — the container leases and releases them |

A provider gets providers because it manages lifetimes; a consumer gets
resources because it manages none. Treating an injected provider as if it were
the resource is the single most common mistake in a new module.

```javascript
import { Provider } from '@3sln/ngin';

export class DataProvider extends Provider {
  static deps = ['logger'];

  constructor({ logger }) {
    super();
    this.loggerProvider = logger; // a Provider, not a logger
  }

  async obtain() {
    const logger = await this.loggerProvider.obtain();
    logger.log('Data connection created.');
    this.loggerProvider.release(logger);
    return { fetchData: () => 'some data' };
  }

  async release(resource) { /* per-consumer teardown */ }
  async dispose() { /* permanent shutdown, on engine.dispose() */ }
}
```

`release` ends one consumer's lease. `dispose` happens once, at the end of the
world. Conflating them is how a resource gets torn down while someone is still
holding it.

## The two shapes you will actually write

Most resources in a real graph are singletons, and ngin has both forms so you
do not hand-roll them:

```javascript
// Already built. Handed out as-is; nothing to await.
Provider.fromSingleton(new PlayerStore())

// Built on first use, torn down on dispose. Use this when construction is
// async or can fail.
Provider.fromLazySingleton(() => initDB(), db => db.close())
```

Laziness is not a micro-optimisation here — it is error routing. An eager
`await db.init()` at construction turns a failure to open IndexedDB (private
mode, a blocked upgrade, exhausted quota) into an unhandled rejection at module
evaluation time. Lazily, it surfaces to whoever asked for the database.

`release` on a lazy singleton is deliberately a no-op: a consumer that finishes
with it has not finished the resource.

## Leasing

A consumer does not call `obtain` — it declares `static deps` and the container
leases for it. Where you need resources outside an action or query, lease
explicitly:

```javascript
// Manual: you hold it, you release it.
const lease = await container.lease({ node: { id, capability: 'read' } });
try { /* lease.resources.node */ } finally { await lease.release(); }

// Scoped: obtained, run, released — even if fn throws.
await container.use({ db: {} }, ({ db }) => db.query(sql));
```

If one obtain in a lease fails, the ones already obtained are released before
the error propagates. Partial leases do not leak.

## Instance deps: parameterising a provider

`static deps` names *what* you need. An instance's `this.deps` says *which one*:

```javascript
export class ScanCollection extends Action {
  static deps = ['vfs', 'tasks', 'lifecycle'];

  constructor({ collectionId = 'default' } = {}) {
    super();
    this.collectionId = collectionId;
    // How the claim provider learns what it is claiming.
    this.deps = { claim: { collectionId } };
  }
}
```

This is what makes a provider able to represent *a specific grant*, *a specific
lease*, *a specific connection* — rather than one global thing.

## Providers as policy

Because `obtain` runs before `execute`, a provider is the right place for any
rule that should stop work from happening at all:

```javascript
// Resolves the node, asserts the capability, and returns a handle whose
// methods are the ones that capability permits. Denial happens in `lease()`.
node: NodeAccessProvider
```

Two properties follow, and neither is available from an interceptor:

- It works identically for actions **and queries** — ngin runs interceptors on
  the dispatcher only, so a query would escape one entirely.
- The action never sees a raw id it could take somewhere else.

See [Ethos §3](/?c=%2Fethos.md).

## What the container derives so you don't

- **Build order** — from the declared graph. A cycle is refused by name.
- **Teardown order** — reverse construction order, always.
- **Laziness** — nothing is built until something needs it.

Every one of those was, at some point in one of these projects, a hand-written
list that had to stay in agreement with another hand-written list.
