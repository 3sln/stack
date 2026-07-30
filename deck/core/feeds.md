# Feeds

A feed is an `EventTarget`. There are two, and the difference is who is
listening.

| Feed | Scope | Use it for |
| --- | --- | --- |
| `dispatchFeed` | one dispatch | progress and results for *this caller* |
| `engineFeed` | the whole engine | facts other parts of the system may care about |

Both are handed to `execute` in its second argument. `engine.dispatch(action)`
returns the `dispatchFeed`.

## Why a feed instead of a return value

A return value can say one thing, once, at the end. A great deal of real work
needs to say something **now** and something else **later** — and every time
that has been modelled as a return value in these projects, it grew into the
same three-part tangle:

| Hand-rolled | What it actually was |
| --- | --- |
| a `{ task, alreadyRunning, done }` tuple | a feed — `started` now, `result` later |
| progress poked into a registry, then polled by clients | a feed, with a second mechanism bolted on beside it |
| a `try/finally` releasing a claim | a resource lifetime (see [Providers](/?c=%2Fcore%2Fproviders.md)) |

As a feed, the same work reads as what it is:

```javascript
async execute({ vfs, tasks, lifecycle, claim }, { dispatchFeed, signal }) {
  if (!claim.held) {
    // An ordinary answer, not a failure. The dispatch completes normally.
    emit(dispatchFeed, 'started', { task: null, alreadyRunning: true });
    return;
  }
  const { task, done } = tasks.begin(spec, handle => this.#scan({ ... }));

  // The caller has something to answer with long before there is a result.
  emit(dispatchFeed, 'started', { task, alreadyRunning: false });
  emit(dispatchFeed, 'result', { result: await done });
}
```

The caller awaits whichever moment it needs:

```javascript
const feed = engine.dispatch(new ScanCollection({ collectionId }));
const { task, alreadyRunning } = await feed.next('started');   // respond now
```

An SSE endpoint or a progress bar can subscribe to the same feed rather than
having a parallel polling mechanism invented for it.

## `next()` and abort

`feed.next(events)` waits for one of them. By contract, **aborting pre-empts
anything waiting** — an abort rejects the waiters, which is what you want for a
caller who gave up.

It is not what you want for the work's own completion:

```javascript
// ⚠️ Pre-empted by abort — but an aborted scan still produces a partial
// result with `stopped: true` that a resumable scan needs to report.
feed.next(['result'])

// Naming `abort` opts out of the pre-emption.
feed.next(['result', 'abort'])
```

Get this right at the seam once. It is not obvious from the outside, and it is
the kind of thing that looks like it works until the first cancellation.

## Cancellation has one shape

Three different events all mean *stop*, and an action should read them as three
lines in one place:

```javascript
signal.aborted        // the caller gave up
handle.cancelled      // someone clicked Cancel
lifecycle.closing     // the process is going down
```

That last one is the interesting one: making shutdown a **dependency** rather
than a closure variable means an action that must stop when the server is going
down has to *say so* in its `deps`. You can grep for which ones did.

## Cost

`dispatch()` returns synchronously and the action runs on a later macrotask, so
anything the action does happens at least one turn of the event loop after the
call. Usually microseconds and irrelevant — but it moves a real boundary. Work
that used to be decided *before* a function returned is now decided one turn
later, and two requests arriving in the same tick can both get through a check
that a synchronous version would have caught.

Whether that is a problem depends on whether the guarantee you need is *"never
concurrently"* (still held, by the lease) or *"never twice"* (not held). Know
which one you are relying on.
