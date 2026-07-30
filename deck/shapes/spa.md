# Shape: the single-page app

A client application that owns its document: an engine, a provider graph of
browser capabilities, business logic, and a dodo tree. This is the shape most of
our client code takes, whether it owns the whole site or ships as a bundle into
a page a service rendered.

## The composition root

One file. It builds the engine, mounts the tree, and dispatches the first
action. It is the only file in the app that names concrete providers.

```javascript
import { Engine, Provider } from '@3sln/ngin';
import { dodo } from '../shared/stack.js';
import { PlayerStore, PlayerApi, OfflineStore, AudioController } from './providers/resources.js';
import * as bl from './bl/player.js';
import { playerApp } from './ui/compositions/playerApp.js';

const engine = new Engine({
  providers: {
    state: Provider.fromSingleton(new PlayerStore()),
    api:   Provider.fromSingleton(new PlayerApi(slug)),
    store: Provider.fromSingleton(new OfflineStore(slug)),
    audio: Provider.fromSingleton(new AudioController()),
  },
});

dodo.reconcile(document.getElementById('app'), [playerApp(engine)]);
engine.dispatch(new bl.Init(ticket));
```

Global listeners that are genuinely global — keyboard shortcuts, media keys,
`beforeinstallprompt`, `popstate` — also belong here, dispatching actions. They
are not a component's business.

## Providers are browser capabilities

Everything the platform offers gets wrapped: IndexedDB, `Audio`, the network,
`localStorage`, the microphone, a WebRTC peer, speech synthesis. The test is
whether you could run the business logic in a test harness without a browser. If
not, something reached for a global.

Use `fromLazySingleton` wherever construction can fail — opening IndexedDB in
private mode, requesting a microphone, a blocked upgrade — so the failure
reaches whoever asked rather than becoming an unhandled rejection at import time.

## Reactivity: the Cell seam

dodo's reactivity speaks the **Cell** protocol — `{ onDirty(fn), getValue() }` —
which is push-to-invalidate, pull-to-read, so any number of state changes
coalesce into a single render. ngin's `engine.query()` hands back an
observer-style observable. Those are different protocols, and the adaptation
belongs at the composition seam, in one shared module per project:

```javascript
// shared/stack.js — import dodo and watch from here, never from the package,
// so every vnode reconciles through one dodo instance.
import * as dodo from '@3sln/dodo';
import { cell, effect, fromObservable, watch } from '@3sln/dodo/reactive';

export { dodo, cell, effect, watch };

// The cell stays disconnected until a `watch` subscribes and disconnects again
// when the last one detaches, so the query is torn down along with the UI that
// asked for it.
export const queryCell = (observable, { initial } = {}) =>
  initial === undefined
    ? fromObservable(observable)
    : fromObservable(observable, { initial });
```

That last property is the point: query lifetime is tied to UI lifetime
automatically. Nothing has to remember to unsubscribe.

## Compositions

A composition injects the engine, reads queries through `watch`, and dispatches
actions in response to events from components.

```javascript
export default ({ engine, controlBar, trackList }) => {
  const state = queryCell(engine.query(new PlayerState()));

  return alias(() =>
    div({ className: 'player' },
      watch(state, s => trackList(s.tracks)),
      controlBar().on({
        'send-text': e => engine.dispatch(new SendText(e.detail)),
      })));
};
```

Compositions are the *only* UI modules that may import from `bl/`. If a
component imports an Action, the boundary is gone.

## Components

Dumb. No engine, no business logic, no imports from `bl/`. They render what they
are handed and announce what happened with
[bubbling DOM events](/?c=%2Fcore%2Fboundaries.md).

Build them with `tag(props, ...children)`; props use `$styling`, `$classes`,
`$attrs` and `$dataset`; chain `.on({ event: fn })`, `.key(k)` and `.opaque()`.

```javascript
// Hyphenated names must go through $attrs: a bare prop is assigned as a
// property, and `aria-label` is not one.
button({ className: 'play', $attrs: { 'aria-label': 'Play' } }, icons.play())
  .on({ click: () => fire('toggle-play') })
```

There is no `.style()` helper. Use `$styling`.

Use `alias` rather than a plain function for anything reusable: it is memoized
(a plain function re-runs on every parent render), and it is backed by a stable
`<udom-alias>` element, which is what makes `.key()`, `.on()` and
`this.dispatchEvent` possible at all.

An alias builder is invoked as `builder.apply(hostElement, args)`, so write it
as a `function` whenever it needs `this`. An arrow function's `this` is fixed
where it is written and `apply` cannot change it — but arrows *inside* the
builder are fine, and are the normal way to write handlers.

For performance-critical rendering or a third-party library that owns its own
DOM (charts, maps, editors), drop to `special({ attach, update, detach })`. Both
`alias` and `special` nodes can be reconciled directly onto a custom element,
which is how dodo integrates with Web Components.

## Routing is a Query

The URL is state that something observes, so it is a query, with actions to
navigate. Parse it into a `{ view, params }` shape and let the root composition
switch on it.

Resolve redirects during parsing rather than after the first render — a share
link that lands on `/?uri=…` should route to the importer immediately, not flash
the dashboard on the way.

## The one place a bound engine is allowed

Actions cannot dispatch actions. A few cross-cutting flows genuinely need one
action's completion to start another — audio `ended` → play next track,
create → navigate to the new thing. Bind the engine once at the composition root
and let those specific flows use it:

```javascript
bl.bindEngine(engine);
```

Keep the list short and keep it visible. It is an escape hatch, and it is the
first thing to check when a call graph stops being followable.
