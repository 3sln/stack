# Actions and Queries

Two verbs and one noun: **Actions** do something, **Queries** observe something.
Both declare `static deps` and receive already-obtained resources. Neither knows
what is above it.

## Actions

```javascript
import { Action } from '@3sln/ngin';

export class AddTodo extends Action {
  static deps = ['todos'];

  constructor(text) { super(); this.text = text; }

  async execute({ todos }, { dispatchFeed, engineFeed, state, signal }) {
    if (signal.aborted) return;
    await todos.add({ text: this.text });
    engineFeed.dispatchEvent(new CustomEvent('todos-updated'));
  }
}
```

The constructor takes the payload; `execute` takes the resources. That split is
what lets an action be constructed anywhere and executed only where the
container is.

### `dispatch` returns a feed, not a promise

This is the single sharpest edge in the library, and it fails quietly:

```javascript
await engine.dispatch(new AddTodo('milk'));   // ⚠️ resolves immediately
```

`dispatch` returns the `dispatchFeed` — an `EventTarget`. Awaiting it resolves
on the next tick, so callers run their follow-up work (navigate away, close the
dialog) before the action has touched the database, and an action that throws
fails silently.

Await the terminal event instead:

```javascript
// Resolves when the action completes; rejects with whatever it threw.
export const dispatch = action => engine.dispatch(action).next('complete');

// Fire-and-forget, for handlers with nothing to sequence after them —
// but failures are still reported rather than swallowed.
export const fire = action =>
  dispatch(action).catch(err =>
    console.error(`${action?.constructor?.name} failed:`, err));
```

Give every project those two helpers once, at the composition root, and no call
site has to remember.

### Actions cannot dispatch Actions

There is no engine in an action's dependencies, deliberately: action chaining is
how a call graph becomes impossible to follow. When work genuinely must trigger
work, emit on `engineFeed` and let something subscribed decide — or, for the
handful of cross-cutting cases in a client app (audio "ended" → next track,
post-create navigation), keep one bound engine at the composition root and
dispatch there.

## Queries

A query is booted on its **first** subscriber and killed after its **last**.

```javascript
import { Query } from '@3sln/ngin';

export class Todos extends Query {
  static deps = ['todos'];

  async boot({ todos }, { notify }) {
    notify(await todos.all());
    const onUpdate = data => notify(data);
    todos.subscribe(onUpdate);
    // Assign `this.kill` dynamically when cleanup needs the closure.
    this.kill = async () => todos.unsubscribe(onUpdate);
  }
}
```

The handle from `engine.query(...)` is a minimal RxJS-style observable with
`subscribe`, plus `peek` for a one-off read. An active query answers `peek` from
its last value; an inactive one is answered by `fetch`.

### Two things every live query needs

Both of these were absent from every query in one project until they caused
bugs, and they are worth building in from the start:

- **Ordering.** One action often fires several events, so refreshes overlap.
  Whichever read finishes last wins, which is not necessarily the newest one.
  Sequence them and let only the most recently started refresh notify.
- **Error containment.** A read that throws rejects the boot promise and kills
  the query silently, leaving the view stuck on its last value forever.

```javascript
function live(engineFeed, events, notify, fetcher) {
  let seq = 0;
  const run = async () => {
    const mine = ++seq;
    try {
      const value = await fetcher();
      if (mine === seq) notify(value);        // ordering
    } catch (err) {
      console.error('Query refresh failed:', err);  // containment
    }
  };
  for (const event of events) engineFeed.addEventListener(event, run);
  return { run, stop: () => { /* removeEventListener */ } };
}
```

### When a Query is the wrong tool

A boot/kill subscription store is built for something *watched over time*. A
one-shot read that answers a single HTTP request and is never observed again is
not that: you pay the whole subscription lifecycle to deliver one value.

Rule of thumb: **queries where there is a subscriber; plain resource methods
where there is a caller.** In practice this means client apps are full of
queries and stateless request handlers have almost none — but a long-lived
server object that clients watch (a Durable Object, a daemon's session) is
squarely back in query territory.

## Interceptors

Cross-cutting concerns around work — logging, transactions, auth, retries — with
`enter`, `leave` and `error`, unwound in reverse. An interceptor that calls
`handled()` clears the error, and outer interceptors see a success.

**Interceptors wrap queries too**, as of ngin 0.0.5. They were an actions-only
idea, but the concerns people reach for them with are not about actions — they
are about work, and a query is work. The context says which kind it was called
for, `action` or `query`, so one interceptor can serve both and tell them apart.

Where a query is entered matters:

- A **live** query is entered before it boots — before its boot action is
  dispatched and before its resources are leased — and left once it has been
  killed and its lease released.
- A query answered by `fetch` (a one-shot subscribe, or peeking at an inactive
  one) has the same shape as a dispatch: enter, fetch, leave.

Interceptors have no say in results. What reaches subscribers, and what `peek()`
resolves with, is the query's own value whatever the hooks return.
